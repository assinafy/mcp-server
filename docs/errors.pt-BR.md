# Tratamento de erros

*Português (Brasil) · [English (US)](errors.md)*

Falhas de execução retornam `isError: true` e uma descrição em texto. Resultados
JSON bem-sucedidos também usam `structuredContent`. Falhas de autorização antes
da execução usam códigos HTTP. Conectar, listar ferramentas e verificar uma
assinatura publicamente não exigem acesso à workspace.

## Autenticação e transporte

Se o token venceu, tente renovar a autorização existente. Solicite novo
consentimento apenas quando a renovação falhar ou forem necessárias permissões
adicionais. Em `invalid_target`, confira a URL
`https://mcp.assinafy.com.br/mcp` e procure a Assinafy se o erro persistir.
Veja [autenticação](authentication.pt-BR.md).

| Status HTTP | Significado e recuperação |
|---|---|
| 400 | Requisição inválida, tentativa de substituir credenciais, Authorization duplicado, token na URL, chave JSON repetida, ferramenta desconhecida ou lote acima de 20 mensagens |
| 401 | Token ausente, inválido ou vencido; também pode indicar requisição anônima acima de 64 KiB. Renove a autorização ou siga o challenge `resource_metadata` se não houver autorização utilizável |
| 403 com `insufficient_scope` | Aprove os escopos indicados em `WWW-Authenticate`, preservando os já concedidos. A operação ainda não começou |
| Outro 403 | Origem, host ou acesso não permitido; confira a configuração do cliente e a requisição |
| 405 em GET de event-stream | Use POST com Streamable HTTP; não há stream SSE independente |
| 415 | O POST em `/mcp` exige `Content-Type: application/json` |
| 413 | Requisição autenticada acima de 40 MiB; requisições anônimas têm limite de 64 KiB |
| 429 | Limite de requisições; respeite `Retry-After` |
| 503 | Serviço de autorização temporariamente indisponível ou incapaz de fornecer uma permissão já concedida. Use espera progressiva; novo consentimento não corrige uma falha do serviço |

Não coloque credenciais ou outra workspace nos argumentos ou em `_meta`.
Permissões insuficientes são verificadas antes da execução, mas revogações ou
mudanças de acesso durante um fluxo ainda podem produzir uma resposta HTTP `200`
com `isError: true`. Confira o resultado MCP, não apenas o status HTTP.

## Events

`events/subscribe` e `events/unsubscribe` retornam erros JSON-RPC:

| Código | Significado | Ação |
|---|---|---|
| `-32015` | Callback não verificado; `data.reason` pode ser `invalid_url`, `timeout`, `connection_failed` ou `challenge_failed` | Use HTTPS público e responda exatamente `{"challenge": …}`; depois tente inscrever novamente |
| `-32602` | Evento, documento, entrega, segredo ou cursor inválido; documento inacessível ou limite de inscrições atingido | Corrija a requisição; repeti-la sem alterações não resolve |
| `-32603` | Serviço temporariamente indisponível ou inscrição alterada/cancelada simultaneamente | Use espera progressiva e renove antes de `refreshBefore` |

Uma entrega com falha permanente ou cinco tentativas esgotadas é abandonada; o
monitoramento continua. Veja [Events](events.pt-BR.md).

## Erros da API

Os erros preservam status, mensagem, código textual e erros por campo quando
fornecidos. `ContractChanged`, `PlanRestricted` e `CreditCeilingExceeded` continuam
identificáveis mesmo que a mensagem mude. Uma falha dentro do envelope da resposta
da API também deve ser tratada como erro.

Exemplos de mensagens que o cliente pode receber:

```text
API error 404: Página não encontrada.
API error 403: Permissão insuficiente. (Bearer error="insufficient_scope", scope="documents:write")
Document ID is required
digital_certificate_requires_virtual: DigitalCertificate signers require method virtual
validating "arguments": ...
```

## Novas tentativas

Leituras seguras (`GET`, `HEAD`, `OPTIONS`) podem receber até três novas tentativas
automáticas para `429` e `5xx` transitórios, com espera progressiva. Escritas
(`POST`, `PUT`, `PATCH`, `DELETE`) nunca são repetidas automaticamente.
`Retry-After` aceita segundos ou data HTTP; `X-Rate-Limit-Reset` em segundos também
é reconhecido. Se a espera exigida ultrapassar o orçamento automático de oito
segundos, o erro retorna imediatamente para que o cliente aguarde antes de tentar.

`404` e `422` exigem correção ou conferência dos parâmetros. Um conflito ao criar
um contato durante o envio pode ser resolvido por busca do email exato. Isso não
autoriza repetir uma solicitação de assinatura com resposta incerta.

Uma falha de `assinafy_send_pdf_for_signature` depois do upload informa o ID do
documento preservado. Inspecione-o antes de continuar para evitar documentos ou
notificações duplicados. Se o token vencer dentro de `max_wait_secs` mais dois
minutos, nada é enviado; reduza a espera ou renove o token primeiro.

Se ainda não houver assignment e o estado permitir, continue com
`assinafy_create_assignment` e os contatos confirmados. Acompanhe assignments
existentes em vez de recriá-los. Leituras não enviam notificações. Corrigir um
canal de contato não verificado pode invalidar links anteriores; envie um lembrete
somente após a correção bem-sucedida e com autorização. Canais já verificados em
um documento em andamento não podem ser alterados.

## Conflitos de contratos

Um `403` pode indicar que o plano não inclui gestão de contratos; novo
consentimento não altera o plano. Em `409 ContractChanged`, leia os termos e
atividades atuais, revise a alteração concorrente e só então envie uma atualização
autorizada com o novo `updated_at`. Nunca remova a condição para forçar a escrita.
Uma renovação repetida pode retornar `400` porque o `term` original não é mais atual.

Lembretes de ciclo de vida seguem as preferências da conta e dependem do plano.
`contract.activity` observa atividades enquanto a conexão e a inscrição estão
ativas. Não envia outro lembrete, não garante entrega após esgotar tentativas e
não autoriza uma alteração automática do contrato.
