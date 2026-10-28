# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Título** | Outbox + worker para webhooks outbound de mudança de status |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2025-10-23 |
| **Revisores** | Marcos (PM), Bruno (Orders), Diego (Platform), Sofia (Security) |

## Resumo executivo (TL;DR)

Proponho notificar clientes B2B sobre mudanças de status de pedido via **webhooks HTTP outbound**, usando **Transactional Outbox no MySQL** na mesma transação de `changeStatus`, e um **worker Node em processo separado** que faz polling a cada 2s, assina o corpo com **HMAC-SHA256** (secret por endpoint) e entrega com semântica **at-least-once** (`X-Event-Id`). Retries: 5 tentativas com backoff até DLQ + replay ADMIN. Detalhamento de contratos e fluxos fica no [FDD](./FDD.md); decisões pontuais nos [ADRs](./adrs/README.md).

## Contexto e problema

Clientes como Atlas, MaxDistribuição e Nova Cargo hoje fazem polling em `GET /orders`. Isso gera carga, atraso e atrito comercial (risco de churn até o fim do trimestre). O OMS já controla bem o ciclo de vida do pedido (máquina de estados, estoque, histórico), mas **não possui** eventos, filas nem notificações externas.

A reunião de 2025-10-23 fechou a direção técnica; este RFC consolida a proposta para revisão formal do time.

## Proposta técnica

### Visão geral

```
API (changeStatus)
  └─ txn MySQL: update order + history (+ stock) + INSERT webhook_outbox
                                                      │
Worker (processo separado, poll 2s) ◄─────────────────┘
  └─ HTTP POST → URL do cliente (HMAC, headers de evento)
       └─ sucesso → delivered | falha → retry/backoff → DLQ
```

1. **Configuração**: módulo `webhooks` com CRUD de endpoints por `customer_id` (URL HTTPS, lista de status inscritos, secret gerado pela plataforma, rotação com grace 24h).
2. **Produção do evento**: na transação existente de mudança de status, se houver webhook do customer inscrito naquele `toStatus`, inserir outbox com payload snapshot (`order.status_changed`).
3. **Consumo**: worker dedicado lê pendentes, envia HTTP (timeout 10s), registra deliveries, aplica política de retry e move para DLQ quando esgotar.
4. **Operação**: endpoint ADMIN de replay da DLQ; histórico recente de deliveries por webhook; logs Pino; sem dashboard visual nesta fase.

### Fronteiras desta proposta

- **Entra:** webhooks outbound de status; outbox MySQL; worker; HMAC; CRUD + deliveries + DLQ replay.
- **Não entra aqui (detalhe):** schemas Prisma campo a campo, payloads de exemplo e matriz completa de erros — ver [FDD](./FDD.md).
- **Fora de produto nesta fase:** e-mail em falhas, rate limit outbound, dashboard, arquivamento automático do outbox, multi-worker.

## Alternativas consideradas

### A1 — HTTP síncrono dentro do serviço de pedidos

Disparar o webhook no meio (ou ao fim) de `changeStatus`, aguardando resposta do cliente.

- **Trade-off que motivou o descarte:** a disponibilidade do endpoint externo passaria a condicionar a transação do pedido — timeout ou 5xx do cliente bloquearia outros fluxos ou forçaria rollback da mudança de status, o que é semanticamente errado (`[09:04]` Bruno/Larissa; `[09:06]` Diego).

### A2 — Redis Streams (ou “fila Redis”) como barramento

Publicar o evento em Redis e consumir com workers.

- **Trade-off que motivou o descarte:** introduz Redis Cluster e carga operacional nova para um time pequeno; o MySQL já está na txn crítica do pedido — outbox resolve consistência sem infra adicional (`[09:07]` Diego/Larissa).

### A3 — Exactly-once ponta a ponta (mencionado para contraste)

Coordenar ack dual-side para cada entrega.

- **Trade-off:** complexidade desproporcional; mercado resolve com at-least-once + id de dedupe (`[09:25]` Diego). Registrado em [ADR-004](./adrs/ADR-004-at-least-once-x-event-id.md).

## Questões em aberto

1. **Rate limiting outbound** — Se houver rajadas de mudanças de status, podemos inundar o cliente. Decisão da reunião: observar em produção e só então implementar limites (`[09:38]`–`[09:39]`). Precisamos de um critério de gatilho (ex.: erros 429 recorrentes) antes do próximo ciclo.
2. **Deadline comercial Atlas (fim de novembro)** — Marcos precisa confirmar com o cliente se a estimativa de ~3 sprints (incluindo review de segurança de Sofia ≥2 dias úteis) cabe na expectativa (`[09:45]`–`[09:47]`). Impacta priorização e escopo de endurecimento de auth no CRUD.

## Impacto e riscos

| Risco | Impacto | Mitigação |
| --- | --- | --- |
| Cliente processa evento duplicado | Dados/ações duplicadas no ERP do cliente | Contrato at-least-once + docs de dedupe via `X-Event-Id` |
| Crescimento da tabela outbox | Pressão de storage/queries no MySQL | Índice status+created_at; arquivamento ~30d em fase futura |
| Worker único vira gargalo | Atraso &gt;10s sob pico | Monitorar lag do outbox; multi-worker só com redesenho de ordenação |
| Secret vazado | Spoofing de webhooks daquele endpoint | Secret por endpoint + rotação 24h + HTTPS-only |
| Estimativa de 3 sprints estourar | Risco comercial Atlas | Confirmar deadline; cortar polish (não o núcleo outbox/HMAC) |

## Decisões relacionadas

- [ADR-001 — Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Retry, backoff e DLQ](./adrs/ADR-002-retry-backoff-e-dlq.md)
- [ADR-003 — HMAC-SHA256 por endpoint](./adrs/ADR-003-hmac-sha256-por-endpoint.md)
- [ADR-004 — At-least-once e X-Event-Id](./adrs/ADR-004-at-least-once-x-event-id.md)
- [ADR-005 — Worker polling processo separado](./adrs/ADR-005-worker-polling-processo-separado.md)
- [ADR-006 — Reuso dos padrões do projeto](./adrs/ADR-006-reuso-padroes-do-projeto.md)

## Próximos passos

1. Revisores comentarem este RFC (especialmente questões em aberto).
2. Engenheiros usarem o [FDD](./FDD.md) como especificação de implementação.
3. Sofia incluir review de segurança no caminho crítico do secret/HMAC.
