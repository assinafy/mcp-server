# Assinafy MCP Server

*[Português (Brasil)](README.md) · English (US)*

MCP server for Assinafy document signing and contract lifecycle management.
It exposes 29 tools over Streamable HTTP:
27 focused signing tools and two CLM tools with fixed actions. The public REST
contract is the [Assinafy API reference](https://api.assinafy.com.br/v1/docs).

## Connect

Add `https://mcp.assinafy.com.br/mcp` as a remote MCP connection in your client.
Sign in to Assinafy in your browser, choose the workspace, and authorize the
permissions you need. Your client maintains the OAuth connection and refreshes
access. Use the option below that matches your client:

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

## Try it in your client

After connecting, copy these prompts into your assistant conversation. Use a PDF
without sensitive data and a recipient who authorized the test. Replace
`signer@example.invalid` with that authorized email: the example address is
fictional and cannot receive messages. Real sends and reminders may use credits.

### 1. Check the connection without sending anything

> List up to five recent documents in my workspace with their names, IDs, and
> statuses. Do not change or send any documents.

**Expected result:** documents from the selected workspace, or an empty list.
For authorization errors, see [authentication](docs/authentication.md).
Then ask: “Show pending signers and activities for document ID.”
Replace `ID` with the returned value; the assistant should clarify duplicate names.

### 2. Prepare a PDF, check the cost, and send by email

Attach `signing-test.pdf` in a client that can provide files to MCP tools:

> Prepare this PDF for Email signing by Example Signer,
> signer@example.invalid. Estimate the cost without sending invitations. Show
> the document ID, required credits, and anything preventing the send.

**Expected result:** an uploaded document and a cost estimate, without new
contacts or invitations. The PDF estimate uploads the file; keep the document ID.
After reviewing the result, replace `ID` and `LIMIT` with your chosen values:

> Send the prepared document ID to Example Signer,
> signer@example.invalid, using Email, with a total limit of LIMIT credits.
> Reuse the document; do not upload it again.

**Expected result:** document and assignment IDs, with the signers.
The recipient receives the invitation and signs in Assinafy. Do not repeat the
send after a timeout: read the document and check whether an assignment exists.

### 3. Use a saved template

> List my templates. For the template I choose, show the signer roles and fields
> I need to fill in before creating a document.

After choosing a template and supplying all required role and field values:

> Estimate the cost of this template with the recipients and values provided.
> Do not create or send the document yet.

**Expected result:** roles/fields and a cost estimate. To send, ask
“Create and send with these values, with a total limit of LIMIT credits.”
Keep the returned ID and wait for processing before checking signatures.

### 4. Try another signing method

For a prepared document without an assignment, choose the method explicitly:

| Method | Example request |
|---|---|
| WhatsApp | “Estimate WhatsApp signing for document ID by the selected contact, with WhatsApp notification.” |
| A1/A3 certificate | “Estimate digital certificate signing for document ID by the selected contact, with Email notification.” |
| Biometrics | “Estimate ID-document and facial verification signing for document ID by the selected contact, with Email notification.” |

**Expected result:** cost and eligibility, or the API's reason it cannot proceed.
Check the contact's phone number for WhatsApp and government ID (CPF/CNPJ) for
certificate or biometric signing. After reviewing, request the send using the
same method and a credit limit. Certificate signers must be alone in their step.
The person provides their certificate, photos, and liveness check in Assinafy,
never to the assistant.

### 5. Track, remind, and download the result

> Check document ID and tell me who still needs to sign. Show recent activities
> and the available notification history.

For a reminder: “Estimate the cost of reminding the selected pending signer on
document ID.” After reviewing, ask “Send that reminder.”

> When document ID is certificated, download the signed PDF and verify its
> authenticity using the verification code returned by Assinafy.

**Expected result:** current status; after certification, the file and verification
result. Your client must be able to save the file returned by the MCP tool.
If signatures are still pending, check again later or use Events below.
At `pending_identity_review`, the account must review the evidence in Assinafy.

### 6. Receive Events and inspect contracts

> Monitor document ID and notify me when it is certificated.

**Expected result:** a subscription in an Events-capable client and a notification
when a new certification occurs while monitoring. Keep the connection active.
If the client does not support Events, ask for another status check;
the prompt alone does not enable background monitoring.

> List my tracked contracts and show the terms and end date of the contract
> I choose. Do not renew or terminate anything.

**Expected result:** current terms, or a notice that the plan does not include
this feature. A contract's end date is different from its signing deadline.

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
