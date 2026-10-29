# Error handling

Tool execution errors are returned with `isError: true` and a plain-text
description. Successful JSON object results also use MCP `structuredContent`.
Authorization failures occur before tool execution and use HTTP status codes
instead. Only a request naming a tool that needs a workspace is challenged;
connecting, listing tools and the public verification tool answer without one.

## Authentication and transport

If `codex mcp login assinafy` reports `Protected resource metadata missing required
resource field`, check the deployed server version. Versions before `v3.0.1`
returned a tool manifest at `GET /mcp` without OAuth metadata, which Codex treated
as an incomplete discovery document. Deploy `v3.0.1` or later and retry the same
login command; no change to the client's MCP URL or credentials is required.

If login returns `invalid_target`, the authorization server has rejected the
MCP resource `https://mcp.assinafy.com.br/mcp`. This was reproduced in production
with both Codex and Claude Code on 2026-09-19, after successful discovery. Ask the
Assinafy operator to enable that exact resource in the provider's accepted
audiences, then retry login. Keep the MCP URL unchanged. Consent and token
exchange cannot complete until the provider accepts the resource; reinstalling
the client or supplying API credentials does not resolve this rejection.

| HTTP status | Meaning and recovery |
|---|---|
| 400 | Malformed request, credential override attempt, duplicate Authorization headers, a token in the query string, a message repeating a JSON key, an unknown tool name, or a batch over 20 messages |
| 401 | A tool needing a workspace was called with no token, an unusable one, or a credential that is not an Assinafy OAuth token; also an unauthenticated request over 64 KiB. Follow the `resource_metadata` challenge and reconnect |
| 403 with `insufficient_scope` | The validated grant lacks a tool permission. Consent for the scopes in `WWW-Authenticate`, in addition to those already granted, and retry; the workflow has not started |
| Other 403 | Forbidden Origin, unexpected public Host, or an area no OAuth token may reach; correct the client, proxy, or the request |
| 405 on an event-stream GET | Expected: this stateless server does not provide a standalone SSE stream; use POST Streamable HTTP |
| 415 | A POST to `/mcp` whose `Content-Type` is not `application/json` |
| 413 | A verified request exceeds the 40 MiB limit. An anonymous request is capped at 64 KiB; a bearer token is verified after at most 64 KiB plus one byte, before buffering a larger body |
| 429 | Assinafy is rate-limiting authorization checks; honour the `Retry-After` header |
| 503 | Introspection/exchange is unavailable, malformed, redirected, rejected the MCP service credentials/target, or refused or withheld a scope the grant already holds. That last is the MCP service client's policy, which consenting again cannot change. Retry transient failures and ask the operator to check persistent ones |

The HTTP authorization responses above carry no body beyond the status text, so
no Assinafy description or credential leaks through them. Upstream messages do
reach the caller, verbatim, inside tool results — see below. The server does not
refresh a client's tokens; refresh is the client's job. A
credential or workspace override in tool arguments or `_meta` returns a tool
error before the document operation runs.

The MCP checks permissions returned by the issuer before executing any tool.
A missing tool scope returns HTTP `403 insufficient_scope` before mutations.
Assinafy also enforces authorization on every business-API call; revocation,
membership changes or other upstream failures during a workflow can still
produce a `200` JSON-RPC result with `isError: true`. A long workflow renews the
server's short-lived API token as it goes; if Assinafy refuses a renewal, the
tool stops with such an error instead of retrying. Inspect a retained document
ID before retrying a partially completed workflow.

The server refuses to start if its introspection/exchange client settings are
missing, issuer metadata lacks those capabilities, or a registered tool declares
no scopes without being listed as public. Resource registration alone does not
satisfy the new token-separation contract; deploy the API capabilities and inject
the operator credentials before upgrading from v3.0.1.
`/healthz` reports only that this process is serving; it is
deliberately independent of Assinafy. See [authentication](../README.en.md#workspaces-and-permissions).

## MCP Events

`events/subscribe` and `events/unsubscribe` return JSON-RPC errors (see
[Events](events.md)):

| Code | Meaning | Client action |
|---|---|---|
| `-32015` | Callback verification failed; `data.reason` is `invalid_url`, `timeout`, `connection_failed` or `challenge_failed` | Fix the callback (HTTPS, public, replies exactly `{"challenge": …}`) and subscribe again |
| `-32602` | Invalid request: unknown event, bad `document_id`, unsupported delivery, invalid secret, non-null `cursor`, the document is not accessible in the workspace, or the subscription limit is reached | Correct the request; it will not succeed unchanged |
| `-32603` | Temporary: Events are starting, Assinafy is unavailable, the grant has no `grant_id`, the store could not be saved, or the subscription changed or was unsubscribed concurrently | Retry with backoff; refresh before `refreshBefore` |

Webhook deliveries that fail permanently, or after five attempts, are abandoned and
logged; the subscription keeps monitoring.

## Error types

### `APIError`

Represents an upstream failure reported by either a non-2xx HTTP response or a
non-2xx status inside an Assinafy response envelope.

| Field | Type | Description |
|---|---|---|
| `StatusCode` | int | HTTP status code |
| `Message` | string | `message` or `error` field from the response body |
| `ResponseData` | any | Parsed upstream response data, for a non-2xx HTTP response |
| `Challenge` | string | The upstream `WWW-Authenticate` header, when there was one |

The Assinafy API returns two error body shapes, both of which carry a
`message` that is surfaced verbatim (messages are Portuguese-language):

- `{"status":400,"data":null,"message":"..."}`
- Framework route-not-found: `{"name":"Not Found","message":"Página não encontrada.","code":0,"status":404}`

Common status codes:

| Code | Meaning |
|---|---|
| 400 | Bad request - check required fields |
| 401 | Unauthorized - invalid or missing credentials |
| 403 | Forbidden - insufficient permissions |
| 404 | Not found - resource does not exist |
| 409 | Conflict - resource already exists |
| 422 | Unprocessable entity - validation errors in request body |
| 429 | Rate limited - back off and retry |
| 500 | Internal server error - contact Assinafy support |

### `ValidationError`

Represents invalid input or an invalid upstream success response (for example,
an empty document ID, invalid email address, or missing created-resource ID).

### `NetworkError`

Wraps request construction, transport, response-read, and response-decode
failures, including timeouts, connection refusal, and DNS lookup failure.

## Error message format

When a tool fails, the MCP content will be a single `TextContent` item:

```
API error 404: Página não encontrada.
```

When the upstream response carried a `WWW-Authenticate` header, it follows the
message in parentheses — often the only place a missing permission is named:

```
API error 403: Permissão insuficiente. (Bearer error="insufficient_scope", scope="documents:write")
```

A validation failure this server raises itself is the bare message, with no
prefix:

```
Document ID is required
```

Coded validation failures prefix the message with a stable machine-readable code:

```
digital_certificate_requires_virtual: DigitalCertificate signers require method virtual
```

A failure of the tool's input schema is raised by the MCP SDK before the handler
runs, and is prefixed differently:

```
validating "arguments": ...
```

## Retrying

The client already retries automatically, so most transient failures never reach
the caller:

- **Automatic retry.** HTTP `429` and transient `5xx` responses receive at most
  three automatic retries, with bounded exponential backoff, only on safe read
  methods (`GET`, `HEAD`, `OPTIONS`). Mutating `POST`, `PUT`, `PATCH`, and
  `DELETE` requests are never retried automatically. Numeric and HTTP-date
  `Retry-After` values are honored; `X-Rate-Limit-Reset` seconds are also
  accepted for compatibility.
- `404` and `422` responses indicate a problem with the request parameters and
  should not be retried without fixing the input.
- `409` during the internal create-or-reuse signer step is followed by one
  exact-email lookup, allowing the operational workflow to return the signer
  created by a concurrent request.

For `assinafy_send_pdf_for_signature`, a failure after upload reports the
retained document ID. Do not blindly retry the whole workflow after an
uncertain assignment response; inspect or delete the retained document first
to avoid duplicate documents or notifications. If the connection's access token
ends within `max_wait_secs` plus two minutes, the tool uploads nothing and says
how many seconds remain; retry with a shorter wait or after the client renews.

If the retained document has no assignment and its state permits signing, use
`assinafy_create_assignment` to continue with confirmed signer IDs. An existing
assignment should be tracked instead of recreated. Reading a document performs no
document mutation or notification. A successful signer-contact correction may
invalidate earlier access links; use an authorized resend after checking the
updated document. Assinafy rejects changes to channels already verified on an
in-flight document.

## Contract conflicts and reminders

CLM errors preserve the API's status and message. `403` can mean the workspace
plan does not include tracking; additional consent will not change a plan.
`409 ContractChanged` means the supplied `updated_at` is stale: read the contract
or amendment again, review the change, then submit an authorized update.
Never remove the precondition merely to force the write through.
A repeat renewal can return `400` because its source `term` is no longer current.
On an uncertain write, inspect current terms and activities before proceeding.

Lifecycle reminders are scheduled by Assinafy, follow member notification
preferences and pause without a tracking plan. `contract.activity` only observes
new activities while its client keeps authorization and subscriptions refreshed.
It does not send another reminder, guarantee delivery after retries are exhausted,
or authorize an automatic contract mutation.
