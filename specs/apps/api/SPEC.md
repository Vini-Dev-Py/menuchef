# Spec de Aplicação — Backend API (`apps/api`)

> **Documento autocontido.** Especifica completamente o backend do Menu Chef: stack Go `1.27.4`, `chi` v5 + **`Huma v2`** (OpenAPI 3.1 + RFC 9457), arquitetura modular (Clean Architecture + CQRS), modelagem PostgreSQL `19.0` (com `uuidv7()` nativo, Keyset Pagination e Full-Text Search PT-BR `pt_unaccent + pg_trgm`), **2 instâncias Redis isoladas** (`redis-state` + `redis-cache`), idempotência (`Idempotency-Key`), concorrência atômica de cupons com estorno automático, LGPD B2B2C, tempo real via SSE + Pub/Sub, pagamento mocado via Adapter (`MockPaymentAdapter`) e armazenamento WebP via Garage S3.

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
| **Estado Crítico & Pub/Sub** | **`redis-state` (`go-redis/v9` + `alexedwards/scs/v2`)** | Instância dedicada com `noeviction` + AOF: sessões opacas `HttpOnly` com revogação instantânea e `RenewToken`, Pub/Sub SSE e rate limiting — jamais sofre eviction por cache |
| **Cache de Leitura** | **`redis-cache` (`go-redis/v9` + `x/sync/singleflight`)** | Instância dedicada `allkeys-lru` (sem disco): cache Cache-Aside de cardápio público, horários e RBAC com TTL + jitter (±15%), proteção contra *stampede* e invalidação explícita pós-commit (`UNLINK`) |
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
    ├── platform/                    # Infra compartilhada: config, db (pgxpool + WithTx), redis (state + cache), httpserver (chi + huma), idempotency, logger, clock, errors
    └── modules/
        ├── lead/                    # Captação comercial da landing (onboarding assistido)
        ├── identity/                # Usuários globais, login/logout, sessões redis-state, convites, contas de cliente, LGPD (export/anonimização, legal_acceptances)
        ├── tenant/                  # Empresas, filiais, vínculos (memberships), RBAC, troca de contexto, suporte auditado
        ├── catalog/                 # Categorias, produtos, variações/adicionais, promoções, upload WebP, busca FTS
        ├── delivery/                # Horários de funcionamento, pausa da loja, frete fixo e faixas de CEP, cupons atômicos
        ├── order/                   # Ciclo de vida do pedido, máquina de estados, display_number, histórico/reorder, estorno de cupom
        ├── payment/                 # PaymentGatewayPort + MockPaymentAdapter (Pix/Cartão mocado) + extrato
        ├── realtime/                # Hub SSE em memória + backplane redis-state Pub/Sub + Web Push dispatcher
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
- **`branches`**: `id (uuidv7)`, `company_id (FK)`, `name`, `slug (UNIQUE)`, `phone`, `address_json`, `logo_url`, `primary_color_hex`, `is_paused`, `pause_reason`, `accepts_delivery`, `accepts_pickup`, `accepts_pay_on_delivery`, `shipping_mode (fixed|cep_range)`, `fixed_shipping_cents`, `min_order_cents`, `opening_hours_json`.
- **`memberships`**: `id (uuidv7)`, `user_id (FK)`, `company_id (FK)`, `branch_id (FK nullable p/ company_owner)`, `role (company_owner|branch_manager|kitchen_operator|dispatcher)`, `status (active|revoked)`, `UNIQUE (user_id, company_id, branch_id)`.
- **`invites`**: `id (uuidv7)`, `email`, `company_id`, `branch_id`, `role`, `token_hash (UNIQUE)`, `expires_at`, `accepted_at`.
- **`categories`**: `id (uuidv7)`, `branch_id (FK)`, `name`, `sort_order`, `is_active`.
- **`products`**: `id (uuidv7)`, `branch_id (FK)`, `category_id (FK)`, `name`, `description`, `price_cents (bigint)`, `promo_price_cents (bigint nullable)`, `image_url`, `is_available`, `sort_order`, `search_vector (tsvector)`, `search_text_trgm (text)`.
- **`option_groups`** & **`option_items`**: `id (uuidv7)`, `product_id (FK)`, `name`, `min_select`, `max_select`, `is_required` / `name`, `price_cents`, `is_available`.
- **`cep_shipping_ranges`**: `id (uuidv7)`, `branch_id (FK)`, `cep_start (char(8))`, `cep_end (char(8))`, `fee_cents (bigint)`, `estimated_minutes`, `is_active` — com constraint nativa **`EXCLUDE USING gist (branch_id WITH =, int4range(cep_start::int, cep_end::int, '[]') WITH &&) WHERE (is_active)`** (extensão `btree_gist`, mapeando `SQLSTATE 23P01` para `409 CEP_RANGE_OVERLAP`).
- **`coupons`**: `id (uuidv7)`, `branch_id (FK)`, `code`, `discount_type (fixed|percentage|free_shipping)`, `discount_value`, `min_order_cents`, `max_uses (int nullable)`, `max_uses_per_customer (int nullable)`, `used_count (int NOT NULL DEFAULT 0)`, `valid_from`, `valid_until`, `is_active`, `UNIQUE (branch_id, lower(code))`.
- **`coupon_redemptions`**: `id (uuidv7)`, `coupon_id (FK)`, `order_id (FK UNIQUE)`, `customer_identifier_hash (char(64))`, `created_at` — garante controle atômico por cliente e idempotência de estorno em cancelamentos.
- **`idempotency_keys`**: `scope_key (text)`, `idempotency_key (uuid)`, `request_hash (char(64))`, `status (in_progress|completed)`, `response_status (int nullable)`, `response_body (jsonb nullable)`, `locked_until (timestamptz)`, `expires_at (timestamptz)`, `PRIMARY KEY (scope_key, idempotency_key)`.
- **`orders`**: `id (uuidv7)`, `branch_id (FK)`, `customer_user_id (FK nullable)`, `display_number (int)`, `tracking_token (UNIQUE)`, `version (int NOT NULL DEFAULT 1)` (Optimistic Locking), `fulfillment_mode (delivery|pickup)`, `payment_timing (online|on_delivery)`, `payment_method (pix_mock|card_mock|cash|card_machine|pix_door)`, `change_for_cents (bigint nullable)`, `status`, `customer_name`, `customer_phone`, `delivery_address_json`, `subtotal_cents`, `shipping_cents`, `discount_cents`, `total_cents`, `coupon_id`, `rejection_or_cancel_reason`, `created_at`, `confirmed_at`, `completed_at`.
- **`order_items`** & **`order_item_options`**: snapshot imutável do nome, preço unitário em centavos, quantidade, observação (`"sem cebola"`) e adicionais escolhidos no momento do fechamento (mudanças futuras no cardápio nunca alteram pedidos históricos).
- **`payments`**: `id (uuidv7)`, `order_id (FK)`, `branch_id (FK)`, `provider (mock)`, `provider_ref`, `method (pix|card)`, `amount_cents`, `platform_fee_cents (0 quando on_delivery)`, `status (pending|approved|refused|expired|refunded)`, `pix_copy_paste`, `expires_at`, `settled_at`.
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

---

## 5. Autenticação, RBAC, Cache Redis e Defesa contra SQL Injection

1. **Sessão Opaca em `redis-state` (`alexedwards/scs`):**
   - Cookie `__Host-mc_session` (`HttpOnly; Secure; SameSite=Lax; Path=/`). **Zero tokens no `localStorage`.**
   - Payload de sessão: `user_id`, `active_company_id`, `active_branch_id`, `auth_at`.
   - Toda chamada de login e troca de contexto (`POST /api/v1/auth/context`) executa `RenewToken` (rotação de Session ID).
2. **Proteção CSRF nativa (Go 1.27):** `http.NewCrossOriginProtection()` acoplado como middleware global em todas as rotas mutáveis (`POST`, `PUT`, `PATCH`, `DELETE`).
3. **Cache-Aside & Invalidação em `redis-cache`:**
   - Decorator envolvendo `BranchMenuReadRepository` (`mc:menu:branch:{id}:v1`, TTL 5m ±15% jitter), `BranchPublicMetaReader` (`mc:branch:slug:{slug}:v1`, TTL 60s ±15%) e `MembershipPermissionReader` (`mc:rbac:user:{userId}:v1`, TTL 2m ±15%).
   - Toda leitura cacheada usa `golang.org/x/sync/singleflight` contra *cache stampede*; falha no `redis-cache` degrada transparentemente para o PostgreSQL.
   - Todo Command de escrita que altera cardápio, status de pausa/horário, vínculo de equipe ou suspensão de empresa dispara `UNLINK` das chaves afetadas **somente após `tx.Commit()`** bem-sucedido.
   - **Nunca cachear invariantes de escrita:** checkout, CEP no fechamento, cupons e máquina de estados do pedido leem sempre do PostgreSQL em transação.
4. **SQL Injection Zero:**
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
- Para limite por cliente (`max_uses_per_customer`), adquire **`pg_advisory_xact_lock`** escopado em `(LockNSCouponCustomer, hash(coupon_id, customer_identifier_hash))` (evita *phantom read* sem bloquear outros clientes do mesmo cupom), conta usos em `coupon_redemptions` e insere o novo vínculo com `UNIQUE (coupon_id, order_id)`.
- **Estorno automático:** quando um pedido com cupom vai para `canceled_by_customer` (antes de `confirmed`), `rejected_by_store` ou expiração de Pix mocado, a transação executa `DELETE FROM coupon_redemptions WHERE order_id = $1 RETURNING coupon_id` e, somente se deletou, `UPDATE coupons SET used_count = GREATEST(used_count - 1, 0) WHERE id = $1`.

### 6.3 Transactions, Locks & Advisory Locks (`pgxpool` + PostgreSQL 19)
- **Isolamento padrão:** `READ COMMITTED` com `pgx.BeginTxFunc` (rollback garantido).
- **Hierarquia obrigatória de timeouts em `ConnConfig.RuntimeParams`:** `lock_timeout = 2000` (2s, `55P03` ERROR) `<` `idle_in_transaction_session_timeout = 4000` (4s) `<` `statement_timeout = 5000` (5s, `57014` ERROR) `<` `transaction_timeout = 15000` (15s, `25P04` FATAL — rede de segurança) `<` HTTP request timeout (20s). Overrides locais usam **sempre `SET LOCAL`** (nunca `SET` puro no pool).
- **Optimistic Locking em `orders`:** toda transição de status executa `UPDATE orders SET status = $next, version = version + 1 WHERE id = $1 AND branch_id = $2 AND status = $expected AND version = $v RETURNING ...`. Zero linhas → `409 Conflict` (`INVALID_STATE_TRANSITION`).
- **Upsert de `branch_order_counters` (`display_number`):** executado como **último write** dentro da transação de `CreateOrder`, segurando o lock da linha da filial por < 1–2 ms.
- **Advisory Locks estritamente transaction-scoped (`pg_advisory_xact_lock` / `pg_try_advisory_xact_lock`):** **proibido `pg_advisory_lock` (session-level)** com `pgxpool`. Namespaces `int32` centralizados em `internal/platform/db/locks.go` para: (1) cupom por cliente (`max_uses_per_customer`), (2) jobs cron singleton (`pg_try_advisory_xact_lock` na limpeza LGPD/retenção) e (3) reordenação em lote do cardápio por filial. Lotes de worker (expiração de Pix pendente) usam **`SELECT ... FOR UPDATE SKIP LOCKED`**.
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
- **Pagar na Entrega / Retirada:** não chama `PaymentGatewayPort`; registra `platform_fee_cents = 0` e envia o pedido direto para `received`.

### 6.5 LGPD & Privacidade no MVP
- **Ledger de aceite:** `POST /public/leads`, `POST /auth/invites/accept` e `POST /auth/register-customer` exigem versão aceita de Termos/Privacidade e gravam registro imutável em `legal_acceptances`.
- **Exportação do titular:** `GET /api/v1/customer/privacy/export` retorna JSON com perfil, endereços, aceites legais e pedidos do cliente autenticado.
- **Exclusão / Anonimização:** `POST /api/v1/customer/privacy/delete-account` revoga todas as sessões do cliente no `redis-state`, remove endereços salvos e anonimiza PII em `users` e `orders` (`customer_name = "Cliente Anonimizado"`, telefone/e-mail substituídos por hash irreversível, `delivery_address_json = NULL`), **preservando intactos `order_items`, valores e métricas financeiras das filiais**.
- **Jobs de retenção (coordenados por `pg_try_advisory_xact_lock`):** rotina periódica remove `idempotency_keys` expiradas (>24h), convites expirados (>7d) e leads inativos (>24 meses).

---

## 7. Tempo Real (SSE + `redis-state` Pub/Sub)

- **Endpoints SSE (registrados em `chi` puro, fora do wrapper JSON do `Huma v2`):**
  - Fila da cozinha (autenticado + permissão `orders:read`): `GET /api/v1/admin/branches/{branchId}/orders/stream`
  - Acompanhamento do cliente (público via `tracking_token`): `GET /api/v1/public/orders/{trackingToken}/stream`
- **Arquitetura multi-réplica sem sticky sessions (Publicação pós-commit + Snapshot):**
  1. Command de pedido (`CreateOrder`, `UpdateOrderStatus`, `CancelOrder`) faz **`tx.Commit()` no PostgreSQL primeiro** e publica o evento JSON no `redis-state` Pub/Sub (`branch:{branchId}:orders` e `order:{trackingToken}`) **somente após o commit** (sem tabela extra de Outbox no MVP; reconciliação garantida pelo snapshot do banco).
  2. Cada réplica da API mantém um Hub em memória multiplexado por uma única conexão `redis-state` Pub/Sub.
  3. Ao conectar ou reconectar (com `Last-Event-ID`), o handler envia um **snapshot inicial do banco** (`event: snapshot`) e depois transmite eventos incrementais (`order_created`, `order_status_changed`).
  4. Handler limpa o `WriteTimeout` com `http.NewResponseController(w).SetWriteDeadline(time.Time{})`, envia `X-Accel-Buffering: no`, `Cache-Control: no-cache, no-transform` e heartbeat `: ping\n\n` a cada 15s.

---

## 8. Contratos HTTP Principais (`/api/v1` — OpenAPI 3.1 via `Huma v2`)

> **Versionamento:** evolução aditiva dentro de `/api/v1`; mudanças incompatíveis exigem `/api/v2` ou ciclo `Deprecation`/`Sunset` (RFC 8594). A spec OpenAPI 3.1 é exportada em build/CI por `go run ./cmd/api openapi > openapi.json`.

### Público (Landing & Storefront)
- `GET /api/v1/health/live` · `GET /api/v1/health/ready`
- `POST /api/v1/public/leads` — registra contato de empresa interessada + `accepted_privacy_version` (rate-limited)
- `GET /api/v1/public/branches/{branchSlug}/menu` — dados da filial, status (aberta/fechada/pausada), categorias, produtos, adicionais e promoções (cacheado em `redis-cache`)
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
- **Operação & Entrega:** `PUT /schedule`, `POST /pause` / `POST /resume`, CRUD `shipping-ranges` (CEP), CRUD `coupons`
- **Pedidos:** `GET /orders` (filtros dinâmicos via `go-jet` + Cursor Pagination), `GET /orders/stream` (SSE em `chi` puro), `POST /orders/{id}/confirm`, `POST /orders/{id}/reject` (estorna cupom), `POST /orders/{id}/start-production`, `POST /orders/{id}/mark-ready`, `POST /orders/{id}/dispatch`, `POST /orders/{id}/complete`, `GET /orders/{id}/ticket` (dados formatados para impressão térmica)
- **Equipe & Filiais:** `POST /api/v1/admin/companies/{companyId}/branches`, `GET/POST/PATCH /memberships`
- **Dashboard & Extrato:** `GET /dashboard?from=&to=`, `GET /payments/statement?from=&to=`
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

- [ ] **Unitários (`go test -race ./...`):** >= 80% cobertura em `domain/` e `application/`, testando máquina de estados de pedidos, estratégias de frete (fixo/CEP/retirada), estratégias de cupom, `Money` em centavos e `MockPaymentAdapter`.
- [ ] **Integração (`testcontainers-go` — PG 19 + `redis-state` + `redis-cache` + Garage):**
  - Migrations `up` e `down` limpas (incluindo `btree_gist` e `EXCLUDE USING gist` em `cep_shipping_ranges`, rejeitando faixas sobrepostas com `23P01` → `409`);
  - Busca FTS encontra "Açaí" buscando `"acai"` e "X-Salada" buscando `"x-salda"`, confirmando via `EXPLAIN` uso de índice GIN;
  - Payloads de SQL Injection em `q` e filtros dinâmicos são neutralizados sem erro 500 e sem vazamento;
  - Isolamento multi-tenant validado: usuário da Filial A recebe `403` ao tentar ler/mutar recurso da Filial B;
  - Troca de contexto rotaciona o token do cookie `HttpOnly` no `redis-state`;
  - Cache `redis-cache` valida invalidação `UNLINK` pós-commit, `singleflight` sob concorrência e fallback transparente se `redis-cache` estiver indisponível;
  - **Idempotência (`Idempotency-Key`):** replay `201` (payload igual), `409 Conflict` (requisição paralela `in_progress`) e `422 Unprocessable Entity` (payload divergente);
  - **Concorrência & Locks:**
    - 50 goroutines simultâneas sobre cupom com `max_uses = 5` → exatamente 5 aprovações e 45 rejeições (`409`), mais teste concorrente de `max_uses_per_customer = 1` via `pg_advisory_xact_lock` e estorno automático em cancelamento/recusa/expiração;
    - Corrida simultânea cliente cancela vs cozinha confirma no mesmo pedido → Optimistic Locking (`version`) garante exatamente 1 sucesso e 1 `409 Conflict` sem deadlock;
    - 2 réplicas concorrentes disparando o cron de retenção LGPD via `pg_try_advisory_xact_lock` → exatamente 1 executa;
    - Validação de `SET LOCAL lock_timeout` (`55P03`) retornando conexão saudável ao `pgxpool`;
  - **LGPD:** `GET /customer/privacy/export` retorna JSON completo e `POST /customer/privacy/delete-account` anonimiza 100% do PII mantendo inalterados totais financeiros da filial;
  - Upload de PNG/JPEG converte e persiste `.webp` no Garage S3 e rejeita arquivo falsificado (magic bytes inválidos);
  - SSE entrega eventos publicados pós-commit no `redis-state` em < 200ms e envia snapshot inicial na reconexão.
- [ ] **Estático & Contrato:** `golangci-lint` (`depguard` + `gosec`), `sqlc diff` e `go run ./cmd/api openapi` (zero drift com `openapi-typescript`) passando sem avisos.
