# MCP Events

This server implements the [OpenAI MCP Events integration](https://developers.openai.com/plugins/build/mcp-events)
for MCP 2.0 (`2026-07-28`). ChatGPT Work can monitor a document and act when
its status changes, its certificated artifacts become ready, or contract lifecycle
activities are recorded.

This is an advertised extension with `events/list`, `events/subscribe`, and
`events/unsubscribe`, not the core MCP subscription API. Clients discover it
through the `events` capability in `server/discover`.

## Enable

Set `MCP_EVENTS_STORE` to a JSON file on persistent storage writable by the
server user, for example `/app/data/events/events.json`. Leave it empty to
disable Events. The server advertises `events` in `server/discover` only when
enabled, and only then do `events/*` methods exist or require OAuth scopes.
Existing tools and older MCP clients continue to work.

The store's *directory* must be writable and persistent: every save writes a
temporary file beside the store and renames it into place. Mount a directory,
not a single file, and do not point the path at a symlink; the server checks
both at startup and refuses to enable Events otherwise.

Use one server replica for the Events store. A process lock prevents two processes
from using the same file. A process that finds the lock held still starts and
serves every tool; it takes the store over within 30 seconds of the holder
exiting, and until then `events/*` return a retryable error. With Kubernetes,
prefer the `Recreate` strategy (or a StatefulSet) so the old and new pods do not
overlap for long. Different store files do not share subscriptions, so
multiple replicas need a shared subscription database and delivery queue before
enabling this feature. The store is written atomically with permissions `0600`;
it contains callback signing secrets and must be protected like a secret. Keep it
outside the source checkout and back it up securely. OAuth tokens are never written
to the store. Events requires outbound HTTPS and DNS access.

### Deployment checklist

`values.production.yml` leaves Events disabled until durable storage is mounted.
The image creates `/app/data/events`, owned by uid/gid `65532`. To enable Events:

1. Mount a persistent volume at `/app/data/events`, writable by uid/gid `65532`
   (`fsGroup: 65532`). With a ReadWriteOnce volume, also set
   `strategy: {type: Recreate}` on the `mcp` web entry so a rolling update does
   not wait on the volume attachment. Pin the deployment to one replica.
2. Set `MCP_EVENTS_STORE: "/app/data/events/events.json"` only after the mount is
   present. Never use the container filesystem for the production store.
3. Allow egress over HTTPS and DNS to the public internet, because ChatGPT
   supplies the callback URLs.

## Events and subscription methods

| Event | Trigger |
|---|---|
| `document.status_changed` | A monitored document's observed status changes |
| `document.certificated` | A monitored document reaches `certificated`, when signed artifacts are ready |
| `contract.activity` | New contract notices, warnings, renewals and changes in its activities ([details](#contract-lifecycle-events)) |

All events require exactly one filter, `document_id`. Resolve a document's name
with `assinafy_list_documents` before subscribing. The initial status is a baseline:
subscribing to an already certificated document does not send a completion event.
Use the existing document tools to inspect or download it immediately instead.

`events/list`, `events/subscribe`, and `events/unsubscribe` use the same `/mcp`
endpoint and OAuth authorization as the tools, and the MCP SDK applies the same
protocol rules (headers, request `_meta`, error status codes) as for every other
method. They require `account:read` and `documents:read`. ChatGPT supplies the
webhook URL (at most 2 KiB) and `whsec_` signing secret. The server verifies a
signed callback challenge before accepting application delivery: the reply must be
exactly `{"challenge": "<value>"}`, so an endpoint that merely echoes requests does
not qualify. It accepts only HTTPS public destinations, pins validated DNS
addresses at connection time, and follows no redirects. Unknown optional request
fields, such as the draft's `maxAgeMs`, are ignored; a key that differs from a
known one only by letter case is refused.

Example subscription (ChatGPT supplies the real callback and secret):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "events/subscribe",
  "params": {
    "name": "document.certificated",
    "arguments": { "document_id": "document-example" },
    "delivery": {
      "mode": "webhook",
      "url": "https://receiver.example.com/mcp-events/callback-example",
      "secret": "whsec_<base64-encoded-24-to-64-byte-key>"
    },
    "cursor": null
  }
}
```

Subscriptions are idempotent for the same authorization grant, workspace, event,
document, and callback. Argument order does not change identity. Ownership is the
introspected `grant_id`, which stays the same across access-token rotation, so a
refresh with a new bearer keeps the same subscription. Re-consent issues a new grant
and therefore a new subscription. Subscribing fails if introspection omits
`grant_id`. Another grant cannot refresh or unsubscribe it, even in the same workspace. Limits are 16 subscriptions per workspace and 32 in total, checked before any outbound request. Each live subscription costs about one introspection per minute and a token exchange every few minutes, sharing Assinafy's per-IP limits (50 exchanges and 60 introspections per minute) with interactive tool calls; raise the total only with a dedicated rate-limit allowance.
A subscribe whose subscription changes while its checks run starts over; one that
an unsubscribe overtook fails with a retryable error rather than reviving it.

The granted lifetime defaults to ten minutes and never exceeds OAuth expiry.
`ttlMs` is clamped to between 30 seconds and ten minutes, never rejected;
`ttlMs: null` requests an unlimited lifetime, but the server grants a finite one.
The lifetime starts after the server's checks finish. An expired subscription is
kept, without checks or deliveries, for two more minutes so that a late refresh
still finds its pending event. ChatGPT must refresh before `refreshBefore`.
A replacement signing secret is verified and uses a one-minute overlap with the
previous key. Successful callback verification is cached for at most ten minutes.

Unsubscribe with the original event name, arguments, and delivery URL; a secret
is not required. The response is `{}`. Unsubscribe is idempotent.

## Delivery and recovery

The server checks subscribed documents every 30 seconds using the existing
OAuth exchange and API client. It does not change the workspace's Assinafy webhook
configuration. Checks run sequentially; upstream latency and subscription volume
can lengthen a scan. A fresh access check is required before each delivery, with
OAuth decisions cached by the existing authorization layer for at most one minute.
The API separately enforces grant lineage. Revoked access, a grant that lost its
scopes, workspace mismatches, or document access denial (403/404) end the
subscription. Outages, rate limits, a scope withheld only by token exchange, and
incomplete API responses are temporary: nothing is delivered until a check
succeeds again. During `events/subscribe`, only an access denial is reported as
invalid parameters; anything else is a retryable internal error.

A delivery contains `eventId`, `name`, `timestamp`, `data`, and `cursor: null`.
For document events, `data` contains only `document_id`, `status`, and `previous_status`. The timestamp
is the document's `updated_at` when it falls between the previous check and now,
otherwise the time the change was observed.
Signatures use Standard Webhooks HMAC-SHA256 with `webhook-id`,
`webhook-timestamp`, `webhook-signature`, and `X-MCP-Subscription-Id` headers.

The pending delivery is persisted before sending. Transient failures (network,
`408`, `429`, `5xx`) receive up to five attempts, with exponential backoff and the
same event ID; each attempt is signed again. A `2xx` acknowledges receipt. `410`,
`413` and other permanent replies end that delivery without retry, as do exhausted
retries: the event is abandoned and logged (`event_delivery_abandoned`, status only,
no URLs, secrets or tokens), and the subscription keeps monitoring. A shutdown
during a delivery does not count as an attempt. A reply to a delivery signed before
a refresh rotated the secret is ignored and the event is retried with the new key.
Store failures are logged with their cause (`events_store_failed`).

Subscriptions and pending deliveries survive restart. In-memory OAuth credentials
do not: ChatGPT must refresh the subscription before checks or delivery resume.
There is no historical replay (`cursor: null`, `truncated: false`). Checks observe
current state, so intermediate changes and changes during an interrupted or expired
subscription can be missed. The outbox holds one pending event per subscription;
newer status changes are observed after it is acknowledged. Make downstream write
tools idempotent because a crash after receipt but before the acknowledgment is
saved can cause redelivery.

This is webhook delivery to ChatGPT, with periodic checks as the Assinafy event
source. MCP polling/streaming delivery and `gap`/`terminated` control notifications
are not exposed. Production acceptance still requires a configured ChatGPT plugin,
live OAuth consent, and a real human signing ceremony.

## Contract lifecycle events

`contract.activity` uses the same `document_id` filter, scopes, subscription
limits, callback verification, persistence and refresh rules as document events.
Find a contract with `assinafy_browse_contracts` and subscribe for each document
whose lifecycle you want to monitor. A root's subscription observes changes to
that root; subscribe to an amendment document to observe its link changes.

The source is `GET /v1/documents/{documentId}/activities`, not locally calculated
calendar deadlines. It delivers the API's `document_contract_updated`,
`document_contract_renewal_notice`, `document_contract_expiring`,
`document_contract_expired`, `document_contract_renewed`,
`document_contract_terminated`, `document_contract_amended` and
`document_contract_amendment_updated` activities. Assinafy continues to own its
hourly lifecycle, plan checks and owner emails. Watching a document does not
create an Assinafy reminder or duplicate those emails.

The initial latest activity ID is a baseline, including on a document without
terms yet. Historical activities are not replayed. Later scans page activities
in descending ID order and select the oldest unseen CLM activity. One pending
delivery is persisted at a time, with its activity watermark; a refresh or restart
does not reset it. Pagination shares the ten-second upstream check deadline;
incomplete or failed reads never advance the watermark. A busy document can
need multiple scans to drain its backlog, and sustained API latency can delay it.
The existing five-attempt/permanent-failure abandonment policy still applies.

For this event, `data` contains `document_id`, `activity_id` (a string to preserve
large IDs), and `activity` (the upstream event code); `end_date` and `days_before`
are included when supplied. The timestamp is the recorded activity time. Actor
names, tax IDs, amounts, metadata, termination reasons and full activity payloads
are not delivered. Read current terms with `assinafy_browse_contracts` before
acting, since an alert may refer to an earlier term. Deduplicate `eventId` on
retries, or `(document_id, activity_id)` across independent subscriptions.

For a reminder far in the future, the client must continue renewing its OAuth
connection and subscription. The ten-minute grant is not a durable scheduled job.
No tokens are persisted and monitoring cannot resume after a restart until the
client refreshes. Use Assinafy's owner emails for lifecycle reminders independent
of the MCP client. Changes to contract terms never grant permission to send a
separate email, renew or terminate automatically from an event.
