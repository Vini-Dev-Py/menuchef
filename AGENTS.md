# Menu Chef — guia para agentes

O Menu Chef é um SaaS brasileiro de cardápio digital e fila de pedidos para restaurantes com várias empresas e filiais. O cliente pede no navegador, sem app. A cozinha opera um painel PWA. O modelo comercial é **sem comissão, apenas mensalidade**.

A fonte de verdade da implementação é `specs/`. Cada spec é autocontida. `docs/pesquisa/` é a pesquisa de mercado e de visual que originou as decisões; não substitui a spec.

## Mapa do monorepo

| Caminho | Papel |
| :--- | :--- |
| `apps/api` | Backend Go 1.27 (ainda não existe no repositório). Modular monolith, Clean Architecture, `chi` + Huma v2, `pgx`, `sqlc`, `go-jet`, `goose`. |
| `apps/landing` | Site institucional em TanStack Start (SSG). Domínio `menuchef.com.br`. |
| `apps/storefront` | Cardápio do cliente. Domínio `pedir.menuchef.com.br/{filial}`. |
| `apps/admin` | Painel PWA da empresa, da filial e da plataforma. Domínio `app.menuchef.com.br`. |
| `specs/` | Contratos de produto, arquitetura, infra, UX, testes e de cada app. |
| `docs/pesquisa/` | Pesquisa de concorrentes, paleta, logo e protótipos. Histórico, não contrato. |
| `infra/` | Docker Compose, Traefik e middlewares. Previsto; ainda não está no repositório. |
| `.cursor/skills/` | Skills deste repositório. `sites-cinematograficos` é para protótipo visual de estudo. A landing do produto segue `specs/apps/landing/SPEC.md`. |

Índice das specs: `specs/README.md`.

## Comandos

Nenhum app foi gerado ainda. Os comandos abaixo são os **previstos** pelas specs. Não invente um atalho paralelo.

Subir o ambiente (previsto):

```bash
docker compose -f infra/docker-compose.base.yml -f infra/docker-compose.dev.yml up -d
```

Testes (previstos):

```bash
cd apps/api && go test -race ./...
cd apps/api && go test -tags=integration ./...
pnpm exec vitest run
pnpm exec cypress run
```

O Cypress roda em `tests/e2e` contra `infra/docker-compose.test.yml`.

Lint (previsto):

```bash
golangci-lint run
pnpm exec tsc --noEmit
```

A spec de qualidade aceita ESLint ou Biome nos frontends. Use o que o app tiver quando existir.

Codegen (previsto):

```bash
cd apps/api && sqlc generate
cd apps/api && goose -dir migrations postgres "$DATABASE_URL" up
go run ./cmd/api openapi > openapi.json
pnpm exec openapi-typescript openapi.json -o src/shared/api/schema.d.ts
```

Saúde prevista: `GET /api/v1/health/live` e `GET /api/v1/health/ready` (Postgres, a instância Redis e Garage).

## Convenções

- **Clean Architecture na API.** `domain` e `application` não importam `chi`, `huma`, `pgx`, `redis` nem `net/http`. HTTP fica em `presentation`. SQL fica em `infrastructure`.
- **Dinheiro em centavos inteiros** (`int64` / `bigint`). Sem `float`.
- **Chave primária `UUIDv7`.** Token público, convite e sessão não usam UUIDv7. O número da comanda é `display_number` por filial e por `business_date`.
- **Escopo de tenant obrigatório.** Query de tabela da filial ou da empresa leva predicado `branch_id` ou `company_id`. Exceção só na allowlist `apps/api/queries/tenant_scope_allowlist.txt`, com motivo. RLS do Postgres fica fora do MVP.
- **Vínculo único.** `memberships` usa `UNIQUE NULLS NOT DISTINCT (user_id, company_id, branch_id)`. `company_owner` tem `branch_id` nulo. Os outros papéis exigem filial.
- **Fuso da filial.** `branches.timezone` é IANA (`America/Sao_Paulo` por default, `America/Manaus` é caso de teste). Horário de funcionamento, "aberto agora", dia do dashboard e horário mostrado ao cliente usam esse fuso. Instante no banco é `timestamptz` em UTC. O processo roda com `TZ=UTC`. Não use `time.Local`.
- **Sem comissão.** Não existe taxa da plataforma por pedido nem por pagamento. A mensalidade é a receita do Menu Chef. Taxa de gateway real, quando existir, é do provedor.
- **Redis único.** Sessão (`mc:sess:`, TTL 24h), cache (`mc:cache:`, TTL obrigatório), rate-limit (`mc:rl:`) e Pub/Sub (`mc:ps:`) na mesma instância. Política `volatile-ttl`, AOF `everysec`, sem RDB.
- **Cupom por cliente (23/11/2026).** `max_uses_per_customer` só vale para conta logada, identificada por `customer_user_id`. Convidado recebe `422 COUPON_REQUIRES_ACCOUNT` e a sacola fica. Cupom sem esse limite continua para convidado. Não há hash de telefone.
- **Git.** Integração em `develop`. Branches `docs/`, `feat/` e `fix/`. Commit em português, no imperativo, dizendo o porquê. Sem rebase e sem force push, salvo pedido explícito do dono. PR aponta para `develop`.

## O que o agente não faz

- Não trata `docs/pesquisa/` como especificação de implementação.
- Não deixa duas specs discordantes. Se a regra aparece em mais de um arquivo, atualiza todos, inclusive o índice em `specs/README.md` quando o resumo mudaria.
- Não cria `platform_fee_cents`, comissão por pedido ou segunda instância Redis.
- Não liga `ROW LEVEL SECURITY` neste ciclo.
- Não identifica cliente de cupom por hash de telefone, nem implementa verificação por WhatsApp no MVP (isso é melhoria futura, pela API oficial do WhatsApp Business).
- Não grava token ou sessão em `localStorage` / `sessionStorage`.
- Não edita `.cursor/skills/sites-cinematograficos` salvo pedido explícito.
- Não inventa endpoint, tabela ou marco que a spec não tenha. Se a spec não cobre o caso, pare e registre a lacuna em vez de escolher no escuro.

## Como usar as specs

1. Leia a spec do app que vai mudar e a spec geral do assunto (arquitetura, infra, testes ou UX).
2. Implemente o critério de aceite que já está escrito. Os números de teste (Manaus às `2026-11-23T01:30:00Z`, 50 goroutines no cupom, `23505` no owner duplicado) não são exemplo solto: são o contrato.
3. Skills em `.cursor/skills/` detalham o como. A spec ganha se as duas divergirem.
