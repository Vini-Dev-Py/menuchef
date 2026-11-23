---
name: "migration-goose-sqlc-tenant"
description: "Cria migration goose e query sqlc com escopo de tenant. Use ao adicionar tabela, coluna, índice, constraint ou SQL de leitura/escrita da API do Menu Chef."
---

# Migration goose + query sqlc

Use ao mexer em `apps/api/migrations` ou `apps/api/queries`. A spec de schema está em `specs/apps/api/SPEC.md` (seção 4) e as regras de isolamento em `specs/geral/02-arquitetura-e-padroes.md` e `specs/geral/05-qualidade-testes-observabilidade.md`.

## Migration

- SQL puro no goose (`-- +goose Up` / `-- +goose Down`). Sem AutoMigrate.
- PK `UUID NOT NULL DEFAULT uuidv7()`. Instante é `timestamptz`. Data civil do pedido é `orders.business_date`, calculada no fuso da filial.
- Tabela da filial tem `branch_id`. Tabela da empresa tem `company_id`. Índice de listagem começa por essa coluna.
- `memberships`: `UNIQUE NULLS NOT DISTINCT (user_id, company_id, branch_id)` e `CHECK` que obriga `branch_id` nulo só em `company_owner`.
- `branches.timezone TEXT NOT NULL DEFAULT 'America/Sao_Paulo'`.
- `coupon_redemptions.customer_user_id UUID NULL` referenciando `users`. Sem `customer_identifier_hash`. Sem `UNIQUE (coupon_id, customer_user_id)`.
- `payments` não tem coluna de taxa da plataforma.
- Não habilite `ROW LEVEL SECURITY` nem crie policy. Isso fica fora do MVP.

## Query

- Um arquivo `.sql` por consulta, nome `-- name: VerboEntidade`.
- Predicado `branch_id` ou `company_id` em toda tabela multi-tenant (`orders`, `products`, `coupons`, `payments`, `memberships`, `branches` e as demais listadas na spec de testes).
- Allowlist só para query que a spec já excepcionou (slug público, login por e-mail, leads, lista de empresas da plataforma). Linha: `NomeSqlc<TAB>motivo`. Motivo vazio não passa.
- `ORDER BY` dinâmico não entra no SQL. O handler escolhe uma query da allowlist.

## Teste mínimo da migration

Suba Postgres 19 com Testcontainers, rode `goose up` e `goose down`, e cubra o caso que a constraint existe para pegar: segundo owner com `branch_id` nulo devolve `23505`; faixa de CEP sobreposta devolve `23P01`.
