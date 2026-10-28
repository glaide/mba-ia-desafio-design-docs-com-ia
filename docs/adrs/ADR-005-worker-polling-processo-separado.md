# ADR-005: Worker em processo separado com polling

- **Status:** Aceito
- **Data:** 2025-10-23 (reunião técnica)
- **Decisores:** Larissa (Tech Lead), Diego (Platform), Bruno (Orders)

## Contexto

O disparo HTTP dos webhooks não pode rodar dentro do request path da API de pedidos: aumenta latência do `changeStatus`, acopla falhas de rede externa ao processo HTTP e dificulta escalar API e entrega de forma independente. Ainda assim, o time quer evitar fila externa. Com outbox no MySQL, o worker precisa de um mecanismo simples para descobrir pendências.

Meta de “tempo real” dos clientes: notificação em menos de 10 segundos.

## Decisão

1. Implementar um **processo Node separado** (entrypoint `src/worker.ts`, script `npm run worker`), espelhando o papel de `src/server.ts` para a API.
2. O worker faz **polling** na tabela `webhook_outbox` a cada **2 segundos**, buscando lotes `pending` ordenados por `created_at`.
3. Usa o **mesmo** `DATABASE_URL` / stack Prisma, mas **instância própria** de `PrismaClient` (não compartilha processo com a API).
4. Enquanto houver um único worker, a ordenação por `created_at` garante ordem por `order_id`; multi-worker/particionamento fica como limitação conhecida para o futuro.
5. Timeout HTTP de cada tentativa: **10 segundos**.

## Alternativas Consideradas

1. **Worker no mesmo processo da API** (setInterval no `server.ts`) — Descartada: falha/restart do worker derruba a API e vice-versa; acoplamento indesejado (`[09:11]` Diego).
2. **Push via trigger MySQL** — Descartada (ver ADR-001).
3. **Fila Redis / broker** — Descartada nesta fase (ver ADR-001).

## Consequências

**Positivas**

- Deploy e scaling independentes de API e worker.
- Poll de 2s atende a meta de &lt;10s com folga.
- Implementação simples, alinhada ao stack atual.

**Negativas / trade-offs**

- Latência de ponta a ponta tem piso ~2s no pior caso de timing do poll.
- Single-worker é premissa de ordenação; escalar horizontalmente exige redesenho (documentar como limitação).
- Polling contínuo gera carga periódica no MySQL (mitigada por índice e lotes pequenos).
