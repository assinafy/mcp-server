# Conversational tool reference

The server exposes **11 tools covering the same 24 operations**. Tools match chat
tasks; related actions share an entry point. Every grouped call requires an
explicit `action`. Its `oneOf` schema enforces the selected action's required
fields and rejects arguments belonging to another action before any API call.
Deletion and public verification keep their standalone inputs.

| Tool | Actions | MCP annotation |
|---|---|---|
| `assinafy_find_documents` | `list`, `get`, `activities`, `notifications` | Read-only |
| `assinafy_prepare_document` | `upload`, `rename` | Destructive write hint (rename) |
| `assinafy_request_signatures` | `from_pdf`, `from_document`, `from_template` | Non-destructive write |
| `assinafy_follow_up_assignment` | `resend`, `set_expiration` | Destructive write hint (expiration) |
| `assinafy_find_signers` | `list`, `get` | Read-only |
| `assinafy_save_signer` | `create`, `update` | Destructive write hint (update) |
| `assinafy_find_templates` | `list`, `get` | Read-only |
| `assinafy_check_fields` | `list`, `get`, `validate` | Read-only |
| `assinafy_download_document` | `artifact`, `page` | Read-only |
| `assinafy_delete_document` | No action argument | Destructive write |
| `assinafy_verify_document` | No action argument | Read-only; public |

All tools are open-world and marked non-idempotent for client retry purposes.
Annotations apply to the whole tool, so a tool containing an update advertises a
destructive hint even for its additive action. Read-only tasks remain separate
from writes. An explicit user request already authorizes its specified action;
ask only for missing details, ambiguous matches, or a new action outside that request.

Authorization is lazy: connecting, listing tools and public verification work
without a token. Workspace operations use the caller's OAuth grant. The workspace
is discovered automatically; `account_id` may repeat it but cannot override it.
See [authentication](../README.en.md#workspaces-and-permissions) for scopes and
[the client guide](../README.en.md#conversational-behavior) for chat flows.

## Find documents and check progress

`assinafy_find_documents` supports:

| Action | Required arguments besides `action` | Optional arguments |
|---|---|---|
| `list` | None | `page`, `per_page`, `search`, `sort`, `status`, `account_id` |
| `get` | `document_id` | None |
| `activities` | `document_id` | None |
| `notifications` | `document_id`, `assignment_id` | None |

`list` returns `{data:[...]}` with pagination `meta` when supplied by Assinafy.
Pages start at 1; `per_page` is capped upstream at 100. `sort` accepts `name` or
`updated_at`, optionally prefixed by `-`. Resolve a user's description to a
specific document, then `get` its status, pages, artifacts, assignment and signers.
`activities` returns event-specific payloads and request origin information;
`notifications` returns WhatsApp delivery details. Email delivery history is in
the document's assignment signer records.

```json
{"action":"list","search":"Service agreement","status":"pending_signature"}
```

```json
{"action":"get","document_id":"DOCUMENT_ID"}
```

Inspect pending people and signing order before a reminder. A 100% signature
summary can still mean `certificating`; wait for `certificated` and the requested
artifact. Reads never send invitations. Use bounded status checks.

## Prepare without sending

`assinafy_prepare_document` supports `upload` with required `file_name` and
`file_base64`, and optional `account_id`; `rename` requires `document_id` and
`name` (1–255 characters). Upload uses the same PDF validation and limits as
`request_signatures` with `from_pdf`, but creates no contacts or invitations.
Renaming is allowed only before an assignment exists, in the states permitted
by Assinafy. The API normalizes the name.

Keep the document ID, inspect it, resolve signer IDs, then request signatures
with `action: "from_document"`. Preparation is optional when the user has already
asked to send a new PDF and supplied its recipients.

## Request signatures

`assinafy_request_signatures` is the single entry point for sending. Choose one
source explicitly; do not pass inputs from multiple sources. Invitations may
consume the workspace's document allowance and notification credits. Use the
recipients and action authorized by the user.

### `from_pdf`: new PDF and email recipients

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
| `max_wait_secs` | no | Processing timeout, 1–600; default 30. Waits over about two minutes can outlive the delegated API token; the error keeps the uploaded document ID for `from_document` |
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

### `from_document`: existing document and signer IDs

Required: `document_id`, `method` (`virtual` or `collect`), and a non-empty
`signers` array of objects with `id`. Optional: `entries`, `message`, RFC 3339
`expires_at`, and `copy_receivers` (signer IDs).

Inspect the document first. If an assignment already exists, track it instead
of sending again. `virtual` can start from `uploaded`, `metadata_processing` or
`metadata_ready`; Assinafy advances the document when metadata is ready.
`collect` requires `metadata_ready` and non-empty page/field placements in
`entries`. Get page IDs and dimensions from `find_documents` and field IDs from
`check_fields`. Every placement contains `signer_id`, `field_id` and
`display_settings` (`left`, `top`, `width`, `height`, `fontSize`, optionally
`fontFamily` and `backgroundColor`). Geometry uses 150-DPI page-image pixels;
the server checks page readiness and bounds before creating the assignment.

Signers also accept `verification_method` (`Email`, `Whatsapp`,
`DigitalCertificate`), `notification_methods` (`Email`, `Whatsapp`) and `step`.
When ordering is used, every signer supplies a step; steps are contiguous from 1,
and people on the same step sign in parallel. Assinafy validates eligibility and
signing order. Certificate signers must be alone in their step.

```json
{"action":"from_document","document_id":"DOCUMENT_ID","method":"virtual","signers":[{"id":"SIGNER_ID"}]}
```

### `from_template`: saved template and role assignments

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
| `signers[].verification_method` | no | `Email`, `Whatsapp`, or `DigitalCertificate`; rejected otherwise |
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

Discover roles and placements with `assinafy_find_templates`. Match configurable
field names or placement labels using `assinafy_check_fields`; ambiguous matches
need clarification. `editor_fields` takes the placement's `field_id`, not its
placement ID or label. Validate values before creation. Template creation, layout
and role editing remain in Assinafy.

## Follow up on an existing assignment

`assinafy_follow_up_assignment` takes the actual document and assignment IDs:

| Action | Required arguments besides `action` | Result |
|---|---|---|
| `resend` | `document_id`, `assignment_id`, `signer_id` | Notification result; `is_sent` does not mean signed |
| `set_expiration` | `document_id`, `assignment_id`, RFC 3339 `expires_at` | Updated assignment |

Inspect current pending signers before an authorized reminder. Each resend may
consume notification credits; an uncertain response requires inspection before
retrying. Changing expiration does not imply that a reminder was sent.

## Find and save contacts

`assinafy_find_signers` supports `list` with optional `account_id`, `search`,
`page`, `per_page`, and `get` with required `signer_id` and optional `account_id`.
Resolve exact contacts and clarify ambiguous matches before sending.

`assinafy_save_signer` supports:

| Action | Required arguments besides `action` | Optional arguments |
|---|---|---|
| `create` | `full_name` | `email`, `whatsapp_phone_number`, `account_id` |
| `update` | `signer_id` and at least one contact field | `full_name`, `email`, `whatsapp_phone_number`, `government_id`, `account_id` |

Saving a contact sends no signing request. Standalone creation may return a
conflict; look up the existing contact before retrying. Contacts are shared in
the workspace. Assinafy restricts edits to channels already verified on an
in-flight document. Updating an unverified channel invalidates old links/codes;
use an authorized reminder only after a successful correction.

## Discover saved templates

`assinafy_find_templates` supports `list` with optional `page`, `per_page`,
`search`, `account_id`, and `get` with required `template_id` and optional
`account_id`. The list returns `{data:[...]}` plus optional pagination `meta`.
Inspect roles, tags, default document tags, pages and editor-field placements.

`get` uses `GET /accounts/{accountId}/templates/{templateId}`, a compatibility
route absent from the public OpenAPI document. Confirm availability in the target
environment; listing is the documented discovery path. Finding a template does
not generate a document. Use `request_signatures` with `from_template` to send.

## Find and validate fields

`assinafy_check_fields` supports:

| Action | Required arguments besides `action` | Optional arguments |
|---|---|---|
| `list` | None | `account_id`, `include_inactive`, `include_standard` |
| `get` | `field_id` | `account_id` |
| `validate` | `values`: array of `{field_id,value}` | `account_id` |

These actions are read-only, including validation's upstream POST. Validation
forwards `values` as the root JSON array and preserves JSON types, explicit null
and exact numbers. Check each result's `success` and `error_message` before using
values. Keep identifiers with leading zeros as strings. Template editor values
must be strings even when they represent numbers or dates.

Field definitions and value validation do not create fields or template placements.
There is no automatic field-name resolver; match IDs using discovered metadata.

## Download a document or page

`assinafy_download_document` requires `action`:

| Action | Required arguments besides `action` | Optional arguments | Result |
|---|---|---|---|
| `artifact` | `document_id` | `artifact`: `original` (default), `certificated`, `certificate-page`, `pades`, `bundle` | `{document_id,artifact,base64}` |
| `page` | `document_id`, `page_id` | None | `{document_id,page_id,base64}` |

Download signed artifacts only after certification and availability are confirmed.
Binary responses are capped at 64 MiB before base64 encoding. Download only the
bytes needed; use the client's file tools to save them without printing base64.

## Delete or verify

`assinafy_delete_document` takes `document_id`, with no `action`, and returns a
short success message. The user's request must authorize deletion, and Assinafy
enforces deletable states. Never use it as automatic cleanup after an uncertain send.

`assinafy_verify_document` takes `hash`, with no `action`: the signed document's
actual Assinafy SHA-1 signature hash. This is the only tool reachable without
connecting. It sends no credential upstream and returns the verification object,
including `is_valid` when supplied. The server refuses to start if another tool
has no scope policy without being explicitly reviewed as public.

## Migrating from per-operation tools

The old names are no longer registered; refresh the client's tool catalog and
update explicit calls. Inputs and result shapes remain the same, with `action`
added for grouped calls. `assinafy_download_document` also now requires `action`.
Deletion and verification are unchanged.

| Previous tool | Current tool | Add `action` |
|---|---|---|
| `assinafy_send_document_for_signature` | `assinafy_request_signatures` | `from_pdf` |
| `assinafy_create_assignment` | `assinafy_request_signatures` | `from_document` |
| `assinafy_create_document_from_template` | `assinafy_request_signatures` | `from_template` |
| `assinafy_list_documents` | `assinafy_find_documents` | `list` |
| `assinafy_get_document` | `assinafy_find_documents` | `get` |
| `assinafy_list_document_activities` | `assinafy_find_documents` | `activities` |
| `assinafy_list_whatsapp_notifications` | `assinafy_find_documents` | `notifications` |
| `assinafy_upload_document` | `assinafy_prepare_document` | `upload` |
| `assinafy_rename_document` | `assinafy_prepare_document` | `rename` |
| `assinafy_reset_assignment_expiration` | `assinafy_follow_up_assignment` | `set_expiration` |
| `assinafy_resend_notification` | `assinafy_follow_up_assignment` | `resend` |
| `assinafy_list_signers` | `assinafy_find_signers` | `list` |
| `assinafy_get_signer` | `assinafy_find_signers` | `get` |
| `assinafy_create_signer` | `assinafy_save_signer` | `create` |
| `assinafy_update_signer` | `assinafy_save_signer` | `update` |
| `assinafy_list_templates` | `assinafy_find_templates` | `list` |
| `assinafy_get_template` | `assinafy_find_templates` | `get` |
| `assinafy_list_fields` | `assinafy_check_fields` | `list` |
| `assinafy_get_field` | `assinafy_check_fields` | `get` |
| `assinafy_validate_fields` | `assinafy_check_fields` | `validate` |
| `assinafy_download_document` | `assinafy_download_document` | `artifact` |
| `assinafy_download_document_page` | `assinafy_download_document` | `page` |

## Result and error behavior

Successful JSON objects use both MCP text content and `structuredContent`.
Collection and empty fixed-operation responses use `{data: ...}` with pagination
`meta` where available. Binary bytes are base64 inside an object. Upstream errors
use `isError: true` and retain Assinafy's status/message where supplied.

The fixed actions use only the configured methods, routes, query parameters and
body fields from the [Assinafy API](https://api.assinafy.com.br/v1/docs). Callers cannot
supply arbitrary URLs, HTTP methods or credentials. Path identifiers reject
empty values, separators, percent escapes, query/fragment characters, traversal and control
characters. The MCP SDK validates all input schemas.

Writes are never retried automatically. After an uncertain send, inspect the
retained document before deciding what to do. Use `from_document` only if no
assignment exists and its state permits signing. See [errors](errors.md) for
recovery and [Assinafy API reference](https://api.assinafy.com.br/v1/docs) for capabilities outside this catalog.
