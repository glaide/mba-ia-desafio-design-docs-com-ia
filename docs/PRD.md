# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O Order Management System (OMS) em produção gerencia pedidos B2B com máquina de estados, estoque e auditoria de status, mas **não notifica** sistemas externos. Clientes integradores (Atlas, MaxDistribuição, Nova Cargo) descobrem mudanças via polling em `GET /orders`, o que é lento, custoso e gera pressão comercial até o fim do trimestre.

Esta feature entrega **webhooks HTTP outbound**: a plataforma avisa o cliente quando o status do pedido muda, com configuração por endpoint, assinatura HMAC, retries e histórico de entregas. A decisão técnica foi tomada em reunião (ver `TRANSCRICAO.md`); este PRD fixa o *porquê* e o *quê* em nível de produto.

## Problema e motivação

- **Problema:** clientes não recebem push de mudança de status; polling gera atraso e carga.
- **Motivação comercial:** Atlas quer a capacidade até o fim de novembro; atraso aumenta risco de migração para concorrente.
- **Motivação operacional:** liberar o OMS de ser “consultado o tempo todo” e oferecer contrato estável de integração.

## Público-alvo e cenários de uso

| Público | Cenário |
| --- | --- |
| Integradores B2B (Atlas, MaxDistribuição, Nova Cargo) | Recebem POST no ERP/WMS quando pedido vai para `SHIPPED` / `DELIVERED` (ou subset configurado) |
| Time interno de operações / ADMIN | Reprocessa falhas permanentes via replay da DLQ |
| Engenheiros da plataforma | Mantêm worker, outbox e secrets sem dashboard dedicado nesta fase |
| PM / Developer Portal | Publica guia de integração (HMAC, dedupe `X-Event-Id`) |

## Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Percepção de tempo real | Latência entre commit do status e 1ª tentativa HTTP bem-sucedida (cliente saudável) | **&lt; 10 segundos** |
| Cobertura de manutenção típica | Janela de retry antes da DLQ | ~**15 horas** (5 tentativas, backoff acordado) |
| Adoção piloto | Clientes B2B com ao menos 1 webhook ativo | Atlas + demais citados na reunião assim que a API estiver estável |
| Entrega no trimestre | Feature utilizável em produção | Alvo fim de novembro (a confirmar com Atlas) |

## Escopo

### Incluso

- Webhooks **somente outbound** (plataforma → cliente).
- Notificação em mudança de status de pedido, com filtro por lista de status por endpoint.
- CRUD de configuração (URL HTTPS, statuses, active); secret gerado pela plataforma.
- Rotação de secret com grace period de 24h.
- Entrega at-least-once com `X-Event-Id` para dedupe no cliente.
- Histórico recente de deliveries (~100).
- DLQ + replay por usuário ADMIN.
- Documentação de integração no portal (responsabilidade do PM).

### Fora de escopo

1. **E-mail** quando o webhook falha repetidamente — adiado para fase futura (`[09:37]`–`[09:38]`).
2. **Dashboard visual** para o cliente — nesta fase apenas API; frontend é projeto separado (`[09:39]`–`[09:40]`).
3. **Rate limiting outbound** — observar primeiro; decidir depois (`[09:38]`–`[09:39]`).
4. **Arquivamento automático** de linhas delivered do outbox (~30 dias) — fora desta feature (`[09:08]`).
5. **Payload com items do pedido** — cliente consulta `GET /orders/:id` (`[09:43]`).
6. **Exactly-once** e **HTTP síncrono** / **Redis** como transport — descartados tecnicamente (ver RFC/ADRs).

## Requisitos funcionais

| ID | Requisito |
| --- | --- |
| FR-01 | Sistema deve enviar webhooks outbound quando o status de um pedido muda e existir endpoint ativo inscrito naquele status. |
| FR-02 | Cliente deve registrar webhook autenticado via JWT, informando URL HTTPS, `customerId` (body/path, não derivado do JWT) e lista de statuses; a plataforma gera e devolve o secret. |
| FR-03 | Cliente deve poder listar, atualizar e remover/desativar webhooks autenticado via JWT. |
| FR-04 | Sistema deve filtrar eventos no momento do enqueue (só grava outbox se houver inscrição). |
| FR-05 | Sistema deve expor histórico das últimas ~100 deliveries (sucesso/falha, payload, resposta, latência). |
| FR-06 | Sistema deve assinar o body com HMAC-SHA256 e enviar `X-Signature`; secret único por endpoint. |
| FR-07 | Sistema deve permitir rotação de secret com overlap de 24h. |
| FR-08 | Sistema deve garantir at-least-once e enviar `X-Event-Id` estável para deduplicação. |
| FR-09 | Após 5 falhas com backoff, o evento vai para DLQ; ADMIN pode reprocessar via API. |
| FR-10 | Latência alvo de notificação &lt; 10s sob condições normais (poll 2s). |
| FR-11 | Replay de DLQ deve ser auditado (quem reprocessou). |
| FR-12 | Payload do evento `order.status_changed` inclui ids, statuses, `order_number`, `total_cents`, sem items. |

## Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| NFR-01 | Consistência: evento de outbox e mudança de status na mesma transação SQL. |
| NFR-02 | Segurança: HTTPS-only; HMAC; secret por endpoint; role ADMIN no replay. |
| NFR-03 | Resiliência: timeout HTTP 10s; 5 retries; backoff 1m/5m/30m/2h/12h. |
| NFR-04 | Payload máximo 64KB; acima disso erro (sem truncar). |
| NFR-05 | Observabilidade: logs estruturados; métricas de sucesso/falha/lag; correlation por `event_id`. |
| NFR-06 | Ordenação por `order_id` sob premissa de worker único (limitação conhecida). |

## Decisões e trade-offs principais

- **Outbox MySQL vs Redis/sync HTTP:** consistência e simplicidade operacional &gt; latência sub-segundo.
- **At-least-once vs exactly-once:** simplicidade e padrão de mercado &gt; garantia absoluta de uma entrega.
- **API-only vs dashboard:** entregar valor de integração antes de UX visual.
- **Retry longo (até ~15h) vs poucas tentativas:** cobre manutenção de ~2h dos clientes.

Detalhes em [RFC](./RFC.md) e [ADRs](./adrs/README.md).

## Dependências

- OMS atual (auth JWT, pedidos, customers, Prisma/MySQL).
- Disponibilidade de processo worker em deploy (`npm run worker`).
- Portal do desenvolvedor / docs de integração (Marcos).
- Review de segurança (Sofia) no caminho HMAC/secrets (≥2 dias úteis).
- Confirmação do deadline Atlas (Marcos).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente não implementa dedupe e duplica efeitos | Média | Alto | Docs claros + exemplos com `X-Event-Id`; reforço no onboarding Atlas |
| Deadline fim de novembro não cabe em ~3 sprints | Média | Alto | Confirmar com Atlas; priorizar núcleo (outbox/HMAC/CRUD) sobre polish |
| Endpoint do cliente fora por &gt;15h | Baixa | Médio | DLQ + replay ADMIN; fase futura de alerta por e-mail |
| Vazamento de secret | Baixa | Alto | Secret por endpoint, rotação, HTTPS, redaction em logs |
| Lag do outbox sob pico sem rate limit | Média | Médio | Métricas de lag; rate limit como follow-up se necessário |

## Critérios de aceitação

1. Cliente registra webhook HTTPS e recebe secret uma vez.
2. Ao mudar status inscrito, o cliente recebe POST assinado em &lt;10s (ambiente saudável).
3. Status não inscrito **não** gera entrega.
4. Falha temporária do cliente é retentada conforme backoff; após 5 falhas aparece na DLQ.
5. ADMIN reprocessa item da DLQ; operador comum recebe 403.
6. Rotação de secret mantém aceitação dual por 24h.
7. Documentação de integração publicada cobrindo HMAC e dedupe.
8. Itens explicitamente fora de escopo (e-mail, dashboard) **não** estão no MVP.

## Estratégia de testes e validação

- **Unitários:** filtro de statuses, validação HTTPS, scheduling de backoff, montagem de assinatura HMAC.
- **Integração:** `changeStatus` + outbox na mesma txn (commit e rollback); worker delivery success/fail; replay ADMIN.
- **Contrato:** payloads/headers conforme FDD; códigos `WEBHOOK_*`.
- **Segurança:** Sofia valida rotação, storage de secret e ausência de secret em GETs/logs.
- **Aceite de produto:** piloto com Atlas em staging medindo latência p50/p95 vs meta de 10s.
- **Não-objetivos de teste nesta fase:** carga multi-worker, rate limit, UI.
