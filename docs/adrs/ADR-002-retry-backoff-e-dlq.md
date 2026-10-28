# ADR-002: Retry com backoff exponencial e DLQ

- **Status:** Aceito
- **Data:** 2025-10-23 (reunião técnica)
- **Decisores:** Larissa (Tech Lead), Diego (Platform), Bruno (Orders)

## Contexto

Endpoints de clientes B2B podem ficar indisponíveis por manutenção (já houve janelas de ~2h). Sem retry, notificações se perdem. Com retry infinito, eventos de clientes que sumiram ficam eternamente “pendentes” e poluem o outbox. É necessário um limite claro, janela útil de recuperação e um destino para falhas permanentes com capacidade de reprocessamento manual.

## Decisão

1. Até **5 tentativas** de entrega HTTP por evento.
2. Backoff exponencial entre tentativas: **1m / 5m / 30m / 2h / 12h** (janela total ~15h após a primeira falha).
3. Após esgotar as tentativas, mover o evento para a tabela separada **`webhook_dead_letter`** (payload, motivo da falha, timestamp).
4. Expor **`POST /admin/webhooks/dead-letter/:id/replay`** (role `ADMIN`) para reenfileirar como `pending`, com auditoria de quem reprocessou.

## Alternativas Consideradas

1. **Retry indefinido** — Descartada: eventos ficam presos para sempre se o cliente desaparecer (`[09:15]` Diego).
2. **Apenas 3 tentativas** — Descartada: janela ~30 min é agressiva demais frente a manutenções de ~2h já observadas (`[09:16]` Bruno/Diego).
3. **Marcar outbox como `failed` sem tabela DLQ** — Descartada em favor de tabela separada para facilitar debug e reprocessamento (`[09:17]`–`[09:18]` Diego).

## Consequências

**Positivas**

- Cobre janelas de manutenção típicas dos clientes sem intervenção manual imediata.
- DLQ separada deixa falhas permanentes inspecionáveis e reprocessáveis.
- Limite de 5 tentativas evita crescimento infinito de retries.

**Negativas / trade-offs**

- Entrega pode ocorrer até ~15h após a falha inicial (não é “só poucos minutos”).
- Replay exige operação ADMIN e processo operacional documentado.
- Durante backoff, o cliente pode receber eventos fora da ordem temporal absoluta entre pedidos distintos (ordenação garantida apenas por `order_id` com worker único).
