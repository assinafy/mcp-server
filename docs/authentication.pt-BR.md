# Autenticação

*Português (Brasil) · [English (US)](authentication.md)*

Conecte-se a `https://mcp.assinafy.com.br/mcp`, entre na Assinafy, escolha uma
workspace e aprove as permissões. Para usar o serviço hospedado, não é necessário
informar API key, client secret ou criar uma aplicação OAuth. Veja as
[instruções de conexão](../README.md#conectar).

## Conexão e workspace

O serviço oferece OAuth 2.1, autorização no navegador com PKCE S256 e Client ID
Metadata Documents (CIMD). Clientes compatíveis descobrem as configurações pelo
endpoint MCP. Guarde os tokens no armazenamento seguro do cliente; nunca os
inclua em prompts, argumentos de ferramentas, URLs ou metadados de requisição.

Cada conexão seleciona uma workspace. O argumento `account_id` pode repetir a
workspace conectada, mas não pode substituí-la. Crie uma conexão separada para
cada workspace adicional.

A descoberta e o catálogo de ferramentas são públicos. Operações da workspace
exigem OAuth. `assinafy_verify_document` é uma consulta pública e dispensa conexão.

## Permissões

| Escopo | Finalidade |
|---|---|
| `account:read` | Identificar a workspace conectada |
| `documents:read` | Ler documentos, contatos, campos, status, artefatos e atividades |
| `documents:write` | Enviar arquivos, preparar assinaturas, validar campos, enviar solicitações e lembretes, atualizar contatos e contratos |
| `templates:read` | Ler templates e papéis |
| `templates:write` | Criar documentos a partir de templates |

O cliente pode solicitar todos os escopos ou um subconjunto para um fluxo
limitado. Algumas operações exigem vários escopos, incluindo leitura de documentos
ou templates. Em `403 insufficient_scope`, solicite os escopos indicados no
challenge e preserve os já concedidos. Restrição de plano não é falta de escopo.

## Renovação e recuperação

O cliente guarda e renova os tokens OAuth. Renove a autorização existente antes
do vencimento do token de acesso; a renovação normal não exige novo consentimento
no navegador. Reconecte se a autorização for revogada, a renovação deixar de ser
válida ou for necessário selecionar outra workspace. Escopos adicionais podem
exigir consentimento.

Events exige `account:read` e `documents:read`. Renove o token e chame
`events/subscribe` novamente antes de `refreshBefore`. Renovar apenas o token não
renova a inscrição. A mesma autorização mantém a identidade da inscrição; um
novo consentimento cria outra autorização.

Use espera progressiva e respeite `Retry-After` em limites de uso ou falhas
transitórias. Um `503` não significa, por si só, que é necessário autorizar
novamente. Confira os IDs de documentos preservados antes de repetir um fluxo
parcial de assinatura. Veja [Events](events.pt-BR.md) e
[recuperação de erros](errors.pt-BR.md).
