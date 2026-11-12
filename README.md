# Servidor MCP da Assinafy

*Português (Brasil) · [English (US)](README.en.md)*

Servidor MCP para assinaturas e gestão do ciclo de vida de contratos na
Assinafy. Expõe 29 ferramentas por Streamable HTTP: 27 ferramentas de
assinatura e duas de CLM com ações fixas. O contrato REST público está na
[documentação da API](https://api.assinafy.com.br/v1/docs).

## Conectar

Adicione `https://mcp.assinafy.com.br/mcp` como conexão MCP remota no seu cliente.
Entre na Assinafy pelo navegador, escolha a workspace e autorize as permissões
necessárias. O cliente mantém a conexão OAuth e renova o acesso. Use uma das
opções abaixo conforme o seu cliente:

```bash
codex mcp add assinafy --url https://mcp.assinafy.com.br/mcp
codex mcp login assinafy

claude mcp add --transport http assinafy https://mcp.assinafy.com.br/mcp
```

Em conectores remotos do Claude, use a mesma URL e conclua a autorização. Os escopos
são `account:read`, `documents:read`, `documents:write`, `templates:read` e
`templates:write`; o cliente pode aprovar um subconjunto. Veja
[autenticação](docs/authentication.pt-BR.md). Não é necessário informar API key ou
credenciais de uma aplicação OAuth para conectar.

## Experimente no seu cliente

Após conectar, copie os prompts abaixo na conversa com o assistente. Use um PDF
sem dados sensíveis e uma pessoa que autorizou o teste. Substitua
`signer@example.invalid` pelo email autorizado: esse endereço é fictício e não
recebe mensagens. Envios e lembretes reais podem consumir créditos.

### 1. Confira a conexão sem enviar nada

> Liste até cinco documentos recentes da minha workspace e mostre nome, ID e
> status. Não altere nem envie documentos.

**Resultado esperado:** documentos da workspace escolhida, ou uma lista vazia.
Se houver erro de autorização, siga [autenticação](docs/authentication.pt-BR.md).
Depois, peça: “Mostre os signatários pendentes e as atividades do documento ID.”
Substitua `ID` pelo valor retornado; o assistente deve esclarecer nomes duplicados.

### 2. Prepare um PDF, confira o custo e envie por email

Anexe `teste-assinatura.pdf` em um cliente que consiga fornecer arquivos ao MCP:

> Prepare este PDF para assinatura por Email de Signatário Exemplo,
> signer@example.invalid. Estime o custo sem enviar convites. Mostre o ID do
> documento, os créditos necessários e qualquer impedimento.

**Resultado esperado:** um documento enviado para a Assinafy e uma estimativa,
sem contatos novos ou convites. A estimativa do PDF já faz upload; guarde o ID.
Após conferir o resultado e substituir `ID` e `LIMITE` pelos valores escolhidos:

> Envie o documento ID já preparado para Signatário Exemplo,
> signer@example.invalid, por Email, com limite total de LIMITE créditos.
> Reutilize o documento; não faça outro upload.

**Resultado esperado:** IDs do documento e do assignment, com os signatários.
A pessoa recebe o convite e conclui a assinatura na Assinafy. Não repita o envio
se houver timeout: consulte o documento e confira se o assignment já existe.

### 3. Use um template salvo

> Liste meus templates. Para o template que eu escolher, mostre os papéis de
> signatários e os campos que preciso preencher, antes de criar um documento.

Depois de escolher e fornecer os valores de todos os papéis e campos obrigatórios:

> Estime o custo desse template com os destinatários e valores informados.
> Não crie nem envie o documento ainda.

**Resultado esperado:** papéis/campos e uma estimativa. Para enviar, peça
“Crie e envie usando esses valores, com limite total de LIMITE créditos”.
Guarde o ID retornado e aguarde o processamento antes de consultar as assinaturas.

### 4. Teste outro método de assinatura

Para um documento preparado sem assignment, escolha o método explicitamente:

| Método | Exemplo de pedido |
|---|---|
| WhatsApp | “Estime a assinatura do documento ID por WhatsApp para o contato escolhido, com notificação por WhatsApp.” |
| Certificado A1/A3 | “Estime a assinatura do documento ID por certificado digital para o contato escolhido, com notificação por Email.” |
| Biometria | “Estime a assinatura do documento ID com documento de identidade e reconhecimento facial para o contato escolhido, com notificação por Email.” |

**Resultado esperado:** custo e elegibilidade, ou o impedimento informado pela API.
Confira o telefone do contato para WhatsApp e o CPF/CNPJ para certificado ou
biometria. Após revisar, peça o envio com o mesmo método e um limite de créditos.
Certificado exige signatário sozinho na etapa. Certificado, fotos e prova de vida
são apresentados pela pessoa na Assinafy, nunca enviados ao assistente.

### 5. Acompanhe, lembre e baixe o resultado

> Consulte o documento ID e diga quem ainda precisa assinar. Mostre as atividades
> recentes e o histórico de notificações disponível.

Para um lembrete: “Estime o custo de lembrar o signatário pendente escolhido no
documento ID”. Após conferir, peça “Envie esse lembrete”.

> Quando o documento ID estiver certificated, baixe o PDF assinado e verifique
> sua autenticidade usando o código de verificação retornado pela Assinafy.

**Resultado esperado:** status atual; após a certificação, o arquivo e o resultado
da verificação. O cliente precisa conseguir salvar o arquivo retornado pelo MCP.
Se ainda faltar assinatura, consulte novamente depois ou use Events abaixo.
Se estiver `pending_identity_review`, a conta precisa revisar as evidências na Assinafy.

### 6. Receba Events e consulte contratos

> Monitore o documento ID e me avise quando ele for certificado.

**Resultado esperado:** inscrição em um cliente compatível com Events e aviso
quando houver uma nova certificação durante o monitoramento. Mantenha a conexão
ativa. Se o cliente não suportar Events, peça uma nova consulta de status;
o prompt sozinho não habilita monitoramento em segundo plano.

> Liste meus contratos acompanhados e mostre os termos e a data de término do
> contrato que eu escolher. Não renove nem encerre nada.

**Resultado esperado:** termos atuais, ou indicação de que o plano não oferece
esse recurso. O término contratual é diferente do prazo para assinar.

## Fluxo completo de assinatura

1. Localize documentos, templates, campos, papéis e contatos com as ferramentas de
   listagem/leitura. Resolva nomes ambíguos e preserve os IDs retornados.
2. Para um PDF novo por email, use `assinafy_send_pdf_for_signature`. A ferramenta
   valida, envia uma vez, aguarda metadados, reutiliza/cria contatos e cria um
   assignment virtual por Email. Para preparar sem enviar, use
   `assinafy_upload_document` e depois `assinafy_create_assignment`.
3. Para template, leia todos os papéis e campos, valide os valores de editor e use
   `assinafy_create_document_from_template` com todos os papéis Signer preenchidos.
   Papéis CopyReceiver são opcionais; papéis Editor usam `editor_fields`. Aguarde
   o processamento do template antes de inspecionar o assignment.
4. Escolha `Email`, `Whatsapp`, `DigitalCertificate` ou
   `IdDocumentFaceMatchBasic`. Só um canal de notificação é permitido. Email e
   Whatsapp exigem o canal correspondente; certificado e biometria aceitam ambos.
   A1/A3, documento de identidade e liveness são concluídos pela pessoa na Assinafy.
5. Use `virtual` sem campos ou `collect` com campos posicionados nas páginas.
   Certificado exige `virtual` e signatário sozinho na etapa. Quando houver ordem,
   todos informam etapas contíguas iniciando em 1.
6. Antecipe custos de assignment, template ou lembrete com `estimate_only`. Nos
   envios de PDF, assignment ou template, use `max_total_credits` para limitar
   o gasto; zero permite apenas envio sem créditos. Plano, saldo, elegibilidade
   de identidade e assinatura continuam sob controle da API.
7. Acompanhe documento, assignment, páginas de atividades e notificações. Em
   `pending_identity_review`, uma pessoa autorizada da conta revisa as evidências.
8. Aguarde `certificated`, baixe o artefato base64 e, se necessário, valide o
   `verification_code` retornado por `assinafy_get_document` com
   `assinafy_verify_document`. Passe o código sem alterar; não calcule o hash do PDF.

`assinafy_update_signer` aceita null ou string vazia para limpar email, WhatsApp ou
documento fiscal. `assinafy_set_assignment_expiration` aceita data/data-hora ISO
8601 ou null para remover o prazo. Atividades aceitam `sort`, `page` e `per_page`.
Entradas, respostas, exemplos, estados e recuperação estão em
[ferramentas](docs/tools.pt-BR.md), [schemas completos](docs/tool-payloads.json) e
[erros](docs/errors.pt-BR.md).

Escritas nunca são repetidas automaticamente. Uma falha composta pode deixar um
contato ou documento enviado; preserve o ID, confira o assignment e continue sem
reenviar o PDF. O assistente nunca assina por uma pessoa.

## Extensões MCP Apps e Events

Hosts compatíveis renderizam o recurso `ui://` anunciado com
`text/html;profile=mcp-app`. Ele atende status/lembretes, revisão antes do envio,
posicionamento visual de campos e navegação de contratos. Outros hosts recebem o
mesmo resultado em texto e conteúdo estruturado.

Events permite acompanhar mudanças de status, conclusão da certificação e
atividades de contratos em clientes compatíveis. Peça ao assistente para monitorar
um documento. A conexão OAuth e a inscrição precisam continuar ativas; o cliente
as renova. Veja [Events](docs/events.pt-BR.md) para a integração e as regras de entrega.

Webhooks de conta são geridos na Assinafy. Guarde o segredo único ao criar um
endpoint assinado; OAuth não o lê nem o rotaciona. O receptor verifica Standard
Webhooks sobre o corpo bruto, rejeita timestamps antigos e deduplica delivery IDs.

## Gestão de contratos

Use `assinafy_browse_contracts` para listar, ler ou seguir uma família. Use
`assinafy_update_contract` para definir termos, renovar ou encerrar somente após ler
o `updated_at`; renovação também preserva o termo atual. Em conflito, releia e nunca
sobrescreva silenciosamente. Aditivos são documentos assinados separados, ligados
por `parent_document_id` e `relation_type`. Events informa; não autoriza escrita.
