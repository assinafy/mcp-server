# Referência de ferramentas

*Português (Brasil) · [English (US)](tools.md)*

A conexão oferece **29 ferramentas**: 27 de assinatura e duas de gestão de
contratos com ações específicas. Campos obrigatórios ausentes e argumentos
não reconhecidos são rejeitados antes da operação.

## Catálogo

| Ferramenta | Finalidade |
|---|---|
| `assinafy_list_documents` | Buscar documentos por nome, status ou página |
| `assinafy_get_document` | Ler status, páginas, artefatos, assignment e signatários |
| `assinafy_list_document_activities` | Consultar atividades do documento |
| `assinafy_list_whatsapp_notifications` | Consultar entregas de notificações por WhatsApp |
| `assinafy_upload_document` | Enviar um PDF sem solicitar assinaturas |
| `assinafy_rename_document` | Renomear antes de existir assignment |
| `assinafy_send_pdf_for_signature` | Enviar um PDF novo para assinatura por email |
| `assinafy_create_assignment` | Solicitar assinaturas em documento existente |
| `assinafy_create_document_from_template` | Gerar um documento de template e enviar convites |
| `assinafy_resend_notification` | Lembrar um signatário pendente |
| `assinafy_set_assignment_expiration` | Alterar ou remover o prazo de assinatura |
| `assinafy_list_signers` | Buscar contatos |
| `assinafy_get_signer` | Ler um contato |
| `assinafy_create_signer` | Criar um contato |
| `assinafy_update_signer` | Corrigir um contato |
| `assinafy_list_templates` | Buscar templates |
| `assinafy_get_template` | Ler um template |
| `assinafy_list_fields` | Listar definições de campos |
| `assinafy_get_field` | Ler uma definição de campo |
| `assinafy_validate_fields` | Validar valores sem alterar campos |
| `assinafy_download_document` | Baixar um artefato do documento |
| `assinafy_download_document_page` | Baixar a imagem de uma página |
| `assinafy_delete_document` | Excluir um documento em estado permitido |
| `assinafy_verify_document` | Verificar publicamente um código de assinatura |
| `assinafy_show_document` | Exibir status e ações de acompanhamento |
| `assinafy_review_signature_request` | Revisar destinatários antes de enviar |
| `assinafy_place_fields` | Posicionar campos visualmente |
| `assinafy_browse_contracts` | Listar, ler ou acompanhar a família de um contrato |
| `assinafy_update_contract` | Definir termos, vincular aditivos, renovar ou encerrar |

As anotações MCP informam se cada ferramenta é de leitura, destrutiva,
idempotente ou pode afetar destinatários externos. Elas constam nos
[schemas e exemplos completos](tool-payloads.json), junto com cada entrada,
resposta e operação. Os nomes dos campos e valores da API são os mesmos nos dois
idiomas. Os exemplos usam contatos sintéticos.

Uma solicitação explícita autoriza a operação indicada. Esclareça destinatários
ambíguos ou informações ausentes; não trate uma consulta como autorização para
enviar, cobrar ou excluir. A workspace vem da conexão OAuth: `account_id` pode
repeti-la, mas não substituí-la. Veja [autenticação](authentication.pt-BR.md) e o
[fluxo de assinatura](../README.md#fluxo-completo-de-assinatura).

Use `tools/call` com `{name,arguments}`. Respostas JSON aparecem em texto e em
`structuredContent`; downloads incluem base64 somente em `structuredContent`,
com um resumo curto em texto. Listas preservam `{data,meta?}`. Exclusões retornam
texto. Falhas usam `isError: true`, com status, mensagem e challenge quando fornecido.
Campos opcionais das respostas dependem do estado e das permissões.

## Custos

Use `estimate_only: true` em `assinafy_send_pdf_for_signature`,
`assinafy_create_assignment`, `assinafy_create_document_from_template` ou
`assinafy_resend_notification` para obter `cost_estimate` sem enviar convites ou
lembretes. Confira `has_sufficient_resources`, `blocking_reason`, `total_credits`,
saldos e detalhamento. A API recalcula os recursos no envio efetivo.

A estimativa de PDF envia e prepara o arquivo e retorna `{document,cost_estimate}`,
mas não cria contatos nem convites. Preserve o documento e continue com
`assinafy_create_assignment`; reenviar o PDF criaria uma duplicata. Na estimativa
de assignment, IDs de signatários são opcionais; `collect` ainda exige `entries`.
A estimativa de template exige `template_id` e os `role_id`, mas dispensa nome,
email ou ID de contato. A de lembrete exige os mesmos IDs do lembrete efetivo.

Nos envios de PDF, assignment ou template, `max_total_credits` limita o gasto.
Use o `total_credits` da estimativa; zero permite apenas envio sem créditos.
`409 CreditCeilingExceeded` impede cobrança acima do limite. Um template também
pode falhar depois, se o custo aumentar durante o processamento: inspecione o
documento preservado. O limite não é enviado aos endpoints de estimativa.

## Métodos de verificação

Escolha `Email`, `Whatsapp`, `DigitalCertificate` ou `IdDocumentFaceMatchBasic`.
Exatamente um canal de notificação é permitido. Email e Whatsapp usam seus canais
correspondentes; certificado e biometria aceitam qualquer um dos dois. Configurações
omitidas são inferidas pela Assinafy.

`DigitalCertificate` usa o certificado ICP-Brasil A1/A3 da pessoa e exige
`virtual`, com o signatário sozinho na etapa. `IdDocumentFaceMatchBasic` usa
fotos de documento e prova de vida, conforme plano e elegibilidade. Salve o
`government_id` autorizado no contato. O assistente não recebe chaves privadas,
não realiza prova de vida e não assina pela pessoa.

Se a biometria exigir revisão, o signatário apresenta
`identity_verification_status: InReview`. Após as assinaturas, o documento pode
aguardar em `pending_identity_review`. Uma pessoa autorizada da conta analisa as
evidências na Assinafy. Aprovar pode liberar a certificação; rejeitar cancela o
documento e envia avisos de cancelamento.

## Documentos e acompanhamento

| Ferramenta | Obrigatório | Opcional |
|---|---|---|
| `assinafy_list_documents` | Nada | `page`, `per_page`, `search`, `sort`, `status`, `method`, `tags`, `account_id` |
| `assinafy_get_document` | `document_id` | Nada |
| `assinafy_list_document_activities` | `document_id` | `sort` (`-id` ou `id`), `page`, `per_page` |
| `assinafy_list_whatsapp_notifications` | `document_id`, `assignment_id` | Nada |

A paginação começa em 1 e a API limita documentos e atividades a 50 itens por
página. `sort` de documentos aceita `name` ou `updated_at`, com `-` opcional para
ordem decrescente. Atividades vêm da mais recente por padrão. Percorra as páginas
para obter o histórico completo. Entregas por email aparecem nos registros dos
signatários do assignment; o histórico separado atende WhatsApp.

```json
{"search":"Contrato de serviços","status":"pending_signature"}
```

```json
{"document_id":"DOCUMENT_ID"}
```

Confira pessoas pendentes e ordem de assinatura antes de lembrar alguém. Uma
contagem de 100% assinado ainda pode estar em `certificating`: aguarde
`certificated` e a disponibilidade do artefato. Leituras não enviam convites.
Use verificações de status limitadas, evitando consultas contínuas sem intervalo.

## Preparar sem enviar

`assinafy_upload_document` exige `file_name` e `file_base64`; `account_id` é
opcional. O nome deve terminar em `.pdf`; o PDF decodificado tem limite de 25 MiB
e precisa conter um cabeçalho PDF no primeiro KiB. Nenhum contato ou convite é
criado. `assinafy_rename_document` exige `document_id` e `name` de 1–255 caracteres,
antes de existir assignment e em estado permitido. A API normaliza o nome.

Preserve o ID, confira o documento, resolva os contatos e depois use
`assinafy_create_assignment`. Preparação separada é opcional quando o pedido já
é enviar um PDF novo com destinatários definidos.

## Solicitar assinaturas

### PDF novo

`assinafy_send_pdf_for_signature` valida o PDF, envia uma vez, aguarda o
processamento, reutiliza ou cria contatos pelo email exato e cria um assignment
virtual com verificação e notificação por Email.

| Campo | Obrigatório | Uso |
|---|:---:|---|
| `file_name` | Sim | Nome terminado em `.pdf`; componentes de caminho são descartados |
| `file_base64` | Sim | Bytes do PDF em base64; limite decodificado de 25 MiB |
| `signers` | Sim | Lista não vazia de `{full_name,email}`; emails duplicados são rejeitados sem diferenciar maiúsculas |
| `message` | Não | Mensagem do convite |
| `expires_at` | Não | Data ou data-hora ISO 8601 |
| `max_total_credits` | Não | Limite não negativo de créditos |
| `account_id` | Não | Workspace já autorizada |
| `max_wait_secs` | Não | Espera de 1–600 segundos; padrão 30 |
| `poll_secs` | Não | Intervalo de 1 até `max_wait_secs`; padrão 2 |
| `estimate_only` | Não | Preparar e estimar sem criar contatos ou convites |

```json
{"document":{"id":"document-example"},"assignment":{"id":"assignment-example"},"signer_ids":["signer-example"]}
```

Se o token vencer dentro da espera mais dois minutos, nada é enviado. Uma falha
após o upload informa o ID preservado. A operação não é transacional: inspecione
o resultado antes de repetir ou decidir por uma exclusão autorizada.

### Documento existente

`assinafy_create_assignment` exige `document_id`, `method` (`virtual` ou
`collect`) e `signers` não vazio com `id` em cada entrada. Aceita `entries`,
`message`, `expires_at` ISO 8601, `copy_receivers` com IDs de contatos,
`max_total_credits` e `estimate_only`.

Se já houver assignment, acompanhe-o em vez de enviar novamente. `virtual` aceita
`uploaded`, `metadata_processing` ou `metadata_ready`. `collect` exige
`metadata_ready` e posicionamentos em `entries`. Use páginas e dimensões de
`assinafy_get_document` e campos de `assinafy_list_fields`.

Cada posicionamento inclui `signer_id`, `field_id` e `display_settings` com
`left`, `top`, `width`, `height`, `fontSize` e, opcionalmente, `fontFamily` e
`backgroundColor`. As coordenadas usam pixels da imagem de página em 150 DPI;
respeite seus limites. O envio collect também precisa de `documents:read`.

Os signatários aceitam `verification_method`, `notification_methods` e `step`.
Quando houver ordem, todos informam etapas contíguas desde 1. Pessoas na mesma
etapa assinam em paralelo, exceto certificado, cujo signatário fica sozinho.
`DigitalCertificate` com `collect` retorna `digital_certificate_requires_virtual`
antes da estimativa ou criação, sem enviar ou cobrar um convite inutilizável.

```json
{"document_id":"DOCUMENT_ID","method":"virtual","signers":[{"id":"SIGNER_ID"}]}
```

### Template salvo

`assinafy_create_document_from_template` exige `template_id` e `signers` com uma
entrada por papel Signer. CopyReceiver é opcional; Editor usa `editor_fields`.
Cada entrada informa `role_id` e um `id` existente **ou** `full_name` e `email`.
É possível misturar as duas formas e usar o mesmo email em papéis diferentes.

Aceita `name`, `message`, `expires_at` ISO 8601, `max_total_credits`, `tags` por
nome, `editor_fields` com `{field_id,value}`, `account_id` e `estimate_only`.
Signatários existentes aceitam `verification_method`, `notification_methods` e
`step`. Entradas resolvidas por email usam Email; para WhatsApp, certificado ou
biometria, prepare o contato e forneça seu `id`.

Todos os papéis Signer obrigatórios são verificados antes de criar contatos.
Se houver ordem, todos os papéis Signer informam etapas contíguas desde 1;
CopyReceiver ignora ordem, verificação e notificações. Um certificado fica
sozinho na etapa. Contatos criados podem permanecer se uma etapa posterior falhar.

Templates com campos posicionados para signatários usam `collect` e não aceitam
`DigitalCertificate`. Campos apenas de Editor permitem certificado virtual.
Leia papéis e posições, associe nomes a IDs e esclareça correspondências ambíguas.
`editor_fields` usa o `field_id`, não o ID da posição nem seu rótulo. Seus valores
são strings, mesmo para números e datas; valide-os antes de criar.

O retorno inicial pode estar em `uploaded` com `assignment: null` enquanto o
processamento continua. Consulte `assinafy_get_document`; não crie um segundo
assignment nesse intervalo. A criação e edição do template ficam na Assinafy.

## Lembretes e prazo

| Ferramenta | Obrigatório | Resultado |
|---|---|---|
| `assinafy_resend_notification` | `document_id`, `assignment_id`, `signer_id` | Resultado da notificação; `is_sent` não significa assinado |
| `assinafy_set_assignment_expiration` | `document_id`, `assignment_id`, `expires_at` ISO 8601 ou `null` | Assignment atualizado |

Confira pendências antes do lembrete autorizado. Cada reenvio pode consumir
créditos. Uma resposta incerta exige consulta antes de tentar novamente. Data
sem horário vence às 23:59 UTC; data-hora sem offset é lida como UTC. `null` remove
o prazo. Alterar o prazo não envia um lembrete.

## Contatos

`assinafy_list_signers` aceita `account_id`, `search`, `page` e `per_page`.
`assinafy_get_signer` exige `signer_id` e aceita `account_id`.
`assinafy_create_signer` exige `full_name` e aceita `email`,
`whatsapp_phone_number`, `government_id` e `account_id`.
`assinafy_update_signer` exige `signer_id` e pelo menos um campo de contato;
`account_id` é opcional.

Contatos são compartilhados na workspace. Criar ou editar um contato não solicita
assinatura. Em conflito, consulte o contato existente antes de repetir. `null`
ou string vazia limpa email, WhatsApp ou documento fiscal; `full_name` não pode
ser limpo. Canais já verificados em documento em andamento têm edição restrita.
Corrigir um canal não verificado invalida links/códigos anteriores: só envie um
lembrete autorizado depois da correção. CPF/CNPJ é sensível; salve-o apenas com
autorização para o método de verificação necessário.

## Templates e campos

`assinafy_list_templates` aceita `page`, `per_page`, `search` e `account_id`.
`assinafy_get_template` exige `template_id` e aceita `account_id`. A lista retorna
`{data:[...]}` e `meta` opcional. Confira papéis, tags, páginas e posições de Editor.
A leitura individual usa uma rota de compatibilidade que pode não estar disponível
em todo ambiente; a listagem é a descoberta documentada. Consultar não gera documento.

`assinafy_list_fields` aceita `account_id`, `include_inactive` e `include_standard`.
`assinafy_get_field` exige `field_id` e aceita `account_id`.
`assinafy_validate_fields` exige `values`, uma lista de `{field_id,value}`, e aceita
`account_id`. São consultas, inclusive a validação. Confira `success` e
`error_message` de cada valor. Tipos JSON, null explícito e números exatos são
preservados; identificadores com zeros à esquerda devem ser strings.
Essas operações não criam campos nem posições de template.

## Downloads, exclusão e verificação

`assinafy_download_document` exige `document_id` e aceita `artifact`: `original`
(padrão), `certificated`, `certificate-page`, `pades` ou `bundle`. Retorna
`{document_id,artifact,base64}`. `assinafy_download_document_page` exige
`document_id` e `page_id` e retorna `{document_id,page_id,base64}`.
Confirme certificação e disponibilidade antes de baixar artefatos assinados.
O limite é de 64 MiB antes do base64; salve os bytes sem imprimi-los na conversa.

`assinafy_delete_document` exige `document_id` e retorna uma mensagem curta.
A exclusão precisa estar autorizada e em estado permitido. Não a use como limpeza
automática após envio incerto.

`assinafy_verify_document` exige `hash`: passe sem alteração o `verification_code`
de `assinafy_get_document` após certificação, ou o hash impresso na página de
certificado. Não calcule um hash do PDF. É uma consulta pública, sem credenciais,
e pode retornar `is_valid`.

## Cartões visuais

Clientes compatíveis com [MCP Apps](https://modelcontextprotocol.io/extensions/apps)
exibem cartões interativos. Os demais recebem o mesmo resultado em texto.
Abrir um cartão é somente leitura; ações posteriores usam as ferramentas normais
e a mesma autorização OAuth.

- `assinafy_show_document`: recebe `document_id` e mostra status, assinaturas,
  pendências e prazo, com ações de atualizar e lembrar.
- `assinafy_review_signature_request`: recebe os argumentos de envio de
  `assinafy_create_assignment`, sem `estimate_only`, e mostra destinatários,
  verificação e prazo antes de enviar.
- `assinafy_place_fields`: recebe os argumentos de assignment sem `method`,
  `entries` e `estimate_only`. Permite posicionar campos e enviar com `collect`.
  Em texto, retorna IDs e dimensões de páginas e campos para montar `entries`.

O envio verifica novamente se já existe assignment. O editor exige
`metadata_ready`, ausência de assignment e posições dentro da página. Cartões
não recebem links de assinatura.

## Contratos

`assinafy_browse_contracts` usa `action: list`, `get` ou `related`, sempre em
leitura. `list` aceita `search`, `status` separado por vírgula, `counterparty`,
intervalos de término, `ending_within_days` entre 0 e 3650 em dias de Brasília,
referências externas, ordenação e paginação. `expand: document` inclui nomes.
`get` lê o contrato/aditivo indicado por `document_id`; `related` lê a família.
Filtros de lista não são aceitos nessas duas ações. O resultado usa
`{view:"contracts",request,result}` em texto e no cartão.

`assinafy_update_contract` exige `documents:write` e plano com gestão de contratos:

- `set`: envie `document_id` e `terms` não vazio. Apenas campos fornecidos mudam.
  Para termos ou vínculos existentes, inclua `terms.updated_at` da leitura atual;
  omita na primeira criação. Null segue as regras de limpeza da API.
  `external_references` substitui a lista completa. `notice_period_days` configura
  o aviso; `auto_renew` autoriza renovação automática. `owner_id` identifica um
  membro da workspace, não um contato signatário.
- `renew`: envie `term` e `updated_at` atuais, mesmo com `end_date` explícito;
  `end_date` e `value` são opcionais. Renova contratos vencidos, mas recusa
  encerrados. A mesma requisição não pode renovar duas vezes.
- `terminate`: envie `updated_at` atual e `reason` opcional. Encerra o
  acompanhamento e os avisos, sem cancelar o assignment de assinatura.

Leia primeiro e altere apenas o autorizado. Em `409`, confira a mudança concorrente
antes de uma nova escrita; não substitua a condição silenciosamente. Em timeout,
consulte termos e atividades antes de repetir.

Para um instrumento assinado, envie um documento separado e use `set` com
`terms.parent_document_id`, `relation_type` (`amendment`, `renewal`, `termination`)
e as alterações permitidas. A aplicação ocorre após certificação, na ordem de
assinatura. Confira `apply_error` e `status` se continuar pendente. Null remove
uma alteração de aditivo pendente; `parent_document_id: null` desfaz o vínculo.
Use o `updated_at` do próprio documento, não o do contrato principal.

Prazos contratuais são independentes dos prazos de assinatura. A Assinafy gerencia
avisos, alertas de 45/30 dias e renovação no dia seguinte ao término, em horário de
Brasília, conforme plano e preferências. Events apenas informa novas atividades:
não autoriza renovar, encerrar ou enviar emails adicionais.

```json
{"action":"renew","document_id":"document-example","term":1,"updated_at":"2030-01-01T12:00:00Z"}
```

## Webhooks da conta

Configure-os na Assinafy: até três endpoints em planos pagos e um nos demais.
Guarde o segredo único ao habilitar assinatura. OAuth não o consulta nem o
rotaciona depois; se perdido, é necessário excluir/recriar com autorização.
O catálogo MCP não administra esses endpoints.

Valide Standard Webhooks sobre o corpo original: decodifique a chave após
`whsec_`, calcule HMAC-SHA256 de `id.timestamp.body`, codifique em base64 e compare
com uma assinatura `v1,` em tempo constante. Use `webhook-id`,
`webhook-timestamp` e `webhook-signature`, rejeite timestamps fora de cinco minutos
e deduplique o ID. O timestamp de assinatura usa segundos Unix. Events usa
callbacks próprios, separados desses webhooks.

Veja [Events](events.pt-BR.md), [erros](errors.pt-BR.md),
[schemas completos](tool-payloads.json) e a
[API pública](https://api.assinafy.com.br/v1/docs).
