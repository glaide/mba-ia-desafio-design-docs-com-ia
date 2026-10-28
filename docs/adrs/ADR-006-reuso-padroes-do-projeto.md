# ADR-006: Reuso dos padrões existentes do projeto

- **Status:** Aceito
- **Data:** 2025-10-23 (reunião técnica)
- **Decisores:** Larissa (Tech Lead), Bruno (Orders), Diego (Platform), Sofia (Security)

## Contexto

O OMS já possui convenções claras de módulos, erros, validação, autenticação e logging. Introduzir um “subproduto” de webhooks com stack ou estilos paralelos aumentaria custo de manutenção e onboarding. A feature deve parecer um módulo nativo do sistema, incluindo o ponto de integração no ciclo de vida do pedido.

## Decisão

Reutilizar ao máximo os padrões já presentes no repositório:

1. **Módulo de domínio** em `src/modules/webhooks/` com a mesma estrutura dos demais (`controller`, `service`, `repository`, `routes`, `schemas`), mais `webhook.worker.ts` / `webhook.processor.ts` para a lógica de entrega.
2. **Erros** via hierarquia `AppError` em `src/shared/errors/app-error.ts` e subclasses em `src/shared/errors/http-errors.ts`, com códigos prefixados **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), consumidos pelo `src/middlewares/error.middleware.ts`.
3. **Validação** com Zod + `src/middlewares/validate.middleware.ts` (incluindo regra HTTPS-only na URL).
4. **Auth**: `authenticate` para CRUD de webhooks; `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` para replay de DLQ — mesmo padrão de rotas admin de usuários.
5. **Logging** com Pino em `src/shared/logger/index.ts` (API e worker).
6. **Integração no pedido**: estender a transação de `OrderService.changeStatus` em `src/modules/orders/order.service.ts` chamando uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` — sem injetar o repositório completo de webhooks no `OrderService`.
7. **Registro de rotas/DI**: estender `src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`), montando sob `/api/v1`.
8. **Persistência**: novos models Prisma em `prisma/schema.prisma` (config, outbox, deliveries, dead letter), IDs UUID alinhados ao restante do schema.
9. **Filtro de status** alinhado ao enum/`canTransition` de `src/modules/orders/order.status.ts`.

## Alternativas Consideradas

1. **Microserviço / pacote separado só para webhooks** — Descartada nesta fase: aumenta ops e duplica auth/DB sem necessidade (`[09:30]` Larissa: módulo igual aos outros).
2. **Injetar `WebhookRepository` inteiro no `OrderService`** — Descartada: acoplamento alto; preferir função pura com `tx` (`[09:41]` Bruno/Diego).
3. **Framework de filas / jobs (Bull, etc.)** — Descartada junto com Redis (ADR-001/005).

## Consequências

**Positivas**

- Curva de aprendizado baixa para quem já conhece Orders/Customers.
- Erros, logs e auth consistentes na API.
- Ponto de extensão em `changeStatus` permanece explícito e testável.

**Negativas / trade-offs**

- `changeStatus` ganha uma responsabilidade a mais (chamar publish); precisa de testes de integração cobrindo rollback da txn.
- Worker reutiliza padrões da API mas vive em outro processo — documentação de ops (`npm run worker`) torna-se obrigatória.
- Prefixo `WEBHOOK_` precisa ser mantido com disciplina para não misturar códigos genéricos.
