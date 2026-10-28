# ADR-001: Outbox no MySQL

- **Status:** Aceito
- **Data:** 2025-10-23 (reunião técnica)
- **Decisores:** Larissa (Tech Lead), Diego (Platform), Bruno (Orders)

## Contexto

O OMS precisa notificar clientes B2B quando o status de um pedido muda. A mudança de status já ocorre dentro de uma transação Prisma que atualiza o pedido, grava histórico e pode ajustar estoque. Incluir HTTP síncrono nessa transação acoplaria a disponibilidade do cliente externo à consistência do pedido: se o endpoint do cliente estiver fora, a mudança de status falharia ou ficaria bloqueada por segundos.

A equipe precisa de um mecanismo que garanta que o evento de notificação só exista se a mudança de status for commitada, e vice-versa, sem introduzir infraestrutura nova (Redis, broker) neste momento.

## Decisão

Adotar o padrão **Transactional Outbox** persistido no MySQL já usado pelo OMS:

1. Na mesma transação SQL que altera o status do pedido, inserir uma linha na tabela `webhook_outbox` com o payload do evento já materializado (snapshot).
2. Um worker separado lê registros `pending` e dispara o HTTP para o cliente.
3. Estados do outbox: `pending`, `processing`, `delivered`, `failed` (antes de ir para DLQ).
4. Índice por `status` + `created_at` para o worker buscar lotes pequenos de pendentes.

## Alternativas Consideradas

1. **HTTP síncrono dentro de `changeStatus`** — Descartada: client fora do ar bloquearia ou forçaria rollback indevido da mudança de status (`[09:04]` Bruno/Larissa; `[09:06]` Diego).
2. **Redis Streams / fila Redis** — Descartada: exigiria Redis Cluster e operação adicional; overengineering para o tamanho do time e do problema (`[09:07]` Diego/Larissa).
3. **Trigger MySQL para notificar o worker** — Descartada: MySQL não oferece LISTEN/NOTIFY nativo; solução ficaria artificial (`[09:09]` Diego).

## Consequências

**Positivas**

- Atomicidade: não há mudança de status sem evento enfileirado, nem evento órfão sem commit.
- Zero infra nova: reutiliza MySQL + Prisma já existentes.
- Operação familiar ao time de Orders/Platform.

**Negativas / trade-offs**

- Latência mínima ligada ao intervalo de polling do worker (não é push instantâneo).
- Crescimento da tabela outbox exige política de retenção futura (arquivamento ~30 dias ficou fora desta feature).
- Throughput do outbox compete com o mesmo banco operacional do OMS.
