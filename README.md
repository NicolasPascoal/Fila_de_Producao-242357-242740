# Fila de Produção — Documentação de API

**Serviço 07 — PZaaS (Pizza as a Service)**
Dupla: RA 242357 · RA 242740
Workflow n8n: `Fila_de_Producao-242357-242740`

## Visão geral

A Fila de Produção recebe pedidos aprovados pelo Orquestrador, enfileira, envia ao Forno (Serviço 06) respeitando a capacidade de 2 pizzas simultâneas, e notifica o Orquestrador quando o pedido fica pronto.

Fluxo: `Orquestrador → Fila de Produção → Forno → Fila de Produção → Orquestrador`

## Autenticação

Todas as requisições exigem os headers:

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `x-api-key` | `turma2026` |

## Base URL

```
https://pzaas.online/webhook/242357
```

*(mesmo domínio compartilhado usado pelo Forno e pelo Orquestrador — os exemplos abaixo usam a URL completa)*

> **Nota de status:** até a publicação final na instância compartilhada, o serviço foi validado em ambiente local (`http://localhost:5678/webhook/242357/...`) com testes reais contra o Forno e o Orquestrador de produção. A URL acima entra em operação assim que o workflow for ativado na instância compartilhada da turma.

## Endpoints

### `GET https://pzaas.online/webhook/242357/health`

Healthcheck público, sem autenticação.

**Resposta 200:**
```json
{ "status": "UP", "service": "fila-producao-242357" }
```

---

### `POST https://pzaas.online/webhook/242357/v1/fila/entrada`

Chamado pelo Orquestrador para inserir um pedido na fila de produção.

**Headers adicionais:**

| Header | Obrigatório | Descrição |
|---|---|---|
| `x-pedido-id` | Sim | Identificador único do pedido |

**Body:**
```json
{
  "PedidoID": "30",
  "SaborPizza": "Calabresa"
}
```

**Respostas:**

| HTTP | Situação | Body |
|---|---|---|
| 201 | Pedido enfileirado com sucesso | `{ "queued": true, "pedidoId": "30" }` |
| 400 | Header `x-pedido-id` ausente | `{ "error": "header x-pedido-id obrigatorio" }` |
| 401 | `x-api-key` ausente ou inválida | `{ "error": "x-api-key ausente ou invalida" }` |
| 409 | Pedido já enfileirado (mesmo `x-pedido-id`) | `{ "error": "pedido ja enfileirado", "pedidoId": "30" }` |

---

### `GET https://pzaas.online/webhook/242357/v1/fila/proxima-pronta` *(endpoint auxiliar de inspeção)*

Não faz parte do fluxo automático — usado apenas para depuração manual da fila de pizzas prontas.

**Headers adicionais:** `x-api-key`

| HTTP | Situação | Body |
|---|---|---|
| 200 | Há um pedido pronto | `{ "pedidoId": "30" }` |
| 404 | Fila de prontos vazia | `{ "error": "nenhum pedido pronto no momento" }` |
| 401 | `x-api-key` inválida | `{ "error": "x-api-key ausente ou invalida" }` |

## O que acontece depois do `201` (fluxo interno)

1. O pedido é enfileirado no Redis (`fila-242357:aguardando_forno`).
2. Um processo interno verifica a cada poucos segundos se há vaga (máx. 2 pizzas simultâneas no Forno) e pedidos pendentes.
3. A Fila chama o Forno: `POST https://pzaas.online/webhook/213804/v1/forno` com `x-pedido-id` e `x-api-key: turma2026`.
4. Se o Forno responder `503` (lotado ou indisponível), o pedido volta para a fila e é tentado novamente automaticamente.
5. Se o Forno responder `200` (`status: PRONTO`), a Fila notifica o Orquestrador:
   `POST https://pzaas.online/webhook/v1/pedido-pronto` com `x-pedido-id` e body `{ "status": "Pronto" }`.
6. Eventos de todo o processo (`PEDIDO_ENFILEIRADO`, `ENVIADO_AO_FORNO`, `FORNO_INDISPONIVEL`, `PEDIDO_PRONTO_NOTIFICADO`, `ERRO_FORNO`) são enviados ao Logger (Serviço 09).

## Exemplo real de chamada (capturado em teste de integração contra o Forno de produção)

**Request enviado pela Fila ao Forno real:**
```
POST https://pzaas.online/webhook/213804/v1/forno
Content-Type: application/json
x-api-key: turma2026
x-pedido-id: TESTE-CLAUDE-1788918835550

{ "pizzaName": "Teste-Integracao" }
```

**Response real recebida (HTTP 200):**
```json
{
  "success": true,
  "pedidoId": "TESTE-CLAUDE-1788918835550",
  "forno": "FORNO-02",
  "status": "PRONTO",
  "tamanho": null,
  "tamanhoLabel": "PADRAO",
  "pizzaId": null,
  "pizzaName": "Teste-Integracao",
  "tempoPreparoSegundos": 6,
  "timeSource": "default",
  "startedAt": "2026-09-09T01:54:42.022Z",
  "finishedAt": "2026-09-09T01:54:49.078Z"
}
```

## Resiliência

- Chamadas ao Forno e ao Orquestrador não derrubam o fluxo em caso de falha de rede — o pedido é marcado com erro e o slot de capacidade é sempre liberado.
- Chamadas ao Logger são "fire-and-forget": uma falha no Logger nunca impede o processamento do pedido.
