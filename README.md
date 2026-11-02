# Servidor MCP da Assinafy

*Português · [Read in English](README.en.md)*

Servidor MCP em Go para assinaturas e gestão do ciclo de vida de contratos na
Assinafy. Expõe 29 ferramentas por Streamable HTTP sem estado: 27 ferramentas de
assinatura e duas de CLM com ações fixas. O contrato REST público está na
[documentação da API](https://api.assinafy.com.br/v1/docs).

## Conectar

O servidor é um resource server OAuth 2.1. O cliente descobre os metadados, registra
por CIMD, abre o consentimento no navegador com PKCE S256 e guarda/renova o próprio
token. Cada conexão escolhe uma workspace. O MCP valida issuer, audience, validade,
workspace e escopos, depois troca o token por outro token curto e exclusivo da API.
As credenciais OAuth do operador ficam somente no servidor.

```bash
codex mcp add assinafy --url https://mcp.assinafy.com.br/mcp
codex mcp login assinafy

claude mcp add --transport http assinafy https://mcp.assinafy.com.br/mcp
```

Em conectores remotos do Claude, use a mesma URL e conclua a autorização. Os escopos
são `account:read`, `documents:read`, `documents:write`, `templates:read` e
`templates:write`; o cliente pode aprovar um subconjunto. Veja
[autenticação](docs/authentication.md) e [configuração do cliente](MCP_CLIENT_DOCUMENTATION.md).

## Fluxo completo de assinatura

1. Localize documentos, templates, campos, papéis e contatos com as ferramentas de
   listagem/leitura. Resolva nomes ambíguos e preserve os IDs retornados.
2. Para um PDF novo por email, use `assinafy_send_pdf_for_signature`. A ferramenta
   valida, envia uma vez, aguarda metadados, reutiliza/cria contatos e cria um
   assignment virtual por Email. Para preparar sem enviar, use
   `assinafy_upload_document` e depois `assinafy_create_assignment`.
3. Para template, leia todos os papéis e campos, valide os valores de editor e use
   `assinafy_create_document_from_template` com um signatário por papel.
4. Escolha `Email`, `Whatsapp`, `DigitalCertificate` ou
   `IdDocumentFaceMatchBasic`. Só um canal de notificação é permitido. Email e
   Whatsapp exigem o canal correspondente; certificado e biometria aceitam ambos.
   A1/A3, documento de identidade e liveness são concluídos pela pessoa na Assinafy.
5. Use `virtual` sem campos ou `collect` com campos posicionados nas páginas.
   Certificado exige `virtual` e signatário sozinho na etapa. Quando houver ordem,
   todos informam etapas contíguas iniciando em 1.
6. Antecipe custos de assignment, template ou lembrete com `estimate_only`. Plano,
   saldo, elegibilidade de identidade e assinatura continuam sob controle da API.
7. Acompanhe documento, assignment, páginas de atividades e notificações. Em
   `pending_identity_review`, uma pessoa autorizada da conta revisa as evidências.
8. Aguarde `certificated`, baixe o artefato base64 e, se necessário, valide o hash
   real da assinatura com `assinafy_verify_document`.

Exemplo com email sintético:

```json
{"file_name":"contrato.pdf","file_base64":"<PDF_BASE64>","signers":[{"full_name":"Signatário Exemplo","email":"signer@example.invalid"}]}
```

`assinafy_update_signer` aceita null ou string vazia para limpar email, WhatsApp ou
documento fiscal. `assinafy_set_assignment_expiration` aceita data/data-hora ISO
8601 ou null para remover o prazo. Atividades aceitam `sort`, `page` e `per_page`.
Entradas, respostas, exemplos, estados e recuperação estão em
[ferramentas](docs/tools.md), [schemas completos](docs/tool-payloads.json) e
[erros](docs/errors.md).

Escritas nunca são repetidas automaticamente. Uma falha composta pode deixar um
contato ou documento enviado; preserve o ID, confira o assignment e continue sem
reenviar o PDF. O assistente nunca assina por uma pessoa.

## Extensões MCP Apps e Events

Hosts compatíveis renderizam o recurso `ui://` anunciado com
`text/html;profile=mcp-app`. Ele atende status/lembretes, revisão antes do envio,
posicionamento visual de campos e navegação de contratos. Outros hosts recebem o
mesmo resultado em texto e conteúdo estruturado.

MCP Events é uma extensão opcional anunciada com `events/list`, `events/subscribe`
e `events/unsubscribe`. Monitora `document.status_changed`,
`document.certificated` e `contract.activity`, valida o challenge, assina entregas,
limita inscrições e usa arquivo privado persistente. Exige uma réplica e é separado
das subscriptions do núcleo MCP e dos webhooks da conta. Veja [Events](docs/events.md).

Webhooks de conta são geridos na Assinafy. Guarde o segredo único ao criar um
endpoint assinado; OAuth não o lê nem o rotaciona. O receptor verifica Standard
Webhooks sobre o corpo bruto, rejeita timestamps antigos e deduplica delivery IDs.

## Gestão de contratos

Use `assinafy_browse_contracts` para listar, ler ou seguir uma família. Use
`assinafy_update_contract` para definir termos, renovar ou encerrar somente após ler
o `updated_at`; renovação também preserva o termo atual. Em conflito, releia e nunca
sobrescreva silenciosamente. Aditivos são documentos assinados separados, ligados
por `parent_document_id` e `relation_type`. Events informa; não autoriza escrita.

## Executar e operar

Requer Go 1.27.3+, MCP Go SDK v1.8.0 e HTTPS em produção. O binário não carrega
`.env`; injete segredos em runtime.

```bash
make build
./bin/assinafy-mcp-server -addr :8080
```

| Variável | Função |
|---|---|
| `MCP_RESOURCE_URL` | URL pública obrigatória terminando em `/mcp` |
| `ASSINAFY_OAUTH_ISSUER` | Padrão `https://auth.assinafy.com.br` |
| `ASSINAFY_BASE_URL` | Padrão `https://api.assinafy.com.br/v1` |
| `ASSINAFY_OAUTH_CLIENT_ID` / `ASSINAFY_OAUTH_CLIENT_SECRET` | Cliente de introspecção/troca obrigatório |
| `ASSINAFY_TIMEOUT` | Timeout da API; padrão `30s` |
| `MCP_EVENTS_STORE` | Arquivo privado opcional de Events; uma réplica |
| `MCP_ADDR` | Padrão `:8080` |
| `MCP_TLS_CERT_FILE` / `MCP_TLS_KEY_FILE` | TLS direto opcional |
| `MCP_ALLOWED_ORIGINS` | Origens exatas de navegador, separadas por vírgula |
| `OPENAI_APPS_CHALLENGE` | Token opcional de verificação de domínio OpenAI |
| `LOG_LEVEL` / `LOG_FORMAT` | Padrão `info` / `json` |

`POST /mcp` atende MCP; `GET /mcp` entrega identidade, manifesto e metadados OAuth.
A descoberta também fica em `/.well-known/oauth-protected-resource/mcp`; saúde em
`/healthz`. O proxy preserva host público e headers OAuth/MCP, encerra TLS e aplica
limites por origem e tenant. Requisições: 40 MiB; PDF decodificado: 25 MiB; JSON:
16 MiB; artefato binário: 64 MiB. Veja [segurança e deploy](SECURITY.md).

## Verificar

```bash
make verify
make docker-build
make docker-test
make docker-lint
```

`make verify` executa formatação, consistência de módulos, vet, testes com race,
build e vulnerabilidades alcançáveis. Testes de transporte cobrem as 29 ferramentas
e versões MCP suportadas; testes de payload validam todos os schemas e exemplos.
Integrações com tag exigem credenciais em runtime e destinatário expressamente
autorizado; credenciais sozinhas não autorizam envios, cobranças ou mutações remotas.

No release, renderize o Helm privado, injete credenciais pelo gerenciador de segredos
e mantenha tokens, documentos, certificados e dados reais fora do código, logs e docs.
