# Assinafy MCP client guide

*[Leia em português](README.md) · English*

Connect Assinafy to your assistant to prepare and send documents, track signatures,
follow up with recipients, and download signed copies.

Use `https://mcp.assinafy.com.br/mcp` as the server URL. Sign in to Assinafy,
choose a workspace, and approve the requested permissions. You do not need to
create an OAuth application or enter a client ID, client secret, API key, or
workspace header.

You can connect and view the tool catalog before signing in. Authorization is
requested when a tool needs access to your workspace.

## Codex

```bash
codex mcp add assinafy --url https://mcp.assinafy.com.br/mcp
codex mcp login assinafy
```

Complete the browser sign-in, choose your workspace, and approve the permissions.
See [Codex MCP documentation](https://developers.openai.com/codex/mcp).

## Claude Code

```bash
claude mcp add --transport http assinafy https://mcp.assinafy.com.br/mcp
claude mcp login assinafy
```

You can also open `/mcp` in Claude Code to authenticate Assinafy. Complete the
browser sign-in and select your workspace. See
[Claude Code MCP documentation](https://code.claude.com/docs/en/mcp).

## Claude remote connectors

In Claude's connector settings, add a custom remote connector named Assinafy with
URL `https://mcp.assinafy.com.br/mcp`, then connect and complete Assinafy consent.
Leave optional OAuth client credentials unset when CIMD is supported. Availability
of custom connectors depends on the client's plan and organization settings.
A local command registration is for Claude Code; a hosted connector needs the
publicly reachable HTTPS URL.

See [Claude connector authentication](https://claude.com/docs/connectors/building/authentication).

## ChatGPT

Configure a remote MCP connection with OAuth and the Assinafy URL. Select CIMD
if a registration choice is offered, leave optional client credentials unset,
and complete the Assinafy sign-in and workspace selection. See
[ChatGPT authentication](https://developers.openai.com/plugins/build/auth).

## VS Code / GitHub Copilot Chat

Add this entry to VS Code's MCP configuration:

```json
{
  "servers": {
    "assinafy": {
      "type": "http",
      "url": "https://mcp.assinafy.com.br/mcp"
    }
  }
}
```

Start the server and complete the browser authorization prompt. VS Code selects
CIMD when advertised and supports requesting additional scopes when a tool needs
them. See [VS Code authentication support](https://code.visualstudio.com/updates/v1_106#_authentication-client-id-metadata-document-authentication-flow)
and [MCP configuration](https://code.visualstudio.com/docs/agents/reference/mcp-configuration).

## Workspaces and permissions

One connection authorizes one workspace. Account-scoped tools discover that account
automatically. An `account_id` argument may repeat it but cannot select another one.
Connect each additional workspace separately. Never put API keys, OAuth tokens, or
any other credential in prompts, tool arguments, or `_meta`; requests carrying them
are rejected.

You configure the MCP URL. Discovery advertises
`account:read documents:read documents:write templates:read templates:write` for the whole
catalog. Login requests this full set by default so one consent covers all tools.
You can explicitly choose fewer scopes in your client or consent screen; actions
covered by those permissions still work. A narrower grant may need additional
consent before sending. Assinafy requires `templates:write` to generate a
document from a saved template, and `documents:write` to validate field values.
Request `offline_access` when the client supports refresh tokens for background
access.

A missing permission returns HTTP `403 insufficient_scope` with the scopes to
approve before the workflow starts. Reauthorize with the requested permissions
and try again. Other failures can still interrupt a workflow after it starts;
inspect any retained `document_id` before retrying.

For Codex, explicitly authorize the complete document and template flow with:

```bash
codex mcp login assinafy --scopes account:read,documents:read,documents:write,templates:read,templates:write,offline_access
```

For an intentionally read-only document connection, choose:

```bash
codex mcp login assinafy --scopes account:read,documents:read,offline_access
```

Your client stores and refreshes its tokens. If a grant expires or is revoked,
let the client refresh it or reconnect through Assinafy. A connection ends 30
days after you approve it, even when refreshed, so expect to reconnect monthly.
Never paste tokens into the conversation or tool arguments.

## What you can ask

Talk to your assistant in plain language, in any language it understands. It
picks the right tool, looks up names for you and confirms only what is
ambiguous. Names, emails and dates below are examples.

**Check on documents**

| You say | What happens |
|---|---|
| “Which documents are still waiting for signatures?” | Lists documents in `pending_signature`, newest first |
| “Has everyone signed the Acme service agreement?” | Finds the document and reports who signed and who is pending |
| “Who hasn't signed the NDA yet, and when were they invited?” | Reads each pending signer's step and invitation history |
| “What happened with the office lease this week?” | Shows the document's event timeline |
| “Did the WhatsApp invitation to Carlos arrive?” | Reads the WhatsApp delivery history for that assignment |

**Send for signature**

| You say | What happens |
|---|---|
| “Send contract.pdf to Ana Lima, ana@example.com, for signature.” | Uploads the PDF, waits for processing and emails Ana an invitation |
| “Upload proposal.pdf but don't send it yet. Call it Proposal v2.” | Uploads and renames it; no one is notified |
| “Send Proposal v2 to Ana first, then Bruno once she signs.” | Requests signatures on the uploaded document with signing order |
| “Send it with the message ‘Please sign by Friday’ and make it expire on 30 June.” | Adds the invitation message and expiry to the request |
| “Send the lease to Ana and copy finance@example.com without asking them to sign.” | Saves the contact, then sends with a copy-only recipient |

To send a file, attach it or point your assistant to it. The assistant reads the
file and sends its contents; the server never opens paths on your computer.

**Use templates**

| You say | What happens |
|---|---|
| “What templates do we have?” | Lists saved templates with their roles and fields |
| “Use the NDA template for Carla Souza, carla@example.com, with company name Acme Ltda.” | Matches the field, validates the value, fills the role and sends |
| “Is 123.456.789-09 valid for the CPF field?” | Validates the value without creating anything |

**Manage contacts**

| You say | What happens |
|---|---|
| “Is Bruno Costa already in our contacts?” | Searches the workspace's signers |
| “Add Bruno Costa, bruno@example.com, as a contact.” | Creates the contact; no invitation is sent |
| “Change Carla's WhatsApp number to +55 11 91234-5678.” | Updates the contact, subject to Assinafy's verification rules |

**Follow up**

| You say | What happens |
|---|---|
| “Remind Ana to sign the service agreement.” | Checks she is still pending, then sends one reminder |
| “Give the lease signers until 15 July.” | Changes the assignment's expiry; no reminder is sent |

**Download, verify and clean up**

| You say | What happens |
|---|---|
| “Download the signed copy of the Acme agreement.” | Checks that certification has finished, then downloads the certified PDF |
| “Show me the first page of the contract.” | Downloads a page preview |
| “Is this signature valid? Hash 3f2a…” | Checks the public signature record; works without signing in |
| “Delete the draft Proposal v1.” | Deletes it if Assinafy still allows deletion in its current state |

Your assistant cannot sign, accept terms or enter verification codes for
someone else, and cannot create or edit templates. Those stay in Assinafy.

## Conversational behavior

Start from what the user wants to accomplish. Resolve names and reuse IDs from
previous results; do not ask the user to know API identifiers. Ask one concise
question when a recipient, template role, or document match is ambiguous. An
explicit instruction to send, remind, or delete already authorizes that action;
do not repeatedly ask for the same confirmation.

| User intent | Tool flow |
|---|---|
| “Send this PDF to Ana.” | `assinafy_request_signatures` with `action: "from_pdf"` after resolving the authorized contact |
| “Use our NDA template for this customer.” | Find the template, match roles and fields, validate values, then request signatures with `action: "from_template"` |
| “Has Ana signed the agreement?” | Find the document, then inspect it with `action: "get"`; report the pending people and current state |
| “Remind Ana.” | Inspect the current assignment, then `assinafy_follow_up_assignment` with `action: "resend"` for that signer |
| “Get the signed copy.” | Inspect certification and artifact availability, then download with `action: "artifact"` and `artifact: "certificated"` |

The model selects the tool and action; the conversation should report the result
and useful next step in ordinary language. Keep raw IDs and base64 out of normal
chat replies. Reads never send invitations. Preparation never sends invitations.

## Document flow

The server provides 11 tools covering 24 document operations. Your assistant
chooses tools and actions from the catalog; you describe what you want to do.
See the [tool reference](docs/tools.md) for inputs and examples.

```mermaid
flowchart TD
    A[Connect and authorize one workspace] --> B{Document source}
    B --> C[Find an existing document]
    B --> D[Upload a PDF or use a template]
    D --> E[Prepare and request signatures]
    C --> F[Read status, assignment, signers and activities]
    E --> F
    F --> G{Current state}
    G -->|Pending| H[Check delivery, remind or update expiration]
    H --> F
    G -->|Certificating| F
    G -->|Certificated| I[Download and verify]
    G -->|Failed or rejected| J[Inspect and choose recovery]
```

### 1. Find the document and check its current state

Use `assinafy_find_documents` (`action: "list"`) with `search`, `status`, `page`, and `per_page`.
It also supports `sort`, which accepts `name` or `updated_at`, optionally prefixed
by `-` to reverse the order.
Pages start at 1 and contain at most 100 records; follow returned pagination `meta`
instead of assuming the first page contains every document.

For example, call `assinafy_find_documents` (`action: "list"`) with:

```json
{"action":"list","status":"pending_signature","sort":"-updated_at","page":1,"per_page":25}
```

Once the correct document is identified, call `assinafy_find_documents` (`action: "get"`):

```json
{"action":"get","document_id":"DOCUMENT_ID"}
```

Use IDs returned by Assinafy in every later call. Do not invent IDs or infer the
assignment ID from the document ID. Document details expose the current `status`,
`is_closed`, `assignment`, signer records, page IDs, artifacts, and any decline
reason supplied by Assinafy. The documented lifecycle is:

| State | Next step |
|---|---|
| `uploading`, `uploaded`, `metadata_processing` | Processing is incomplete. Recheck later; collect assignments need prepared page metadata. |
| `metadata_ready` | Inspect or rename the PDF, then prepare its signature request. |
| `pending_signature` | Inspect pending signers, signing order, delivery history, and expiration. |
| `certificating` | All signatures may be present while final artifacts are still being generated. |
| `certificated` | Download the available signed artifacts. This state is not deletable under the documented API. |
| `expired` | Inspect the assignment and decide whether to update its expiration. Assinafy validates whether the update is allowed. |
| `rejected_by_signer`, `rejected_by_user` | Read the decline details and choose the next action with the user. A reminder does not reverse a rejection. |
| `failed` | Read document activities and the error before choosing recovery. |

Status checks are individual reads, not subscriptions. Use bounded polling when
asked to monitor a document; stop at a terminal state or the agreed timeout.

### 2. Inspect signing progress and delivery

Use `assinafy_find_documents` (`action: "get"`) to identify the actual pending people. The assignment
provides `id`, `signers[].id`, `completed`, `step`, `notified`,
`notification_history`, and signing URLs where available. Missing optional fields
mean the API has not supplied that information. A signer waiting for an earlier
step is not necessarily suffering a delivery failure.

Call `assinafy_find_documents` (`action: "activities"`) with `document_id` for the event timeline,
including each event's timestamp and payload. Inspect `notification_history` for
sent/failed events and error details. For WhatsApp delivery, call
`assinafy_find_documents` (`action: "notifications"`) with `document_id` and `assignment_id`.
Treat returned access links and codes as sensitive and share them only with their intended authorized recipients.

**100% signing progress does not prove the certificated PDF is ready.** Check the
document's lifecycle state and artifact availability before downloading it.

### 3. Resend a reminder

Read the document again, identify the intended pending signer and active signing
step, and make sure the user's instruction authorizes that reminder. OAuth scopes
authorize access to an operation; they do not select a recipient for the user.
A reminder spends notification credits. An explicit request to remind that
recipient is sufficient authorization; ask only if the intended recipient or action
is unclear. Call `assinafy_follow_up_assignment` (`action: "resend"`) with:

```json
{
  "action": "resend",
  "document_id": "DOCUMENT_ID",
  "assignment_id": "ASSIGNMENT_ID",
  "signer_id": "SIGNER_ID"
}
```

`is_sent: true` reports the resend result; it does not mean the recipient signed.
Read delivery history and signing progress afterward. Do not repeatedly resend
because progress is unchanged. An uncertain response may follow a successful
send; inspect activities before trying again. Respect rate limits and retry timing.

### 4. Update expiration or correct recipient information

Use `assinafy_follow_up_assignment` (`action: "set_expiration"`) with the IDs from current document
details and an explicit RFC 3339 timestamp:

```json
{
  "action": "set_expiration",
  "document_id": "DOCUMENT_ID",
  "assignment_id": "ASSIGNMENT_ID",
  "expires_at": "2027-01-31T23:59:59-03:00"
}
```

Choose the actual date and timezone requested by the user; the example is only a
format illustration. Read the updated assignment to confirm the resulting date.
Do not assume that updating expiration also sent a new invitation.

Find contacts through `assinafy_find_signers` (`action: "list"`) or `assinafy_find_signers` (`action: "get"`).
`assinafy_save_signer` (`action: "update"`) supports `full_name`, `email`, `whatsapp_phone_number`, and
`government_id`. A contact is shared within the workspace, so review the intended
change before applying it. Assinafy blocks changes to a channel already verified
on an in-flight document. Changing an unverified channel invalidates old access
links/codes; after a successful correction, use an authorized resend to deliver
the replacement link. Preserve and report an upstream refusal.

### 5. Prepare and send a new PDF

For the common email flow, call `assinafy_request_signatures` (`action: "from_pdf"`):

```json
{
  "action": "from_pdf",
  "file_name": "agreement.pdf",
  "file_base64": "BASE64_PDF_BYTES",
  "signers": [
    {
      "full_name": "Example Signer",
      "email": "signer@example.invalid"
    }
  ],
  "message": "Please review and sign",
  "max_wait_secs": 30
}
```

The address is fictional. Supply only recipients authorized for the actual task.
The tool uploads the PDF, waits for processing, creates or reuses email contacts,
and starts an Email-verified assignment with Email invitations. Save
`document.id`, `assignment.id`, and `signer_ids` from the result.

For preparation before sending, use this sequence:

1. `assinafy_prepare_document` (`action: "upload"`) accepts `file_name` and `file_base64`, returns a
   document, and sends no invitations. PDFs may contain up to 25 MiB and 2,000 pages;
   Assinafy enforces the page limit. The MCP server never opens a local file path.
2. `assinafy_find_documents` (`action: "get"`) checks readiness and supplies page IDs and dimensions.
   Use `assinafy_prepare_document` (`action: "rename"`) with `document_id` and `name` while renaming is
   allowed: before an assignment exists, in `uploaded` or `metadata_ready`.
3. `assinafy_find_signers` (`action: "list"`) / `assinafy_save_signer` (`action: "create"`) supply the signer IDs. Creating
   a contact alone sends no signing request. The standalone create operation can
   return a conflict; look up the existing contact before retrying.
4. `assinafy_request_signatures` (`action: "from_document"`) starts signing for the existing document using
   signer IDs. It spends the workspace's document allowance and, for WhatsApp,
   notification credits — use recipients and actions the user has authorized. Keep the returned assignment
   and signer identities for follow-up.

Example `assinafy_request_signatures` (`action: "from_document"`) arguments:

```json
{
  "action": "from_document",
  "document_id": "DOCUMENT_ID",
  "method": "virtual",
  "signers": [
    {
      "id": "SIGNER_ID",
      "verification_method": "Email",
      "notification_methods": [
        "Email"
      ],
      "step": 1
    }
  ],
  "message": "Please review and sign"
}
```

The assignment tool supports `virtual` and `collect`, optional `copy_receivers`
(signer IDs), `message`, `expires_at`, and ordered signing through `step`.
Verification methods are `Email`, `Whatsapp`, and `DigitalCertificate`;
notifications use the documented Email/WhatsApp channels. WhatsApp needs the
appropriate contact and subscription; certificate signing needs the signer's
required identity information and enabled account feature. Assinafy validates
these requirements. No response carries a cost or a balance: creating a document
and sending a WhatsApp notification draw on the workspace's document allowance and
notification credits at the published rates, and the remaining allowance is visible
in Assinafy rather than through this server. Use the user's existing authorization for the requested action; ask only for missing
recipient or document choices, and never repeat a call because progress looks unchanged.

For ordered signing, every signer must specify a step if any does; steps must be
contiguous from 1. People in one step sign in parallel. A digital-certificate
signer must be alone in their step.

For `collect`, wait for `metadata_ready`. Use `assinafy_check_fields` (`action: "list"`) (optionally
`include_standard: true`) and `assinafy_check_fields` (`action: "get"`) for field definitions. Pass
`entries[]` containing `page_id` and `fields[]`; each placement contains
`signer_id`, `field_id`, and `display_settings` with `left`, `top`, `width`,
`height`, and `fontSize`. Coordinates use the page image's 150-DPI pixels.
The server rejects placements outside the selected page before creating an
assignment. Page previews are available through `assinafy_download_document` (`action: "page"`).

#### Validate field values

Use `assinafy_check_fields` (`action: "list"`) and `assinafy_check_fields` (`action: "get"`) to inspect definitions,
then validate proposed values with `assinafy_check_fields` (`action: "validate"`):

```json
{
  "action": "validate",
  "values": [
    {
      "field_id": "CUSTOMER_NAME_FIELD_ID",
      "value": "Sample Company"
    },
    {
      "field_id": "CONTRACT_TOTAL_FIELD_ID",
      "value": "1250.00"
    }
  ]
}
```

The MCP converts `values` to the API's JSON array body. Validation does not create
a document or send invitations. Inspect every result's `success` and
`error_message`; a successful HTTP request can report an invalid value. Keep
identifiers with leading zeros as strings. Template `editor_fields[].value`
always takes a string, even when the value represents a number or date.

### 6. Generate a document from a template

Use `assinafy_find_templates` (`action: "list"`) and read its roles, pages, and editor fields.
`assinafy_find_templates` (`action: "get"`) supplies a direct lookup where the environment supports
that compatibility route. The list response is the documented source when the
individual route is unavailable.

For a request such as “fill `customer_name` and `contract_total` in the Service
contract template”, resolve the names before creating anything:

1. Select the requested template from `assinafy_find_templates` (`action: "list"`), following
   pagination, and inspect its processing status, `roles`, and `pages[].fields`.
2. Match the requested names to placement `label` values or to field definitions
   returned by `assinafy_check_fields` (`action: "list"`) / `assinafy_check_fields` (`action: "get"`). Join a definition's
   `id` to the placement's `field_id`, and check the placement's `role_id` against
   the template's editor role. Fill only fields already configured on that template.
3. Use the placement's **`field_id`**, not its placement `id` or its label, in
   `editor_fields`. Field names are configurable; they are not tool argument names.
   The client performs this mapping from metadata; the server accepts IDs.
4. If names are missing or ambiguous, ask the user to identify the intended field.
   Do not guess. Repeated placements sharing one `field_id` have one supplied
   value; the API does not offer per-placement values in this request.
5. Validate the proposed values and resolve one signer per template role before the
   authorized creation call.

Template creation and page-layout/role editing are documented only in the internal
API and are not exposed by this server. Configure the saved template in Assinafy
before using the public template document workflow.

`assinafy_request_signatures` (`action: "from_template"`) fills every role. Each entry gives either
an existing signer `id` or a `full_name` and `email` that the server creates or
reuses — one or the other, never both in the same entry, so a template can mix
directory signers and new contacts in a single call. Entries also take the
documented `verification_method`, `notification_methods`, and `step`. An entry
resolved from an email defaults to Email verification and notification; naming a
notification method instead lets Assinafy infer the matching verification method.
Template entries accept at most one notification channel: `Email` or `Whatsapp`.
The server validates every entry before creating contacts. Role eligibility and
signing order are still checked by Assinafy, so a later failure can leave newly
created contacts behind.

The call accepts a custom name, message, expiration, `tags` (tag **names**, merged
with the template defaults), and `editor_fields` containing `{field_id,value}`.
Creation can send invitations and spends the workspace's document allowance, so
use the user-authorized recipients and document. Save the returned document ID, then call
`assinafy_find_documents` (`action: "get"`) for the assignment and subsequent status checks.

The creation arguments look like this (substitute IDs discovered from the selected
template and signer directory):

```json
{
  "action": "from_template",
  "template_id": "TEMPLATE_ID",
  "name": "Service agreement.pdf",
  "signers": [
    {
      "role_id": "EDITOR_ROLE_ID",
      "id": "PREPARER_SIGNER_ID",
      "step": 1
    },
    {
      "role_id": "CUSTOMER_ROLE_ID",
      "full_name": "Sample Customer",
      "email": "customer@example.com",
      "step": 2
    }
  ],
  "editor_fields": [
    {
      "field_id": "CUSTOMER_NAME_FIELD_ID",
      "value": "Sample Company"
    },
    {
      "field_id": "CONTRACT_TOTAL_FIELD_ID",
      "value": "1250.00"
    }
  ]
}
```

Subsequent status checks, reminders, expiration changes, and downloads use the
returned document's IDs just as they do for uploaded PDFs.

### 7. Inspect page previews

`assinafy_download_document` (`action: "page"`) takes `document_id` and a returned `page_id`.
It returns image bytes as base64. Download only the pages needed for the task.

### 8. Download signed artifacts and verify

Once certification finishes, call `assinafy_download_document` (`action: "artifact"`) with the document
ID and `artifact: "certificated"`. Its result contains `document_id`, the artifact
name, and `base64`. Decode and save the bytes through the client's supported file
tools; do not print a large base64 payload in the conversation.

The tool also accepts `original`, `certificated`,
`certificate-page`, `pades`, or `bundle`. Artifact availability depends on the
lifecycle and signing method; an unavailable artifact is not a successful download.
Binary responses are limited to 64 MiB before base64 encoding.

Call `assinafy_verify_document` with the signed document's Assinafy SHA-1 signature
hash. Use the actual verification hash supplied with the signed document, not the
document ID or an invented value. This is the one tool that works without
connecting: it checks a public signature record and touches nothing belonging to
a workspace. Report the returned `is_valid` result.

People complete signatures, signer-side rejection, and required verification in
Assinafy. Those endpoints use a signer's access code, outside the workspace OAuth2
grant. These tools do not accept terms, enter verification codes, or sign on
someone else's behalf.

### 9. Recover a partial operation or delete

A multi-step send can fail after the upload succeeds. Its error preserves the
created document ID. Inspect that document and its activities before continuing:

- An existing assignment means the signature request may already have succeeded.
  Continue tracking it; do not create another assignment or resend blindly.
- A prepared document without an assignment can continue through
  `assinafy_request_signatures` (`action: "from_document"`) using confirmed signer IDs. It does not need another
  upload. Check processing status before placing fields.
- If cleanup is requested, use `assinafy_delete_document` only when the current
  document state permits it. This action needs authorization from the user.

No mutating API request is automatically retried by the MCP server.

## Errors and reconnection

| Result | Client action |
|---|---|
| HTTP 401 | Let the client refresh its grant or reconnect through Assinafy consent. |
| HTTP 403 with `insufficient_scope` | Authorize the requested scope set and retry after consent. |
| Tool error naming a missing scope | Assinafy refused the call. A multi-step tool may have completed earlier steps; read its error for a retained `document_id` and continue from there. |
| Tool result with `isError: true` | Read the Assinafy error and inspect the current document before retrying a mutation. |
| Invalid contact, expired assignment, or unavailable artifact | Correct the request or wait for the required lifecycle state. |
| Rate limit or uncertain network response | Respect retry timing; inspect state and delivery history before repeating a write. |
| HTTP 429 | Assinafy is rate-limiting authorization checks; honour `Retry-After`. |
| HTTP 503 during authentication | The service is temporarily unavailable. Retry later; contact Assinafy support if it persists. |

See [error recovery](docs/errors.md) for full response semantics and
[tool inputs](docs/tools.md) for field reference.

## Check the connection

Refresh the tool list in your client after reconnecting or updating it. You can
also inspect the available catalog without signing in:

```bash
curl -sS -X POST https://mcp.assinafy.com.br/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

If login reports `invalid_target`, `invalid_client`, or missing resource metadata,
confirm that the server URL is exactly `https://mcp.assinafy.com.br/mcp` and that
your client is up to date. If the error persists, contact Assinafy support with
the client name, version, and error message. Do not include tokens or credentials.
