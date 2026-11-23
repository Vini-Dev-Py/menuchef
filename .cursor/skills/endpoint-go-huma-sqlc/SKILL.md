---
name: "endpoint-go-huma-sqlc"
description: "Cria endpoint Go no apps/api com Huma v2, sqlc e Clean Architecture. Use ao adicionar rota, handler, command, query ou DTO da API do Menu Chef."
---

# Endpoint Go (Huma + sqlc)

Use quando for criar ou alterar um endpoint em `apps/api`. Leia `specs/apps/api/SPEC.md` e `specs/geral/02-arquitetura-e-padroes.md` antes de escrever código.

## Onde cada peça mora

```text
internal/modules/<modulo>/
  domain/                         # entidade, value object, erro, interface do repositório
  application/commands|queries/   # um use case por arquivo
  application/dto/
  infrastructure/postgres/        # sqlc / go-jet
  presentation/http/              # struct do Huma, mapeamento HTTP
```

`domain` e `application` não importam `chi`, `huma`, `pgx`, `redis` nem `net/http`.

## Passos

1. Nomeie o use case pelo ato (`ConfirmOrder`, `GetBranchMenu`). Sem service com dezenas de métodos.
2. Escreva a query em `apps/api/queries/`, com bind `$1`. Tabela de filial ou empresa leva `branch_id` ou `company_id` no `WHERE` ou no `JOIN`. Se a query for entrada pública por slug ou tela da plataforma, registre o nome e o motivo em `queries/tenant_scope_allowlist.txt`.
3. Rode `sqlc generate` (previsto). Página 1 e página com cursor são queries separadas. Sem `OFFSET` e sem `WHERE $1 IS NULL OR ...`.
4. Declare input e output do Huma só em `presentation/http`. Mapeie para o DTO da aplicação. Erro de domínio vira RFC 9457 com `code` estável (`COUPON_REQUIRES_ACCOUNT`, `COUPON_CUSTOMER_LIMIT_REACHED`, `CEP_OUT_OF_DELIVERY_AREA`).
5. Rota mutável de pedido ou pagamento exige `Idempotency-Key` quando a spec da rota pedir. SSE fica no `chi` puro, fora do Huma. Não ligue `middleware.URLFormat`.
6. Dinheiro é `int64` em centavos. Id de entidade é UUIDv7 gerado no domínio. "Aberto agora" e dia de dashboard usam `branches.timezone`, com o processo em `TZ=UTC`.

## Não faça

- Não busque pedido, produto, cupom ou pagamento só por `id`.
- Não coloque taxa da plataforma no payload. O extrato leva valor, método e status.
- Não cacheie checkout, cupom, CEP de fechamento nem estado do pedido. Cache de leitura usa o prefixo `mc:cache:` e TTL. `UNLINK` só depois do `Commit`.
- Cupom com `max_uses_per_customer` exige `customer_user_id`. Convidado recebe 422 pedindo login. O lock é `(coupon_id, customer_user_id)`.
