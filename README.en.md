# Assinafy MCP Server

*[Português (Brasil)](README.md) · English (US)*

MCP server for Assinafy document signing and contract lifecycle management.
It exposes 29 tools over Streamable HTTP:
27 focused signing tools and two CLM tools with fixed actions. The public REST
contract is the [Assinafy API reference](https://api.assinafy.com.br/v1/docs).

## Connect

The server is an OAuth 2.1 resource server. The client discovers protected-resource
metadata, registers through CIMD, opens browser consent with PKCE S256, and stores
and refreshes its own token. A connection selects one Assinafy workspace. The MCP
validates issuer, audience, expiry, workspace, and scopes, then exchanges that token
for a separate short-lived API token. Operator OAuth credentials remain server-side.

```bash
codex mcp add assinafy --url https://mcp.assinafy.com.br/mcp
codex mcp login assinafy

claude mcp add --transport http assinafy https://mcp.assinafy.com.br/mcp
```

For Claude remote connectors, add the same URL and finish browser authorization.
Scopes are `account:read`, `documents:read`, `documents:write`, `templates:read`,
and `templates:write`; a client may approve a smaller set. See
[authentication](docs/authentication.md). You do not need an API key or your own
OAuth application credentials to connect.

## Complete signing flow

1. Find documents, templates, fields, roles, and contacts with the list/get tools.
   Resolve ambiguous names and keep the returned IDs.
2. For a new email PDF, call `assinafy_send_pdf_for_signature`. It validates and
   uploads once, waits for metadata, reuses or creates contacts, and sends a virtual
   Email assignment. For preparation without sending, use
   `assinafy_upload_document`, then `assinafy_create_assignment`.
3. For a saved template, read every role and placed field, validate editor values,
   then call `assinafy_create_document_from_template` with every Signer role filled.
   CopyReceiver roles are optional; Editor roles use `editor_fields`. Wait for
   template processing before inspecting the assignment.
4. Choose `Email`, `Whatsapp`, `DigitalCertificate`, or
   `IdDocumentFaceMatchBasic` per signer. Exactly one notification channel is
   allowed. Email and Whatsapp verification require the matching channel;
   certificate and biometric verification accept either. A1/A3 certificate,
   ID-document, and liveness ceremonies are completed by the human signer.
5. Use `virtual` for field-free signing or `collect` with page and field placements.
   Certificate signers require `virtual` and must be alone in their signing step.
   Ordered steps must be present for every signer and contiguous from 1.
6. Preview assignment, template, or reminder cost with `estimate_only`. Set
   `max_total_credits` on PDF, assignment, or template sends to cap spending. Zero
   permits only a zero-credit send. Paid features, identity eligibility, balances,
   and signing remain enforced by Assinafy.
7. Track the document, assignment, activity pages, and notification history.
   At `pending_identity_review`, an authorized account user reviews the evidence.
8. Wait for `certificated`, download the requested base64 artifact, and optionally
   pass the `verification_code` from `assinafy_get_document` unchanged to
   `assinafy_verify_document`. Do not compute a hash of the PDF.

Synthetic email example:

```json
{"file_name":"agreement.pdf","file_base64":"<BASE64_PDF>","signers":[{"full_name":"Example Signer","email":"signer@example.invalid"}]}
```

`assinafy_update_signer` accepts null or an empty string to clear email, WhatsApp,
or government ID. `assinafy_set_assignment_expiration` accepts an ISO 8601 date or
date-time, or null to remove the deadline. Activity history accepts `sort`, `page`,
and `per_page`. Full inputs, outputs, examples, status handling, and recovery are in
[tool behavior](docs/tools.md), [payload schemas](docs/tool-payloads.json), and
[errors](docs/errors.md).

Writes are never automatically retried. A composite failure can leave a contact or
uploaded document; retain the document ID, inspect its assignment, and continue
without uploading again. Human signatures are never performed by the assistant.

## MCP Apps and Events extensions

Apps-capable hosts render the shared `ui://` resource advertised with
`text/html;profile=mcp-app`. It powers document status and reminders, signature
review, visual field placement, and contract browsing. Other hosts receive the same
tool result as text and structured content.

Events lets compatible clients follow document status changes, completed
certification, and contract activities. Ask your assistant to watch a document.
The OAuth connection and subscription must stay active; the client renews them.
See [Events](docs/events.md) for integration and delivery behavior.

Account webhooks are managed in Assinafy. Save a signed endpoint's one-time secret;
OAuth cannot retrieve or rotate it. Receivers verify Standard Webhooks signatures
over raw bytes, reject stale timestamps, and deduplicate delivery IDs.

## Contract management

Use `assinafy_browse_contracts` to list, read, or follow a contract family. Use
`assinafy_update_contract` to set terms, renew, or terminate only after reading the
current `updated_at`; renewal also retains the current term. Re-read conflicts and
never overwrite silently. Amendments are separate signed documents linked through
`parent_document_id` and `relation_type`. Events notify; they never authorize writes.
