# Especificações do Menu Chef

Índice mestre das especificações técnicas, arquiteturais e de produto do **Menu Chef** (SaaS multi-empresa e multi-filial de cardápio digital e gestão de delivery/retirada).

> **Princípio de independência:** Nenhuma spec depende de outra para ser compreendida, planejada ou validada. Cada documento é **autocontido** (escopo, contratos explícitos, decisões com trade-offs, regras de negócio, critérios de aceite e plano de testes). Este `README.md` atua apenas como mapa de navegação.

---

## Mapa de Especificações

### Specs Gerais (`specs/geral/`)

| Documento | Responsabilidade |
| :--- | :--- |
| [01-visao-produto-e-marcos.md](geral/01-visao-produto-e-marcos.md) | Visão de produto derivada de `docs/pesquisa/`, perfis, onboarding assistido, escopo IN/OUT e **Marcos Sólidos (M0–M6)** com Definition of Done |
| [02-arquitetura-e-padroes.md](geral/02-arquitetura-e-padroes.md) | Clean Architecture, CQRS, SOLID, Object Calisthenics, Design Patterns, IDs (`UUIDv7`), Paginação Cursor/Keyset, Sessão `HttpOnly`, SQLi, **Cache Redis**, **Idempotência & Cupons Atômicos**, **Transactions & Locks (`pg_advisory_xact_lock`, Optimistic Locking, Split-Tx)**, **OpenAPI 3.1 (`Huma v2`)** e **LGPD** |
| [03-infra-docker-traefik.md](geral/03-infra-docker-traefik.md) | Docker, Docker Compose, Traefik v3.7 com socket-proxy, PostgreSQL 19.0, **Redis isolado (`redis-state` + `redis-cache`)**, Garage v2.5 (S3), isolamento de redes e topologia same-origin |
| [04-identidade-visual-e-ux.md](geral/04-identidade-visual-e-ux.md) | Paleta Brasa clara/escura, marca Concha com vapor, Mobile-First (mobile, tablet, desktop), acessibilidade WCAG 2.2 AA, Tailwind CSS v4, shadcn/ui v4 (Base UI), Motion e React Joyride v3 |
| [05-qualidade-testes-observabilidade.md](geral/05-qualidade-testes-observabilidade.md) | Pirâmide de testes (Go unit + Testcontainers concorrência/locks/idempotência/LGPD, Vitest + RTL, Cypress E2E), padronização de erros RFC 9457, observabilidade (`log/slog`, `pg_stat_lock`) e gates de CI (OpenAPI/TS drift) |

### Specs por Aplicação (`specs/apps/`)

| Aplicação | Domínio (Produção) | Documento | Resumo |
| :--- | :--- | :--- | :--- |
| **API** (`apps/api`) | `/api/*` (same-origin em todos os domínios) | [apps/api/SPEC.md](apps/api/SPEC.md) | Backend **Go 1.27.4**, `chi` v5 + **`Huma v2`** (OpenAPI 3.1), `pgx/v5` + `sqlc` + `go-jet` + `goose`, **PostgreSQL 19.0** FTS, **`redis-state` + `redis-cache`**, **Locks/Transactions (`pg_advisory_xact_lock` + Split-Tx)**, Idempotência, Cupons Atômicos, LGPD, Garage S3 e `MockPaymentAdapter` |
| **Landing** (`apps/landing`) | `menuchef.com.br` | [apps/landing/SPEC.md](apps/landing/SPEC.md) | **TanStack Start (SSG/Prerender)**, React 19, Tailwind v4, Motion, WebP, **SEO/AEO/GEO**, `llms.txt` v2 + `.md`, `sitemap.xml`, `robots.txt`, páginas SSG **`/termos-de-uso` e `/politica-de-privacidade` (LGPD B2B2C)** e leads |
| **Storefront** (`apps/storefront`) | `pedir.menuchef.com.br/{filial}` | [apps/storefront/SPEC.md](apps/storefront/SPEC.md) | Cardápio digital Mobile-First (React 19 + Vite + TanStack Router/Query/Form + `openapi-fetch`), FTS, adicionais, sacola persistida, checkout com **`Idempotency-Key`**, pagamento mocado/na hora, tracking SSE e **autoatendimento LGPD** |
| **Admin PWA** (`apps/admin`) | `app.menuchef.com.br` | [apps/admin/SPEC.md](apps/admin/SPEC.md) | Painel **PWA** (`vite-plugin-pwa` v2 + TanStack Table v9 + `openapi-fetch` + React Joyride v3) para **Empresas/Filiais** (troca de contexto sem relogar, Kanban SSE, som híbrido + Web Push + Wake Lock, modo escuro, cardápio/cupons/CEP, impressão 1-clique) e **Admin da Plataforma** (onboarding assistido, aceite legal e suporte auditado) |

---

## Resumo dos Marcos Sólidos (M0–M6)

| Marco | Nome | Escopo Principal | Independência / Entrega Verificável |
| :--- | :--- | :--- | :--- |
| **M0** | Fundação, Infra & Contratos | Monorepo, Docker Compose, Traefik v3.7, PostgreSQL 19.0, `redis-state` + `redis-cache`, Garage v2.5, `Huma v2` OpenAPI 3.1 + `openapi-typescript`, skeletons de CI e harnesses de teste | Ambiente sobe com 1 comando; healthchecks, roteamento same-origin e drift check OpenAPI/TS verdes |
| **M1** | Landing Page, LGPD & Captação de Leads | `apps/landing` SSG/Prerender, WebP, SEO/AEO/GEO, `llms.txt` v2, `sitemap.xml`, `robots.txt`, **`/termos-de-uso` e `/politica-de-privacidade` (LGPD B2B2C)** e endpoint de lead com aceite versionado | `curl` entrega HTML completo; Lighthouse >= 95; lead + aceite legal persistidos e testados E2E |
| **M2** | Identidade, Multi-tenancy & Onboarding Assistido | Sessão `redis-state` `HttpOnly`, RBAC Empresa/Filial, Admin da Plataforma criando empresa + 1ª filial + convite, ledger `legal_acceptances`, seletor de contexto no Painel PWA | 1 login acessa N empresas e M filiais com papéis distintos sem relogar |
| **M3** | Cardápio da Filial, FTS, Cupons & CEP | CRUD de categorias/produtos/adicionais, upload WebP (Garage), cache `redis-cache` (`singleflight` + invalidação `UNLINK`), FTS (`pt_unaccent + pg_trgm`), CEP, horários/pausa e `apps/storefront` | Cliente navega e busca no celular sem instalar app; loja pausada bloqueia compra |
| **M4** | Sacola, Checkout, Idempotência & Cupons Atômicos | Sacola persistida, entrega/retirada, validação de CEP, **`Idempotency-Key` (3 estados)**, **concorrência atômica de cupons + estorno automático**, pagamento mocado (Pix/Cartão) e na hora | Pedido fechado ponta a ponta; testes de concorrência (50 goroutines) e idempotência verdes |
| **M5** | Operação da Cozinha em Tempo Real (PWA) | Kanban SSE + `redis-state` Pub/Sub, estados explícitos da cozinha, cancelamento pré-confirmação (com estorno de cupom), alerta sonoro híbrido + Wake Lock + Web Push, modo escuro, impressão 1-clique e React Joyride v3 | Pedido aparece na fila < 1s, muda status sincronizado com o cliente e imprime comanda |
| **M6** | Histórico, Dashboard, LGPD Cliente & Hardening | Histórico do cliente ("pedir de novo"), **exportação JSON + exclusão/anonimização LGPD**, dashboard operacional da filial, extrato de pagamentos mocados, suporte auditado e suite E2E completa | Critérios de segurança (OWASP/SQLi/CSRF/LGPD), performance e E2E 100% verdes |

---

## Stack Consolidada (Referência: Novembro/2026)

| Camada | Escolha | Versão / Padrão |
| :--- | :--- | :--- |
| **Backend & Contratos** | Go + `chi` v5 + **`Huma v2` (`humachi`)** + `pgx/v5` + `sqlc` + `go-jet` + `goose` | Go `1.27.4` (`uuid.NewV7`, `encoding/json/v2`, `http.CrossOriginProtection`), OpenAPI 3.1 + RFC 9457 |
| **Banco de Dados** | PostgreSQL | `19.0` (nativo `uuidv7()`, `unaccent` + `pg_trgm` FTS PT-BR, Keyset Pagination, `REPACK CONCURRENTLY`) |
| **Estado / Cache / PubSub** | **2 instâncias Redis isoladas** (`redis-state` + `redis-cache`) | `8.0-alpine` (`redis-state`: `noeviction` + AOF para `scs`/PubSub/rate-limit; `redis-cache`: `allkeys-lru` + `singleflight` + jitter) |
| **Object Storage** | Garage (S3-compatible) atrás de `MediaStoragePort` | `v2.5.x` (substitui MinIO CE arquivado em abr/2026) |
| **Frontend & SDK Tipado** | React 19 + TypeScript + Vite + Tailwind CSS v4 + **`openapi-typescript` / `openapi-fetch`** | `shadcn/ui` v4 (`@base-ui/react`), Motion (`reducedMotion="user"`), React Joyride `3.2.0` |
| **Ecossistema TanStack** | Start (Landing SSG), Router, Query v5, Table v9 (`manualPagination`), Form + Zod | 100% type-safe de ponta a ponta (Go → OpenAPI 3.1 → TypeScript) |
| **PWA** | `vite-plugin-pwa` v2 (`injectManifest`, `registerType: 'prompt'`) | Workbox 7, Screen Wake Lock, Web Push + som híbrido em foreground |
| **Infra & Proxy** | Docker + Docker Compose + Traefik + `docker-socket-proxy` | Traefik `v3.7`, TLS automático, HTTP/3, redes isoladas |
| **Testes** | Go `testing` + `testcontainers-go` · Vitest + RTL · Cypress | Pirâmide completa (Unit + Integration + Concorrência + E2E) |
