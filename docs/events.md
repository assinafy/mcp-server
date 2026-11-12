# Events

*[Português (Brasil)](events.pt-BR.md) · English (US)*

Events lets compatible clients watch documents and receive notifications when
their status changes, certification finishes, or contract activities occur.
Ask your assistant to monitor a document using the hosted Assinafy MCP connection.

Events uses the same OAuth connection as the tools. It does not change your
Assinafy account webhooks or authorize actions such as sending reminders, renewing
a contract, or terminating it. Human signers complete signatures and identity
checks in Assinafy.

## Available events

| Event | Notification |
|---|---|
| `document.status_changed` | The monitored document's observed status changed |
| `document.certificated` | Certification completed and signed artifacts are ready |
| `contract.activity` | A new contract notice, warning, renewal, or change was recorded |

Each subscription watches one `document_id`. Find the document with
`assinafy_list_documents` or `assinafy_browse_contracts` and use the returned ID.
Subscribing establishes a baseline: an already certificated document does not
produce a new completion event. Use `assinafy_get_document` to inspect it instead.

## Client integration

Events is an advertised MCP extension for MCP 2.0 (`2026-07-28`). Check the
`events` capability in `server/discover`, then call `events/list` to discover
event names and their input and payload schemas. It is separate from core MCP
resource subscriptions. See the [MCP Events integration](https://developers.openai.com/plugins/build/mcp-events).

`events/list`, `events/subscribe`, and `events/unsubscribe` use the same `/mcp`
endpoint and require `account:read` and `documents:read`. Your MCP client supplies
the protocol headers and request metadata. For direct requests, include
`Content-Type: application/json`, the OAuth `Authorization` header,
`MCP-Protocol-Version: 2026-07-28`, and `Mcp-Method` matching the requested method.

Example subscription with synthetic document and callback values:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "events/subscribe",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "example-client",
        "version": "1.0.0"
      }
    },
    "name": "document.certificated",
    "arguments": { "document_id": "document-example" },
    "delivery": {
      "mode": "webhook",
      "url": "https://receiver.example.com/events/callback-example",
      "secret": "whsec_<base64-encoded-24-to-64-byte-key>"
    },
    "cursor": null
  }
}
```

Example successful result (IDs and times are illustrative):

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "complete",
    "id": "sub_example",
    "refreshBefore": "2030-01-01T12:10:00Z",
    "cursor": null,
    "truncated": false
  }
}
```

The callback must use public HTTPS, accept signed verification requests, and
reply with exactly `{"challenge":"<received challenge>"}` before deliveries
begin. Redirects are not followed. The callback URL is limited to 2 KiB. Keep the
callback signing secret private; it is separate from the OAuth token.

## Refresh and unsubscribe

Subscriptions are idempotent for the same OAuth grant, workspace, event, document,
and callback. Refreshing an access token preserves that identity; new browser
consent creates a new grant. Another grant cannot refresh or unsubscribe it.

The subscription lifetime defaults to ten minutes and never exceeds the access
token's remaining lifetime. Optional `ttlMs` is clamped between 30 seconds and
ten minutes; `null` still receives a finite lifetime. Refresh the OAuth token as
needed and call `events/subscribe` again before `refreshBefore`. Token renewal
alone does not renew a subscription. See [authentication](authentication.md).

An expired subscription pauses checks and delivery. Its pending event remains
available for a two-minute grace period for a late refresh. After a restart or
reconnection, refresh subscriptions before monitoring resumes. If an earlier
subscription is no longer available, subscribing starts a new baseline.

Call `events/unsubscribe` with the original event name, arguments, and delivery
URL; the signing secret is not required. It returns an empty result and is
idempotent. Limits are 16 subscriptions per workspace and 32 in total. Remove an
unneeded subscription before adding another when a limit is reached.

## Delivery and recovery

Document checks normally run every 30 seconds; API latency and subscription volume
can delay notifications. Events observes changes while monitoring is active and
does not provide historical replay (`cursor: null`, `truncated: false`). Intermediate
changes and changes during interrupted monitoring can be missed. Read current
document or contract details before acting on an event.

Each delivery contains `eventId`, `name`, `timestamp`, `data`, and `cursor: null`.
Document events include `document_id`, `status`, and `previous_status` in `data`:

```json
{
  "eventId": "evt_example",
  "name": "document.certificated",
  "timestamp": "2030-01-01T12:05:00Z",
  "data": {
    "document_id": "document-example",
    "status": "certificated",
    "previous_status": "pending_signature"
  },
  "cursor": null
}
```

Verify Standard Webhooks HMAC-SHA256 signatures over the raw request bytes using
`webhook-id`, `webhook-timestamp`, and `webhook-signature`. The subscription ID is
in `X-MCP-Subscription-Id`. Reject stale timestamps and deduplicate `eventId`.
A replacement signing secret is verified and has a one-minute overlap with the
previous key.

A `2xx` response acknowledges delivery. Network failures, `408`, `429`, and `5xx`
receive up to five attempts with backoff and the same event ID. Permanent errors
such as `410` and `413`, or exhausted retries, end that delivery; monitoring
continues. Duplicate delivery is possible, so make downstream actions idempotent.
Revoked access, missing permissions, or loss of document access ends monitoring.
Temporary API failures pause checks. See [error recovery](errors.md#mcp-events).

## Contract activities

`contract.activity` reports `document_contract_updated`,
`document_contract_renewal_notice`, `document_contract_expiring`,
`document_contract_expired`, `document_contract_renewed`,
`document_contract_terminated`, `document_contract_amended`, and
`document_contract_amendment_updated` activities.

Its `data` contains `document_id`, `activity_id` as a string, and `activity`.
`end_date` and `days_before` are included when supplied. The timestamp is the
recorded activity time. Names, tax IDs, amounts, metadata, termination reasons,
and full activity payloads are not delivered.

Subscribe separately to an amendment document to follow its link changes.
Historical activities are not replayed. Deduplicate `(document_id, activity_id)`
across independent subscriptions and read current terms with
`assinafy_browse_contracts` before acting.

Keep the OAuth connection and subscription renewed for long-running monitoring.
Assinafy's owner emails provide lifecycle reminders independent of an MCP client.
An Events subscription does not create a second reminder or grant permission to
send a separate email, renew, or terminate a contract automatically.
