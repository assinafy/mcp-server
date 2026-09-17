# Guia do cliente MCP da Assinafy

*Português · [Read in English](README.en.md)*

Conecte uma vez com **OAuth2 da Assinafy e CIMD** e conduza o ciclo de vida do
documento pelas ferramentas MCP: preparar e enviar, acompanhar status e entrega,
reenviar lembretes, atualizar a expiração, baixar artefatos e retomar trabalho
incompleto.

Use a URL HTTPS de Streamable HTTP terminada em `/mcp`; substitua o host de
exemplo pelo endpoint informado pelo operador. Claude e Codex descobrem suas
identidades CIMD automaticamente. O cliente não cria aplicativo OAuth nem informa
client ID, client secret, chave de API ou cabeçalho de workspace. Entre na
Assinafy, escolha um workspace e aprove as permissões solicitadas. O servidor
impõe esse workspace em todas as operações.

Conectar e listar as ferramentas não exigem login, então o cliente já mostra o
que o servidor oferece; o consentimento é pedido quando uma ferramenta precisa do
seu workspace. Os clientes abaixo se identificam por um Client ID Metadata
Document, que é como a Assinafy os reconhece; veja
[situação da conexão](#situação-da-conexão).

## Codex

```bash
codex mcp add assinafy --url https://mcp.assinafy.com.br/mcp
codex mcp login assinafy
```

O Codex descobre o recurso protegido e o servidor de autorização, usa sua própria
URL CIMD hospedada e abre o consentimento da Assinafy. Escolha o workspace e
aprove as permissões. Nada além disso pertence à configuração do usuário.
Veja a [documentação MCP do Codex](https://developers.openai.com/codex/mcp).

## Claude Code

```bash
claude mcp add --transport http assinafy https://mcp.assinafy.com.br/mcp
```

Abra `/mcp` no Claude Code e autentique na Assinafy. O Claude Code descobre o
suporte a CIMD pelo issuer; não informe client ID nem client secret. Veja a
[documentação MCP do Claude Code](https://code.claude.com/docs/en/mcp).

## Conectores remotos do Claude

Nas configurações de conectores do Claude, adicione um conector remoto
personalizado chamado Assinafy com a URL `https://mcp.assinafy.com.br/mcp`,
conecte e conclua o consentimento. Deixe as credenciais OAuth opcionais em branco
quando houver CIMD. A disponibilidade de conectores personalizados depende do
plano e das configurações da organização. O registro por linha de comando é do
Claude Code; um conector hospedado precisa da URL HTTPS pública.

O Claude escolhe CIMD quando os metadados do issuer anunciam ao mesmo tempo
`client_id_metadata_document_supported: true` e `none` em
`token_endpoint_auth_methods_supported`. O MCP da Assinafy verifica os dois na
inicialização. Veja a
[autenticação de conectores do Claude](https://claude.com/docs/connectors/building/authentication).

## ChatGPT

Configure uma conexão MCP remota com OAuth e a URL da Assinafy. Escolha CIMD
quando a configuração oferecer a opção de registro. O ChatGPT suporta
autenticação de cliente público com `none`; o cliente não precisa de client
secret. Sua identidade de metadados e seu redirect são diferentes dos do Codex,
então o servidor de autorização precisa permitir os dois de forma independente.
Veja a [autenticação do ChatGPT](https://developers.openai.com/plugins/build/auth)
e os requisitos de registro de cliente.

## VS Code / GitHub Copilot Chat

Adicione esta entrada à configuração MCP do VS Code:

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

Inicie o servidor e conclua a autorização no navegador. O VS Code escolhe CIMD
quando anunciado e sabe pedir escopos adicionais quando uma ferramenta precisa.
Veja o [suporte a autenticação do VS Code](https://code.visualstudio.com/updates/v1_106#_authentication-client-id-metadata-document-authentication-flow)
e a [configuração MCP](https://code.visualstudio.com/docs/agents/reference/mcp-configuration).

## Workspaces e permissões

Uma conexão autoriza um workspace. As ferramentas com escopo de conta descobrem
essa conta automaticamente. O argumento `account_id` pode repeti-la, mas não
seleciona outra. Conecte cada workspace adicional separadamente. Nunca coloque
chaves de API, tokens OAuth ou qualquer outra credencial em prompts, argumentos
de ferramenta ou `_meta`; requisições que os carregam são recusadas.

Você configura a URL do MCP e nada mais. Seu cliente se identifica à Assinafy, e
a tela de consentimento pede
`account:read documents:read documents:write templates:read` — todo o catálogo de
ferramentas — de modo que uma aprovação cobre tudo e o primeiro envio não é
recusado no meio do caminho. `templates:write` nunca é pedido: nenhuma ferramenta
altera um template. Peça `offline_access` quando o cliente usar refresh tokens
para acesso em segundo plano.

**A Assinafy impõe as permissões, e este servidor as reporta.** Os access tokens
são opacos, então o servidor não consegue ler o que a concessão permite e não
recusa uma escrita antecipadamente. Uma chamada recusada devolve a mensagem da
própria Assinafy nomeando o escopo que falta. Isso importa nas ferramentas de
várias etapas: com uma concessão parcial, `assinafy_send_document_for_signature`
pode enviar o PDF e ser recusada na etapa seguinte. O erro carrega o
`document_id` preservado, então continue por `assinafy_create_assignment` em vez
de enviar o arquivo de novo. Aprovar o conjunto completo na conexão é o que torna
isso raro.

Para o Codex, autorize explicitamente o fluxo completo de documentos e templates:

```bash
codex mcp login assinafy --scopes account:read,documents:read,documents:write,templates:read,offline_access
```

O cliente guarda e renova os próprios tokens. O MCP não oferece ferramenta de
login, não aceita refresh tokens e não guarda credenciais do cliente. Conexões
expiradas ou revogadas retornam HTTP 401; reconecte quando a renovação não
restaurar a concessão.

## Fluxo do documento

Todas as operações de documento passam pelo MCP com a concessão OAuth2 da
conexão. O cliente cuida de access tokens, refresh tokens e novo consentimento. O
servidor publica 28 ferramentas de documento e fornece instruções de fluxo na
inicialização do MCP. Atualize a lista de ferramentas do cliente depois de uma
atualização do servidor. Não há ferramenta de login nem necessidade de chamar a
API REST pela conversa.

```mermaid
flowchart TD
    A[Conectar e autorizar um workspace] --> B{Origem do documento}
    B --> C[Localizar um documento existente]
    B --> D[Enviar um PDF ou usar um template]
    D --> E[Preparar, estimar custo, solicitar assinaturas]
    C --> F[Ler status, assignment, signatários e atividades]
    E --> F
    F --> G{Estado atual}
    G -->|Pendente| H[Conferir entrega, lembrar ou atualizar expiração]
    H --> F
    G -->|Certificando| F
    G -->|Certificado| I[Baixar e verificar]
    G -->|Falha ou recusa| J[Inspecionar e escolher a recuperação]
```

### 1. Localizar o documento e conferir seu estado

Use `assinafy_list_documents` com `search`, `status`, `page` e `per_page`. Ele
também aceita `sort`, `method` (`virtual` ou `collect`) e IDs de tag separados por
vírgula em `tags`. Todas as tags informadas precisam corresponder. As páginas
começam em 1 e trazem no máximo 100 registros; siga o `meta` de paginação
retornado em vez de supor que a primeira página contém todos os documentos.

Por exemplo, chame `assinafy_list_documents` com:

```json
{"status":"pending_signature","sort":"-updated_at","page":1,"per_page":25}
```

Identificado o documento correto, chame `assinafy_get_document`:

```json
{"document_id":"DOCUMENT_ID"}
```

Use os IDs devolvidos pela Assinafy em todas as chamadas seguintes. Não invente
IDs nem deduza o ID do assignment a partir do ID do documento. Os detalhes do
documento expõem o `status` atual, `is_closed`, `assignment`, os registros de
signatários, IDs de página, artefatos e qualquer motivo de recusa informado pela
Assinafy. O ciclo de vida documentado é:

| Estado | Próximo passo |
|---|---|
| `uploading`, `uploaded`, `metadata_processing` | O processamento não terminou. Verifique depois; assignments `collect` precisam dos metadados de página prontos. |
| `metadata_ready` | Inspecione ou renomeie o PDF e prepare a solicitação de assinatura. |
| `pending_signature` | Inspecione signatários pendentes, ordem de assinatura, histórico de entrega e expiração. |
| `certificating` | Todas as assinaturas podem estar presentes enquanto os artefatos finais ainda são gerados. |
| `certificated` | Baixe os artefatos assinados disponíveis. Este estado não é excluível pela API documentada. |
| `expired` | Inspecione o assignment e decida se atualiza a expiração. A Assinafy valida se a atualização é permitida. |
| `rejected_by_signer`, `rejected_by_user` | Leia os detalhes da recusa e escolha o próximo passo com o usuário. Um lembrete não reverte uma recusa. |
| `failed` | Leia as atividades do documento e o erro antes de escolher a recuperação. |

As conferências de status são leituras individuais, não assinaturas de eventos.
Use consultas com limite quando pedirem para acompanhar um documento; pare em um
estado terminal ou no tempo combinado.

### 2. Inspecionar progresso e entrega

Use `assinafy_get_document` para identificar quem realmente está pendente. O
assignment traz `id`, `signers[].id`, `completed`, `step`, `notified`,
`notification_history` e as URLs de assinatura quando disponíveis. Campos
opcionais ausentes significam que a API não forneceu aquela informação. Um
signatário aguardando uma etapa anterior não está necessariamente sofrendo falha
de entrega.

Chame `assinafy_list_document_activities` com `document_id` para a linha do tempo
dos eventos, incluindo data e payload de cada um. Inspecione
`notification_history` em busca de eventos enviados/falhos e detalhes de erro.
Para entrega por WhatsApp, chame `assinafy_list_whatsapp_notifications` com
`document_id` e `assignment_id`. Mensagens de WhatsApp em staging podem ser
simuladas. Trate links e códigos de acesso retornados como sensíveis e
compartilhe apenas com os destinatários autorizados.

**100% de progresso não prova que o PDF certificado está pronto.** Confira o
estado do ciclo de vida e a disponibilidade do artefato antes de baixá-lo.

### 3. Estimar e reenviar um lembrete

Leia o documento de novo, identifique o signatário pendente e a etapa ativa, e
confirme que a instrução do usuário autoriza aquele lembrete. Escopos OAuth
autorizam o acesso a uma operação; eles não escolhem um destinatário pelo
usuário.

Chame `assinafy_estimate_resend_cost` com:

```json
{
  "document_id":"DOCUMENT_ID",
  "assignment_id":"ASSIGNMENT_ID",
  "signer_id":"SIGNER_ID"
}
```

Verifique o custo retornado, os saldos, `has_sufficient_resources` e qualquer
`blocking_reason`. Estimar não envia mensagem. Quando autorizado, chame
`assinafy_resend_notification` com os mesmos identificadores.

`is_sent: true` informa o resultado do reenvio; não significa que a pessoa
assinou. Leia o histórico de entrega e o progresso depois. Não reenvie
repetidamente só porque o progresso não mudou. Uma resposta incerta pode vir
depois de um envio bem-sucedido; inspecione as atividades antes de tentar de
novo. Respeite limites de taxa e o tempo de nova tentativa.

### 4. Atualizar a expiração ou corrigir dados do destinatário

Use `assinafy_reset_assignment_expiration` com os IDs dos detalhes atuais do
documento e um timestamp RFC 3339 explícito:

```json
{
  "document_id":"DOCUMENT_ID",
  "assignment_id":"ASSIGNMENT_ID",
  "expires_at":"2027-01-31T23:59:59-03:00"
}
```

Escolha a data e o fuso realmente pedidos pelo usuário; o exemplo ilustra apenas
o formato. Leia o assignment atualizado para confirmar a data resultante. Não
suponha que atualizar a expiração também enviou um novo convite.

Encontre contatos por `assinafy_list_signers` ou `assinafy_get_signer`.
`assinafy_update_signer` aceita `full_name`, `email`, `whatsapp_phone_number` e
`government_id`. Um contato é compartilhado dentro do workspace, então revise a
alteração pretendida antes de aplicá-la. A Assinafy bloqueia mudanças em um canal
já verificado em um documento em andamento. Alterar um canal não verificado
invalida links e códigos antigos; depois de uma correção bem-sucedida, use um
reenvio autorizado para entregar o link novo. Preserve e informe uma recusa da
Assinafy.

### 5. Preparar e enviar um novo PDF

Para o fluxo comum por e-mail, chame `assinafy_send_document_for_signature`:

```json
{
  "file_name":"contrato.pdf",
  "file_base64":"BYTES_DO_PDF_EM_BASE64",
  "signers":[{"full_name":"Pessoa de Exemplo","email":"signer@example.invalid"}],
  "message":"Revise e assine o documento",
  "max_wait_secs":30
}
```

O endereço é fictício. Informe apenas destinatários autorizados para a tarefa
real. A ferramenta envia o PDF, aguarda o processamento, cria ou reutiliza
contatos de e-mail e inicia um assignment com verificação e convite por Email.
Guarde `document.id`, `assignment.id` e `signer_ids` do resultado.

Para preparar antes de enviar, use esta sequência:

1. `assinafy_upload_document` aceita `file_name` e `file_base64`, devolve um
   documento e não envia convites. PDFs podem ter até 25 MiB e 2.000 páginas; a
   Assinafy impõe o limite de páginas. O servidor MCP nunca abre um caminho de
   arquivo local.
2. `assinafy_get_document` confere a prontidão e fornece IDs e dimensões de
   página. Use `assinafy_rename_document` com `document_id` e `name` enquanto
   renomear é permitido: antes de existir um assignment, em `uploaded` ou
   `metadata_ready`.
3. `assinafy_list_signers` / `assinafy_create_signer` fornecem os IDs dos
   signatários. Criar um contato sozinho não envia solicitação de assinatura. A
   criação avulsa pode retornar conflito; procure o contato existente antes de
   tentar de novo.
4. `assinafy_estimate_assignment_cost` precifica os métodos de verificação e
   notificação pretendidos sem enviar convites. IDs de signatário não são
   necessários para a estimativa; signatários virtuais por Email podem ser
   representados por entradas `{}`.
5. `assinafy_create_assignment` inicia a assinatura do documento existente usando
   os IDs de signatário. Guarde o assignment e as identidades retornadas para o
   acompanhamento.

Exemplo de argumentos de `assinafy_create_assignment`:

```json
{
  "document_id":"DOCUMENT_ID",
  "method":"virtual",
  "signers":[
    {"id":"SIGNER_ID","verification_method":"Email","notification_methods":["Email"],"step":1}
  ],
  "message":"Revise e assine o documento"
}
```

A ferramenta de assignment aceita `virtual` e `collect`, `copy_receivers`
opcionais (IDs de signatário), `message`, `expires_at` e ordem de assinatura por
`step`. Os métodos de verificação são `Email`, `Whatsapp` e
`DigitalCertificate`; as notificações usam os canais Email/WhatsApp
documentados. WhatsApp exige o contato e a assinatura de plano adequados;
assinatura com certificado exige os dados de identidade do signatário e o
recurso habilitado na conta. Use as estimativas de custo para os valores atuais.
A Assinafy valida esses requisitos.

Na assinatura ordenada, se um signatário informar `step`, todos precisam
informar; os valores devem ser contíguos a partir de 1. Pessoas na mesma etapa
assinam em paralelo. Um signatário com certificado digital fica sozinho na etapa
dele.

Para `collect`, aguarde `metadata_ready`. Use `assinafy_list_fields`
(opcionalmente `include_standard: true`) e `assinafy_get_field` para as
definições de campo. Passe `entries[]` com `page_id` e `fields[]`; cada
posicionamento contém `signer_id`, `field_id` e `display_settings` com `left`,
`top`, `width`, `height` e `fontSize`. As coordenadas usam os pixels da imagem da
página em 150 DPI. O servidor recusa posicionamentos fora da página escolhida
antes de criar o assignment. As prévias de página estão em
`assinafy_download_document_page`.

#### Validar valores de campo

Use `assinafy_list_fields` e `assinafy_get_field` para inspecionar as definições
e valide os valores propostos com `assinafy_validate_fields`:

```json
{
  "values": [
    {"field_id": "ID_DO_CAMPO_NOME", "value": "Empresa Exemplo"},
    {"field_id": "ID_DO_CAMPO_TOTAL", "value": "1250.00"}
  ]
}
```

O MCP converte `values` no corpo em array JSON da API. A validação não cria
documento nem envia convites. Verifique `success` e `error_message` de cada
resultado; uma requisição HTTP bem-sucedida pode reportar um valor inválido.
Mantenha como texto os identificadores com zeros à esquerda. O
`editor_fields[].value` de template sempre recebe uma string, mesmo quando o
valor representa número ou data.

### 6. Gerar um documento a partir de um template

Use `assinafy_list_templates` e leia seus papéis, páginas e campos de editor.
`assinafy_get_template` oferece a consulta direta onde o ambiente suporta essa
rota de compatibilidade. A resposta da listagem é a fonte documentada quando a
rota individual não está disponível.

Para um pedido como “preencha `customer_name` e `contract_total` no template de
contrato de serviço”, resolva os nomes antes de criar qualquer coisa:

1. Selecione o template pedido em `assinafy_list_templates`, seguindo a
   paginação, e inspecione seu status de processamento, `roles` e
   `pages[].fields`.
2. Relacione os nomes pedidos aos `label` dos posicionamentos ou às definições
   devolvidas por `assinafy_list_fields` / `assinafy_get_field`. Ligue o `id` da
   definição ao `field_id` do posicionamento e confira o `role_id` do
   posicionamento contra o papel de editor do template. Preencha apenas campos já
   configurados naquele template.
3. Use o **`field_id`** do posicionamento em `editor_fields` — não o `id` do
   posicionamento nem seu rótulo. Nomes de campo são configuráveis; eles não são
   nomes de argumento de ferramenta. O cliente faz esse mapeamento pelos
   metadados; o servidor aceita IDs.
4. Se os nomes faltarem ou forem ambíguos, peça ao usuário que identifique o
   campo pretendido. Não adivinhe. Posicionamentos repetidos com o mesmo
   `field_id` recebem um único valor; a API não oferece valor por posicionamento
   nesta requisição.
5. Valide os valores propostos, resolva um signatário por papel do template e
   estime o custo antes da criação autorizada.

Criação de template e edição de layout e papéis são documentadas apenas na API
interna e não são expostas por este servidor. Configure o template salvo na
Assinafy antes de usar o fluxo público de documento por template.

Estime primeiro com `assinafy_estimate_template_document_cost`:

```json
{"template_id":"TEMPLATE_ID","signers":[{"role_id":"ROLE_ID"}]}
```

Escolha a ferramenta de criação conforme a informação disponível:

- `assinafy_create_document_from_template` recebe um contato
  `{role_id,full_name,email}` por papel, cria ou reutiliza os contatos e usa
  verificação e notificação por Email.
- `assinafy_create_template_document` recebe mapeamentos `{role_id,id}` de
  signatários existentes e aceita as opções documentadas de verificação,
  notificação, etapa de assinatura e tags. Suas `tags` são **nomes** de tag,
  mesclados com os padrões do template.

Ambas aceitam nome personalizado, mensagem, expiração e `editor_fields` com
`{field_id,value}`. A criação pode enviar convites. Guarde o ID do documento
retornado e chame `assinafy_get_document` para o assignment e as conferências de
status seguintes.

Com IDs de signatário existentes, os argumentos de criação ficam assim
(substitua pelos IDs descobertos no template e no diretório de signatários):

```json
{
  "template_id": "TEMPLATE_ID",
  "name": "Contrato de serviço.pdf",
  "signers": [
    {"role_id": "ID_DO_PAPEL_EDITOR", "id": "ID_DO_SIGNATARIO_PREPARADOR"},
    {"role_id": "ID_DO_PAPEL_CLIENTE", "id": "ID_DO_SIGNATARIO_CLIENTE"}
  ],
  "editor_fields": [
    {"field_id": "ID_DO_CAMPO_NOME", "value": "Empresa Exemplo"},
    {"field_id": "ID_DO_CAMPO_TOTAL", "value": "1250.00"}
  ]
}
```

Use `assinafy_create_template_document` nessa chamada. As conferências de status,
lembretes, mudanças de expiração e downloads seguintes usam os IDs do documento
retornado, como em um PDF enviado.

### 7. Inspecionar prévias de página

`assinafy_download_document_page` recebe `document_id` e um `page_id` retornado.
Devolve os bytes da imagem em base64. Baixe apenas as páginas necessárias.

### 8. Baixar artefatos assinados e verificar

Concluída a certificação, chame `assinafy_download_document` com o ID do
documento e `artifact: "certificated"`. O resultado traz `document_id`, o nome do
artefato e `base64`. Decodifique e salve os bytes pelas ferramentas de arquivo do
cliente; não imprima um base64 grande na conversa.

A ferramenta também aceita `original`, `certificated`, `certificate-page`,
`pades` e `bundle`. A disponibilidade do artefato depende do ciclo de vida e do
método de assinatura; um artefato indisponível não é um download bem-sucedido. As
respostas binárias são limitadas a 64 MiB antes da codificação base64.

Chame `assinafy_verify_document` com o hash SHA-1 de assinatura do documento
assinado. Use o hash de verificação real fornecido com o documento, não o ID do
documento nem um valor inventado. Esta é a única ferramenta que funciona sem
conectar: ela consulta um registro público de assinatura e não toca em nada
pertencente a um workspace. Informe o `is_valid` retornado.

As pessoas concluem assinaturas, recusas e verificações exigidas dentro da
Assinafy. Esses endpoints usam o código de acesso do signatário, fora da
concessão OAuth2 do workspace. Estas ferramentas não aceitam termos, não digitam
códigos de verificação e não assinam no lugar de ninguém.

### 9. Retomar uma operação parcial ou excluir

Um envio de várias etapas pode falhar depois do upload. O erro preserva o ID do
documento criado. Inspecione esse documento e suas atividades antes de
continuar:

- Um assignment existente indica que a solicitação de assinatura pode já ter dado
  certo. Continue acompanhando; não crie outro assignment nem reenvie às cegas.
- Um documento preparado sem assignment continua por
  `assinafy_create_assignment` com os IDs de signatário confirmados. Não precisa
  de novo upload. Confira o status de processamento antes de posicionar campos.
- Se pedirem limpeza, use `assinafy_delete_document` apenas quando o estado atual
  do documento permitir. Essa ação precisa de autorização do usuário.

Nenhuma requisição de escrita é repetida automaticamente pelo servidor MCP.

## Erros e reconexão

| Resultado | Ação do cliente |
|---|---|
| HTTP 401 | Deixe o cliente renovar a concessão ou reconectar pelo consentimento da Assinafy. |
| HTTP 403 com `insufficient_scope` | Autorize o conjunto de escopos pedido e tente de novo após o consentimento. |
| Erro de ferramenta nomeando um escopo ausente | A Assinafy recusou a chamada. Uma ferramenta de várias etapas pode ter concluído etapas anteriores; leia o erro em busca de um `document_id` preservado e continue dali. |
| Resultado de ferramenta com `isError: true` | Leia o erro da Assinafy e inspecione o documento atual antes de repetir uma escrita. |
| Contato inválido, assignment expirado ou artefato indisponível | Corrija a requisição ou aguarde o estado necessário do ciclo de vida. |
| Limite de taxa ou resposta de rede incerta | Respeite o tempo de nova tentativa; inspecione o estado e o histórico de entrega antes de repetir uma escrita. |
| HTTP 429 | A Assinafy está limitando as verificações de autorização; respeite o `Retry-After`. |
| HTTP 503 na autenticação | A Assinafy estava inacessível durante a verificação do token; tente de novo. Não existe alternativa por chave de API no fluxo do cliente. |

## Situação da conexão

O servidor está no ar em `https://mcp.assinafy.com.br/mcp`. Conectar e listar as
28 ferramentas já funciona sem credencial — experimente:

```bash
curl -sS -X POST https://mcp.assinafy.com.br/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Concluir um login depende de dois pontos no servidor de autorização da Assinafy,
os dois de configuração e não de código:

| | |
|---|---|
| **Confiança no cliente** | O cliente se identifica por um Client ID Metadata Document, e a Assinafy só aceita um sob um prefixo da sua lista de confiança. Um cliente fora dela é recusado com `invalid_client`. Alguns clientes então informam que o servidor "não suporta registro dinâmico de cliente" — essa mensagem descreve o plano B deles, não a causa. O Claude Code já é confiável. |
| **Registro do recurso** | O cliente envia o valor `resource` que este servidor publica, `https://mcp.assinafy.com.br/mcp`. A Assinafy responde `invalid_target` enquanto essa URL não for registrada como um recurso para o qual ela emite tokens. |

As credenciais OAuth2 do cliente ficam no armazenamento dele. O servidor não tem
nenhuma: ele verifica o bearer token apresentando-o à Assinafy, repassa-o sem
alteração e não guarda nada.
