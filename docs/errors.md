# Error handling

*[Português (Brasil)](errors.pt-BR.md) · English (US)*

Tool execution errors are returned with `isError: true` and a plain-text
description. Successful JSON object results also use MCP `structuredContent`.
Authorization failures occur before tool execution and use HTTP status codes
instead. Only a request naming a tool that needs a workspace is challenged;
connecting, listing tools and the public verification tool answer without one.

## Authentication and transport

For an expired access token, refresh the existing OAuth grant first. Request new
consent only when refresh fails or additional permissions are needed. If login
returns `invalid_target`, confirm the MCP URL is
`https://mcp.assinafy.com.br/mcp` and contact Assinafy if the error persists.
See [authentication](authentication.md).

| HTTP status | Meaning and recovery |
|---|---|
| 400 | Malformed request, credential override attempt, duplicate Authorization headers, a token in the query string, a message repeating a JSON key, an unknown tool name, or a batch over 20 messages |
| 401 | A tool needing a workspace was called with no token, an unusable one, or a credential that is not an Assinafy OAuth token; also an unauthenticated request over 64 KiB. Refresh the existing grant, or follow the `resource_metadata` challenge if no usable grant remains |
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

## MCP Events

`events/subscribe` and `events/unsubscribe` return JSON-RPC errors (see
[Events](events.md)):

| Code | Meaning | Client action |
|---|---|---|
| `-32015` | Callback verification failed; `data.reason` is `invalid_url`, `timeout`, `connection_failed` or `challenge_failed` | Fix the callback (HTTPS, public, replies exactly `{"challenge": …}`) and subscribe again |
| `-32602` | Invalid request: unknown event, bad `document_id`, unsupported delivery, invalid secret, non-null `cursor`, the document is not accessible in the workspace, or the subscription limit is reached | Correct the request; it will not succeed unchanged |
| `-32603` | Temporary: The service is temporarily unavailable, or the subscription changed or was unsubscribed concurrently | Retry with backoff; refresh before `refreshBefore` |

Webhook deliveries that fail permanently, or after five attempts, are abandoned and
logged; the subscription keeps monitoring.

## API errors

Tool errors preserve the API's status, message, string error code, and field-level
errors when supplied. `ContractChanged`, `PlanRestricted` and
`CreditCeilingExceeded` remain distinguishable even when the message changes.
Treat a failure reported inside an API response envelope as an error too.

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

The server retries some upstream read failures before returning an error:

- **Automatic retry.** HTTP `429` and transient `5xx` responses receive at most
  three automatic retries, with bounded exponential backoff, only on safe read
  methods (`GET`, `HEAD`, `OPTIONS`). Mutating `POST`, `PUT`, `PATCH`, and
  `DELETE` requests are never retried automatically. Numeric and HTTP-date
  `Retry-After` values are honored; `X-Rate-Limit-Reset` seconds are also
  accepted for compatibility. When the requested wait exceeds the eight-second
  automatic retry budget, the refusal is returned immediately; the caller must
  wait before trying again. The client never shortens a server-mandated wait.
  Document processing polls stop on a returned API error; they do not start a
  second retry cycle or discard the refusal in favor of a polling timeout.
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

CLM errors preserve the API's status, code, message and field errors. `403` can mean the workspace
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
