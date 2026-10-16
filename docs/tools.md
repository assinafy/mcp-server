# Conversational tool reference

The server exposes **27 tools**, one per operation. Each tool takes only its own
arguments; none takes an `action`. Input schemas are strict: missing required
fields and unknown arguments are rejected before any API call.

| Tool | Purpose | Read-only | Destructive | Idempotent | Open-world |
|---|---|:---:|:---:|:---:|:---:|
| `assinafy_list_documents` | Find documents by search, status or page | yes | no | yes | no |
| `assinafy_get_document` | Status, pages, artifacts, assignment and signers | yes | no | yes | no |
| `assinafy_list_document_activities` | Document events | yes | no | yes | no |
| `assinafy_list_whatsapp_notifications` | WhatsApp delivery details | yes | no | yes | no |
| `assinafy_upload_document` | Upload a PDF without sending | no | no | no | no |
| `assinafy_rename_document` | Rename before an assignment exists | no | yes | no | no |
| `assinafy_send_pdf_for_signature` | New PDF to email recipients | no | no | no | yes |
| `assinafy_create_assignment` | Request signatures on an existing document | no | no | no | yes |
| `assinafy_create_document_from_template` | Generate a document from a template and send it | no | no | no | yes |
| `assinafy_resend_notification` | Remind a pending signer | no | no | no | yes |
| `assinafy_set_assignment_expiration` | Change the deadline | no | no | no | no |
| `assinafy_list_signers` | Find contacts | yes | no | yes | no |
| `assinafy_get_signer` | Read one contact | yes | no | yes | no |
| `assinafy_create_signer` | Create a contact | no | no | no | no |
| `assinafy_update_signer` | Correct a contact | no | yes | no | no |
| `assinafy_list_templates` | Find templates | yes | no | yes | no |
| `assinafy_get_template` | Read one template | yes | no | yes | no |
| `assinafy_list_fields` | Field definitions | yes | no | yes | no |
| `assinafy_get_field` | One field definition | yes | no | yes | no |
| `assinafy_validate_fields` | Validate field values | yes | no | yes | no |
| `assinafy_download_document` | Download an artifact | yes | no | yes | no |
| `assinafy_download_document_page` | Download a page image | yes | no | yes | no |
| `assinafy_delete_document` | Delete a document | no | yes | no | no |
| `assinafy_verify_document` | Public verification by hash | yes | no | yes | no |
| `assinafy_show_document` | Status card ([visual card](#visual-cards)) | yes | no | yes | no |
| `assinafy_review_signature_request` | Review before sending ([visual card](#visual-cards)) | yes | no | yes | no |
| `assinafy_place_fields` | Field placement editor ([visual card](#visual-cards)) | yes | no | yes | no |

Every tool declares all four MCP annotations. Only the four tools that send
messages to people are open-world. An explicit user request already authorizes
the operation it names; ask only for missing details, ambiguous matches, or a new
operation outside that request.

Authorization is lazy: connecting, listing tools and public verification work
without a token. Workspace operations use the caller's OAuth grant. The workspace
is discovered automatically; `account_id` may repeat it but cannot override it.
See [authentication](../README.en.md#workspaces-and-permissions) for scopes and
[the client guide](../README.en.md#conversational-behavior) for chat flows.

## Complete payloads

[Tool payload schemas and examples](tool-payloads.json) documents every tool's
complete input and response schema, request/response examples, annotations and
REST route or workflow. Shared response objects use `$defs` references. Tests
check every input schema against the advertised catalog and validate examples.
The examples use synthetic contacts and an actual synthetic PDF fixture.

MCP calls use `tools/call` with `{name,arguments}`. Successful JSON results carry
the same JSON in text and `structuredContent`. REST response envelopes are
unwrapped. Lists retain `{data,meta?}`. Document deletion returns text. Failures
use `isError: true`, preserving API status/message and any authentication
challenge. Optional response fields depend on resource state and permissions.

## Costs within the signing flow

Set `estimate_only: true` on `assinafy_send_pdf_for_signature`,
`assinafy_create_assignment`, `assinafy_create_document_from_template`, or
`assinafy_resend_notification` to receive `{cost_estimate: {...}}` without sending.
No separate cost tools are registered. Check `has_sufficient_resources`,
`blocking_reason`, `total_credits`, balances and breakdown before sending.
The API determines current costs and rechecks resources at actual creation.

The PDF preview validates and uploads the PDF, waits for processing, and returns
`{document,cost_estimate}`. It creates no contacts or invitations. Retain its ID
and send with `assinafy_create_assignment`; uploading again creates a duplicate.
Assignment previews require `document_id`, `method` and signing choices; signer
IDs are optional for previews. Collect previews also require `entries` but do not
read page geometry. Actual collect sends require `documents:read` to check bounds.
Template previews require `template_id` and one `role_id` per signer; name/email
or contact ID is optional for this preview and no contact is created. Reminder
previews require the same document, assignment and signer IDs as a reminder.
Send through the same tool with `estimate_only` false/omitted after checking the
result, using the prepared document rather than rerunning the PDF upload.

## Biometric and certificate verification

Choose `DigitalCertificate` for the signer's own ICP-Brasil A1/A3 certificate,
or `IdDocumentFaceMatchBasic` for ID photos and liveness. The signer completes
these checks in Assinafy; the MCP does not handle private certificate keys or
perform a human signature. Configure `government_id` on the contact first.
Certificate signers must be alone in their signing step.

If biometric verification needs manual review, `identity_verification_status`
shows `InReview` on the assignment signer. After all signatures, the document
waits at `pending_identity_review` until the account reviews it in Assinafy.
Approval can release certification; rejection cancels the document and sends
cancellation notices. The existing document/status tools show these states.

## Account webhooks

Configure account webhooks in Assinafy. Paid plans support up to three endpoints;
other accounts have one. Create with signing enabled and store the one-time
secret securely. OAuth applications cannot retrieve or rotate it later; a lost
secret requires an explicitly authorized delete/recreate in the account's
integration. There are no account-webhook administration tools in this catalog.

Receivers verify Standard Webhooks `webhook-id`, `webhook-timestamp` and
`webhook-signature` over the exact raw body: decode the key after `whsec_`,
compute HMAC-SHA256 of `id.timestamp.body`, base64-encode it and compare against a
`v1,` signature in constant time. Reject timestamps outside five minutes and
deduplicate the delivery ID. Body timestamps are Unix seconds. See the
[official contract](https://api.assinafy.com.br/v1/docs). MCP Events use their
own callbacks and do not modify these account endpoints.

## Find documents and check progress

| Tool | Required | Optional |
|---|---|---|
| `assinafy_list_documents` | None | `page`, `per_page`, `search`, `sort`, `status`, `method`, `tags`, `account_id` |
| `assinafy_get_document` | `document_id` | None |
| `assinafy_list_document_activities` | `document_id` | None |
| `assinafy_list_whatsapp_notifications` | `document_id`, `assignment_id` | None |

`assinafy_list_documents` returns `{data:[...]}` with pagination `meta` when
supplied by Assinafy. Pages start at 1; `per_page` is capped upstream at 100.
`sort` accepts `name` or `updated_at`, optionally prefixed by `-`. Resolve a
user's description to a specific document, then use `assinafy_get_document` for
its status, pages, artifacts, assignment and signers.
`assinafy_list_document_activities` returns event-specific payloads and request
origin information; `assinafy_list_whatsapp_notifications` returns WhatsApp
delivery details. Email delivery history is in the document's assignment signer records.

```json
{"search":"Service agreement","status":"pending_signature"}
```

```json
{"document_id":"DOCUMENT_ID"}
```

Inspect pending people and signing order before a reminder. A 100% signature
summary can still mean `certificating`; wait for `certificated` and the requested
artifact. Reads never send invitations. Use bounded status checks.

## Prepare without sending

`assinafy_upload_document` requires `file_name` and `file_base64`, with optional
`account_id`. It uses the same PDF validation and limits as
`assinafy_send_pdf_for_signature`, but creates no contacts or invitations.
`assinafy_rename_document` requires `document_id` and `name` (1–255 characters).
Renaming is allowed only before an assignment exists, in the states permitted
by Assinafy. The API normalizes the name.

Keep the document ID, inspect it, resolve signer IDs, then request signatures
with `assinafy_create_assignment`. Preparation is optional when the user has
already asked to send a new PDF and supplied its recipients.

## Request signatures

Three tools send signing requests; pick the one that matches the source.
Invitations may consume the workspace's document allowance and notification
credits. Use the recipients authorized by the user.

### `assinafy_send_pdf_for_signature`: new PDF and email recipients

The common email-signing workflow: validates everything, uploads the PDF, waits for it to become ready, creates or reuses signers by exact email, then creates a virtual assignment with Email verification and Email notification.

Input:

| Field | Required | Description |
|---|:---:|---|
| `file_name` | yes | PDF filename ending in `.pdf`; path components are discarded |
| `file_base64` | yes | Base64 PDF bytes; decoded limit 25 MiB and content must contain a PDF header in the first 1 KiB |
| `signers` | yes | Non-empty array of `{full_name,email}`; duplicate emails are rejected case-insensitively |
| `message` | no | Invitation message |
| `expires_at` | no | RFC 3339 timestamp |
| `account_id` | no | Authorized workspace ID; discovered automatically in OAuth mode |
| `max_wait_secs` | no | Processing timeout, 1–600; default 30. If the connection's access token ends within this wait plus two minutes, nothing is uploaded and the error says how many seconds remain. A timeout keeps the uploaded document ID for `assinafy_create_assignment` |
| `poll_secs` | no | Poll interval, 1 through `max_wait_secs`; default 2 |

Result:

```json
{
  "document": {"id": "..."},
  "assignment": {"id": "..."},
  "signer_ids": ["..."]
}
```

The operation is not transactional. If work fails after upload, the error names the retained document ID. It intentionally does not auto-delete data after an uncertain upstream response.

### `assinafy_create_assignment`: existing document and signer IDs

Required: `document_id`, `method` (`virtual` or `collect`), and a non-empty
`signers` array of objects with `id`. Optional: `entries`, `message`, RFC 3339
`expires_at`, and `copy_receivers` (signer IDs).

Inspect the document first. If an assignment already exists, track it instead
of sending again. `virtual` can start from `uploaded`, `metadata_processing` or
`metadata_ready`; Assinafy advances the document when metadata is ready.
`collect` requires `metadata_ready` and non-empty page/field placements in
`entries`. Get page IDs and dimensions from `assinafy_get_document` and field IDs
from `assinafy_list_fields`. Every placement contains `signer_id`, `field_id` and
`display_settings` (`left`, `top`, `width`, `height`, `fontSize`, optionally
`fontFamily` and `backgroundColor`). Geometry uses 150-DPI page-image pixels;
the server checks page readiness and bounds before creating the assignment.

Signers also accept `verification_method` (`Email`, `Whatsapp`,
`DigitalCertificate`, `IdDocumentFaceMatchBasic`), `notification_methods` (`Email`, `Whatsapp`) and `step`.
When ordering is used, every signer supplies a step; steps are contiguous from 1,
and people on the same step sign in parallel. Assinafy validates eligibility and
signing order. Certificate signers must be alone in their step, and their CPF/CNPJ
must be saved on the contact with `assinafy_create_signer` or `assinafy_update_signer`
when authorized. Biometric verification (`IdDocumentFaceMatchBasic`) needs its
account feature and a valid CPF. Email verification pairs with Email notification;
Whatsapp pairs with Whatsapp. Certificate/biometric verification accepts either
channel. Exactly one notification method is allowed; omitted settings are inferred.

```json
{"document_id":"DOCUMENT_ID","method":"virtual","signers":[{"id":"SIGNER_ID"}]}
```

### `assinafy_create_document_from_template`: saved template and role assignments

This operational wrapper fills every template role, taking either an existing signer ID or a contact to create or reuse, then invokes the documented template-document endpoint. One call may mix both.

Input:

| Field | Required | Description |
|---|:---:|---|
| `template_id` | yes | Template ID |
| `signers` | yes | Non-empty array, one entry per role; role IDs must be unique |
| `signers[].role_id` | yes | Template role the entry fills |
| `signers[].id` | no | Existing signer ID; give this **or** `full_name` and `email`, never both |
| `signers[].full_name` | no | Contact name; required when `id` is omitted |
| `signers[].email` | no | Contact email; required when `id` is omitted. An exact match is reused |
| `signers[].verification_method` | no | `Email`, `Whatsapp`, `DigitalCertificate`, or `IdDocumentFaceMatchBasic`; rejected otherwise |
| `signers[].notification_methods` | no | At most one channel: `Email` or `Whatsapp`. Assinafy infers verification when it is omitted |
| `signers[].step` | no | Signing order, 1 or greater, contiguous from 1; all entries must set it if any does |
| `name` | no | Generated document name |
| `message` | no | Invitation message |
| `expires_at` | no | RFC 3339 timestamp |
| `tags` | no | Tag **names**, merged with the template defaults |
| `editor_fields` | no | `{field_id,value}` entries to bake into the document |
| `account_id` | no | Authorized workspace ID; discovered automatically in OAuth mode |

An entry resolved from an email defaults to Email verification and notification;
supplying `notification_methods` alone leaves the matching verification method to
Assinafy. Returns the created document. A single email may fill multiple different
roles; signer lookup prevents duplicate account signers.

Every entry is validated before contact creation, including notification channels.
Assinafy still validates role eligibility and signing order. Template creation is
not transactional: a later upstream failure can leave newly created contacts.

Discover roles and placements with `assinafy_list_templates`. Match configurable
field names or placement labels using `assinafy_list_fields`; ambiguous matches
need clarification. `editor_fields` takes the placement's `field_id`, not its
placement ID or label. Validate values before creation. Template creation, layout
and role editing remain in Assinafy.

## Follow up on an existing assignment

Both tools take the actual document and assignment IDs:

| Tool | Required | Result |
|---|---|---|
| `assinafy_resend_notification` | `document_id`, `assignment_id`, `signer_id` | Notification result; `is_sent` does not mean signed |
| `assinafy_set_assignment_expiration` | `document_id`, `assignment_id`, RFC 3339 `expires_at` | Updated assignment |

Inspect current pending signers before an authorized reminder. Each resend may
consume notification credits; an uncertain response requires inspection before
retrying. Changing expiration does not imply that a reminder was sent.

## Find and save contacts

`assinafy_list_signers` takes optional `account_id`, `search`, `page` and
`per_page`; `assinafy_get_signer` requires `signer_id` with optional `account_id`.
Resolve exact contacts and clarify ambiguous matches before sending.

| Tool | Required | Optional |
|---|---|---|
| `assinafy_create_signer` | `full_name` | `email`, `whatsapp_phone_number`, `government_id`, `account_id` |
| `assinafy_update_signer` | `signer_id` and at least one of `full_name`, `email`, `whatsapp_phone_number`, `government_id` | `account_id` |

Saving a contact sends no signing request. Standalone creation may return a
conflict; look up the existing contact before retrying. Contacts are shared in
the workspace. Assinafy restricts edits to channels already verified on an
in-flight document. Updating an unverified channel invalidates old links/codes;
use an authorized reminder only after a successful correction. Save CPF/CNPJ in `government_id` only when authorized for certificate or biometric
verification. Assinafy normalizes and validates the value; treat it as sensitive data.

## Discover saved templates

`assinafy_list_templates` takes optional `page`, `per_page`, `search` and
`account_id`; `assinafy_get_template` requires `template_id` with optional
`account_id`. The list returns `{data:[...]}` plus optional pagination `meta`.
Inspect roles, tags, default document tags, pages and editor-field placements.

`assinafy_get_template` uses `GET /accounts/{accountId}/templates/{templateId}`,
a compatibility route absent from the public OpenAPI document. Confirm
availability in the target environment; listing is the documented discovery path.
Finding a template does not generate a document. Use
`assinafy_create_document_from_template` to send.

## Find and validate fields

| Tool | Required | Optional |
|---|---|---|
| `assinafy_list_fields` | None | `account_id`, `include_inactive`, `include_standard` |
| `assinafy_get_field` | `field_id` | `account_id` |
| `assinafy_validate_fields` | `values`: array of `{field_id,value}` | `account_id` |

These tools are read-only, including validation's upstream POST. Validation
forwards `values` as the root JSON array and preserves JSON types, explicit null
and exact numbers. Check each result's `success` and `error_message` before using
values. Keep identifiers with leading zeros as strings. Template editor values
must be strings even when they represent numbers or dates.

Field definitions and value validation do not create fields or template placements.
There is no automatic field-name resolver; match IDs using discovered metadata.

## Download a document or page

| Tool | Required | Optional | Result |
|---|---|---|---|
| `assinafy_download_document` | `document_id` | `artifact`: `original` (default), `certificated`, `certificate-page`, `pades`, `bundle` | `{document_id,artifact,base64}` |
| `assinafy_download_document_page` | `document_id`, `page_id` | None | `{document_id,page_id,base64}` |

Download signed artifacts only after certification and availability are confirmed.
Binary responses are capped at 64 MiB before base64 encoding. Download only the
bytes needed; use the client's file tools to save them without printing base64.

## Delete or verify

`assinafy_delete_document` takes `document_id` and returns a short success
message. The user's request must authorize deletion, and Assinafy enforces
deletable states. Never use it as automatic cleanup after an uncertain send.

`assinafy_verify_document` takes `hash`: the signed document's actual Assinafy
SHA-1 signature hash. This is the only tool reachable without connecting. It
sends no credential upstream and returns the verification object, including
`is_valid` when supplied. The server refuses to start if another tool has no
scope policy without being explicitly reviewed as public.

## Visual cards

Three display tools render interactive cards in hosts that support
[MCP Apps](https://modelcontextprotocol.io/extensions/apps), such as Claude (web
and desktop) and ChatGPT. Every other client, including Claude Code, receives the
same result as a plain-text summary, so no workflow depends on the UI. Each tool is
read-only: a card's buttons call the regular tools above, under the same OAuth grant.

| Tool | Input | Card | Text summary |
|---|---|---|---|
| `assinafy_show_document` | `document_id` | Status, who signed, who is pending, expiry; **Remind** the next pending signer and **Refresh** | Status, signed count, pending names, expiry |
| `assinafy_review_signature_request` | The `assinafy_create_assignment` sending arguments (without `estimate_only`) | Recipients with emails and verification, expiry; one **Send** button | Recipients and “nothing has been sent” |
| `assinafy_place_fields` | The `assinafy_create_assignment` arguments except `method`, `entries` and `estimate_only` | Fullscreen editor: place signature, initials, date and custom fields on each page, then **Send** with `method: "collect"` | Page IDs and sizes in 150-DPI pixels and field IDs, to build `entries` |

Send buttons first recheck the document and stop if it already has a signature
request, so a reloaded card or a send from the chat cannot duplicate invitations.
After a send, the card tells the assistant what happened. Cards never receive
signing URLs. `assinafy_place_fields` requires a document in `metadata_ready` with
no assignment; placements are validated against each page's bounds before sending.

## Migrating from grouped tools

Grouped tools that took an `action` argument are no longer registered. Refresh
the client's tool catalog, call the matching tool and drop `action`; the other
arguments and result shapes are unchanged. Signer creation and updates accept
`government_id` for supported identity verification methods.

| Previous tool and `action` | Current tool |
|---|---|
| `assinafy_find_documents` `list` | `assinafy_list_documents` |
| `assinafy_find_documents` `get` | `assinafy_get_document` |
| `assinafy_find_documents` `activities` | `assinafy_list_document_activities` |
| `assinafy_find_documents` `notifications` | `assinafy_list_whatsapp_notifications` |
| `assinafy_prepare_document` `upload` | `assinafy_upload_document` |
| `assinafy_prepare_document` `rename` | `assinafy_rename_document` |
| `assinafy_request_signatures` `from_pdf` | `assinafy_send_pdf_for_signature` |
| `assinafy_request_signatures` `from_document` | `assinafy_create_assignment` |
| `assinafy_request_signatures` `from_template` | `assinafy_create_document_from_template` |
| `assinafy_follow_up_assignment` `resend` | `assinafy_resend_notification` |
| `assinafy_follow_up_assignment` `set_expiration` | `assinafy_set_assignment_expiration` |
| `assinafy_find_signers` `list` / `get` | `assinafy_list_signers` / `assinafy_get_signer` |
| `assinafy_save_signer` `create` / `update` | `assinafy_create_signer` / `assinafy_update_signer` |
| `assinafy_find_templates` `list` / `get` | `assinafy_list_templates` / `assinafy_get_template` |
| `assinafy_check_fields` `list` / `get` / `validate` | `assinafy_list_fields` / `assinafy_get_field` / `assinafy_validate_fields` |
| `assinafy_download_document` `artifact` | `assinafy_download_document` |
| `assinafy_download_document` `page` | `assinafy_download_document_page` |

## Result and error behavior

Successful JSON objects use both MCP text content and `structuredContent`.
Collection and empty fixed-operation responses use `{data: ...}` with pagination
`meta` where available. Binary bytes are base64 inside an object. Upstream errors
use `isError: true` and retain Assinafy's status/message where supplied.

The fixed operations use only the configured methods, routes, query parameters and
body fields from the [Assinafy API](https://api.assinafy.com.br/v1/docs). Callers cannot
supply arbitrary URLs, HTTP methods or credentials. Path identifiers reject
empty values, separators, percent escapes, query/fragment characters, traversal and control
characters. The MCP SDK validates all input schemas.

Writes are never retried automatically. After an uncertain send, inspect the
retained document before deciding what to do. Use `assinafy_create_assignment`
only if no assignment exists and its state permits signing. See [errors](errors.md)
for recovery and the [Assinafy API reference](https://api.assinafy.com.br/v1/docs) for capabilities outside this catalog.
