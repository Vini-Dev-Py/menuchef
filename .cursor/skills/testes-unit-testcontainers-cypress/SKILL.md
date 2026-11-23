---
name: "testes-unit-testcontainers-cypress"
description: "Escreve testes do Menu Chef: unitário Go ou Vitest, integração com Testcontainers e E2E Cypress. Use ao cobrir use case, query, fuso, cupom, tenant, Redis ou fluxo de tela."
---

# Testes (unitário, Testcontainers, Cypress)

Use ao adicionar ou corrigir teste. O catálogo está em `specs/geral/05-qualidade-testes-observabilidade.md` e na seção de aceite da spec do app. Nomeie pelo comportamento (`TestCreateOrder_RejectsWhenCepOutsideBranchRange`). Estrutura AAA. Relógio injetável. Sem `time.Sleep` para esperar regra de negócio.

## Unitário

- Go: `go test -race ./...` em `domain` e `application`, com ports falsos. Meta de cobertura 80% nessas duas camadas.
- Front: Vitest + Testing Library + MSW. Conta em centavos, Zod e o texto visível do cupom e do horário.
- O gate `TestSQLCQueries_RequireTenantPredicate` lê os `.sql` do sqlc e falha se tabela multi-tenant não tiver predicado `branch_id` ou `company_id` fora da allowlist. Roda sem banco.

## Integração (Testcontainers)

Postgres 19, **uma** instância Redis e Garage. Casos que não podem faltar quando a mudança os toca:

- Segundo `company_owner` com `branch_id` NULL → `23505`.
- Duas empresas: UUID da empresa B na sessão da A não devolve corpo da B. Id inexistente e id da outra empresa, na rota já escopada, devolvem o mesmo 404.
- Grade 10:00–22:00. Em `2026-11-23T01:30:00Z`, Manaus está aberta e São Paulo fechada. Pedido em `2026-11-23T03:30:00Z` cai no dia 22 em Manaus e no dia 23 em São Paulo.
- 50 goroutines num cupom com `max_uses = 5` → 5 sucessos e 45 conflitos.
- Cupom com `max_uses_per_customer = 1`: convidado recebe `422 COUPON_REQUIRES_ACCOUNT` sem resgate; o segundo uso da mesma conta recebe `409 COUPON_CUSTOMER_LIMIT_REACHED`; cupom sem esse limite fecha para convidado.
- Cache sem TTL é recusado. Com `maxmemory` baixo, `volatile-ttl` tira chave `mc:cache:` curta e deixa `mc:sess:` de 24h.
- Idempotência: replay 201, em voo 409, payload diferente 422.

## Cypress

Em `tests/e2e`, viewports 375, 768 e 1440. Seletor por `data-testid` ou role. Seed só com `APP_ENV=test`. Cubra o fluxo da spec do app, não um happy path genérico. Falha grava screenshot.
