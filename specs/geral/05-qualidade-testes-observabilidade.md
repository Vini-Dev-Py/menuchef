# Spec Geral 05 — Qualidade, Pirâmide de Testes, Erros e Observabilidade

> **Documento autocontido.** Define a estratégia completa de engenharia de qualidade do Menu Chef: testes unitários, testes de integração com infraestrutura real (**Testcontainers**), testes **E2E com Cypress** (multi-viewport), padronização de erros de domínio/HTTP, observabilidade estruturada (`log/slog`, correlationId, métricas) e Quality Gates de CI.

---

## 1. Pirâmide de Testes

```text
                ▲
               / \       E2E (Cypress) — Fluxos críticos de negócio
              /   \      Multi-viewport: Mobile (375), Tablet (768), Desktop (1440)
             /─────\
            /       \    Integração — Repositories Postgres 19 (FTS/SQLi/Migrations),
           /         \   Redis (scs/PubSub), Garage S3, HTTP Handlers + Middlewares
          /───────────\
         /             \ Unitários — Domain (Entities, VOs, Strategies, State Machine),
        /               \ Application (Commands/Queries com mocks de Ports), Frontend (Vitest + RTL)
       ───────────────────
```

**Padrão obrigatório em todos os testes:** estrutura clara **AAA (Arrange / Act / Assert)** ou **Given / When / Then**, nomes descritivos de comportamento (ex.: `TestCreateOrder_RejectsWhenCepOutsideBranchRange`), determinismo total (zero `time.Sleep` arbitrário, relógio injetável `Clock` no domínio).

---

## 2. Testes por Camada e Ferramentas

### 2.1 Backend Go (`apps/api`)
- **Testes Unitários (`go test ./... -race`):**
  - Ferramentas: stdlib `testing` + `github.com/stretchr/testify/assert` & `require` (mocks leves gerados ou escritos à mão via interfaces pequenas).
  - Alvo: 100% das invariantes de `domain/` (`Money`, `CEP`, `OrderStatus.CanTransitionTo`, cálculo de frete/cupom, validação de adicionais obrigatórios) e `application/` (handlers CQRS).
  - Flag `-race` obrigatória em CI (detecção de data races, especialmente no Hub SSE em memória).
- **Testes de Integração (`go test -tags=integration ./...`):**
  - Ferramenta: **`testcontainers-go`** subindo containers reais de **PostgreSQL 19.0**, **uma instância Redis** e **Garage v2.5**.
  - Alvo:
    - Aplicação limpa de todas as migrations `goose up` / `goose down`.
    - Queries `sqlc` e `go-jet` contra banco real, incluindo **Full-Text Search PT-BR** (acentuação, typos com `pg_trgm`, ordenação de relevância e uso de índice GIN).
    - **Vínculo único:** inserir duas vezes `company_owner` para o mesmo `(user_id, company_id)` com `branch_id` NULL falha com `23505` (`UNIQUE NULLS NOT DISTINCT`). O segundo vínculo do mesmo usuário na mesma filial também falha. `company_owner` com `branch_id` preenchido, ou papel de filial com `branch_id` NULL, falha no `CHECK`. Owner da empresa e `branch_manager` de uma filial da mesma empresa coexistem.
    - **Isolamento entre duas empresas (IDOR):** semear empresa A (filial A1, usuário UA) e empresa B (filial B1, usuário UB). Com sessão de UA, ler ou alterar pedido, produto, cupom, pagamento, dashboard ou membership pelo UUID da B não devolve corpo da B. Pausar filial, mudar preço ou confirmar pedido da B é recusado. Id inexistente e id da outra empresa, na rota já filtrada pelo `branch_id` do contexto, devolvem o **mesmo** status (`404`) e o mesmo formato. `branchId` de path sem vínculo devolve `403`, sem nome da outra empresa. UA que também opera a B, com contexto em B, vê a fila da B e não a da A.
    - **Gate estático de queries (roda também sem Testcontainers, no job de unitários):** `TestSQLCQueries_RequireTenantPredicate` lê os `.sql` do `sqlc` em `apps/api/queries/`. Tabelas multi-tenant: `branches`, `memberships`, `invites`, `categories`, `products`, `option_groups`, `option_items`, `cep_shipping_ranges`, `coupons`, `coupon_redemptions`, `orders`, `order_items`, `order_item_options`, `payments`, `branch_order_counters`, `support_access_audit_logs`. Comentário e literal string são ignorados. A query passa se o SQL tem predicado `branch_id` ou `company_id` (`=`, `<>`, `IN`, `= ANY`, ou `ON` de JOIN amarrando a coluna a parâmetro ou a chave já escopada). `WHERE id = $1` sozinho em `orders` falha. Exceção só pela allowlist `apps/api/queries/tenant_scope_allowlist.txt`, uma query por linha (`NomeSqlc<TAB>motivo`). Motivo vazio falha. Nome inexistente falha. Entradas previstas: `GetBranchByPublicSlug` (resolve a filial pelo slug; as queries seguintes recebem o `branch_id`), `GetUserByEmail` (`users` é global), `ListLeads` e `InsertLead` (fila da plataforma), `ListCompaniesForPlatform` (visão do `platform_admin`). A allowlist não cobre query de filial que poderia levar `branch_id`.
    - **Fuso da filial:** grade 10:00–22:00. No instante `2026-11-23T01:30:00Z`, filial `America/Manaus` está aberta (21:30 do dia 22) e filial `America/Sao_Paulo` está fechada (22:30 do dia 22). Pedido com `created_at = 2026-11-23T03:30:00Z` tem `business_date` `2026-11-22` em Manaus e `2026-11-23` em São Paulo; o dashboard do dia 23 inclui o pedido só em São Paulo. Default de filial nova é `America/Sao_Paulo`. `GMT-4` e `Brasilia` devolvem `422`. Horário exibido ao cliente e na comanda é o relógio da filial, não UTC.
    - Fluxo real de sessão `alexedwards/scs` no prefixo `mc:sess:` (TTL 24h) + `http.CrossOriginProtection` + rotação de Session ID na troca de filial.
    - Cache no prefixo `mc:cache:`: invalidação `UNLINK` pós-commit, `singleflight` sob carga paralela, helper que recusa `SET` sem TTL, e fallback ao Postgres quando o `GET` de cache falha. Com `maxmemory` baixo no container de teste, encher chaves `mc:cache:*` de TTL curto não apaga uma chave `mc:sess:` com TTL de 24h (`volatile-ttl`). `CONFIG GET maxmemory-policy` devolve `volatile-ttl` e o AOF está ligado.
    - **Idempotência (`Idempotency-Key`):** replay `201` para mesmo payload, `409 Conflict` para requisição simultânea `in_progress`, e `422 Unprocessable Entity` para mesma chave com payload diferente.
    - **Concorrência de cupons & Locks:** disparo simultâneo de **50 goroutines** tentando resgatar cupom com `max_uses = 5` garantindo exatamente 5 sucessos e 45 rejeições (`409`). **Limite por cliente (decisão de 23/11/2026):** `max_uses_per_customer = 1` com dois clientes logados — cada um consome uma vez; a segunda tentativa do mesmo `customer_user_id` devolve `409 COUPON_CUSTOMER_LIMIT_REACHED`; o lock é `pg_advisory_xact_lock` em `(coupon_id, customer_user_id)`. Convidado nesse cupom recebe `422 COUPON_REQUIRES_ACCOUNT` com a mensagem *"Este cupom tem limite por cliente. Entre na sua conta para usá-lo."*, sem linha em `coupon_redemptions` e sem mexer em `used_count`. Cupom com `max_uses_per_customer` nulo fecha para convidado. O schema não tem `customer_identifier_hash`. Estorno automático em cancelamento pré-confirmação, recusa da loja e expiração de Pix mocado devolve a vaga global e a vaga daquele `customer_user_id`. Corrida simultânea cliente cancela vs cozinha confirma (`Optimistic Locking` via `orders.version` → exatamente 1 sucesso e 1 `409`); faixas de CEP sobrepostas rejeitadas pela constraint `EXCLUDE USING gist` (`23P01` → `409`); cron singleton via `pg_try_advisory_xact_lock`; e `SET LOCAL lock_timeout` (`55P03`) devolvendo conexão saudável ao `pgxpool`.
    - **LGPD:** `GET /api/v1/customer/privacy/export` retorna JSON completo do titular e `POST /api/v1/customer/privacy/delete-account` anonimiza 100% do PII mantendo inalterados os totais financeiros da filial. A exclusão não estorna cupom.
    - Pub/Sub nos canais `mc:ps:` da instância Redis única + entrega de eventos SSE com reconexão (`Last-Event-ID`).
    - Testes negativos de segurança (payloads clássicos de SQL Injection em busca e filtros).
    - Confirmação de schema: não existe coluna `platform_fee_cents`. Não há `ENABLE ROW LEVEL SECURITY` nas migrations do MVP.

### 2.2 Frontends TypeScript (`apps/{landing,storefront,admin}`)
- **Testes Unitários e de Componente:**
  - Ferramentas: **Vitest** + **React Testing Library** + **@testing-library/user-event** + **MSW (Mock Service Worker)** para mocks de rede em nível de protocolo.
  - Alvo:
    - Schemas Zod, cálculos de sacola (em centavos inteiros), formatação monetária `BRL`, regras de exibição de status/horário no fuso da filial (Manaus aberto e São Paulo fechado no instante `2026-11-23T01:30:00Z`, grade 10:00–22:00) e geração/renovação de `Idempotency-Key`.
    - Cupom com limite por cliente no checkout convidado: mensagem *"Este cupom tem limite por cliente. Entre na sua conta para usá-lo."* e CTA de login sem esvaziar a sacola. Cupom sem esse limite segue no fluxo convidado.
    - Componentes críticos de interação (modal de adicionais obrigatórios, seletor de empresa/filial, ações do card Kanban por teclado, comportamento com `prefers-reduced-motion`).
  - Type-check estrito: `tsc --noEmit` com `strict: true` (zero `any` implícito ou explícito não justificado).

### 2.3 Testes End-to-End (**Cypress**)
Localizados em `tests/e2e/`, executando contra a stack real (`infra/docker-compose.test.yml`) ou ambiente local completo com banco seedado de forma determinística:

- **Suítes E2E Obrigatórias:**
  1. `landing.cy.ts` — renderização SSR/SSG, meta tags SEO, presença de `/llms.txt`, `/sitemap.xml`, `/robots.txt`, páginas SSG **`/termos-de-uso` e `/politica-de-privacidade`**, envio do formulário de contato (lead) com aceite legal e validação de erro em campos inválidos.
  2. `platform-onboarding.cy.ts` — login do Administrador da Plataforma → criação de empresa + 1ª filial → geração de convite → aceite do convite pelo dono com registro de aceite legal → acesso ao painel e conclusão/skip do tour React Joyride.
  3. `context-switching.cy.ts` — 1 usuário vinculado a 2 empresas/filiais com papéis distintos (`company_owner` vs `kitchen_operator`) alterna contexto via seletor/URL sem relogar e verifica alteração de permissões e isolamento de fila.
  4. `storefront-checkout-guest.cy.ts` — mobile (`375x812`): busca FTS com typo/sem acento → seleção de produto com adicional obrigatório + observação → persistência da sacola após reload → CEP fora de área bloqueado → CEP válido + cupom → clique duplo protegido por `Idempotency-Key` → pagamento **Pix mocado** aprovado → tela de comprovante e número curto `#001`.
  5. `storefront-checkout-card-and-door.cy.ts` — checkout com **Cartão mocado** (cenário recusa + retentativa aprovada) e checkout **Retirada / Pagar na entrega com troco**.
  6. `kitchen-realtime-flow.cy.ts` — dois contextos sincronizados (Storefront + Painel Admin PWA): pedido entra na fila em tempo real via SSE → cliente cancela antes de confirmar (sucesso + cupom estornado) → segundo pedido confirmado pela cozinha (cancelamento bloqueado para cliente) → `in_production` → `ready` → `out_for_delivery` → `completed` → preview de impressão da comanda.
  7. `customer-history-reorder-lgpd.cy.ts` — cliente cria conta opcional (com aceite legal) → consulta histórico → aciona "Pedir de novo" → exporta dados JSON (LGPD) → exclui/anonimiza conta validando logout imediato e preservação do faturamento no dashboard da filial.
  8. `support-audit.cy.ts` — Administrador da Plataforma acessa fila de loja em modo suporte somente-leitura com motivo → valida registro imutável na trilha de auditoria.
  9. `responsive-and-a11y.cy.ts` — verifica ausência de overflow horizontal em `375px`, `768px` e `1440px`, mais checagem automatizada **`cypress-axe`** (WCAG 2.2 AA).

- **Boas práticas Cypress:**
  - Seletores estáveis via `data-testid` ou roles acessíveis (`findByRole`), nunca classes Tailwind frágeis.
  - Reset determinístico de estado antes de cada spec via endpoint/comando de seed habilitado **apenas** quando `APP_ENV=test` (nunca compilado/habilitado em produção).
  - Gravação de screenshots em falha e vídeos apenas em falhas no CI.

---

## 3. Padronização de Erros (Domínio → HTTP → UI)

Nunca engolir exceptions ou retornar erros genéricos sem contexto técnico nos logs.

### Contrato de Erro HTTP (RFC 9457 — Problem Details emitido pelo `Huma v2`)
Todas as respostas `4xx`/`5xx` da API seguem formato JSON consistente:

```json
{
  "type": "https://menuchef.com.br/errors/cep-out-of-delivery-area",
  "title": "Unprocessable Entity",
  "status": 422,
  "code": "CEP_OUT_OF_DELIVERY_AREA",
  "detail": "Não entregamos neste CEP no momento.",
  "request_id": "019a2f7c-8e1a-7b3c-9f21-4a7c8b9d0e1f",
  "errors": [
    { "location": "body.address.cep", "message": "Nenhuma faixa ativa cobre o CEP 01310-100" }
  ]
}
```

### Mapeamento de Erros de Domínio
| Erro de Domínio | HTTP Status | Comportamento na UI |
| :--- | :--- | :--- |
| `ValidationError` / `IdempotencyPayloadMismatchError` | `400 Bad Request` / `422 Unprocessable Entity` | Mensagem inline ao lado de cada campo (nunca apenas toast) ou regeneração controlada da `Idempotency-Key` |
| `UnauthorizedError` | `401 Unauthorized` | Redireciona para login preservando URL de retorno |
| `ForbiddenError` | `403 Forbidden` | Exibe estado de acesso negado sem vazar existência de outros tenants |
| `NotFoundError` | `404 Not Found` | Estado vazio/não encontrado amigável |
| `ConflictError` / `InvalidStateTransitionError` / `IdempotencyInProgressError` / `CouponExhaustedError` / `CouponCustomerLimitError` / `CepRangeOverlapError` | `409 Conflict` | Atualiza estado da tela, informa transição/esgotamento ou aguarda `Retry-After` em requisição em voo |
| `CouponRequiresAccountError` | `422 Unprocessable Entity` (`COUPON_REQUIRES_ACCOUNT`) | No checkout convidado, pede login para usar o cupom e mantém a sacola |
| `RateLimitExceededError` | `429 Too Many Requests` | Informa aguardo respeitando header `Retry-After` |
| Erro interno inesperado | `500 Internal Server Error` | Mensagem genérica ao usuário + `request_id` para suporte; stack/detalhe **apenas** no log estruturado do servidor |

---

## 4. Observabilidade (Logs, Tracing e Métricas)

### Logs Estruturados (`log/slog` — stdlib Go)
- Formato JSON em produção/staging; texto legível em dev local.
- Campos obrigatórios em toda entrada de log HTTP/UseCase: `timestamp`, `level`, `request_id` (correlationId gerado/propagado por `chi` middleware), `method`, `path`, `status`, `duration_ms`, `user_id` (quando autenticado), `company_id`, `branch_id` (quando em contexto).
- **Redaction obrigatória (LGPD/Segurança):** jamais logar senhas, hashes, cookies, tokens, CPF/CNPJ completo ou dados de cartão.

### Métricas & Saúde (Prometheus / OpenTelemetry Ready)
- **Métricas críticas monitoradas (p50 / p95 / p99):**
  - Latência e taxa de erro por rota HTTP (`http_request_duration_seconds`).
  - Saturação do pool PostgreSQL via `pgxpool.Pool.Stat()` (`acquired_conns`, `idle_conns`, `acquire_duration_ms`, `canceled_queries`) + contenção de locks via view **`pg_stat_lock`** (PG 19, incluindo `locktype = 'advisory'`) e contadores de `SQLSTATE 55P03`/`40P01`.
  - Taxa de hit/miss, fallback do cache e despejo da instância Redis (`cache_requests_total{result}`, `redis_evicted_keys_total`, `redis_up`).
  - Latência da busca FTS de produtos (`catalog_fts_duration_seconds`).
  - Conexões SSE ativas por filial e falhas de publicação no Pub/Sub (`sse_active_connections`, `realtime_publish_errors_total`).
  - Taxa de conversão de checkout, colisões de `Idempotency-Key`, `COUPON_REQUIRES_ACCOUNT` e erros do `PaymentGatewayPort`.
- **Healthchecks:**
  - `GET /api/v1/health/live` — liveness do processo Go (sem I/O externo).
  - `GET /api/v1/health/ready` — readiness com ping (timeout 2s) em PostgreSQL 19, **a instância Redis** e Garage S3. Um ping só. Redis fora falha o ready: sessão, Pub/Sub, rate-limit e cache são o mesmo processo.

---

## 5. Quality Gates (CI / Pull Requests)

Nenhum PR pode ser mergeado em `develop` ou `main` sem passar em:
1. **Lint & Estático:** `golangci-lint` (incluindo `depguard` para Clean Architecture + `gosec` para segurança) + ESLint/Biome + `tsc --noEmit` nos 3 frontends.
2. **Codegen & Contract Drift Check:**
   - `sqlc diff` / verificação de que código SQL gerado está sincronizado com migrations e queries `.sql`;
   - **OpenAPI 3.1 & TS Client Drift Check:** `go run ./cmd/api openapi` + `openapi-typescript` + `git diff --exit-code` garantindo zero divergência entre handlers `Huma v2` e tipos TypeScript dos 3 frontends.
3. **Unit Tests:** `go test -race ./...` (cobertura alvo >= 80% em `domain/` e `application/`) + `vitest run` nos 3 frontends. Inclui `TestSQLCQueries_RequireTenantPredicate` (gate de `branch_id`/`company_id` com allowlist).
4. **Integration Tests:** `go test -tags=integration ./...` com Testcontainers (concorrência de cupons, limite por cliente logado, fuso de Manaus, vínculo `NULLS NOT DISTINCT`, IDOR entre duas empresas, idempotência, Redis único e LGPD).
5. **E2E Smoke & Critical Flows:** `cypress run` contra `docker-compose.test.yml`.
6. **Build & Bundle Check:** build Docker multi-stage de todas as 4 aplicações sem erros + verificação de `curl` SSG na Landing (incluindo `/termos-de-uso` e `/politica-de-privacidade`).
