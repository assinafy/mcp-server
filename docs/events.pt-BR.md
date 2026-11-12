# Events

*Português (Brasil) · [English (US)](events.md)*

Events permite receber notificações sobre mudanças de status de documentos,
conclusão da certificação e atividades de contratos. Em um cliente compatível,
peça ao assistente para monitorar um documento pela conexão MCP da Assinafy.

A conexão OAuth é a mesma das ferramentas. Events não altera os webhooks da conta
nem autoriza ações como enviar lembretes, renovar ou encerrar contratos. A pessoa
signatária conclui as assinaturas e verificações de identidade na Assinafy.

## Eventos disponíveis

| Evento | Notificação |
|---|---|
| `document.status_changed` | O status observado do documento mudou |
| `document.certificated` | A certificação terminou e os artefatos assinados estão disponíveis |
| `contract.activity` | Uma nova atividade de aviso, alteração ou renovação foi registrada |

Cada inscrição acompanha um `document_id`. Localize o documento com
`assinafy_list_documents` ou `assinafy_browse_contracts` e reutilize o ID retornado.
A inscrição estabelece um estado inicial: um documento já certificado não gera
um novo evento de conclusão. Consulte-o com `assinafy_get_document`.

## Integração do cliente

Events é uma extensão anunciada para MCP 2.0 (`2026-07-28`). Confira a capability
`events` em `server/discover` e use `events/list` para descobrir nomes e schemas
de entrada e payload. É independente das inscrições de recursos do núcleo MCP.
Veja a [integração MCP Events](https://developers.openai.com/plugins/build/mcp-events).

`events/list`, `events/subscribe` e `events/unsubscribe` usam `/mcp` e exigem
`account:read` e `documents:read`. O cliente MCP fornece headers e metadados do
protocolo. Em requisições diretas, use `Content-Type: application/json`,
`Authorization` com OAuth, `MCP-Protocol-Version: 2026-07-28` e `Mcp-Method` com o
mesmo método da requisição.

Exemplo de inscrição com documento e callback sintéticos:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "events/subscribe",
  "params": {
    "_meta": {
      "io.modelcontextprotocol/protocolVersion": "2026-07-28",
      "io.modelcontextprotocol/clientCapabilities": {},
      "io.modelcontextprotocol/clientInfo": {
        "name": "example-client",
        "version": "1.0.0"
      }
    },
    "name": "document.certificated",
    "arguments": { "document_id": "document-example" },
    "delivery": {
      "mode": "webhook",
      "url": "https://receiver.example.com/events/callback-example",
      "secret": "whsec_<base64-encoded-24-to-64-byte-key>"
    },
    "cursor": null
  }
}
```

Exemplo de resultado; ID e horário são ilustrativos:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resultType": "complete",
    "id": "sub_example",
    "refreshBefore": "2030-01-01T12:10:00Z",
    "cursor": null,
    "truncated": false
  }
}
```

O callback deve usar HTTPS público, aceitar verificações assinadas e responder
exatamente `{"challenge":"<challenge recebido>"}` antes das entregas. Redirecionamentos
não são seguidos. A URL tem limite de 2 KiB. Proteja o segredo de assinatura do
callback; ele é diferente do token OAuth.

## Renovação e cancelamento

Inscrições são idempotentes para a mesma autorização OAuth, workspace, evento,
documento e callback. Renovar o token mantém essa identidade; novo consentimento
cria outra autorização. Uma autorização diferente não pode renovar nem cancelar
a inscrição anterior.

A duração padrão é de dez minutos e nunca ultrapassa o tempo restante do token.
O `ttlMs` opcional é limitado entre 30 segundos e dez minutos; `null` também
recebe duração finita. Renove o token quando necessário e repita
`events/subscribe` antes de `refreshBefore`. Renovar só o token não renova a
inscrição. Veja [autenticação](authentication.pt-BR.md).

Após vencer, a inscrição pausa verificações e entregas. O evento pendente fica
disponível por mais dois minutos para uma renovação tardia. Após reconectar ou
reiniciar o serviço, renove a inscrição para retomar o monitoramento. Se ela não
estiver mais disponível, uma nova inscrição estabelece outro estado inicial.

Use `events/unsubscribe` com nome do evento, argumentos e URL de entrega originais;
o segredo não é necessário. O resultado é vazio e a operação é idempotente.
O limite é de 16 inscrições por workspace e 32 no total. Remova uma inscrição
que não seja mais necessária antes de adicionar outra quando atingir o limite.

## Entrega e recuperação

As verificações normalmente ocorrem a cada 30 segundos; latência da API e volume
de inscrições podem atrasar notificações. Não há reprodução histórica
(`cursor: null`, `truncated: false`). Mudanças intermediárias ou ocorridas durante
uma interrupção podem não ser observadas. Leia o documento ou contrato atual
antes de agir a partir de um evento.

Cada entrega contém `eventId`, `name`, `timestamp`, `data` e `cursor: null`.
Eventos de documento incluem `document_id`, `status` e `previous_status` em `data`:

```json
{
  "eventId": "evt_example",
  "name": "document.certificated",
  "timestamp": "2030-01-01T12:05:00Z",
  "data": {
    "document_id": "document-example",
    "status": "certificated",
    "previous_status": "pending_signature"
  },
  "cursor": null
}
```

Verifique a assinatura Standard Webhooks HMAC-SHA256 sobre os bytes originais do
corpo usando `webhook-id`, `webhook-timestamp` e `webhook-signature`. O ID da
inscrição está em `X-MCP-Subscription-Id`. Rejeite timestamps antigos e deduplique
`eventId`. A troca do segredo é verificada e mantém sobreposição de um minuto com
a chave anterior.

Uma resposta `2xx` confirma o recebimento. Falhas de rede, `408`, `429` e `5xx`
recebem até cinco tentativas com espera progressiva e o mesmo ID de evento.
Erros permanentes como `410` e `413`, ou tentativas esgotadas, encerram aquela
entrega; o monitoramento continua. Pode haver duplicidade: ações posteriores
devem ser idempotentes. Revogação, falta de permissões ou perda de acesso ao
documento encerram o monitoramento. Falhas transitórias da API pausam as
verificações. Veja [recuperação de erros](errors.pt-BR.md#events).

## Atividades de contratos

`contract.activity` informa `document_contract_updated`,
`document_contract_renewal_notice`, `document_contract_expiring`,
`document_contract_expired`, `document_contract_renewed`,
`document_contract_terminated`, `document_contract_amended` e
`document_contract_amendment_updated`.

O `data` contém `document_id`, `activity_id` como string e `activity`.
`end_date` e `days_before` aparecem quando fornecidos. O timestamp corresponde à
atividade registrada. Nomes, documentos fiscais, valores, metadados, motivos de
encerramento e payloads completos de atividades não são entregues.

Inscreva-se separadamente no documento de aditivo para acompanhar alterações de
seu vínculo. Atividades históricas não são reproduzidas. Deduplique
`(document_id, activity_id)` entre inscrições independentes e leia os termos
atuais com `assinafy_browse_contracts` antes de agir.

Mantenha a conexão OAuth e a inscrição renovadas para monitoramento prolongado.
Os emails da Assinafy aos responsáveis oferecem lembretes independentes do MCP.
Uma inscrição não cria um segundo lembrete nem concede permissão para enviar
outro email, renovar ou encerrar um contrato automaticamente.
