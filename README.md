# Design Docs com IA — Sistema de Webhooks de Pedidos

## Sobre o desafio

Este repositório entrega o pacote de design docs da feature **Sistema de Webhooks de Notificação de Pedidos** para o OMS existente. A partir da reunião registrada em [`TRANSCRICAO.md`](./TRANSCRICAO.md) e do código Node/TypeScript + Prisma, a documentação descreve o *porquê* (PRD), a proposta técnica para revisão (RFC), as decisões fechadas (ADRs), o *como implementar* (FDD) e a rastreabilidade de cada item (Tracker).

Nenhuma alteração foi feita em `src/`, `prisma/` ou `tests/`: o código permanece como contexto e referência de integração. O enunciado original do desafio está no repositório base ([mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)); este README documenta o **processo de produção** da entrega.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Cursor (Composer / Agent)** | Leitura do repositório, geração e refinamento dos Markdowns, revisão contra checklist |
| **Agentes de exploração (Task/explore)** | Extração estruturada da transcrição (decisões, fora de escopo, open questions) e mapeamento dos pontos de integração no código OMS |
| **Plano Cursor (Plan mode)** | Quebra da entrega em ordem ADRs → RFC → FDD → PRD → Tracker → README antes de escrever |

## Workflow adotado

1. **Contextualização** — exploração paralela da `TRANSCRICAO.md` e dos módulos `orders`, `errors`, `auth`, `routes`, `logger`, `schema.prisma`.
2. **ADRs primeiro** — seis decisões principais formalizadas em MADR (`docs/adrs/`), incluindo uma que cita arquivos reais do código.
3. **RFC** — proposta concisa para revisão, com alternativas descartadas, questões em aberto e links aos ADRs.
4. **FDD** — especificação acionável (fluxos, contratos, erros `WEBHOOK_*`, resiliência, observabilidade, integração com paths reais).
5. **PRD** — consolidação de produto (problema, FRs, métrica &lt;10s, fora de escopo, riscos).
6. **Tracker** — tabela cruzando cada item a timestamp da call ou path de código.
7. **README do processo** — este arquivo, após o pacote estar completo.
8. **Revisão** — checklist de aceite do enunciado, item a item.

## Prompts customizados

### Prompt 1 — Extrair decisões e o que NÃO entra

```text
Leia TRANSCRICAO.md por completo. Extraia:
1) decisões fechadas (outbox MySQL, retry/DLQ, HMAC, at-least-once,
   worker polling, reuso de padrões) com timestamps [hh:mm] e falante;
2) requisitos funcionais e não funcionais;
3) itens explicitamente descartados ou adiados (fora de escopo);
4) questões em aberto;
5) alternativas consideradas e o trade-off do descarte.
Não invente nada que não esteja na transcrição. Se algo for ambíguo, marque como aberto.
```

### Prompt 2 — FDD ancorado no código existente

```text
Com base na transcrição e nos arquivos reais do OMS, escreva docs/FDD.md.
Obrigações:
- fluxos outbox → worker → retry → DLQ;
- ≥4 endpoints com request/response/status codes;
- matriz de erros com prefixo WEBHOOK_;
- seção "Integração com o sistema existente" citando ≥4 paths reais
  (ex.: order.service.ts changeStatus, app-error/http-errors,
  auth.middleware requireRole, routes/index.ts, schema.prisma);
- NÃO alterar código; NÃO inventar arquivos;
- NÃO repetir o nível de detalhe do RFC (RFC = proposta; FDD = implementação).
```

## Iterações e ajustes

1. **Primeira passagem da transcrição vs. “gerar PRD genérico”** — um pedido vago misturava itens adiados (e-mail, dashboard, rate limit) como requisitos. Correção: prompt explícito pedindo *fora de escopo* com timestamps e revisão manual removendo exatamente-once/Redis/sync HTTP do escopo positivo.
2. **RFC inchado com payloads** — a primeira versão do RFC copiava contratos HTTP que pertencem ao FDD. Ajuste: cortar exemplos longos do RFC, manter só visão de arquitetura + alternativas + open questions, e mover contratos para o FDD.
3. **Paths inventados** — checagem cruzada com o repo garantiu que a seção de integração cita apenas arquivos existentes (`order.service.ts`, `order.status.ts`, `auth.middleware.ts`, `routes/index.ts`, `app.ts`, `shared/errors/*`, `shared/logger`, `prisma/schema.prisma`).
4. **Tracker como rede de segurança** — ao montar o TRACKER, qualquer frase sem `[hh:mm] Nome` ou path real foi reescrita ou removida (ex.: métricas formais de SLI % que a call não definiu).

Ciclos principais até o pacote final: **4** (exploração → ADRs/RFC → FDD/PRD → Tracker/README + checklist).

## Como navegar a entrega

Ordem sugerida de leitura:

1. [`docs/PRD.md`](./docs/PRD.md) — problema, escopo, FRs, métricas  
2. [`docs/RFC.md`](./docs/RFC.md) — proposta técnica e o que ainda está aberto  
3. [`docs/adrs/`](./docs/adrs/README.md) — decisões pontuais  
4. [`docs/FDD.md`](./docs/FDD.md) — como implementar  
5. [`docs/TRACKER.md`](./docs/TRACKER.md) — de onde veio cada item  
6. [`TRANSCRICAO.md`](./TRANSCRICAO.md) — fonte primária da reunião  

Estrutura:

```text
.
├── README.md                 ← processo (este arquivo)
├── TRANSCRICAO.md            ← não alterar
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/
│       ├── ADR-001-outbox-no-mysql.md
│       ├── ADR-002-retry-backoff-e-dlq.md
│       ├── ADR-003-hmac-sha256-por-endpoint.md
│       ├── ADR-004-at-least-once-x-event-id.md
│       ├── ADR-005-worker-polling-processo-separado.md
│       └── ADR-006-reuso-padroes-do-projeto.md
├── src/                      ← referência; não alterado
├── prisma/                   ← referência; não alterado
└── tests/                    ← referência; não alterado
```
