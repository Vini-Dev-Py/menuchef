# Spec Geral 03 — Infraestrutura, Docker, Docker Compose e Traefik v3.7

> **Documento autocontido.** Especifica a topologia de containers, isolamento de redes, roteamento reverse proxy com **Traefik v3.7**, hardenização via `docker-socket-proxy`, serviços de dados (**PostgreSQL 19.0**, **Redis**, **Garage v2.5**), build multi-stage de imagens (Go e Vite/Nginx) e estratégia de ambientes (local dev, staging e produção).

---

## 1. Visão Geral da Topologia de Domínios (Same-Origin `/api`)

Para eliminar latência de *CORS preflight* (`OPTIONS`), permitir cookies de sessão `__Host-mc_session` (`HttpOnly; Secure; SameSite=Lax`) isolados por origem e manter cada frontend totalmente desacoplado, adotamos **Subdomínios por App com `/api` Same-Origin** roteados pelo **Traefik v3.7**:

| App / Serviço | Domínio Produção | Domínio Local (`.localhost`) | Roteamento Traefik |
| :--- | :--- | :--- | :--- |
| **Landing (SSG)** | `menuchef.com.br` | `menuchef.localhost` | `Host(\`menuchef.com.br\`)` → `landing:8080` |
| **Cardápio Digital** | `pedir.menuchef.com.br` | `pedir.menuchef.localhost` | `Host(\`pedir.menuchef.com.br\`)` → `storefront:8080` |
| **Painel Admin PWA** | `app.menuchef.com.br` | `app.menuchef.localhost` | `Host(\`app.menuchef.com.br\`)` → `admin:8080` |
| **API Go** | `*/api/*` nos 3 domínios | `*/api/*` nos 3 domínios | `(Host(...) \|\| Host(...) \|\| Host(...)) && PathPrefix(\`/api\`)` → `api:8081` |
| **Mídia Pública (WebP)** | `cdn.menuchef.com.br` | `cdn.menuchef.localhost` | `Host(\`cdn.menuchef.com.br\`)` → Garage S3 public endpoint (read-only) |

### Por que essa topologia?
1. **Zero CORS no fluxo principal:** a Landing envia leads para `/api/v1/public/leads`, o Storefront chama `/api/v1/public/*` e `/api/v1/customer/*`, e o Admin chama `/api/v1/admin/*` e `/api/v1/platform/*` — sempre na mesma origem.
2. **Prioridade automática do Traefik v3:** o Traefik calcula prioridade pelo tamanho da string `rule`, portanto `Host(...) && PathPrefix(\`/api\`)` sempre tem precedência sobre `Host(...)` sem números mágicos frágeis.
3. **Sem `StripPrefix`:** a API Go registra todas as rotas diretamente com prefixo `/api/v1/...`, evitando bugs de redirect e SSE.
4. **Regra v3:** nunca usar matchers multi-valor do Traefik v2 (`Host(\`a\`, \`b\`)` foi removido no v3); sempre usar operador booleano explícito `Host(\`a\`) || Host(\`b\`)`.

---

## 2. Isolamento de Redes & Segurança do Traefik

### Três Redes Isoladas no Docker
1. `proxy` (bridge externa): conecta **apenas** Traefik aos containers voltados para HTTP (`landing`, `storefront`, `admin`, `api`, endpoint público de mídia do `garage`).
2. `socket-tier` (bridge `--internal`, sem acesso à internet): conecta **apenas** `traefik` ao `docker-socket-proxy`.
3. `app-internal` (bridge `--internal` em produção): conecta `api` ↔ `postgres`, `redis` e porta interna/admin do `garage`. **Bancos de dados jamais entram na rede `proxy` e jamais publicam `ports:` no host em produção.** Há um único serviço Redis.

### Proibição de Montar `/var/run/docker.sock` no Traefik
Montar o socket do Docker diretamente no Traefik (mesmo `:ro`) concede privilégio equivalente a root no host em caso de comprometimento do proxy.
- **Solução obrigatória:** usar **`tecnativa/docker-socket-proxy`** (ou `wollomatic/socket-proxy`) na rede `socket-tier` com `CONTAINERS=1`, `NETWORKS=1`, `POST=0`, `BUILD=0`, `EXEC=0`.
- **Exposição explícita:** `--providers.docker.exposedbydefault=false` e `--providers.docker.network=proxy`.

---

## 3. Serviços de Dados e Versões Pinadas (Novembro/2026)

> **Regra de estabilidade:** nunca usar tag `:latest` em produção ou staging. Todas as imagens são pinadas por versão minor/patch verificada.

| Serviço | Imagem Recomendada | Justificativa & Configuração Crítica |
| :--- | :--- | :--- |
| **Proxy Reverso** | `traefik:v3.7` | Versões v3.0–v3.6 são EOL; v3.7 suporta HTTP/3 (QUIC), TLS ACME automático (`acme.json` com `chmod 600`) e access logs JSON estruturados |
| **Socket Proxy** | `tecnativa/docker-socket-proxy:0.3` | Isola leitura do Docker daemon em rede interna |
| **PostgreSQL** | `postgres:19.0-alpine` | Release estável de nov/2026 (suporte até nov/2031); traz **`uuidv7()` nativo**, `REPACK (CONCURRENTLY)`, AIO e extensões `unaccent` + `pg_trgm` para FTS PT-BR |
| **Redis (instância única)** | `redis:8.0-alpine` (`redis`) | Sessão `alexedwards/scs`, Pub/Sub SSE, rate-limit e cache na mesma instância, separados por prefixo (`mc:sess:`, `mc:cache:`, `mc:rl:`) e por canal (`mc:ps:`). Comando: `--appendonly yes --appendfsync everysec --save "" --maxmemory 256mb --maxmemory-policy volatile-ttl --client-output-buffer-limit "pubsub 32mb 8mb 60"`. `volatile-ttl` despeja primeiro a chave de TTL mais curto (cache e rate-limit) e preserva a sessão de 24h. `allkeys-lru` e a dupla `redis-state` + `redis-cache` ficam de fora |
| **Object Storage S3** | `dxflrs/garage:v2.5.0` | **MinIO CE foi arquivado em abr/2026** (imagens descontinuadas e CVEs sem patch). Garage v2.5 consome ~50 MB de RAM, suporta S3 (`PutObject`, `GetObject`, presigned URLs), sem credenciais default inseguras, atrás de `MediaStoragePort` (fácil troca para Cloudflare R2 / AWS S3 em produção cloud) |

### Detalhes Críticos de SSE no Traefik v3
Streams **Server-Sent Events (`text/event-stream`)** (fila do Kanban e rastreamento de pedido) exigem atenção especial no proxy:
- **NÃO aplicar middleware `compress` nas rotas `/api/v1/*/stream`** — compressão gzip faz buffer de pequenos payloads JSON e quebra a entrega em tempo real.
- Criar dois routers/middlewares distintos para a API:
  - `chain-api`: `sec-headers` + `api-ratelimit` + `compress-default` (rotas REST JSON normais).
  - `chain-api-sse`: `sec-headers` + `sse-ratelimit` (sem compressão; `flushInterval: -1` ou `100ms` no `serversTransport` + header `X-Accel-Buffering: no` emitido pelo Go).
- API Go desabilita `WriteTimeout` apenas no handler SSE via `http.ResponseController.SetWriteDeadline(time.Time{})` e envia heartbeat `: ping\n\n` a cada 15 segundos.

---

## 4. Middlewares Centralizados (`infra/traefik/dynamic/middlewares.yml`)

Definidos uma única vez via File Provider (`@file`) para evitar duplicação de labels:

- **`sec-headers`:**
  - `stsSeconds: 31536000`, `stsIncludeSubdomains: true`, `stsPreload: true`
  - `contentTypeNosniff: true`
  - `frameDeny: true`
  - `referrerPolicy: "strict-origin-when-cross-origin"`
  - Remoção de headers `Server` e `X-Powered-By`
  - CSP específica por app (ex.: permitindo `img-src` do domínio `cdn.menuchef.com.br` e `blob:`/`data:` onde necessário, bloqueando `unsafe-eval`).
- **`api-ratelimit`:** `average: 100`, `burst: 50` por IP (complementado por rate limit mais estrito por rota sensível no prefixo `mc:rl:` da instância Redis única, dentro da API Go).
- **`compress-default`:** Gzip/Brotli para assets estáticos e respostas JSON (excluído das rotas SSE).
- **Dashboard Traefik:** `--api.insecure=false` em produção; acessível apenas via túnel SSH ou rota protegida com `basicAuth` + `ipAllowList`.

---

## 5. Padrão de Dockerfiles Multi-Stage & Containers Non-Root

### 5.1 Backend Go (`apps/api/Dockerfile`)
- **Stage 1 (`builder`):** `golang:1.27.4-alpine`, `CGO_ENABLED=0`, `go mod download` com cache de camada, build estático com `-trimpath -ldflags="-s -w"`.
- **Stage 2 (`runtime`):** `gcr.io/distroless/static-debian12:nonroot` (ou `alpine` minimal non-root UID `10001`), contendo apenas o binário `/app/api`, CA certificates e **tzdata completo** (para `time.LoadLocation` resolver `America/Sao_Paulo`, `America/Manaus` e as demais zonas IANA). O processo roda com `TZ=UTC`. O fuso de negócio é `branches.timezone`, nunca o fuso do container e nunca `time.Local`.
- **Healthcheck:** subcomando interno `/app/api --healthcheck` chamando `http://127.0.0.1:8081/api/v1/health/live` (evita precisar de `curl`/`wget` dentro de imagem distroless).
- **Graceful Shutdown:** escuta `SIGINT`/`SIGTERM`, drena conexões HTTP ativas e encerra streams SSE com timeout de 15s antes de fechar `pgxpool` e o cliente único do Redis.

### 5.2 Frontends Vite / TanStack Start (`apps/{landing,storefront,admin}/Dockerfile`)
- **Stage 1 (`builder`):** `node:24-slim` + `pnpm` (Corepack), build de produção (`pnpm build`).
  - `apps/landing`: executa SSG/Prerender gerando HTML completo de todas as rotas (incluindo `/termos-de-uso` e `/politica-de-privacidade`), `/llms.txt`, arquivos `.md`, `/sitemap.xml`, `/robots.txt` e assets **WebP**.
  - `apps/storefront` e `apps/admin`: geram bundle estático otimizado (`apps/admin` inclui Service Worker `sw.js` compilado pelo `vite-plugin-pwa`).
- **Stage 2 (`runtime`):** `nginxinc/nginx-unprivileged:1.27-alpine` escutando na porta `8080` (non-root).
- **Regras obrigatórias do `nginx.conf`:**
  - `apps/storefront` e `apps/admin` (SPA): `try_files $uri $uri/ /index.html;` + `Cache-Control: no-cache, no-store, must-revalidate` em `/index.html` e `/sw.js` + `Cache-Control: public, max-age=31536000, immutable` em `/assets/`.
  - `apps/landing` (SSG): `try_files $uri $uri/ $uri.html =404;` — **retorna HTTP `404` real** para rotas inexistentes (nunca Soft 404 com `200 OK`!), garantindo saúde de SEO/AEO/GEO.
- **Variáveis de ambiente:** frontends usam caminho relativo `/api/v1` (mesma origem via Traefik), permitindo promover a **mesma imagem Docker** de staging para produção sem rebuild.

---

## 6. Estratégia de Ambientes (`docker-compose`)

| Arquivo | Uso | Características |
| :--- | :--- | :--- |
| `infra/docker-compose.base.yml` | Definição comum dos serviços (`postgres`, `redis`, `garage`, `api`, `landing`, `storefront`, `admin`) | Sem portas expostas no host para bancos; variáveis via `.env`. Um serviço `redis`, não dois |
| `infra/docker-compose.dev.yml` | Desenvolvimento local (override) | Traefik em HTTP (`*.menuchef.localhost`), Vite dev servers com HMR opcional, `air` (hot-reload Go) opcional, portas locais de debug (`127.0.0.1:5432`, `127.0.0.1:6379`) e dashboard Traefik em `127.0.0.1:8080` |
| `infra/docker-compose.test.yml` | Execução de suite **Cypress E2E** em CI | Stack completa buildada em modo produção, banco PostgreSQL limpo/efêmero (`tmpfs`), seed determinístico e container Cypress headless |
| `infra/docker-compose.prod.yml` | Produção / Staging | Traefik v3.7 com TLS Let's Encrypt (`acme.json` `0600`), HTTP/3, `restart: unless-stopped`, `security_opt: [no-new-privileges:true]`, limites de `mem_limit`/`cpus` por container e zero portas de banco publicadas |

---

## 7. Backup, Persistência e Operação

- **Volumes nomeados explícitos:** `pg_data`, `redis_data` (AOF da instância única), `garage_meta`, `garage_data`, `traefik_acme`.
- **Política de backup (Produção):** dump lógico diário (`pg_dump --format=custom`) + retenção off-site (S3/R2 externo via `rclone`) e teste periódico de restore em ambiente efêmero.
- **Migrations em deploy:** container/job efêmero dedicado rodando `goose up` com credencial `menuchef_migrator` **antes** do rollout da nova versão da API (`menuchef_app`).

---

## 8. Critérios de Aceite & Plano de Testes da Infraestrutura

- [ ] `docker compose -f infra/docker-compose.base.yml -f infra/docker-compose.dev.yml up -d` sobe 100% dos serviços saudáveis (`healthy`) em menos de 60s em máquina limpa.
- [ ] Containers `postgres`, `redis` e admin do `garage` não estão acessíveis fora da rede `app-internal` em `prod`. Não existe serviço `redis-state` nem `redis-cache`.
- [ ] Container `traefik` não possui acesso direto a `/var/run/docker.sock` (somente via `docker-socket-proxy` na rede `socket-tier`).
- [ ] O serviço `redis` roda com `appendonly yes`, `appendfsync everysec`, `save ""`, `maxmemory-policy volatile-ttl` e um único volume `redis_data`. `CONFIG GET maxmemory-policy` devolve `volatile-ttl`. A API sobe com `TZ=UTC` e consegue carregar `America/Manaus`.
- [ ] Teste automatizado valida:
  - `GET https://menuchef.localhost/` → `200 OK` (HTML prerenderizado) e `GET https://menuchef.localhost/rota-inexistente` → `404 Not Found`;
  - `GET https://pedir.menuchef.localhost/api/v1/health/ready` → `200 OK` (validando PG 19 + a instância Redis + Garage);
  - Stream SSE `/api/v1/health/sse-smoke` (ou rota SSE autenticada) entrega eventos em < 200ms sem buffering do Traefik;
  - Todos os containers rodam como usuário non-root (`id -u` != `0`).
