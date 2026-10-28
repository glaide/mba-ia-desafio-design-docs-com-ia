# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint

- **Status:** Aceito
- **Data:** 2025-10-23 (reunião técnica)
- **Decisores:** Sofia (Security), Larissa (Tech Lead), Diego (Platform)

## Contexto

Webhooks outbound enviam dados de pedidos para URLs controladas pelos clientes. O receptor precisa autenticar que a requisição veio da plataforma e não de um terceiro. Um segredo global compartilhado por todos os clientes cria risco sistêmico: vazamento de um secret compromete todos os endpoints. URLs sem TLS exporiam payloads e assinaturas em trânsito.

## Decisão

1. Assinar o **corpo HTTP** com **HMAC-SHA256** usando um **secret único por endpoint** de webhook.
2. Enviar a assinatura no header **`X-Signature`**.
3. Permitir **rotação de secret** pela API, com **grace period de 24h** em que secret antigo e novo são válidos em paralelo; depois disso o antigo deixa de valer.
4. Aceitar apenas URLs **HTTPS** na criação/atualização (validação Zod rejeita `http://`).
5. O secret é **gerado pela plataforma** no `POST` de criação e retornado uma vez ao cliente (não escolhido pelo cliente).

## Alternativas Consideradas

1. **Secret global da plataforma** — Descartada: um leak compromete todos os clientes (`[09:21]` Sofia).
2. **Autenticação mutual TLS / OAuth no receptor** — Não adotada nesta fase; HMAC é padrão de mercado (Stripe/GitHub) e cabe no prazo (`[09:20]` Sofia).
3. **Permitir HTTP em ambientes internos** — Descartada: regra única HTTPS-only (`[09:23]` Sofia).

## Consequências

**Positivas**

- Blast radius de vazamento limitado a um endpoint.
- Rotação sem downtime de verificação no cliente (24h de overlap).
- Alinhamento com práticas conhecidas dos integradores B2B.

**Negativas / trade-offs**

- Cliente precisa implementar verificação HMAC e gerenciar rotação.
- Durante o grace period, dois secrets válidos aumentam ligeiramente a superfície de ataque temporal.
- Secret retornado no create precisa de cuidado no client (não logar; tratar como credencial).
