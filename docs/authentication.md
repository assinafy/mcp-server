# Authentication

*[Português (Brasil)](authentication.pt-BR.md) · English (US)*

Connect to `https://mcp.assinafy.com.br/mcp`, sign in to Assinafy, choose a
workspace, and approve the requested permissions. You do not need an API key,
client secret, or your own OAuth application to use the hosted server.
See the [connection instructions](../README.en.md#connect).

## Connection and workspace

The server supports OAuth 2.1, browser authorization with PKCE S256, and Client ID
Metadata Documents (CIMD). Compatible clients discover the authorization settings
from the MCP endpoint. Keep tokens in the client's secure credential storage;
never put them in a prompt, tool arguments, URLs, or request metadata.

One connection selects one workspace. Tool arguments cannot switch it: an
`account_id` argument may repeat the connected workspace but cannot override it.
Create a separate connection for each additional workspace.

Discovery and the tool catalog are public. Workspace operations require OAuth.
`assinafy_verify_document` is a public lookup and does not require a connection.

## Permissions

| Scope | Purpose |
|---|---|
| `account:read` | Resolve the connected workspace |
| `documents:read` | Read documents, contacts, fields, status, artifacts, and activities |
| `documents:write` | Upload, prepare, validate fields, send, remind, update contacts, and change contracts |
| `templates:read` | Read templates and roles |
| `templates:write` | Create documents from templates |

Clients can request the full set for the complete tool catalog or a smaller set
for a limited workflow. Some operations require several scopes; for example,
preparing a signing request may also need document or template read access.
On `403 insufficient_scope`, request the permissions named in the challenge,
retaining the scopes already granted. A plan restriction is not a missing scope.

## Refresh and recovery

The client stores and refreshes its OAuth tokens. Refresh the existing grant
before the access token expires; normal token renewal does not require new
browser consent. Reconnect when the grant is revoked, refresh is no longer valid,
or a different workspace is needed. Additional scopes may require consent.

Events requires `account:read` and `documents:read`. Renew the OAuth token and call
`events/subscribe` again before the returned `refreshBefore`. A token refresh
alone does not renew an Events subscription. Refreshing the same grant preserves
subscription identity; new consent creates a different grant.

Use backoff and honor `Retry-After` for rate limits or temporary failures. A `503`
does not by itself mean the user must authorize again. Inspect retained document
IDs before retrying any partially completed signing workflow. See
[Events](events.md) and [error recovery](errors.md).
