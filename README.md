# Assinafy MCP — Client Reference

A hosted Model Context Protocol Streamable HTTP server for system-to-system
integrations with the [Assinafy](https://assinafy.com.br) electronic-signature
platform. The MCP server is a thin, multi-tenant proxy in front of the public
Assinafy REST API (`https://api.assinafy.com.br/v1`). Every request carries the
caller's API key and account ID; no per-tenant state is stored on the MCP
server itself.

- **Endpoint:** `https://mcp.assinafy.com.br/mcp`
- **Transport:** [MCP Streamable HTTP](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports#streamable-http) (stateless)
- **Protocol version:** `2025-11-25`
- **Auth:** per-request HTTPS headers (or per-call `_meta` / `arguments.auth`)

## Endpoint methods

The `/mcp` endpoint accepts two HTTP methods:

| Method | Purpose |
|---|---|
| `GET /mcp` | Returns a small JSON manifest: server name, version, MCP protocol version, transport, statelessness, and the full list of registered tools (name, title, description, annotations). Useful for capability discovery, browser smoke tests, and uptime probes. |
| `POST /mcp` | The JSON-RPC entry point for all MCP operations (`initialize`, `tools/list`, `tools/call`, etc.). |

`GET /mcp` sample:

```bash
curl -sS https://mcp.assinafy.com.br/mcp
```

```json
{
  "name": "assinafy-mcp-server",
  "version": "0.1.0",
  "protocol": "2025-11-25",
  "transport": "streamable-http",
  "stateless": true,
  "status": "ok",
  "tools": [
    {
      "name": "assinafy_list_documents",
      "title": "List Documents",
      "description": "List documents in an Assinafy workspace with optional pagination and search.",
      "annotations": { "readOnlyHint": true, "openWorldHint": true, "title": "List Documents" }
    }
    // ... 39 more
  ]
}
```

The server runs in **stateless mode**: there is no `initialize` handshake to
persist, no `Mcp-Session-Id` to track, and `GET /mcp` does not open an SSE
stream — clients fire each `tools/call` as a self-contained POST with the
required headers.

## Required headers

Every POST must include the following headers:

| Header | Required | Notes |
|---|---|---|
| `Content-Type: application/json` | yes | JSON-RPC 2.0 payload. |
| `Accept: application/json, text/event-stream` | yes | MCP returns each response as a single SSE `message` event over `text/event-stream`. |
| `MCP-Protocol-Version: 2025-11-25` | yes | Required by the MCP spec on every request to a Streamable HTTP server. |
| `X-Api-Key` | yes | Your Assinafy API key. Matches the official Assinafy REST API auth header. |
| `X-Assinafy-Account-Id` | yes | Your workspace (account) ID. |
| `X-Assinafy-Webhook-Secret` | conditional | Only required for `assinafy_verify_webhook_signature`, and only when `secret` is not passed inline as a tool argument. |

Bearer-token authentication is **not** supported.

The public `mcp.assinafy.com.br` host sits behind Cloudflare; clients without
a recognisable `User-Agent` header may be challenged. Standard SDKs and `curl`
send one by default — `urllib`-style minimal clients should set one explicitly.

## Per-call credentials (no HTTP headers)

Clients that cannot attach custom headers (most browser embeds, some hosted
chat platforms) can supply the same three credentials per `tools/call`. The
server checks, in order: HTTP headers → `_meta` → `arguments.auth` →
`arguments.assinafy` → `arguments.credentials`.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "assinafy_list_documents",
    "_meta": {
      "api_key": "<ASSINAFY_API_KEY>",
      "account_id": "<ASSINAFY_ACCOUNT_ID>"
    },
    "arguments": {}
  }
}
```

Recognised keys (snake_case or camelCase): `api_key` / `apiKey` / `x_api_key`,
`account_id` / `accountId`, `webhook_secret` / `webhookSecret`. Any tool that
accepts an explicit `account_id` argument uses that value over the
header/`_meta` value for the single call.

## Calling tools

Every operation flows through standard JSON-RPC `tools/call`. The HTTP
response body is one SSE frame containing the JSON-RPC result.

```bash
curl -sS -X POST https://mcp.assinafy.com.br/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-11-25' \
  -H "X-Api-Key: $ASSINAFY_API_KEY" \
  -H "X-Assinafy-Account-Id: $ASSINAFY_ACCOUNT_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "assinafy_list_documents",
      "arguments": { "page": 1, "per_page": 20 }
    }
  }'
```

Raw response:

```text
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"<JSON>"}],"structuredContent":{...}}}
```

For object-returning tools the result has both:

- `result.content[0].text` — the JSON payload as a string.
- `result.structuredContent` — the same payload as a JSON object.

For tools that return a **bare array** (e.g. `assinafy_get_document_activities`)
the MCP SDK only populates `result.content[0].text`; clients should parse it
with `JSON.parse` when they need the array. Plain-text results
(delete confirmations, etc.) are returned only as `content[0].text`.

Failed tool calls return `result.isError: true` with a human-readable message
in `result.content[0].text`:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "isError": true,
    "content": [{ "type": "text", "text": "API error 400: ..." }]
  }
}
```

The example payloads in the per-tool reference below show the
`structuredContent` object (or `content[0].text` for plain-text / array
results). The wrapping envelope is omitted for brevity.

## Important: file upload requires server-side filesystem access

`assinafy_upload_document` reads the PDF directly from the **MCP server's**
local filesystem using the supplied `file_path`. This works for self-hosted
deployments (you control the path), but on the public hosted endpoint at
`https://mcp.assinafy.com.br/mcp` a remote client cannot place a file where
the server can read it.

Public hosted clients have two options:

1. **Upload via the Assinafy REST API directly** (`POST /v1/accounts/{account_id}/documents`)
   and then use the MCP tools (`assinafy_get_document`, `assinafy_create_assignment`, …)
   to drive the rest of the workflow.
2. **Self-host this MCP server** alongside whatever ingestion pipeline produces
   the PDF, so the server-side filesystem path is meaningful.

Every other tool works end-to-end against the public endpoint.

## Document lifecycle quick reference

| Status | Meaning |
|---|---|
| `uploading` | File still being received. |
| `uploaded` | File received, awaiting metadata extraction. |
| `metadata_processing` | Backend extracting fields/pages. |
| `metadata_ready` | Ready for signing setup. |
| `pending_signature` | Assignment created; waiting on signers. |
| `expired` | Assignment expired without completion. |
| `certificating` | Final signing artifact is being generated. |
| `certificated` | Fully signed; signed PDF + certificate page available. |
| `rejected_by_signer` | A signer rejected the document. |
| `rejected_by_user` | The sender cancelled the request. |
| `failed` | Processing failed. |

`assinafy_wait_document_ready` returns when the document reaches any of
`metadata_ready`, `pending_signature`, or `certificated`.

---

# Tool reference

All 40 tools, organized by resource. Each entry lists arguments (verified
against the input schema in `tools/*.go`) and the response shape observed
against the live REST API. Where the underlying API enum is **case-sensitive**
this is called out — Assinafy's API rejects `"email"` and accepts `"Email"`.

## Documents

### `assinafy_upload_document`

Upload a PDF (max 25 MB) by **server-side filesystem path**. See the upload
note above; only practical for self-hosted MCP deployments.

| Argument | Type | Required | Description |
|---|---|---|---|
| `file_path` | string | yes | Absolute path on the MCP server's filesystem. |
| `account_id` | string | no | Override the account ID for this call. |
| `metadata` | object | no | Arbitrary JSON metadata to attach to the document. |

Response (`structuredContent`):

```json
{
  "resource": "document",
  "id": "2222bbbb2222bbbb2222bbbb2222",
  "account_id": "aaaa0000aaaa0000aaaa0000",
  "name": "sample_contract.pdf",
  "status": "uploaded",
  "artifacts": {
    "original": "https://api.assinafy.com.br/v1/documents/2222bbbb2222bbbb2222bbbb2222/download/original"
  },
  "pages": [],
  "created_at": "2026-05-11T23:25:38Z",
  "updated_at": "2026-05-11T23:25:38Z",
  "is_closed": false
}
```

Immediately after upload, `status` is usually `uploaded` or
`metadata_processing` and `pages` is empty. Call `assinafy_wait_document_ready`
to block until the backend finishes extracting page metadata.

### `assinafy_list_documents`

| Argument | Type | Required | Description |
|---|---|---|---|
| `page` | int | no | 1-based page number. |
| `per_page` | int | no | Results per page (max 100). |
| `search` | string | no | Filter by document name. |
| `sort` | string | no | Sort field; prefix `-` for descending (e.g. `-created_at`). |
| `account_id` | string | no | Override account ID. |

Response:

```json
{
  "data": [
    {
      "id": "1111aaaa1111aaaa1111aaaa111",
      "account_id": "aaaa0000aaaa0000aaaa0000",
      "name": "Sample Agreement.pdf",
      "status": "certificated",
      "is_closed": true,
      "created_at": "2025-03-13T14:24:20Z",
      "updated_at": "2025-03-13T14:26:51Z"
    }
  ],
  "meta": { "current_page": 1, "last_page": 2, "per_page": 3, "total": 4 }
}
```

### `assinafy_get_document`

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |

Response (abridged — `assignment.items` is included when an assignment exists
and contains the full field-placement data the signing UI uses):

```json
{
  "resource": "document",
  "id": "1111aaaa1111aaaa1111aaaa111",
  "account_id": "aaaa0000aaaa0000aaaa0000",
  "name": "Sample Agreement.pdf",
  "status": "certificated",
  "is_closed": true,
  "signing_url": "https://app.assinafy.com.br/sign/1111aaaa1111aaaa1111aaaa111",
  "artifacts": {
    "original": "https://api.assinafy.com.br/v1/documents/.../download/original",
    "thumbnail": "https://api.assinafy.com.br/v1/documents/.../thumbnail",
    "certificated": "https://api.assinafy.com.br/v1/documents/.../download/certificated",
    "certificate-page": "https://api.assinafy.com.br/v1/documents/.../download/certificate-page",
    "bundle": "https://api.assinafy.com.br/v1/documents/.../download/bundle"
  },
  "assignment": {
    "id": "3333cccc3333cccc3333cccc333",
    "sender_email": "sender@example.com",
    "method": "collect",
    "expires_at": null,
    "message": null,
    "signers": [
      { "id": "4444dddd4444dddd4444dddd444", "full_name": "...", "email": "...", "has_accepted_terms": true }
    ],
    "copy_receivers": [],
    "items": [ /* per-page field placements with signer + display_settings */ ],
    "summary": {
      "signer_count": 1,
      "completed_count": 1,
      "signers": [ { "id": "...", "full_name": "...", "email": "...", "completed": true } ]
    },
    "signing_urls": [
      { "signer_id": "4444dddd4444dddd4444dddd444", "url": "https://app.assinafy.com.br/sign/...?email=..." }
    ]
  },
  "created_at": "2025-03-13T14:24:20Z",
  "updated_at": "2025-03-13T14:26:51Z"
}
```

The `assignment.artifacts` URLs follow the document's lifecycle: `certificated`
/ `certificate-page` / `bundle` only appear once the document reaches
`certificated`.

### `assinafy_delete_document`

Deletes the document (and any active assignment) in one call.

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |

Response (`content[0].text`):

```text
Document deleted successfully
```

### `assinafy_get_document_activities`

Chronological event log. Returns a **bare JSON array** in `content[0].text`
(no `structuredContent`).

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |

Response (one entry per event; field shapes shown):

```json
[
  {
    "id": 40549,
    "event": "document_ready",
    "message": "Documento pronto.",
    "origin": null,
    "payload": [],
    "created_at": "2025-03-13T14:26:51Z"
  },
  {
    "id": 40548,
    "event": "signer_signed_document",
    "message": "Signatário Jane Q. Sampleton assinou o documento.",
    "origin": {
      "ip": "203.0.113.1",
      "user-agent": "Mozilla/5.0 ..."
    },
    "payload": {
      "signer_full_name": "Jane Q. Sampleton"
    },
    "created_at": "2025-03-13T14:26:46Z"
  }
]
```

`origin` is `null` for system events and an object (`{ip, user-agent}`) for
signer-driven events. `payload` is `[]` when empty and a map when populated.
The MCP server forwards these fields as raw JSON without coercion.

### `assinafy_wait_document_ready`

Polls the document until its status reaches `metadata_ready`,
`pending_signature`, or `certificated`. The polling happens on the MCP server
side — clients see one HTTP response when the document is ready (or when
`max_wait_secs` elapses).

| Argument | Type | Required | Description |
|---|---|---|---|
| `document_id` | string | yes | |
| `max_wait_secs` | int | no | Max seconds to wait (default 30). |
| `poll_secs` | int | no | Polling interval (default 2). |

Response shape: same as `assinafy_get_document`.

### `assinafy_verify_document`

Verify a signed document by its Assinafy signature hash. This is **not** a
SHA-1 of the file bytes — it is the per-document hash that Assinafy generates
at signing time and renders on the certificate page. Clients usually obtain
it from a printed/exported certificate.

| Argument | Type | Required |
|---|---|---|
| `hash` | string | yes |

Response (when the hash is unknown):

```json
{
  "hash": "deadbeef...",
  "is_valid": false,
  "message": "Documento não assinado ou não encontrado.",
  "id": null,
  "status": null,
  "completed_at": null,
  "completed_count": null,
  "page_count": null,
  "signer_count": null,
  "verified_at": "2026-05-11T23:23:48Z"
}
```

Valid hashes populate `id`, `status`, `signer_count`, `completed_count`,
`completed_at`, and `page_count`. The boolean field is `is_valid`, not `valid`.

### `assinafy_get_signing_progress`

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |

Response:

```json
{ "signed": 1, "total": 1, "pending": 0, "percentage": 100 }
```

`percentage` is a number; integer values are returned without a decimal point.

### `assinafy_create_document_from_template`

Create a document from a saved template, mapping signers to template roles.

| Argument | Type | Required | Description |
|---|---|---|---|
| `template_id` | string | yes | |
| `signers` | array | yes | `[{role_id, id, verification_method?, notification_methods?}]`. |
| `name` | string | no | Custom name for the resulting document. |
| `message` | string | no | Signing invitation message. |
| `expires_at` | string | no | ISO 8601 assignment expiry. |
| `account_id` | string | no | Override account ID. |

`verification_method` and `notification_methods[]` values are **case-sensitive**
and must use the upstream API spelling — see "Enum values" below. Response
shape matches `assinafy_get_document`.

### `assinafy_estimate_document_from_template_cost`

Estimate the credit cost before committing. Same arguments as
`assinafy_create_document_from_template` minus the optional fields.

| Argument | Type | Required |
|---|---|---|
| `template_id` | string | yes |
| `signers` | array | yes |
| `account_id` | string | no |

Response shape: same as `assinafy_estimate_assignment_cost` (see below).

### `assinafy_download_document`

Download a document artifact as base64.

| Argument | Type | Required | Description |
|---|---|---|---|
| `document_id` | string | yes | |
| `artifact` | string | no | `original` (default) or `certificated`. |

Response:

```json
{
  "document_id": "1111aaaa1111aaaa1111aaaa111",
  "artifact": "original",
  "base64": "JVBERi0xLjQK..."
}
```

The base64 decodes to the raw PDF bytes (`%PDF` magic at offset 0).

### `assinafy_download_signed_document`

Convenience wrapper around `assinafy_download_document` with `artifact:
"certificated"`. Errors if the document is not yet certificated.

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |

Response shape: same as `assinafy_download_document`.

### `assinafy_is_fully_signed`

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |

Response:

```json
{ "fully_signed": true }
```

## Signers

### `assinafy_create_signer`

**Idempotent by email.** If the email is already registered, the existing
signer is returned (other fields are not overwritten).

| Argument | Type | Required | Description |
|---|---|---|---|
| `full_name` | string | yes | |
| `email` | string | yes | |
| `whatsapp_phone_number` | string | no | E.164 format. |
| `cpf` | string | no | Brazilian tax ID. Non-digits are stripped automatically. |
| `metadata` | object | no | |
| `account_id` | string | no | |

Response:

```json
{
  "id": "5555eeee5555eeee5555eeee5555",
  "full_name": "Test Signer",
  "email": "test-signer@example.com",
  "has_accepted_terms": false
}
```

`has_accepted_terms` flips to `true` once the signer interacts with their
first signing request. `cpf`, `whatsapp_phone_number`, and `metadata` are
omitted from the response when unset.

### `assinafy_get_signer`

| Argument | Type | Required |
|---|---|---|
| `signer_id` | string | yes |
| `account_id` | string | no |

Response shape: same as `assinafy_create_signer`.

### `assinafy_list_signers`

| Argument | Type | Required |
|---|---|---|
| `page` | int | no |
| `per_page` | int | no |
| `search` | string | no |
| `account_id` | string | no |

Response:

```json
{
  "data": [
    { "id": "bbbb0000bbbb0000bbbb0000", "full_name": "Jane Sampleton", "email": "sender@example.com", "has_accepted_terms": true }
  ],
  "meta": { "current_page": 1, "last_page": 1, "per_page": 5, "total": 3 }
}
```

### `assinafy_update_signer`

Update a signer's contact details. Fails if the signer has active assignments.

| Argument | Type | Required |
|---|---|---|
| `signer_id` | string | yes |
| `full_name`, `email`, `whatsapp_phone_number`, `cpf` | string | at least one |
| `account_id` | string | no |

Response shape: same as `assinafy_create_signer`.

### `assinafy_delete_signer`

| Argument | Type | Required |
|---|---|---|
| `signer_id` | string | yes |
| `account_id` | string | no |

Response (`content[0].text`):

```text
Signer deleted successfully
```

### `assinafy_find_signer_by_email`

| Argument | Type | Required |
|---|---|---|
| `email` | string | yes |
| `account_id` | string | no |

Returns the signer object (same shape as `assinafy_create_signer`) when a
match exists. When no match exists the result is the JSON literal `null`
(`content[0].text == "null"`, no `structuredContent`).

## Assignments

### Enum values (read first)

The Assinafy API rejects the lowercase forms that look idiomatic from
JavaScript / Python. Use the upstream spelling exactly:

| Field | Accepted values |
|---|---|
| `signers[].verification_method` | `Email`, `Whatsapp` |
| `signers[].notification_methods` | `["Email"]`, `["Whatsapp"]`, or both |
| `method` (top-level) | `virtual`, `collect` |

If you pass `"email"` instead of `"Email"` you will get
`API error 400: Método de verificação inválido.` Pairing
`verification_method: "Email"` with an empty `notification_methods` yields
`API error 400: O método de verificação de e-mail requer o método de notificação de e-mail.`

### `assinafy_create_assignment`

Create a signing assignment and notify signers.

| Argument | Type | Required | Description |
|---|---|---|---|
| `document_id` | string | yes | |
| `signers` | array | yes | `[{id, verification_method?, notification_methods?}]`. |
| `method` | string | no | `virtual` (default) or `collect`. |
| `message` | string | no | Invitation message shown to signers. |
| `expires_at` | string | no | ISO 8601. |
| `copy_receivers` | array | no | Signer IDs that receive a copy without signing. |

Response (`structuredContent`) — for a single-signer virtual assignment:

```json
{
  "id": "6666ffff6666ffff6666ffff6666",
  "method": "virtual",
  "sender_email": "sender@example.com",
  "message": "Demo signing request",
  "signers": [
    { "id": "5555eeee5555eeee5555eeee5555", "full_name": "Test Signer", "email": "...", "has_accepted_terms": false }
  ],
  "items": [
    {
      "id": "7777eeee7777eeee7777eeee7777",
      "completed": false,
      "field": { "id": "...", "name": "Virtual", "type": "virtual", "is_active": true, "is_required": false, "is_visible": true },
      "display_settings": [],
      "page": null,
      "signer": { "id": "...", "full_name": "...", "email": "..." },
      "value": null
    }
  ],
  "summary": {
    "signer_count": 1,
    "completed_count": 0,
    "signers": [ { "id": "...", "full_name": "...", "email": "...", "completed": false } ]
  },
  "signing_urls": [
    { "signer_id": "5555eeee5555eeee5555eeee5555", "url": "https://app.assinafy.com.br/sign/<doc>?email=..." }
  ]
}
```

Notes:

- `signing_urls` is an **array of `{signer_id, url}` objects**, not a map.
- `items` enumerates per-page signature placements. For `virtual` assignments
  these are single virtual fields; for `collect` they include `display_settings`
  with pixel coordinates and the originating `page` reference.
- `assignment.expires_at` may be returned as `null`.

### `assinafy_estimate_assignment_cost`

Estimate the credit cost before creating an assignment. Same `signers` /
`method` rules as `assinafy_create_assignment` (you can pass `signers` with
just `verification_method` and `notification_methods`, no `id`, when scoping
out a hypothetical assignment).

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |
| `signers` | array | yes |
| `method` | string | no |

Response:

```json
{
  "breakdown": [],
  "credits": 0,
  "credit_balance": 0,
  "documents": 1,
  "document_balance": 100,
  "extra_document_cost": 0,
  "needs_extra_document": false,
  "has_sufficient_resources": true,
  "total_credits": 0
}
```

`breakdown` populates when the assignment consumes credits (e.g. WhatsApp
notifications); `documents` / `document_balance` track the document budget;
`has_sufficient_resources` is the single boolean to gate the create call on.

### `assinafy_reset_assignment_expiration`

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |
| `assignment_id` | string | yes |
| `expires_at` | string (ISO 8601) | yes |

Response: the updated assignment object (same shape as
`assinafy_create_assignment`).

### `assinafy_resend_notification`

Resend the signing invitation to a single signer.

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |
| `assignment_id` | string | yes |
| `signer_id` | string | yes |

Response:

```json
{ "is_sent": true, "document_id": "doc_...", "signer_id": "sig_..." }
```

### `assinafy_estimate_resend_cost`

Different response shape from `assinafy_estimate_assignment_cost` — this one
is in terms of credits only, not document budget.

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |
| `assignment_id` | string | yes |
| `signer_id` | string | yes |

Response:

```json
{
  "breakdown": [
    { "code": "NotificationEmailResend", "name": "Email Notification Resend", "cost": 0 }
  ],
  "credit_balance": 0,
  "has_sufficient_credits": true,
  "total": 0
}
```

### `assinafy_cancel_signature_request`

Cancel an in-progress signature request for a document. The underlying
endpoint is not part of the public Assinafy Swagger; in practice it returns
`API error 404` on documents in some states (e.g. already-cancelled,
already-certificated). **For routine cleanup, prefer
`assinafy_delete_document`** — it removes the document and any active
assignment in one call regardless of state.

| Argument | Type | Required |
|---|---|---|
| `document_id` | string | yes |
| `reason` | string | yes |
| `account_id` | string | no |

Successful response:

```json
{ "id": "doc_...", "status": "rejected_by_user", "decline_reason": "..." }
```

## Webhooks

### `assinafy_register_webhook`

Register (or replace) the workspace's webhook subscription.

| Argument | Type | Required | Description |
|---|---|---|---|
| `url` | string | yes | HTTPS endpoint receiving events. |
| `email` | string | yes | Contact email for delivery failures. |
| `events` | array | no | Event types to subscribe to; defaults to the full standard set. |
| `is_active` | bool | no | Default `true`. |
| `account_id` | string | no | |

Response:

```json
{
  "url": "https://hooks.example.com/assinafy",
  "email": "ops@example.com",
  "events": ["document_ready", "signer_signed_document"],
  "is_active": true,
  "updated_at": "2026-04-11T14:22:26Z"
}
```

The workspace can have only one subscription. Registering replaces the
existing one. `id` and `created_at` are not part of the standard response.

### `assinafy_get_webhook`

| Argument | Type | Required |
|---|---|---|
| `account_id` | string | no |

A workspace that has never registered a webhook returns an empty-shape object,
not `null`:

```json
{ "url": "", "email": "", "events": [], "is_active": true, "updated_at": "..." }
```

Clients should treat `url == ""` as "no webhook configured."

### `assinafy_delete_webhook`

| Argument | Type | Required |
|---|---|---|
| `account_id` | string | no |

Response (`content[0].text`):

```text
Webhook subscription deleted
```

### `assinafy_inactivate_webhook`

Disable the subscription without deleting it.

| Argument | Type | Required |
|---|---|---|
| `account_id` | string | no |

Response: the subscription object with `is_active: false`.

### `assinafy_list_webhook_event_types`

No arguments. Returns a **bare array** (in `content[0].text`).

```json
[
  { "id": "document_uploaded",        "description": "Triggered when the User has uploaded a Document" },
  { "id": "document_metadata_ready",  "description": "Triggered when the document is ready to be prepared. ..." },
  { "id": "document_prepared",        "description": "Triggered when the User as subject prepares a Document." },
  { "id": "assignment_created",       "description": "Triggered when the User created an assignment ..." },
  { "id": "signature_requested",      "description": "Triggered when the User requested signature of a Document" },
  { "id": "document_ready",           "description": "Triggered when the last Signer ... signs the Document ..." },
  { "id": "signer_created",           "description": "Triggered when the User created a Signer" },
  { "id": "signer_email_verified",    "description": "Triggered when Signer's email has been verified ..." },
  { "id": "signer_whatsapp_verified", "description": "Triggered when Signer's WhatsApp ... has been verified ..." },
  { "id": "signer_signed_document",   "description": "Triggered when Signer completed signing" },
  { "id": "signer_rejected_document", "description": "Triggered when Signer rejected the document" }
  // see live response for the authoritative current list
]
```

This is the canonical list — prefer it over hand-maintained constants.

### `assinafy_list_webhook_dispatches`

| Argument | Type | Required | Description |
|---|---|---|---|
| `event` | string | no | Filter by event type. |
| `delivered` | bool | no | Filter by delivery status. |
| `page`, `per_page` | int | no | Pagination. |
| `account_id` | string | no | |

Response:

```json
{
  "data": [
    {
      "id": "wd_...",
      "event": "signer_signed_document",
      "activity_id": 42,
      "endpoint": "https://hooks.example.com/assinafy",
      "delivered": true,
      "http_status": 200,
      "created_at": 1715184000
    }
  ],
  "meta": { "current_page": 1, "last_page": 1, "per_page": 20, "total": 1 }
}
```

`created_at` on dispatch records is a Unix epoch **integer** (seconds), not an
ISO 8601 string. `endpoint`, `http_status`, `response_body`, and `error` may
be omitted (`null`) when the delivery has not yet been attempted.

### `assinafy_retry_webhook_dispatch`

| Argument | Type | Required |
|---|---|---|
| `dispatch_id` | string | yes |
| `account_id` | string | no |

Response: the updated dispatch record (same shape as the items in
`assinafy_list_webhook_dispatches`).

### `assinafy_verify_webhook_signature`

Validate that a received webhook body matches its `X-Assinafy-Signature`
header using HMAC-SHA256. **The signature is the raw hex digest**, not a
`sha256=<hex>` prefixed value. Pass the secret in any of three ways:

1. `arguments.secret` (per-call).
2. `X-Assinafy-Webhook-Secret` HTTPS header.
3. `_meta.webhook_secret` / `arguments.auth.webhook_secret`.

| Argument | Type | Required | Description |
|---|---|---|---|
| `payload` | string | yes | Raw webhook request body, byte-for-byte. |
| `signature` | string | yes | Value of `X-Assinafy-Signature` (hex digest, no prefix). |
| `secret` | string | no | Webhook secret; falls back to per-request creds. |

Valid signature:

```json
{
  "valid": true,
  "event_type": "signer_signed_document",
  "event_data": { "document_id": "doc_abc", "signer_id": "sig_xyz" }
}
```

Invalid signature (or missing secret):

```json
{ "valid": false }
```

The HMAC comparison is timing-safe. `event_data` is taken from the envelope's
`data` field (or `object`, whichever is present).

## Workspaces

### `assinafy_create_workspace`

| Argument | Type | Required |
|---|---|---|
| `name` | string | yes |
| `primary_color` | string (hex) | no |
| `secondary_color` | string (hex) | no |

Response shape: `WorkspaceResponse` — `{id, name, primary_color?, secondary_color?, created_at}`.

### `assinafy_list_workspaces`

No arguments. Returns every workspace the API key can access, **wrapped in `{data: [...]}`** (the API response is a paginated list).

```json
{
  "data": [
    {
      "id": "aaaa0000aaaa0000aaaa0000",
      "name": "Demo Workspace",
      "is_delete_allowed": true,
      "roles": ["owner"],
      "created_at": "2023-07-18T19:27:29Z"
    }
  ]
}
```

### `assinafy_get_workspace`

| Argument | Type | Required |
|---|---|---|
| `account_id` | string | yes |

Response:

```json
{
  "id": "aaaa0000aaaa0000aaaa0000",
  "name": "Demo Workspace",
  "created_at": "2023-07-18T19:27:29Z"
}
```

`primary_color` / `secondary_color` are present only when the workspace has
them configured.

### `assinafy_update_workspace`

| Argument | Type | Required | Description |
|---|---|---|---|
| `account_id` | string | yes | Workspace to update. |
| `name` | string | no | New display name. |
| `primary_color` | string \| null | no | New color or `null` to clear. |
| `secondary_color` | string \| null | no | New color or `null` to clear. |

Response shape: `WorkspaceResponse`.

### `assinafy_delete_workspace`

| Argument | Type | Required |
|---|---|---|
| `account_id` | string | yes |

Response (`content[0].text`):

```text
Workspace deleted successfully
```

## Templates

### `assinafy_list_templates`

| Argument | Type | Required |
|---|---|---|
| `page`, `per_page` | int | no |
| `search` | string | no |
| `account_id` | string | no |

Response:

```json
{
  "data": [
    {
      "id": "tpl_...",
      "name": "Service Agreement",
      "status": "ready",
      "account_id": "aaaa0000aaaa0000aaaa0000",
      "created_at": "2026-04-01T10:00:00Z",
      "updated_at": "2026-04-02T11:30:00Z"
    }
  ],
  "meta": { "current_page": 1, "last_page": 0, "per_page": 20, "total": 0 }
}
```

Empty workspaces return `data: []` with `total: 0`.

### `assinafy_get_template`

| Argument | Type | Required |
|---|---|---|
| `template_id` | string | yes |
| `account_id` | string | no |

Response:

```json
{
  "id": "tpl_...",
  "name": "Service Agreement",
  "status": "ready",
  "account_id": "aaaa0000aaaa0000aaaa0000",
  "roles": [
    { "id": "role_buyer",  "name": "Buyer" },
    { "id": "role_seller", "name": "Seller" }
  ],
  "created_at": "2026-04-01T10:00:00Z",
  "updated_at": "2026-04-02T11:30:00Z"
}
```

Unknown IDs return `API error 404: Template não encontrado.`

# Errors

Tool-level failures (validation, upstream 4xx/5xx, network) return
`result.isError: true` with the message in `result.content[0].text`:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "isError": true,
    "content": [{ "type": "text", "text": "API error 400: ..." }]
  }
}
```

Common categories:

- **Validation errors.** Missing required arguments, missing credentials, or
  rejected enum values (`Método de verificação inválido` on lowercase
  `verification_method`).
- **Upstream API errors.** The text is prefixed with `API error <code>:`
  followed by the upstream Portuguese-language message (e.g.
  `API error 401: ...`, `API error 404: Página não encontrada.`).
- **Network errors.** Transport-level failures reaching the upstream Assinafy
  API. Surface as `network error: ...`.

Protocol-level failures (malformed JSON-RPC, unknown methods) are returned
through standard JSON-RPC error objects (`response.error`) rather than the
tool envelope.

# Webhook event types (full list)

Subscribe to any subset via `assinafy_register_webhook`. The authoritative
list is whatever `assinafy_list_webhook_event_types` returns — the upstream
catalog evolves.

Documents:

- `document_uploaded`, `document_metadata_ready`, `document_prepared`,
  `document_ready`, `document_processing_failed`

Assignments / signing:

- `assignment_created`, `signature_requested`

Signers:

- `signer_created`, `signer_email_verified`, `signer_whatsapp_verified`,
  `signer_data_confirmed`, `signer_viewed_document`, `signer_signed_document`,
  `signer_rejected_document`, `user_rejected_document`

Templates:

- `template_created`, `template_processed`, `template_processing_failed`
