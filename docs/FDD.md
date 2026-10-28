# FDD — Feature Design Document: Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

O OMS (Express + Prisma/MySQL) já possui ciclo de vida de pedido com máquina de estados, estoque transacional e `order_status_history`. Não há eventos, filas nem notificações externas. Clientes B2B precisam ser avisados quando o status muda, sem polling contínuo em `GET /orders`.

A solução acordada ([RFC](./RFC.md), [ADRs](./adrs/README.md)) é: **outbox no MySQL** na mesma txn de `changeStatus` + **worker separado** com polling + entrega HTTP assinada.

## Objetivos técnicos

1. Enfileirar evento de webhook de forma atômica com a mudança de status.
2. Entregar HTTP outbound com HMAC-SHA256, retries e DLQ.
3. Expor API de configuração (CRUD), histórico de deliveries e replay ADMIN.
4. Reutilizar padrões do repositório (módulos, AppError, Zod, auth, Pino).

## Escopo e exclusões

**Inclui:** models Prisma (config, outbox, delivery, DLQ); módulo `webhooks`; hook em `changeStatus`; worker; contratos HTTP abaixo; códigos `WEBHOOK_*`.

**Exclui:** e-mail de alerta; dashboard; rate limit outbound; arquivamento automático (~30d); multi-worker; items no payload; exactly-once; HTTP síncrono; Redis.

## Fluxos detalhados

### F1 — Criação do evento na outbox

```
changeStatus(orderId, toStatus, userId)
  → prisma.$transaction(tx):
      load order + items
      validate transition (canTransition)
      debit/replenish stock se aplicável
      update orders.status
      insert order_status_history
      publishWebhookEvent(tx, order, from, to):
          list webhooks ativos do customer_id
            cujo statuses[] contém to
          para cada match:
            build payload snapshot (≤64KB)
            insert webhook_outbox
              status=pending, event_id=UUID, payload=JSON,
              attempt_count=0, available_at=now
      return order refreshed
```

Se o insert do outbox falhar, a txn inteira faz rollback (status não muda).

### F2 — Processamento pelo worker

```
loop a cada 2s:
  SELECT pending WHERE available_at <= now
    ORDER BY created_at ASC LIMIT N
    FOR UPDATE SKIP LOCKED (quando multi-instância futura; hoje single-worker)
  marcar processing
  HTTP POST url
    headers: Content-Type, X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id
    body = payload snapshot
    timeout = 10s
  se 2xx:
    outbox → delivered
    insert webhook_delivery (success, latency_ms, response meta)
  senão:
    registrar delivery failure
    se attempt_count < 5: pending + available_at = now + backoff[attempt]
    senão: mover para webhook_dead_letter e marcar failed
```

Backoff: índice 0..4 → `1m, 5m, 30m, 2h, 12h`.

### F3 — Retry

Controlado apenas por `attempt_count` e `available_at` no outbox. O worker não reenvia linhas com `available_at` no futuro. O `event_id` / `X-Event-Id` permanece o mesmo em todas as tentativas.

### F4 — DLQ e replay

Após 5 falhas: copiar para `webhook_dead_letter` (payload, last_error, failed_at, webhook_id, order_id, event_id).

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` (ADMIN):

1. Carrega DLQ; 404 se inexistente.
2. Reinsere outbox `pending` com mesmo `event_id` e payload (ou novo attempt cycle zerado — preferir **mesmo event_id** para dedupe do cliente).
3. Audita em log Pino: `webhook_dlq_replay` com `adminUserId`, `deadLetterId`, `eventId`.
4. Opcional: marca DLQ como `replayed_at`.

## Contratos públicos

Base path: `/api/v1`. Auth: `Authorization: Bearer <JWT>` em todas as rotas abaixo.

### 1. `POST /webhooks` — criar endpoint

**Request**

```json
{
  "customerId": "clx_customer_01",
  "url": "https://atlas.example/hooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `201`**

```json
{
  "id": "clx_wh_01",
  "customerId": "clx_customer_01",
  "url": "https://atlas.example/hooks/orders",
  "statuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_live_a1b2c3d4e5f6",
  "createdAt": "2025-10-23T12:00:00.000Z"
}
```

`secret` só aparece no create (e na rotação). URL `http://` → `400` `WEBHOOK_INVALID_URL`.

### 2. `GET /webhooks?customerId=` — listar

**Response `200`**

```json
{
  "data": [
    {
      "id": "clx_wh_01",
      "customerId": "clx_customer_01",
      "url": "https://atlas.example/hooks/orders",
      "statuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2025-10-23T12:00:00.000Z",
      "updatedAt": "2025-10-23T12:00:00.000Z"
    }
  ]
}
```

Secrets **não** são listados.

### 3. `PATCH /webhooks/:id` — atualizar

**Request**

```json
{
  "url": "https://atlas.example/hooks/orders/v2",
  "statuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `200`** — objeto atualizado (sem secret). `404` → `WEBHOOK_NOT_FOUND`.

### 4. `DELETE /webhooks/:id`

**Response `204`** sem body. Soft-delete (`active=false`) ou hard-delete — implementação: hard-delete se não houver outbox pendente; caso contrário rejeitar com `409` `WEBHOOK_HAS_PENDING_DELIVERIES` **ou** soft-delete. Preferência da feature: **soft-deactivate** (`active=false`) para preservar histórico.

### 5. `GET /webhooks/:id/deliveries` — histórico (~últimas 100)

**Response `200`**

```json
{
  "data": [
    {
      "id": "clx_del_01",
      "eventId": "550e8400-e29b-41d4-a716-446655440000",
      "success": true,
      "httpStatus": 200,
      "latencyMs": 182,
      "attempt": 1,
      "requestPayload": { "event_type": "order.status_changed", "order_id": "..." },
      "responseBodyPreview": "{\"ok\":true}",
      "createdAt": "2025-10-23T12:00:05.000Z"
    }
  ]
}
```

### 6. `POST /webhooks/:id/rotate-secret` — rotação

**Response `200`**

```json
{
  "id": "clx_wh_01",
  "secret": "whsec_live_new_999",
  "previousSecretExpiresAt": "2025-10-24T12:00:00.000Z"
}
```

Durante 24h o worker assina preferencialmente com o secret novo; verificação no cliente deve aceitar ambos. Após `previousSecretExpiresAt`, apenas o novo.

### 7. `POST /admin/webhooks/dead-letter/:id/replay`

Requer `requireRole('ADMIN')`.

**Response `200`**

```json
{
  "deadLetterId": "clx_dlq_01",
  "outboxId": "clx_ob_99",
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending"
}
```

`403` se não ADMIN; `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### Payload outbound (worker → cliente)

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "order.status_changed",
  "timestamp": "2025-10-23T12:00:00.000Z",
  "order_id": "clx_ord_01",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "clx_customer_01",
  "total_cents": 19990
}
```

**Headers**

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | mesmo UUID do `event_id` |
| `X-Signature` | HMAC-SHA256 hex/base64 do body com secret do endpoint |
| `X-Timestamp` | ISO 8601 do envio |
| `X-Webhook-Id` | id do webhook config |

Sem `items`. Detalhe completo via `GET /orders/:id`.

## Matriz de erros previstos

| Código | HTTP | Quando |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | id inexistente em GET/PATCH/DELETE/deliveries/rotate |
| `WEBHOOK_INVALID_URL` | 400 | URL não HTTPS ou inválida |
| `WEBHOOK_SECRET_REQUIRED` | 400 | operação que exige secret e config está inconsistente |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | snapshot &gt; 64KB no enqueue |
| `WEBHOOK_STATUSES_REQUIRED` | 400 | lista de statuses vazia no create/patch |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` inválido |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | replay de id inexistente |
| `WEBHOOK_INACTIVE` | 409 | tentativa de operar endpoint inativo onde não faz sentido |
| `VALIDATION_ERROR` | 400 | Zod genérico (já existente) |
| `UNAUTHORIZED` / `FORBIDDEN` | 401 / 403 | JWT ausente / role insuficiente |

Erros de entrega outbound **não** sobem para a API de pedidos; ficam em deliveries/DLQ/logs do worker.

## Estratégias de resiliência

| Mecanismo | Valor |
| --- | --- |
| Timeout HTTP worker | 10s |
| Poll interval | 2s |
| Retries | 5 |
| Backoff | 1m / 5m / 30m / 2h / 12h |
| Fallback após esgotar | DLQ + replay ADMIN |
| TLS | HTTPS-only nas URLs |
| Consistência | outbox na mesma txn do status |
| Idempotência cliente | `X-Event-Id` estável |

## Observabilidade

**Métricas (a instrumentar no worker/API)**

- `webhook_outbox_pending_count`
- `webhook_delivery_success_total` / `webhook_delivery_failure_total`
- `webhook_delivery_latency_ms` (histograma)
- `webhook_dlq_total`
- `webhook_outbox_lag_seconds` (now − min(created_at) pending)

**Logs (Pino — `src/shared/logger`)**

- `webhook_enqueued` (orderId, eventId, webhookId)
- `webhook_delivery_attempt` (eventId, attempt, httpStatus, latencyMs)
- `webhook_delivery_exhausted` → DLQ
- `webhook_dlq_replay` (adminUserId, …)
- Redação de `secret` / assinaturas (estender `redactPaths`)

**Tracing**

- Propagar `X-Request-Id` da API de mudança de status até o log de enqueue (`orderId` + `eventId` como correlation).
- No worker, span lógico por tentativa (`eventId` + `attempt`) — OpenTelemetry opcional na mesma fase se o time já tiver coletor; caso contrário correlation ids nos logs bastam como MVP de tracing distribuído leve.

## Dependências e compatibilidade

- Node ≥20, Express, Prisma/MySQL, Zod, JWT, Pino — já no projeto.
- Novo script `npm run worker` e entrypoint `src/worker.ts`.
- Migração Prisma para tabelas de webhook.
- Sem novas dependências de broker obrigatórias.
- Compatível com enum `OrderStatus` existente.

## Integração com o sistema existente

| Arquivo | Como integra |
| --- | --- |
| `src/modules/orders/order.service.ts` | Ao final da txn de `changeStatus` (após update + `orderStatusHistory.create`), chamar `publishWebhookEvent(tx, order, from, to)`. Falha no publish aborta a txn. |
| `src/modules/orders/order.status.ts` | Lista de `statuses` do webhook valida contra `OrderStatus` / transições conhecidas; filtro de enqueue usa o `to` já validado por `canTransition`. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Novas subclasses/`UnprocessableEntityError`/`NotFoundError` com códigos `WEBHOOK_*`, no mesmo formato `{ error: { code, message, details } }`. |
| `src/middlewares/auth.middleware.ts` | CRUD: `authenticate`. Replay DLQ: `authenticate` + `requireRole('ADMIN')`. |
| `src/middlewares/error.middleware.ts` + `validate.middleware.ts` | Mesmo pipeline de erros AppError/Zod para rotas do módulo. |
| `src/routes/index.ts` + `src/app.ts` | Registrar `WebhookController` em `Controllers` / `buildControllers` e `router.use('/webhooks', …)` + rota admin. |
| `src/shared/logger/index.ts` | Logger compartilhado API/worker; redaction de secrets. |
| `prisma/schema.prisma` | Models `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter` com IDs UUID. |

Assinatura sugerida:

```ts
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: { id: string; orderNumber: string; customerId: string; totalCents: number },
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void>
```

## Critérios de aceite técnicos

1. Mudança de status com webhook inscrito gera exatamente uma linha outbox na mesma txn (rollback conjunto em falha).
2. Worker entrega &lt;10s na mediana com poll 2s e cliente saudável.
3. HMAC verificável; URL http rejeitada.
4. Após 5 falhas o evento aparece na DLQ; replay ADMIN reenfileira.
5. `X-Event-Id` estável entre retries.
6. Códigos de erro da API usam prefixo `WEBHOOK_`.
7. Testes cobrindo: enqueue na txn, filtro de status, HTTPS validation, backoff scheduling, replay authz.

## Riscos e mitigação

| Risco | Mitigação |
| --- | --- |
| Acoplar demais `OrderService` | Função pura `publishWebhookEvent`; testes de unidade isolados |
| Payload crescer no futuro | Cap 64KB + erro explícito; sem items |
| Lag do outbox sob pico | Métricas de lag; single-worker documentado |
| Secret em logs | Redaction Pino + nunca retornar secret em GET |
