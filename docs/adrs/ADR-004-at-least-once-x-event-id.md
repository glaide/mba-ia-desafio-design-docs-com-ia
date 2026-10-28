# ADR-004: Garantia at-least-once com X-Event-Id

- **Status:** Aceito
- **Data:** 2025-10-23 (reunião técnica)
- **Decisores:** Diego (Platform), Larissa (Tech Lead), Marcos (PM)

## Contexto

Retries, timeouts e falhas de rede podem fazer o worker reenviar o mesmo evento. Garantir exactly-once ponta a ponta exigiria coordenação complexa entre plataforma e cada cliente. O mercado (Stripe, GitHub) resolve isso com entrega at-least-once e um identificador estável para deduplicação no receptor.

## Decisão

1. A plataforma garante entrega **at-least-once** (nunca “no máximo uma vez” como contrato).
2. Cada evento recebe um **UUID** estável no momento em que entra no outbox.
3. Esse UUID é enviado no header **`X-Event-Id`** (e também no campo `event_id` do payload JSON).
4. Documentação para clientes: tratar `X-Event-Id` como chave de deduplicação idempotente.
5. Exactly-once fica explicitamente fora do escopo.

## Alternativas Consideradas

1. **Exactly-once delivery** — Descartada: complexidade alta (coordenação bilateral, store de acknowledgements distribuídos); desproporcional ao benefício (`[09:25]` Diego).
2. **Gerar novo id a cada tentativa de envio** — Descartada: quebraria deduplicação no cliente; o id deve nascer no insert do outbox.

## Consequências

**Positivas**

- Modelo simples de implementar no worker (retry seguro).
- Clientes conseguem deduplicar com store local de ids processados.
- Alinhado a padrões que os integradores já conhecem.

**Negativas / trade-offs**

- Clientes mal implementados podem processar o mesmo evento duas vezes (efeito de negócio duplicado).
- A plataforma não pode “prometer” uma única entrega; o contrato de produto precisa deixar isso explícito.
- Documentação e exemplos de dedupe no portal do desenvolvedor tornam-se obrigatórios (responsabilidade do PM).
