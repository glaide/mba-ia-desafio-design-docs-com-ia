# Tracker de Rastreabilidade

Mapeamento dos itens dos design docs para origem em `TRANSCRICAO.md` ou no código-fonte. Itens sem localização válida não devem permanecer na documentação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Risco de churn Atlas até fim do trimestre/novembro | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo / Métrica | Latência de notificação &lt; 10 segundos | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Webhook outbound em mudança de status | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Registro de webhook com URL, customerId, statuses e secret gerado | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02b | docs/PRD.md | Restrição | customer_id no body/path, não implícito no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar, atualizar e remover webhooks via JWT | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtro de eventos no enqueue por statuses inscritos | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Histórico ~100 deliveries | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | HMAC-SHA256 e X-Signature; secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | DLQ após 5 falhas + replay ADMIN | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Meta &lt;10s com poll 2s | TRANSCRICAO | [09:10] Marcos |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Auditoria de quem fez replay | TRANSCRICAO | [09:36] Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Payload order.status_changed sem items | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Outbox na mesma txn do status | TRANSCRICAO | [09:06] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | HTTPS-only nas URLs | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout 10s e backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Payload máximo 64KB | TRANSCRICAO | [09:23] Sofia |
| PRD-OOS-01 | docs/PRD.md | Fora de Escopo | E-mail em falhas repetidas adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de Escopo | Dashboard visual fora desta fase | TRANSCRICAO | [09:39] Marcos |
| PRD-OOS-03 | docs/PRD.md | Fora de Escopo | Rate limit outbound observar depois | TRANSCRICAO | [09:38] Diego |
| PRD-OOS-04 | docs/PRD.md | Fora de Escopo | Arquivamento outbox ~30d fora da feature | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente sem dedupe processa duplicata | TRANSCRICAO | [09:25] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Deadline novembro vs ~3 sprints | TRANSCRICAO | [09:45] Marcos |
| PRD-DEP-01 | docs/PRD.md | Dependência | Review de segurança Sofia ≥2 dias úteis | TRANSCRICAO | [09:47] Sofia |
| RFC-TLDR-01 | docs/RFC.md | Decisão | Proposta outbox MySQL + worker + HMAC + at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Descarte HTTP síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Descarte Redis Streams / fila Redis | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Descarte exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting outbound em aberto | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | Confirmar deadline Atlas | TRANSCRICAO | [09:46] Marcos |
| RFC-RISK-01 | docs/RFC.md | Risco | Crescimento da tabela outbox | TRANSCRICAO | [09:08] Diego |
| RFC-ADR-01 | docs/RFC.md | Referência | Links para ADRs do pacote | TRANSCRICAO | [09:48] Larissa |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Insert outbox na txn de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | publishWebhookEvent(tx, …) pura | TRANSCRICAO | [09:41] Diego |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Worker poll 2s e entrega HTTP | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Retry/backoff e DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Replay ADMIN da DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks cria config e devolve secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET/PATCH/DELETE de webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | POST admin dead-letter replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Rotação de secret via API | TRANSCRICAO | [09:21] Sofia |
| FDD-HDR-01 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| FDD-PAY-01 | docs/FDD.md | Contrato | Campos do payload e event_type order.status_changed | TRANSCRICAO | [09:43] Diego |
| FDD-ERR-01 | docs/FDD.md | Erro | Prefixo WEBHOOK_* (NOT_FOUND, INVALID_URL, SECRET_REQUIRED) | TRANSCRICAO | [09:28] Bruno |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout HTTP worker 10s | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Reuso Pino para logs | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métricas de delivery/lag propostas no FDD | TRANSCRICAO | [09:29] Bruno |
| FDD-INT-01 | docs/FDD.md | Integração | Extensão de OrderService.changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Alinhamento com canTransition / OrderStatus | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | AppError e subclasses HTTP | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Códigos de erro de domínio existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate + requireRole ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Montagem de rotas /api/v1 | CODIGO | src/routes/index.ts |
| FDD-INT-07 | docs/FDD.md | Integração | DI buildControllers / buildApp | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Logger Pino compartilhado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Novos models no schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-MOD-01 | docs/FDD.md | Integração | Módulo webhooks espelhando estrutura orders | TRANSCRICAO | [09:27] Bruno |
| FDD-WRK-01 | docs/FDD.md | Integração | Entry worker src/worker.ts + npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL decidido | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Trigger MySQL descartado | TRANSCRICAO | [09:09] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Decisão | 5 tentativas + backoff + DLQ separada | TRANSCRICAO | [09:17] Larissa |
| ADR-002-ALT | docs/adrs/ADR-002-retry-backoff-e-dlq.md | Alternativa | 3 retries descartados como agressivos | TRANSCRICAO | [09:16] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-por-endpoint.md | Decisão | HMAC-SHA256 secret por endpoint + grace 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-003-ALT | docs/adrs/ADR-003-hmac-sha256-por-endpoint.md | Alternativa | Secret global descartado | TRANSCRICAO | [09:21] Sofia |
| ADR-004 | docs/adrs/ADR-004-at-least-once-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005 | docs/adrs/ADR-005-worker-polling-processo-separado.md | Decisão | Worker processo separado polling 2s | TRANSCRICAO | [09:11] Diego |
| ADR-005-ORD | docs/adrs/ADR-005-worker-polling-processo-separado.md | Restrição | Ordenação por order_id com single-worker | TRANSCRICAO | [09:12] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-do-projeto.md | Decisão | Reuso máximo dos padrões do OMS | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CODE | docs/adrs/ADR-006-reuso-padroes-do-projeto.md | Integração | Referência explícita a order.service e errors | CODIGO | src/modules/orders/order.service.ts |
| ADR-SNAP-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Snapshot do payload no insert do outbox | TRANSCRICAO | [09:52] Diego |
| ADR-UUID-01 | docs/FDD.md | Decisão | IDs UUID alinhados ao projeto | TRANSCRICAO | [09:51] Larissa |

## Notas de cobertura

- Linhas acima cobrem FRs/NFRs do PRD, alternativas e open questions do RFC, fluxos/contratos/erros/integração do FDD e os seis ADRs.
- Proporção: maioria `TRANSCRICAO` com timestamp `[hh:mm] Nome`; linhas `CODIGO` apontam paths reais do repositório.
- Ao adicionar item novo em qualquer doc, incluir linha correspondente aqui antes do merge.
