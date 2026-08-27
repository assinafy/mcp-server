# Assinafy MCP client guide

This guide describes the deployed MCP surface. The server intentionally exposes
exactly **13 operational document-signing tools**. It does not expose account,
user, API-key, logo, workspace, webhook, tag, field-definition, standalone
signer-management, or signer-session administration tools.

## Endpoint

The server is hosted by Assinafy — there is nothing to deploy. Connect with MCP
Streamable HTTP:

```text
POST https://mcp.assinafy.com.br/mcp
```

## Authentication

The recommended client configuration sends these HTTPS headers on each MCP
request:

```text
X-Api-Key: <ASSINAFY_API_KEY>
X-Assinafy-Account-Id: <ASSINAFY_ACCOUNT_ID>
```

`Authorization: Bearer <token>` also works. The MCP server only passes through
an already-issued bearer token when no API key is present. It does not log in,
exchange an API key, or obtain a bearer token.

Clients unable to attach headers may use per-call `_meta` or
`arguments.auth`:

```json
{
  "name": "assinafy_list_documents",
  "arguments": {
    "auth": {
      "api_key": "<ASSINAFY_API_KEY>",
      "account_id": "<ASSINAFY_ACCOUNT_ID>"
    }
  }
}
```

Headers are safer because inline credentials can enter model context, client
logs, traces, or transcripts.

## Exact tool catalog

| # | Tool | Required input | Effect |
|---:|---|---|---|
| 1 | `assinafy_send_document_for_signature` | `file_name`, `file_base64`, `signers` | Uploads a PDF, waits for processing, creates/reuses email signers, and creates a virtual Email assignment |
| 2 | `assinafy_list_documents` | none | Lists documents with pagination/search/sort/status filters |
| 3 | `assinafy_get_document` | `document_id` | Gets document, artifact, page, assignment, and activity details |
| 4 | `assinafy_get_signing_progress` | `document_id` | Summarizes signed, pending, total, and percentage |
| 5 | `assinafy_download_signed_document` | `document_id` | Downloads the certificated PDF as base64 |
| 6 | `assinafy_download_document` | `document_id` | Downloads an original or selected artifact as base64 |
| 7 | `assinafy_delete_document` | `document_id` | Deletes a document |
| 8 | `assinafy_list_templates` | none | Lists available templates |
| 9 | `assinafy_get_template` | `template_id` | Gets template roles, pages, fields, and defaults |
| 10 | `assinafy_create_document_from_template` | `template_id`, `signers` | Creates/reuses email signers and generates a document from a template |
| 11 | `assinafy_reset_assignment_expiration` | `document_id`, `assignment_id`, `expires_at` | Changes assignment expiry |
| 12 | `assinafy_resend_notification` | `document_id`, `assignment_id`, `signer_id` | Resends one signer notification |
| 13 | `assinafy_verify_document` | `hash` | Verifies a signed-document SHA-1 hash through Assinafy's public endpoint |

Unknown input properties are rejected by the generated MCP schemas.

## Send a PDF for signature

Use `assinafy_send_document_for_signature` for the common end-to-end flow:

```json
{
  "file_name": "agreement.pdf",
  "file_base64": "<BASE64_PDF>",
  "signers": [
    {
      "full_name": "Test Signer",
      "email": "signer@example.com"
    }
  ],
  "message": "Please review and sign",
  "expires_at": "2026-09-30T23:59:59Z"
}
```

Optional inputs are `message`, `expires_at`, `account_id`, `max_wait_secs`
(default 30, maximum 600), and `poll_secs` (default 2). The decoded file limit
is 25 MiB. The file must be a non-empty PDF, every signer must have `full_name`
and a valid `email`, and duplicate emails are rejected case-insensitively.

The result contains the uploaded `document`, created `assignment`, and ordered
`signer_ids`. Assinafy may reuse an existing signer with the exact email.

This workflow is not transactional. If work fails after upload, the MCP error
includes the retained document ID; inspect or delete that document
instead of blindly retrying and creating a duplicate.

## Create from a template

First call `assinafy_list_templates` and `assinafy_get_template` to discover
role IDs, then call:

```json
{
  "template_id": "<TEMPLATE_ID>",
  "signers": [
    {
      "role_id": "<ROLE_ID>",
      "full_name": "Test Signer",
      "email": "signer@example.com"
    }
  ],
  "name": "Generated agreement",
  "message": "Please review and sign",
  "editor_fields": [
    {
      "field_id": "<FIELD_ID>",
      "value": "Approved value"
    }
  ]
}
```

`expires_at`, `editor_fields`, `name`, `message`, and `account_id` are optional.
Each template role ID may appear once. Signers are created or reused by exact
email and use Email verification and notification.

## Follow-up inputs

| Tool | Optional input |
|---|---|
| `assinafy_list_documents` | `page`, `per_page`, `search`, `sort`, `status`, `account_id` |
| `assinafy_download_document` | `artifact`: `original`, `certificated`, `certificate-page`, `pades`, or `bundle` |
| `assinafy_list_templates` | `page`, `per_page`, `search`, `account_id` |
| `assinafy_get_template` | `account_id` |

Account IDs are required only for account-scoped upload, list, signer, and
template operations. Except for public `assinafy_verify_document`, the remaining
document and assignment tools require a valid API key or bearer token, but take
no account ID.

## Side effects and approvals

- Require user confirmation before `assinafy_delete_document`.
- Resetting expiration mutates an active assignment.
- Sending a document, creating from a template, and resending a notification can
  send email and may incur Assinafy usage or cost.
- Downloads return base64 and can consume substantial model context.
- `assinafy_verify_document` is read-only and deliberately sends no workspace
  credential.

## Errors and operational limits

Tool-level failures set MCP `isError: true`. Successful JSON objects are also
returned as `structuredContent`. Safe reads may retry HTTP 429 and transient 5xx
responses; mutating calls are never retried automatically.

The server limits MCP requests to 40 MiB, decoded PDFs to 25 MiB, JSON responses
to 16 MiB, binary responses to 64 MiB, and error responses to 1 MiB.

## Service endpoints

- `GET https://mcp.assinafy.com.br/healthz` and `/readyz` return service status.
- `GET https://mcp.assinafy.com.br/mcp` returns server identity and the exact
  13-tool manifest.
- The hosted endpoint uses HTTPS.
- Your API key and account ID are request-scoped: they are read from each
  request and never stored server-side.
