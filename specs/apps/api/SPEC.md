# Spec de Aplicação — Backend API (`apps/api`)

> **Documento autocontido.** Especifica completamente o backend do Menu Chef: stack Go `1.27.4`, `chi` v5 + **`Huma v2`** (OpenAPI 3.1 + RFC 9457), arquitetura modular (Clean Architecture + CQRS), modelagem PostgreSQL `19.0` (com `uuidv7()` nativo, Keyset Pagination e Full-Text Search PT-BR `pt_unaccent + pg_trgm`), **uma instância Redis** (sessão, Pub/Sub, rate-limit e cache), idempotência (`Idempotency-Key`), concorrência atômica de cupons com estorno automático e limite por cliente só para conta logada, fuso IANA da filial, LGPD B2B2C, tempo real via SSE + Pub/Sub, pagamento mocado via Adapter (`MockPaymentAdapter`) e armazenamento WebP via Garage S3. Modelo comercial: sem comissão, apenas mensalidade. Não existe `platform_fee_cents`.

---

## 1. Responsabilidade e Escopo

A `apps/api` é o núcleo de regras de negócio, persistência e tempo real do Menu Chef. Serve simultaneamente (via roteamento same-origin `/api/v1/*` no Traefik v3.7) os 3 frontends:
1. **Landing (`menuchef.com.br/api/v1/*`):** captação de leads para onboarding assistido (com registro de aceite legal versionado).
2. **Storefront (`pedir.menuchef.com.br/api/v1/*`):** cardápio público da filial, busca FTS, cálculo de frete/cupom, checkout idempotente (guest ou cliente autenticado), simulação de pagamento mocado, stream SSE de acompanhamento e autoatendimento LGPD do cliente.
3. **Admin PWA (`app.menuchef.com.br/api/v1/*`):** autenticação multi-tenant, troca de contexto Empresa/Filial, gestão de cardápio/cupons/CEP/equipe, operação da fila Kanban via SSE, dashboard da filial e governança do Administrador da Plataforma (onboarding assistido, aceites legais e suporte auditado).

---

## 2. Stack Tecnológica & Justificativas (Novembro/2026)

| Componente | Tecnologia | Por que esta escolha (Trade-offs) |
| :--- | :--- | :--- |
| **Linguagem & Runtime** | **Go `1.27.4`** | Performance previsível, binário estático minúsculo, concorrência nativa para milhares de conexões SSE. Traz na stdlib `uuid.NewV7()`, `encoding/json/v2`, Green Tea GC, `http.CrossOriginProtection` e `goroutineleak` profiler |
| **HTTP Router & Contratos OpenAPI 3.1** | **`go-chi/chi/v5` + `Huma v2` (`humachi`)** | `chi` mantém compatibilidade total com `http.Handler` (rotas SSE ficam em `chi` puro); `Huma v2` gera **OpenAPI 3.1** + validação automática + **RFC 9457 Problem Details** diretamente dos structs de `presentation/http` (exportado em CI via `go run ./cmd/api openapi` sem subir banco, alimentando `openapi-typescript` / `openapi-fetch` nos 3 frontends). **Regra obrigatória:** nunca habilitar `middleware.URLFormat` do `chi` nas rotas do Huma |
| **Driver PostgreSQL** | **`jackc/pgx/v5` + `pgxpool`** | Protocolo binário nativo do Postgres (sem wrapper `database/sql`), suporte nativo a `UUID`, `JSONB`, `CopyFrom`, prepared statement cache e estatísticas de pool (`pool.Stat()`) |
| **Acesso a Dados (Estático — 85–90%)** | **`sqlc`** | Compila SQL puro validado contra as migrations em Go type-safe em build-time; zero reflection; latência equivalente a `pgx` puro; casamento ideal com **CQRS** e FTS Postgres |
| **Acesso a Dados (Dinâmico — 10–15%)** | **`go-jet/jet`** | SQL Builder com AST Go tipada gerada do schema para filtros dinâmicos de pedidos/relatórios (substitui `squirrel` e `sqlx`, em manutenção mínima), compartilhando o mesmo `*pgxpool.Pool` |
| **Migrations** | **`pressly/goose`** | SQL puro versionado (`up`/`down`), single source of truth para `sqlc`, nunca `AutoMigrate` |
| **Banco de Dados** | **PostgreSQL `19.0`** | Suporte até nov/2031; `uuidv7()` nativo (RFC 9562), `REPACK (CONCURRENTLY)`, AIO, B-tree skip scan e extensões `unaccent` + `pg_trgm` para busca PT-BR sem motor externo |
| **Redis único** | **`go-redis/v9` + `alexedwards/scs/v2` + `x/sync/singleflight`** | Uma instância: sessão (`mc:sess:`, TTL 24h), Pub/Sub (`mc:ps:`), rate-limit (`mc:rl:`) e cache (`mc:cache:`, TTL obrigatório, jitter ±15%). AOF `everysec`, `save ""`, `maxmemory 256mb`, `maxmemory-policy volatile-ttl`. Um `*redis.Client` só |
| **Object Storage (Mídia WebP)** | **Garage `v2.5` via AWS SDK Go v2** | Encapsulado atrás de `MediaStoragePort` (Adapter Pattern); MinIO CE foi arquivado em abr/2026 |
| **Senhas & Crypto** | `golang.org/x/crypto/argon2` + `crypto/rand` | Argon2id (OWASP) para senhas; `crypto/rand` (256 bits) para convites e `tracking_token` |
| **Logs & Observabilidade** | `log/slog` (stdlib) | Logs JSON estruturados com `request_id` e redaction obrigatória de PII/credenciais |

---

## 3. Estrutura de Módulos (Clean Architecture + CQRS)

```text
apps/api/
├── cmd/api/main.go                  # Composition Root (DI explícita, graceful shutdown, --healthcheck, subcomando `openapi`)
├── migrations/                      # SQL puro versionado (goose) — fonte de verdade do sqlc e go-jet
├── sqlc.yaml                        # Configuração sqlc apontando para migrations/ e queries/
└── internal/
    ├── platform/                    # Infra compartilhada: config, db (pgxpool + WithTx), redis (cliente único), httpserver (chi + huma), idempotency, logger, clock, errors
    └── modules/
        ├── lead/                    # Captação comercial da landing (onboarding assistido)
        ├── identity/                # Usuários globais, login/logout, sessões Redis (mc:sess:), convites, contas de cliente, LGPD (export/anonimização, legal_acceptances)
        ├── tenant/                  # Empresas, filiais, vínculos (memberships), RBAC, troca de contexto, suporte auditado
        ├── catalog/                 # Categorias, produtos, variações/adicionais, promoções, upload WebP, busca FTS
        ├── delivery/                # Horários de funcionamento, pausa da loja, frete fixo e faixas de CEP, cupons atômicos
        ├── order/                   # Ciclo de vida do pedido, máquina de estados, display_number, histórico/reorder, estorno de cupom
        ├── payment/                 # PaymentGatewayPort + MockPaymentAdapter (Pix/Cartão mocado) + extrato
        ├── realtime/                # Hub SSE em memória + backplane Pub/Sub (canais mc:ps:) + Web Push dispatcher
        └── analytics/               # Read models do dashboard operacional da filial
```

Cada módulo segue estritamente as 4 camadas: `domain/`, `application/{commands,queries,dto}/`, `infrastructure/`, `presentation/http/`. O `domain` e `application` são 100% livres de `chi`, `huma`, `pgx` e `redis` (verificado via `golangci-lint` `depguard`).

---

## 4. Modelagem de Dados, IDs & Full-Text Search (PostgreSQL 19.0)

### 4.1 Estratégia Híbrida de Identificadores
- **PK de todas as entidades (`id`):** `UUID NOT NULL PRIMARY KEY DEFAULT uuidv7()` (gerado no domínio Go via `uuid.NewV7()` para respeitar Clean Architecture/CQRS e manter append sequencial na borda direita da B-tree).
- **Número legível do pedido (`display_number`):** `INTEGER NOT NULL` sequencial por filial (`branch_order_counters` atualizado atomicamente na transação do pedido) — exibido como `#001`, `#002` na cozinha e no comprovante do cliente.
- **Tokens públicos / convites (`tracking_token`, `invite_token_hash`):** **nunca `UUIDv7`** (evita vazamento de timestamp). Gerados com `crypto/rand` (32 bytes = 256 bits).

### 4.2 Entidades Principais (Visão Relacional)
- **`leads`**: `id (uuidv7)`, `contact_name`, `company_name`, `phone`, `email`, `branch_count`, `notes`, `status (new|contacted|converted|archived)`, `created_at`.
- **`users`**: `id (uuidv7)`, `email (UNIQUE)`, `name`, `phone`, `password_hash (Argon2id)`, `is_platform_admin`, `tour_state (jsonb)`, `anonymized_at (timestamptz nullable)`, `created_at`.
- **`legal_acceptances`** (append-only, LGPD): `id (uuidv7)`, `actor_type (lead|user|customer)`, `actor_id (uuid)`, `document_type (terms_of_use|privacy_policy)`, `document_version (text)`, `accepted_at`, `ip_hash (char(64))`, `user_agent`.
- **`companies`**: `id (uuidv7)`, `name`, `slug (UNIQUE)`, `document`, `plan_name`, `plan_price_cents (bigint)`, `plan_terms_summary`, `status (active|paused|suspended)`, `status_reason`, `created_at`.
- **`branches`**: `id (uuidv7)`, `company_id (FK)`, `name`, `slug (UNIQUE)`, `phone`, `address_json`, `logo_url`, `primary_color_hex`, `is_paused`, `pause_reason`, `accepts_delivery`, `accepts_pickup`, `accepts_pay_on_delivery`, `shipping_mode (fixed|cep_range)`, `fixed_shipping_cents`, `min_order_cents`, `timezone (text NOT NULL DEFAULT 'America/Sao_Paulo')`, `opening_hours_json`.
- **`memberships`**: `id (uuidv7)`, `user_id (FK)`, `company_id (FK)`, `branch_id (FK nullable só para company_owner)`, `role (company_owner|branch_manager|kitchen_operator|dispatcher)`, `status (active|revoked)`, **`UNIQUE NULLS NOT DISTINCT (user_id, company_id, branch_id)`**, `CHECK ((role = 'company_owner' AND branch_id IS NULL) OR (role <> 'company_owner' AND branch_id IS NOT NULL))`. Revogação é `UPDATE` de `status` na mesma linha. Equivalente, se a ferramenta não emitir `NULLS NOT DISTINCT`: índice único parcial `(user_id, company_id) WHERE branch_id IS NULL` mais `(user_id, company_id, branch_id) WHERE branch_id IS NOT NULL`.
- **`invites`**: `id (uuidv7)`, `email`, `company_id`, `branch_id`, `role`, `token_hash (UNIQUE)`, `expires_at`, `accepted_at`.
- **`categories`**: `id (uuidv7)`, `branch_id (FK)`, `name`, `sort_order`, `is_active`.
- **`products`**: `id (uuidv7)`, `branch_id (FK)`, `category_id (FK)`, `name`, `description`, `price_cents (bigint)`, `promo_price_cents (bigint nullable)`, `image_url`, `is_available`, `sort_order`, `search_vector (tsvector)`, `search_text_trgm (text)`.
- **`option_groups`** & **`option_items`**: `id (uuidv7)`, `product_id (FK)`, `name`, `min_select`, `max_select`, `is_required` / `name`, `price_cents`, `is_available`.
- **`cep_shipping_ranges`**: `id (uuidv7)`, `branch_id (FK)`, `cep_start (char(8))`, `cep_end (char(8))`, `fee_cents (bigint)`, `estimated_minutes`, `is_active` — com constraint nativa **`EXCLUDE USING gist (branch_id WITH =, int4range(cep_start::int, cep_end::int, '[]') WITH &&) WHERE (is_active)`** (extensão `btree_gist`, mapeando `SQLSTATE 23P01` para `409 CEP_RANGE_OVERLAP`).
- **`coupons`**: `id (uuidv7)`, `branch_id (FK)`, `code`, `discount_type (fixed|percentage|free_shipping)`, `discount_value`, `min_order_cents`, `max_uses (int nullable)`, `max_uses_per_customer (int nullable)`, `used_count (int NOT NULL DEFAULT 0)`, `valid_from`, `valid_until`, `is_active`, `UNIQUE (branch_id, lower(code))`.
- **`coupon_redemptions`**: `id (uuidv7)`, `coupon_id (FK)`, `order_id (FK UNIQUE)`, `customer_user_id (FK users, nullable)`, `created_at`. Não há `customer_identifier_hash`. O id da conta é o identificador do limite por cliente (decisão de 23/11/2026). Fica nulo no resgate de convidado em cupom sem limite por cliente. `UNIQUE (coupon_id, order_id)` garante o estorno idempotente. Não há `UNIQUE (coupon_id, customer_user_id)`, porque o teto pode ser maior que 1.
- **`idempotency_keys`**: `scope_key (text)`, `idempotency_key (uuid)`, `request_hash (char(64))`, `status (in_progress|completed)`, `response_status (int nullable)`, `response_body (jsonb nullable)`, `locked_until (timestamptz)`, `expires_at (timestamptz)`, `PRIMARY KEY (scope_key, idempotency_key)`.
- **`orders`**: `id (uuidv7)`, `branch_id (FK)`, `customer_user_id (FK nullable)`, `display_number (int)`, `business_date (date NOT NULL)` (data civil de `created_at` no fuso da filial), `tracking_token (UNIQUE)`, `version (int NOT NULL DEFAULT 1)` (Optimistic Locking), `fulfillment_mode (delivery|pickup)`, `payment_timing (online|on_delivery)`, `payment_method (pix_mock|card_mock|cash|card_machine|pix_door)`, `change_for_cents (bigint nullable)`, `status`, `customer_name`, `customer_phone`, `delivery_address_json`, `subtotal_cents`, `shipping_cents`, `discount_cents`, `total_cents`, `coupon_id`, `rejection_or_cancel_reason`, `created_at`, `confirmed_at`, `completed_at`, `UNIQUE (branch_id, display_number, business_date)`.
- **`order_items`** & **`order_item_options`**: snapshot imutável do nome, preço unitário em centavos, quantidade, observação (`"sem cebola"`) e adicionais escolhidos no momento do fechamento (mudanças futuras no cardápio nunca alteram pedidos históricos).
- **`payments`**: `id (uuidv7)`, `order_id (FK)`, `branch_id (FK)`, `provider (mock)`, `provider_ref`, `method (pix|card)`, `amount_cents`, `status (pending|approved|refused|expired|refunded)`, `pix_copy_paste`, `expires_at`, `settled_at`. Sem `platform_fee_cents`. O Menu Chef não retém comissão sobre o pagamento. Quando houver gateway real, taxa informada pelo provedor pode ser persistida com nome explícito de taxa do provedor, nunca como receita da plataforma.
- **`support_access_audit_logs`** (append-only): `id (uuidv7)`, `platform_admin_user_id`, `company_id`, `branch_id`, `reason`, `accessed_at`, `ip_address`, `user_agent`.

### 4.3 Full-Text Search PT-BR (`pt_unaccent` + `pg_trgm`)
Configuração obrigatória em migration dedicada:

```sql
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TEXT SEARCH CONFIGURATION public.pt_unaccent (COPY = pg_catalog.portuguese);
ALTER TEXT SEARCH CONFIGURATION public.pt_unaccent
  ALTER MAPPING FOR hword, hword_part, word
  WITH unaccent, portuguese_stem;
```

- **Trigger `BEFORE INSERT OR UPDATE`** atualiza `search_vector` (pesos: `A` para `name`, `B` para nome da categoria, `C` para `description`) e `search_text_trgm` (`lower(unaccent(name))` — mantendo `pg_trgm` apenas em campos curtos para performance).
- **Índices obrigatórios:**
  - `CREATE INDEX idx_products_branch_fts ON products USING GIN (search_vector);`
  - `CREATE INDEX idx_products_branch_trgm ON products USING GIN (search_text_trgm gin_trgm_ops);`
  - `CREATE INDEX idx_products_branch_active ON products (branch_id, is_available, sort_order);`
- **Query híbrida segura:** filtra sempre por `branch_id = $1 AND is_available = true`, usa `websearch_to_tsquery('public.pt_unaccent', $2)` combinado com operador indexado `lower(unaccent($2)) <% search_text_trgm` (nunca `WHERE similarity(...) > 0.3`, que força seq scan) e ordena por score combinado (`ts_rank_cd * 2.0 + word_similarity`).

### 4.4 Paginação 100% Cursor Opaco + Keyset (`UUIDv7`)
Todos os endpoints de listagem da API (`GET /orders`, `GET /customer/orders`, `GET /payments/statement`, `GET /products`, `GET /coupons`, `GET /shipping-ranges`, `GET /leads`, `GET /companies`, `GET /support/audit-logs`, `GET /products/search`) seguem o mesmo contrato de **Cursor Opaco (Base64url) + Keyset Pagination** (zero `OFFSET` e zero `COUNT(*)`):
- **Parâmetros:** `?cursor=<base64url>&limit=<int>` (`default: 20`, `max: 100`).
- **Resposta padrão:** `{ "data": [...], "page_info": { "next_cursor": "...", "has_next_page": true, "page_size": 20 } }`.
- **Detecção de `has_next_page`:** toda query busca `LIMIT limit + 1`.
- **Listagens cronológicas puras:** paginam diretamente pela PK `UUIDv7` (`WHERE branch_id = $1 AND id < $2 ORDER BY id DESC LIMIT $3`), sem necessidade de índice composto com `created_at`.
- **Listagens com ordenação customizada (ex.: `total_cents`, `name`, `combined_score`):** usam `id` como tie-breaker determinístico com **direção uniforme** (`ORDER BY col DESC, id DESC` / `ASC, ASC`) e *row-value comparison* `(col, id) < ($2, $3)`, suportadas por índice composto `(branch_id, col DESC, id DESC)`.
- **Separação obrigatória de queries:** em `sqlc` e `go-jet`, a query da **Página 1** (sem cursor) é separada da query de **Páginas Seguintes** (com cursor) — proibindo `WHERE $1 IS NULL OR ...` para evitar *generic plan* ruim no PostgreSQL.
- **Segurança:** cursor inválido ou incompatível com `sort_by`/`sort_dir` retorna `400 Bad Request` (`INVALID_CURSOR`).

### 4.5 Fuso da filial

`branches.timezone` guarda nome IANA. Default `America/Sao_Paulo`. Exemplos válidos: `America/Sao_Paulo`, `America/Manaus`. O domínio rejeita offset (`-04:00`) e apelido (`Brasilia`) com `422`.

`opening_hours_json` é relógio de parede, sem offset. Faixa que cruza a meia-noite é avaliada no fuso da filial. "Aberto agora", abre/fecha, o dia do `GET /dashboard` e o horário de comprovante e comanda usam esse fuso. Pausa é imediata. Instante continua `timestamptz` UTC. A API roda com `TZ=UTC` e converte com `time.LoadLocation(branch.Timezone)` ou `AT TIME ZONE`. Nunca `time.Local`.

`orders.business_date = (created_at AT TIME ZONE branches.timezone)::date`.

O `GET .../menu` devolve `timezone`, `is_open` e o próximo horário já calculados. O storefront não decide sozinho se a loja está aberta.

Casos de teste com relógio fixo:
1. Grade 10:00–22:00. Instante `2026-11-23T01:30:00Z`: Manaus aberta (21:30 do dia 22); São Paulo fechada (22:30 do dia 22).
2. Pedido `created_at = 2026-11-23T03:30:00Z`: `business_date` Manaus `2026-11-22`, São Paulo `2026-11-23`. Dashboard do dia 23 inclui o pedido só em São Paulo.
3. Filial criada sem timezone grava `America/Sao_Paulo`.

---

## 5. Autenticação, RBAC, Redis único e Defesa contra SQL Injection

1. **Sessão Opaca no Redis (`alexedwards/scs`, prefixo `mc:sess:`, TTL absoluto de 24h):**
   - Cookie `__Host-mc_session` (`HttpOnly; Secure; SameSite=Lax; Path=/`). **Zero tokens no `localStorage`.**
   - Payload de sessão: `user_id`, `active_company_id`, `active_branch_id`, `auth_at`.
   - Toda chamada de login e troca de contexto (`POST /api/v1/auth/context`) executa `RenewToken` (rotação de Session ID).
2. **Proteção CSRF nativa (Go 1.27):** `http.NewCrossOriginProtection()` acoplado como middleware global em todas as rotas mutáveis (`POST`, `PUT`, `PATCH`, `DELETE`).
3. **Uma instância Redis para sessão, Pub/Sub, rate-limit e cache:**
   - Política `volatile-ttl` + AOF (`appendonly yes`, `appendfsync everysec`, `save ""`, `maxmemory 256mb`). Despeja primeiro quem expira antes. Cache e rate-limit (TTL curto) saem antes da sessão de 24h. `volatile-lru` não serve: no pico o cardápio é tão quente quanto a sessão. `allkeys-*` pode apagar sessão. Sessão sem TTL não seria limpa e, sozinha, faria o Redis recusar escrita.
   - Chaves: `mc:sess:{id}` (24h), `mc:cache:menu:branch:{id}:v1` (5 min ±15%), `mc:cache:branch:slug:{slug}:v1` (60s ±15%), `mc:cache:rbac:user:{userId}:v1` (2 min ±15%), `mc:rl:{rota}:{sujeito}` (TTL da janela). Canais: `mc:ps:branch:{branchId}:orders` e `mc:ps:order:{trackingToken}` (não são chave).
   - O helper de cache recusa `SET` sem TTL. `singleflight` cobre as leituras. Erro pontual de `GET` de cache cai no Postgres. Se o processo Redis inteiro cai, `/health/ready` falha e login, SSE e rate-limit param juntos.
   - Todo Command que altera cardápio, pausa, horário, fuso, vínculo ou suspensão de empresa dispara `UNLINK` **somente após `tx.Commit()`**.
   - **Nunca cachear invariantes de escrita:** checkout, CEP no fechamento, cupons e máquina de estados do pedido leem sempre do PostgreSQL em transação.
4. **Escopo de tenant obrigatório:** query `sqlc` de tabela multi-tenant traz predicado `branch_id` ou `company_id`. O teste `TestSQLCQueries_RequireTenantPredicate` quebra o CI fora da allowlist `apps/api/queries/tenant_scope_allowlist.txt` (nome, TAB, motivo). RLS do Postgres fica fora do MVP: sem `ENABLE ROW LEVEL SECURITY` e sem policy.
5. **SQL Injection Zero:**
   - 100% das queries em `sqlc` (bind parameters binários) ou `go-jet` (AST tipada).
   - `ORDER BY` dinâmico só aceita enums de allowlist explícita no handler.
   - Usuário PostgreSQL de runtime (`menuchef_app`) tem apenas `SELECT, INSERT, UPDATE, DELETE` (sem DDL).
   - Limite de payload (`http.MaxBytesReader`) e validação estrita via `Huma v2` em todos os endpoints.

---

## 6. Idempotência, Concorrência, Locks/Transactions, Pagamento Mocado & LGPD

### 6.1 Idempotência (`Idempotency-Key` — IETF draft-09)
Obrigatória em `POST /api/v1/public/branches/{branchSlug}/orders` e mutações de pagamento:
- Header `Idempotency-Key: <UUIDv4>` obrigatório (`400 MISSING_IDEMPOTENCY_KEY` se ausente/inválido).
- Persistida em `idempotency_keys` **na mesma transação PostgreSQL** da mutação (TTL 24h):
  - `absent` → cria registro `in_progress` com lease de 30s, processa pedido, grava `response_status`/`response_body` e marca `completed`.
  - `in_progress` (lease ativo) → retorna `409 Conflict` (`IDEMPOTENCY_IN_PROGRESS`) + `Retry-After: 2`.
  - `completed` + SHA-256 do payload idêntico → retorna replay exato (`201 Created`).
  - `completed` + SHA-256 divergente → retorna `422 Unprocessable Entity` (`IDEMPOTENCY_PAYLOAD_MISMATCH`).

### 6.2 Concorrência Atômica de Cupons & Estorno Automático
- No `CreateOrder`, o consumo global ocorre via **`UPDATE` condicional atômico** (`WHERE id = $1 AND branch_id = $2 AND is_active = true AND (max_uses IS NULL OR used_count < max_uses)`). Se zero linhas forem atualizadas → rollback + `409 Conflict` (`COUPON_UNAVAILABLE_OR_EXHAUSTED`).
- **Limite por cliente (decisão do dono em 23/11/2026, opção b):**
  - Vale só quando `max_uses_per_customer` não é nulo. O identificador é `customer_user_id`. Não existe hash de telefone nem coluna `customer_identifier_hash` — essa coluna identificaria convidado, e a opção (a) foi recusada.
  - Convidado recebe `422` (`COUPON_REQUIRES_ACCOUNT`), detalhe *"Este cupom tem limite por cliente. Entre na sua conta para usá-lo."* Não incrementa `used_count` e não grava resgate. O mesmo código sai de `POST .../coupons/validate` e de `POST .../orders`, para o storefront pedir login sem apagar a sacola.
  - Cliente logado: `pg_advisory_xact_lock` em `(LockNSCouponCustomer, hash(coupon_id, customer_user_id))`, conta resgates daquele par e, se já atingiu o teto, devolve `409` (`COUPON_CUSTOMER_LIMIT_REACHED`), detalhe *"Você já usou este cupom o número máximo de vezes."* O resgate grava `customer_user_id`.
  - Cupom com `max_uses_per_customer` nulo continua para convidado. O resgate de convidado deixa `customer_user_id` nulo; o de conta logada preenche o id, sem teto por pessoa.
  - Verificação de telefone por código, via WhatsApp Business API oficial, fica fora do MVP.
- **Estorno automático:** quando um pedido com cupom vai para `canceled_by_customer` (antes de `confirmed`), `rejected_by_store` ou expiração de Pix mocado, a transação executa `DELETE FROM coupon_redemptions WHERE order_id = $1 RETURNING coupon_id` e, somente se deletou, `UPDATE coupons SET used_count = GREATEST(used_count - 1, 0) WHERE id = $1`. O delete também devolve a vaga daquele `customer_user_id`. Exclusão de conta LGPD não estorna cupom.

### 6.3 Transactions, Locks & Advisory Locks (`pgxpool` + PostgreSQL 19)
- **Isolamento padrão:** `READ COMMITTED` com `pgx.BeginTxFunc` (rollback garantido).
- **Hierarquia obrigatória de timeouts em `ConnConfig.RuntimeParams`:** `lock_timeout = 2000` (2s, `55P03` ERROR) `<` `idle_in_transaction_session_timeout = 4000` (4s) `<` `statement_timeout = 5000` (5s, `57014` ERROR) `<` `transaction_timeout = 15000` (15s, `25P04` FATAL — rede de segurança) `<` HTTP request timeout (20s). Overrides locais usam **sempre `SET LOCAL`** (nunca `SET` puro no pool).
- **Optimistic Locking em `orders`:** toda transição de status executa `UPDATE orders SET status = $next, version = version + 1 WHERE id = $1 AND branch_id = $2 AND status = $expected AND version = $v RETURNING ...`. Zero linhas → `409 Conflict` (`INVALID_STATE_TRANSITION`).
- **Upsert de `branch_order_counters` (`display_number`):** executado como **último write** dentro da transação de `CreateOrder`, segurando o lock da linha da filial por < 1–2 ms.
- **Advisory Locks estritamente transaction-scoped (`pg_advisory_xact_lock` / `pg_try_advisory_xact_lock`):** **proibido `pg_advisory_lock` (session-level)** com `pgxpool`. Namespaces `int32` centralizados em `internal/platform/db/locks.go` para: (1) cupom por cliente logado (`hash(coupon_id, customer_user_id)`), (2) jobs cron singleton (`pg_try_advisory_xact_lock` na limpeza LGPD/retenção) e (3) reordenação em lote do cardápio por filial. Lotes de worker (expiração de Pix pendente) usam **`SELECT ... FOR UPDATE SKIP LOCKED`**.
- **Split-Transaction (ZERO I/O externo dentro de Tx):** pagamentos online rodam em **Tx 1** (`order=awaiting_payment`, `payment=pending` → `COMMIT`) → chamada externa ao `PaymentGatewayPort` fora da tx → **Tx 2** (persiste `provider_ref`/aprovação ou estorna cupom em recusa → `COMMIT`).

### 6.4 Pagamento Mocado no MVP (`PaymentGatewayPort`)
No MVP, **não há integração com gateway real**. Toda a lógica de pagamento online passa por:

```go
type PaymentGatewayPort interface {
    CreatePixCharge(ctx context.Context, req PixChargeRequest) (PixChargeResult, error)
    ChargeCard(ctx context.Context, req CardChargeRequest) (CardChargeResult, error)
    Refund(ctx context.Context, providerRef string, amountCents int64) error
}
```

- **Pix Mocado:** gera código EMV copia-e-cola fictício + QR payload + expiração de 15 min (`status: pending`). O endpoint `POST /api/v1/public/orders/{trackingToken}/mock-payment/simulate` (habilitado somente quando `PAYMENT_PROVIDER=mock`) dispara `approve`, `refuse` ou `expire` — acionando o mesmo UseCase de confirmação/expiração (incluindo estorno de cupom em `expire`/`refuse`) que um webhook real usará no futuro.
- **Cartão Mocado:** finais determinísticos (`0000` → `approved`, `0002` → `refused_insufficient_funds`, `0004` → `refused_generic`). Nunca persiste PAN/CVV; armazena apenas `brand` e `last4`.
- **Pagar na Entrega / Retirada:** não chama `PaymentGatewayPort` e envia o pedido direto para `received`. Não há coluna de taxa da plataforma. O extrato (`GET /payments/statement`) lista pedido, `amount_cents`, método e status. Não há linha de comissão do Menu Chef. Taxa de gateway real, no futuro, só aparece rotulada como taxa do provedor.

### 6.5 LGPD & Privacidade no MVP
- **Ledger de aceite:** `POST /public/leads`, `POST /auth/invites/accept` e `POST /auth/register-customer` exigem versão aceita de Termos/Privacidade e gravam registro imutável em `legal_acceptances`.
- **Exportação do titular:** `GET /api/v1/customer/privacy/export` retorna JSON com perfil, endereços, aceites legais e pedidos do cliente autenticado.
- **Exclusão / Anonimização:** `POST /api/v1/customer/privacy/delete-account` revoga todas as sessões do cliente no Redis (`UNLINK` em `mc:sess:`), remove endereços salvos e anonimiza PII em `users` e `orders` (`customer_name = "Cliente Anonimizado"`, telefone/e-mail substituídos por hash irreversível, `delivery_address_json = NULL`), **preservando intactos `order_items`, valores e métricas financeiras das filiais**. Não estorna cupom nem apaga `coupon_redemptions.customer_user_id`.
- **Jobs de retenção (coordenados por `pg_try_advisory_xact_lock`):** rotina periódica remove `idempotency_keys` expiradas (>24h), convites expirados (>7d) e leads inativos (>24 meses).

---

## 7. Tempo Real (SSE + Pub/Sub na instância Redis única)

- **Endpoints SSE (registrados em `chi` puro, fora do wrapper JSON do `Huma v2`):**
  - Fila da cozinha (autenticado + permissão `orders:read`): `GET /api/v1/admin/branches/{branchId}/orders/stream`
  - Acompanhamento do cliente (público via `tracking_token`): `GET /api/v1/public/orders/{trackingToken}/stream`
- **Arquitetura multi-réplica sem sticky sessions (Publicação pós-commit + Snapshot):**
  1. Command de pedido (`CreateOrder`, `UpdateOrderStatus`, `CancelOrder`) faz **`tx.Commit()` no PostgreSQL primeiro** e publica o evento JSON nos canais `mc:ps:branch:{branchId}:orders` e `mc:ps:order:{trackingToken}` **somente após o commit** (sem tabela extra de Outbox no MVP; reconciliação garantida pelo snapshot do banco).
  2. Cada réplica da API mantém um Hub em memória multiplexado por uma conexão Pub/Sub no cliente Redis único.
  3. Ao conectar ou reconectar (com `Last-Event-ID`), o handler envia um **snapshot inicial do banco** (`event: snapshot`) e depois transmite eventos incrementais (`order_created`, `order_status_changed`).
  4. Handler limpa o `WriteTimeout` com `http.NewResponseController(w).SetWriteDeadline(time.Time{})`, envia `X-Accel-Buffering: no`, `Cache-Control: no-cache, no-transform` e heartbeat `: ping\n\n` a cada 15s.

---

## 8. Contratos HTTP Principais (`/api/v1` — OpenAPI 3.1 via `Huma v2`)

> **Versionamento:** evolução aditiva dentro de `/api/v1`; mudanças incompatíveis exigem `/api/v2` ou ciclo `Deprecation`/`Sunset` (RFC 8594). A spec OpenAPI 3.1 é exportada em build/CI por `go run ./cmd/api openapi > openapi.json`.

### Público (Landing & Storefront)
- `GET /api/v1/health/live` · `GET /api/v1/health/ready`
- `POST /api/v1/public/leads` — registra contato de empresa interessada + `accepted_privacy_version` (rate-limited)
- `GET /api/v1/public/branches/{branchSlug}/menu` — dados da filial (incluindo `timezone` e `is_open` no fuso dela), status (aberta/fechada/pausada), categorias, produtos, adicionais e promoções (cache `mc:cache:`)
- `GET /api/v1/public/branches/{branchSlug}/products/search?q=...` — busca FTS PT-BR (`pt_unaccent` + `pg_trgm`)
- `POST /api/v1/public/branches/{branchSlug}/shipping/quote` — valida CEP e retorna taxa/prazo ou erro `CEP_OUT_OF_DELIVERY_AREA`
- `POST /api/v1/public/branches/{branchSlug}/coupons/validate` — valida cupom para subtotal/modo atual
- `POST /api/v1/public/branches/{branchSlug}/orders` — cria pedido (guest ou cliente logado) com header obrigatório `Idempotency-Key`
- `GET /api/v1/public/orders/{trackingToken}` — consulta detalhe/status do pedido
- `GET /api/v1/public/orders/{trackingToken}/stream` — stream SSE de status (`chi` puro)
- `POST /api/v1/public/orders/{trackingToken}/cancel` — cancelamento autônomo pelo cliente (permitido apenas em `awaiting_payment` ou `received`, com estorno automático de cupom)
- `POST /api/v1/public/orders/{trackingToken}/mock-payment/simulate` — simula aprovação/recusa/expiração no MVP mocado

### Identidade, Conta do Cliente & LGPD
- `POST /api/v1/auth/login` · `POST /api/v1/auth/logout` · `GET /api/v1/auth/me` (retorna usuário + lista de empresas/filiais/papéis)
- `POST /api/v1/auth/register-customer` — conta opcional do cliente final (exige `accepted_terms_version` e `accepted_privacy_version`)
- `POST /api/v1/auth/invites/accept` — aceite de convite enviado pelo Admin da Plataforma ou Dono da Empresa (exige versões aceitas de Termos/Privacidade)
- `POST /api/v1/auth/context` — troca `active_company_id`/`active_branch_id` validando vínculo e rotacionando Session ID
- `GET /api/v1/customer/orders` — histórico paginado por cursor (`UUIDv7`) do cliente autenticado
- `POST /api/v1/customer/orders/{orderId}/reorder-preview` — valida itens/preços atuais para "Pedir de novo"
- `GET /api/v1/customer/privacy/export` — exportação JSON dos dados pessoais e pedidos do cliente (LGPD)
- `POST /api/v1/customer/privacy/delete-account` — revogação de sessões + anonimização de PII preservando registros financeiros das filiais (LGPD)

### Painel da Empresa & Filial (`/api/v1/admin/branches/{branchId}/...`)
- **Cardápio:** CRUD de `categories`, `products`, `option-groups`, `option-items`, `PATCH /products/{id}/availability` (esgotado em 1 clique + `UNLINK` pós-commit), `POST /media/upload` (valida magic bytes → converte WebP → grava no Garage)
- **Operação & Entrega:** `PUT /schedule` (grade local + `timezone` IANA), `POST /pause` / `POST /resume`, CRUD `shipping-ranges` (CEP), CRUD `coupons` (`max_uses_per_customer` nulo = vale para convidado; preenchido = só conta logada)
- **Pedidos:** `GET /orders` (filtros dinâmicos via `go-jet` + Cursor Pagination), `GET /orders/stream` (SSE em `chi` puro), `POST /orders/{id}/confirm`, `POST /orders/{id}/reject` (estorna cupom), `POST /orders/{id}/start-production`, `POST /orders/{id}/mark-ready`, `POST /orders/{id}/dispatch`, `POST /orders/{id}/complete`, `GET /orders/{id}/ticket` (dados formatados para impressão térmica)
- **Equipe & Filiais:** `POST /api/v1/admin/companies/{companyId}/branches`, `GET/POST/PATCH /memberships`
- **Dashboard & Extrato:** `GET /dashboard?from=&to=` (datas civis no fuso da filial, convertidas para intervalo `timestamptz` na query), `GET /payments/statement?from=&to=` (valor, método, status; sem taxa da plataforma)
- **Tours:** `PATCH /api/v1/auth/me/tours` (salva progresso do React Joyride)

### Administrador da Plataforma (`/api/v1/platform/...` — exige `is_platform_admin`)
- `GET /leads` · `PATCH /leads/{id}`
- `GET /companies` · `POST /companies` (cria Empresa + 1ª Filial + gera convite do Owner — **onboarding assistido**)
- `PATCH /companies/{id}/status` (suspender/reativar com motivo obrigatório; invalida sessões/cache imediatamente)
- `GET /legal-acceptances` (consulta paginada de aceites legais versionados por empresa/usuário)
- `POST /support/branches/{branchId}/enter` (exige `reason`, grava log imutável em `support_access_audit_logs`, concede acesso **somente leitura** à fila)
- `GET /support/audit-logs`

---

## 9. Critérios de Aceite & Plano de Testes (`apps/api`)

- [ ] **Unitários (`go test -race ./...`):** >= 80% cobertura em `domain/` e `application/`, testando máquina de estados de pedidos, estratégias de frete (fixo/CEP/retirada), estratégias de cupom, `Money` em centavos, `IANATimezone`, "aberto agora" e `MockPaymentAdapter`. Inclui `TestSQLCQueries_RequireTenantPredicate` e o helper de cache que recusa `SET` sem TTL.
- [ ] **Integração (`testcontainers-go` — PG 19 + uma instância Redis + Garage):**
  - Migrations `up` e `down` limpas (incluindo `btree_gist` e `EXCLUDE USING gist` em `cep_shipping_ranges`, rejeitando faixas sobrepostas com `23P01` → `409`); sem coluna `platform_fee_cents`; sem `customer_identifier_hash`; sem `ENABLE ROW LEVEL SECURITY`;
  - Segundo `company_owner` do mesmo usuário na mesma empresa (`branch_id` NULL) falha com `23505`. Papel de filial sem `branch_id`, ou owner com `branch_id`, falha no `CHECK`;
  - Busca FTS encontra "Açaí" buscando `"acai"` e "X-Salada" buscando `"x-salda"`, confirmando via `EXPLAIN` uso de índice GIN;
  - Payloads de SQL Injection em `q` e filtros dinâmicos são neutralizados sem erro 500 e sem vazamento;
  - **IDOR entre duas empresas:** sessão da empresa A não lê nem altera pedido, produto, cupom, extrato, dashboard ou equipe da empresa B. Id inexistente e id da outra empresa, na query já escopada, devolvem o mesmo `404`. Path com filial sem vínculo devolve `403` sem vazar o nome da outra empresa;
  - Fuso: no instante `2026-11-23T01:30:00Z`, grade 10:00–22:00, Manaus aberta e São Paulo fechada. Pedido `2026-11-23T03:30:00Z` cai no dia 22 em Manaus e no dia 23 em São Paulo;
  - Troca de contexto rotaciona o token do cookie `HttpOnly` no prefixo `mc:sess:`;
  - Cache `mc:cache:` valida `UNLINK` pós-commit, `singleflight`, fallback de `GET` e preservação da sessão de 24h quando o `volatile-ttl` despeja cache curto;
  - **Idempotência (`Idempotency-Key`):** replay `201` (payload igual), `409 Conflict` (requisição paralela `in_progress`) e `422 Unprocessable Entity` (payload divergente);
  - **Concorrência & Locks:**
    - 50 goroutines simultâneas sobre cupom com `max_uses = 5` → exatamente 5 aprovações e 45 rejeições (`409`);
    - `max_uses_per_customer = 1`: cada cliente logado consome uma vez; o segundo uso do mesmo `customer_user_id` devolve `409 COUPON_CUSTOMER_LIMIT_REACHED`; convidado recebe `422 COUPON_REQUIRES_ACCOUNT` sem resgate e sem incrementar `used_count`; cupom sem limite por cliente fecha para convidado; estorno devolve a vaga global e a da conta;
    - Corrida simultânea cliente cancela vs cozinha confirma no mesmo pedido → Optimistic Locking (`version`) garante exatamente 1 sucesso e 1 `409 Conflict` sem deadlock;
    - 2 réplicas concorrentes disparando o cron de retenção LGPD via `pg_try_advisory_xact_lock` → exatamente 1 executa;
    - Validação de `SET LOCAL lock_timeout` (`55P03`) retornando conexão saudável ao `pgxpool`;
  - **LGPD:** `GET /customer/privacy/export` retorna JSON completo e `POST /customer/privacy/delete-account` anonimiza 100% do PII mantendo inalterados totais financeiros da filial e sem estornar cupom;
  - Upload de PNG/JPEG converte e persiste `.webp` no Garage S3 e rejeita arquivo falsificado (magic bytes inválidos);
  - SSE entrega eventos publicados pós-commit nos canais `mc:ps:` em < 200ms e envia snapshot inicial na reconexão.
- [ ] **Estático & Contrato:** `golangci-lint` (`depguard` + `gosec`), `sqlc diff`, allowlist de tenant sem linha sem motivo, e `go run ./cmd/api openapi` (zero drift com `openapi-typescript`) passando sem avisos.
