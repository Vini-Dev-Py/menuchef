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
  - Ferramenta: **`testcontainers-go`** subindo containers reais de **PostgreSQL 19.0**, **Redis** e **Garage v2.5**.
  - Alvo:
    - Aplicação limpa de todas as migrations `goose up` / `goose down`.
    - Queries `sqlc` e `go-jet` contra banco real, incluindo **Full-Text Search PT-BR** (acentuação, typos com `pg_trgm`, ordenação de relevância e uso de índice GIN).
    - Isolamento multi-tenant (usuário/filial A nunca lê nem escreve dados da filial B).
    - Fluxo real de sessão `alexedwards/scs` no `redis-state` + `http.CrossOriginProtection` + rotação de Session ID na troca de filial.
    - Cache `redis-cache`: invalidação `UNLINK` pós-commit, `singleflight` sob carga paralela e degradação graciosa se `redis-cache` cair.
    - **Idempotência (`Idempotency-Key`):** replay `201` para mesmo payload, `409 Conflict` para requisição simultânea `in_progress`, e `422 Unprocessable Entity` para mesma chave com payload diferente.
    - **Concorrência de cupons & Locks:** disparo simultâneo de **50 goroutines** tentando resgatar cupom com `max_uses = 5` garantindo exatamente 5 sucessos e 45 rejeições (`409`); validação de `max_uses_per_customer = 1` via `pg_advisory_xact_lock`; estorno automático em cancelamento pré-confirmação, recusa da loja e expiração de Pix mocado; corrida simultânea cliente cancela vs cozinha confirma (`Optimistic Locking` via `orders.version` → exatamente 1 sucesso e 1 `409`); faixas de CEP sobrepostas rejeitadas pela constraint `EXCLUDE USING gist` (`23P01` → `409`); cron singleton via `pg_try_advisory_xact_lock`; e `SET LOCAL lock_timeout` (`55P03`) devolvendo conexão saudável ao `pgxpool`.
    - **LGPD:** `GET /api/v1/customer/privacy/export` retorna JSON completo do titular e `POST /api/v1/customer/privacy/delete-account` anonimiza 100% do PII mantendo inalterados os totais financeiros da filial.
    - Pub/Sub `redis-state` + entrega de eventos SSE com reconexão (`Last-Event-ID`).
    - Testes negativos de segurança (payloads clássicos de SQL Injection em busca e filtros).

### 2.2 Frontends TypeScript (`apps/{landing,storefront,admin}`)
- **Testes Unitários e de Componente:**
  - Ferramentas: **Vitest** + **React Testing Library** + **@testing-library/user-event** + **MSW (Mock Service Worker)** para mocks de rede em nível de protocolo.
  - Alvo:
    - Schemas Zod, cálculos de sacola (em centavos inteiros), formatação monetária `BRL`, regras de exibição de status/horário e geração/renovação de `Idempotency-Key`.
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
| `ConflictError` / `InvalidStateTransitionError` / `IdempotencyInProgressError` / `CouponExhaustedError` / `CepRangeOverlapError` | `409 Conflict` | Atualiza estado da tela, informa transição/esgotamento ou aguarda `Retry-After` em requisição em voo |
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
  - Taxa de hit/miss e erros de fallback do `redis-cache` (`cache_requests_total{cache,result}`).
  - Latência da busca FTS de produtos (`catalog_fts_duration_seconds`).
  - Conexões SSE ativas por filial e falhas de publicação no `redis-state` Pub/Sub (`sse_active_connections`, `realtime_publish_errors_total`).
  - Taxa de conversão de checkout, colisões de `Idempotency-Key` e erros do `PaymentGatewayPort`.
- **Healthchecks:**
  - `GET /api/v1/health/live` — liveness do processo Go (sem I/O externo).
  - `GET /api/v1/health/ready` — readiness com ping (timeout 2s) em PostgreSQL 19, `redis-state`, `redis-cache` e Garage S3.

---

## 5. Quality Gates (CI / Pull Requests)

Nenhum PR pode ser mergeado em `develop` ou `main` sem passar em:
1. **Lint & Estático:** `golangci-lint` (incluindo `depguard` para Clean Architecture + `gosec` para segurança) + ESLint/Biome + `tsc --noEmit` nos 3 frontends.
2. **Codegen & Contract Drift Check:**
   - `sqlc diff` / verificação de que código SQL gerado está sincronizado com migrations e queries `.sql`;
   - **OpenAPI 3.1 & TS Client Drift Check:** `go run ./cmd/api openapi` + `openapi-typescript` + `git diff --exit-code` garantindo zero divergência entre handlers `Huma v2` e tipos TypeScript dos 3 frontends.
3. **Unit Tests:** `go test -race ./...` (cobertura alvo >= 80% em `domain/` e `application/`) + `vitest run` nos 3 frontends.
4. **Integration Tests:** `go test -tags=integration ./...` com Testcontainers (incluindo testes de concorrência de cupons, idempotência, cache Redis e LGPD).
5. **E2E Smoke & Critical Flows:** `cypress run` contra `docker-compose.test.yml`.
6. **Build & Bundle Check:** build Docker multi-stage de todas as 4 aplicações sem erros + verificação de `curl` SSG na Landing (incluindo `/termos-de-uso` e `/politica-de-privacidade`).
