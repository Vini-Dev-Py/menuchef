# Spec Geral 02 — Arquitetura, Padrões de Projeto, IDs e Segurança

> **Documento autocontido.** Define as diretrizes arquiteturais obrigatórias para backend e frontends do Menu Chef: Clean Architecture, CQRS, SOLID, Object Calisthenics, Design Patterns com critérios objetivos de uso, estratégia de identificadores (`UUIDv7`), paginação 100% Cursor Opaco + Keyset, segurança de sessão (`HttpOnly`, zero `localStorage`), defesa contra SQL Injection e IDOR multi-tenant, **uma instância Redis** (sessão, Pub/Sub, rate-limit e cache), **fuso IANA da filial**, idempotência (`Idempotency-Key`), concorrência atômica de cupons, **Transactions & Locks (`pg_advisory_xact_lock`, Optimistic Locking, Split-Tx)**, contratos OpenAPI 3.1 (`Huma v2`) e LGPD B2B2C. Modelo comercial: **sem comissão, apenas mensalidade** (não existe taxa da plataforma por pedido ou por pagamento).

---

## 1. Arquitetura Macro (Modular Monolith + 4 Aplicações Separadas)

O sistema é organizado em **4 aplicações independentes** sob um mesmo repositório (facilitando governança de contratos e testes E2E), onde cada aplicação possui build, deploy e ciclo de vida desacoplados:

```text
Menu Chef/
├── apps/
│   ├── api/          # Backend Go 1.27.4 (Modular Monolith: Clean Architecture + CQRS)
│   ├── landing/      # TanStack Start (SSG/Prerender — SEO/AEO/GEO + llms.txt)
│   ├── storefront/   # React 19 + Vite (Cardápio Digital Mobile-First)
│   └── admin/        # React 19 + Vite + PWA (Painel Empresa/Filial + Admin Plataforma)
├── infra/            # Docker Compose, Traefik v3.7, middlewares dinâmicos, Garage config
├── tests/e2e/        # Suite Cypress E2E cross-app (isolada das unitárias)
└── specs/            # Documentação técnica independente
```

### Por que Modular Monolith em Go na API (e não microsserviços)?
- **Trade-off consciente (YAGNI + Evolução):** microsserviços prematuros introduzem latência de rede, transações distribuídas (Saga/2PC) para fechar pedido/cupom/pagamento e custo operacional elevado sem necessidade de escala independente de times.
- **Módulos de domínio isolados dentro de `apps/api/internal/modules/`:** `identity`, `platform`, `catalog`, `delivery`, `order`, `payment`, `realtime`, `analytics`, `lead`. Cada módulo só se comunica com outro via **interfaces explícitas (Ports)** ou **Domain Events internos**, permitindo extrair um módulo para serviço separado no futuro caso a carga exija.

---

## 2. Clean Architecture & Separação de Camadas

Regra de dependência estrita: **dependências sempre apontam para dentro (`presentation`/`infrastructure` → `application` → `domain`)**. O `domain` não importa frameworks HTTP, drivers SQL, bibliotecas de sessão ou SDKs externos.

### Estrutura por Módulo na API (`apps/api/internal/modules/<modulo>/`)
```text
<modulo>/
├── domain/          # Entities, Value Objects, Domain Errors, Invariants, Repository/Port interfaces
├── application/
│   ├── commands/    # Write UseCases (CreateOrderCommand + CreateOrderHandler)
│   ├── queries/     # Read UseCases (SearchProductsQuery + SearchProductsHandler)
│   └── dto/         # Inputs/Outputs explícitos (nunca vazar Entity de domínio para HTTP)
├── infrastructure/
│   ├── postgres/    # Implementação de Repositories (sqlc + go-jet) e Read Models
│   ├── redis/       # Cache, Pub/Sub, locks
│   └── adapters/    # PaymentGatewayAdapter, MediaStorageAdapter
└── presentation/
    └── http/        # Handlers chi/net/http, request decoding/validation, mapeamento de erro HTTP
```

### Estrutura nos Frontends (`apps/{landing,storefront,admin}/src/`)
Seguimos **Feature-Sliced / Clean Boundaries** sem overengineering:
- `features/<feature>/api/` — chamadas HTTP tipadas + queries/mutations TanStack Query (camada de infraestrutura/adapter).
- `features/<feature>/model/` — tipos de domínio de UI, schemas **Zod**, máquinas de estado e regras puras (testáveis sem DOM).
- `features/<feature>/ui/` — componentes de apresentação puros consumindo primitivas `shadcn/ui` (Base UI).
- `routes/` — definição fina de rotas file-based do TanStack Router (loaders pré-carregam dados no `QueryClient`).
- `shared/` — design system tokens, `httpClient`, tratamento global de erro, utilitários de formatação monetária (`BRL` em centavos).

> **Regra de Dinheiro:** Valores monetários **nunca** usam `float`/`float64` ou `number` decimal com ponto flutuante. No banco (`BIGINT`), na API (Go `int64` encapsulado em Value Object `Money`) e nos DTOs JSON, todos os valores trafegam em **centavos inteiros** (`amount_cents: 3490` = R\$ 34,90), formatados apenas na camada de apresentação.

---

## 3. CQRS (Command Query Responsibility Segregation)

Aplicado no nível de **Application / Data Access** (sem bancos físicos separados no MVP — ambos usam PostgreSQL 19.0, com leitura otimizada e cache Redis onde fizer sentido):

| Lado | Responsabilidade | Implementação na API Go |
| :--- | :--- | :--- |
| **Commands (Write)** | Alterar estado validando todas as invariantes de domínio (ex.: `CreateOrderCommand`, `ConfirmOrderCommand`, `SwitchBranchContextCommand`) | Passam obrigatoriamente pelas Entities/Value Objects do `domain`, rodam em transação (`WithTx`) via `sqlc`, emitem eventos internos (ex.: `OrderStatusChanged` → Redis Pub/Sub) |
| **Queries (Read)** | Retornar projeções enxutas otimizadas para tela (ex.: `SearchBranchMenuQuery`, `GetBranchKanbanQuery`, `GetBranchDashboardQuery`) | Bypassam reconstrução pesada de Aggregates de escrita; consultam Read Models diretos via `sqlc` ou `go-jet` (JOINs enxutos, índices GIN de FTS, cursor pagination) e retornam DTOs específicos |

---

## 4. SOLID, Clean Code e Object Calisthenics

### SOLID aplicado na prática
- **S (Single Responsibility):** 1 UseCase = 1 arquivo/struct (`ConfirmOrderHandler`), 1 responsabilidade de negócio. Sem "God Services" (`OrderService` com 40 métodos).
- **O (Open/Closed):** novos meios de pagamento (Asaas, Mercado Pago), tipos de frete (fixo, faixa de CEP, futuro raio/km) ou tipos de cupom (valor fixo, percentual, frete grátis) entram via novas implementações de interface (**Strategy/Adapter**) sem alterar o fluxo principal do checkout.
- **L (Liskov Substitution):** `MockPaymentAdapter` e futuros gateways reais honram exatamente o mesmo contrato de erros, idempotência e estados definidos em `PaymentGatewayPort`.
- **I (Interface Segregation):** interfaces pequenas definidas **no consumidor** (idioma Go): ex., `OrderReader`, `OrderWriter`, `BranchScheduleChecker` em vez de uma interface `OrderRepository` gigante com 25 métodos.
- **D (Dependency Inversion):** Composition Root em `cmd/api/main.go` injeta `*pgxpool.Pool`, Redis, Garage e Adapters via construtores explícitos (`NewCreateOrderHandler(...)`). Zero variáveis globais ou `init()` com estado.

### Object Calisthenics (Norte prático, não dogma)
1. **Máximo 1 nível de indentação por método/função** sempre que possível — extrair subfunções coesas para loops/transformações.
2. **Proibido `else` desnecessário** — usar **guard clauses** e *early return* (`if err != nil { return ... }`).
3. **Evitar Primitive Obsession em conceitos críticos de domínio:** criar Value Objects com invariantes validadas na construção:
   - `Money` (centavos inteiros, nunca negativo para preço);
   - `CEP` (normalizado 8 dígitos numéricos, validação de formato e comparação de faixas);
   - `PhoneBR` (normalizado E.164/DDD+número);
   - `IANATimezone` (nome IANA carregável, default `America/Sao_Paulo`);
   - `Email`, `Slug`, `OrderStatus` (máquina de estados com método `CanTransitionTo(next)`).
4. **Coleções encapsuladas no domínio:** ex., `OrderItems` garante que não existe pedido vazio, que adicionais obrigatórios foram preenchidos e calcula o subtotal de forma consistente.
5. **Métodos curtos e coesos:** alvo de até ~30 linhas por função; funções de validação e cálculo puras, sem I/O misturado.

---

## 5. Design Patterns — Quando Usar e Quando NÃO Usar

| Pattern | Onde USAR no Menu Chef | Quando NÃO usar (evitar overengineering) |
| :--- | :--- | :--- |
| **Adapter** | `PaymentGatewayPort` (`MockPaymentAdapter` no MVP → futuros gateways), `MediaStoragePort` (Garage S3 → R2/AWS S3), `NotificationPort` (Web Push / futuro WhatsApp) | Não criar Adapter em cima de pacotes estáveis da stdlib (`slog`, `net/http`, `uuid`) |
| **Strategy** | Cálculo de frete (`FixedShippingStrategy`, `CepRangeShippingStrategy`, `PickupZeroShippingStrategy`) e cálculo de desconto de cupom (`FixedDiscountStrategy`, `PercentageDiscountStrategy`, `FreeShippingStrategy`) | Não usar Strategy para um `if` binário que não tem variação real de regra de negócio |
| **Factory** | Criação do agregado `Order` (valida loja aberta, itens, adicionais obrigatórios, estoque/disponibilidade, frete, cupom e estado inicial conforme modo de pagamento) | Não criar Factories vazias para DTOs simples ou structs sem invariantes |
| **Template Method / Pipeline** | Pipeline de processamento de upload de imagem (validar MIME/magic bytes → checar dimensões/tamanho → converter para WebP → gravar no `MediaStoragePort`) e cadeia de middlewares HTTP (`chi`: RequestID → Logger → Recovery → RateLimit → CrossOriginProtection → AuthN → TenantAuthZ) | Não criar frameworks genéricos de pipeline para fluxos lineares de 2 passos |
| **Decorator** | Cache de leitura no Redis envolvendo `BranchMenuReadRepository` e `MembershipPermissionReader` (com invalidação explícita em escrita), além de telemetria/métricas em Ports críticos | Não empilhar 4+ decorators onde uma composição simples resolve |

---

## 6. Decisão Arquitetural: Auto-incremento vs `UUIDv4` vs `UUIDv7`

### Comparativo Técnico (PostgreSQL 19.0 + Go 1.27.4)

| Dimensão | `BIGINT GENERATED ALWAYS AS IDENTITY` | `UUIDv4` (`uuidv4()`) | `UUIDv7` (`uuidv7()` — RFC 9562) |
| :--- | :--- | :--- | :--- |
| **Armazenamento** | 8 bytes | 16 bytes | 16 bytes |
| **B-tree & WAL no PostgreSQL** | Excelente: append sempre na folha mais à direita | Ruim em alto volume: inserção aleatória provoca *page splits*, baixa taxa de cache hit e *WAL Full-Page Images* | **Excelente:** prefixo temporal de 48 bits (ms) + 12 bits sub-ms mantém append sequencial na borda direita da B-tree |
| **Clean Architecture / CQRS** | Fraco: domínio depende de round-trip ao DB para conhecer o ID | Bom: gerado no `domain` antes de persistir | **Excelente:** gerado no `domain` via Go 1.27 stdlib `uuid.NewV7()` ou no DB via PG 19 nativo `DEFAULT uuidv7()` |
| **Segurança (Enumeração / IDOR)** | Alto risco: expõe contagem real de pedidos/empresas (`/orders/105`) e facilita scraping | Seguro: imprevisível | Não expõe contagem de volume, **mas revela o timestamp (ms) de criação** |
| **Paginação por Cursor** | Sim | Não (sem ordem cronológica) | **Sim:** ordenação natural por PK sem `OFFSET` caro (`WHERE id < $cursor ORDER BY id DESC LIMIT 20`) |

### Padrão Híbrido Obrigatório do Menu Chef
1. **Primary Key de todas as entidades (`id`):** **`UUIDv7`** (`uuid PRIMARY KEY DEFAULT uuidv7()` no PostgreSQL 19.0; gerado no domínio Go via `uuid.NewV7()`).
2. **Identificador humano do pedido (`display_number`):** **`INTEGER` sequencial por filial** (ex.: `#101`, `#102`) com constraint `UNIQUE (branch_id, display_number, business_date)`. `business_date` é a data civil de `created_at` no fuso IANA da filial, não a data UTC. A cozinha e o balcão nunca leem um UUID de 36 caracteres.
3. **Tokens, sessões, convites e links públicos de acompanhamento (`tracking_token`):** **NUNCA usar `UUIDv7`** (por conter timestamp). Gerar com `crypto/rand` (mínimo 32 bytes → base64url sem padding = 256 bits de entropia) e armazenar apenas o hash SHA-256 no banco para convites/tokens de uso único.

### 6.1 Fuso da filial (IANA) e instantes em UTC

`branches.timezone` é `TEXT NOT NULL DEFAULT 'America/Sao_Paulo'`. O valor é nome IANA (`America/Sao_Paulo`, `America/Manaus`, `America/Noronha`, `America/Cuiaba`). Offset cru (`-04:00`) e apelido (`Brasilia`) são rejeitados pelo Value Object `IANATimezone` (`time.LoadLocation`) antes de persistir.

`opening_hours_json` guarda relógio de parede da filial, sem offset (`{"mon":[{"open":"10:00","close":"22:00"}]}`). Faixa que cruza a meia-noite (18:00–02:00) é avaliada nesse fuso. "Aberto agora", o texto de abre/fecha, a mensagem de reabertura depois da pausa, o dia do dashboard e o horário da comanda e do comprovante usam esse fuso. A pausa em si é imediata e não depende do relógio.

Instantes (`created_at`, `confirmed_at`, `expires_at`, auditoria) continuam `timestamptz` em UTC. Não usar `timestamp without time zone` para evento. O processo da API roda com `TZ=UTC`; a imagem traz `tzdata` completo. A conversão é explícita (`created_at AT TIME ZONE branches.timezone` ou `time.LoadLocation`). Nunca `time.Local`.

`orders.business_date` é `(created_at AT TIME ZONE branches.timezone)::date`. Ela entra na unicidade do `display_number` e no corte de "pedidos do dia". O payload público da filial traz `timezone`, `is_open` e o próximo horário já calculados no servidor. O browser formata o relógio com esse fuso; não é a fonte de verdade do "aberto agora".

Casos que a implementação tem de cobrir, com relógio injetável:
1. Grade 10:00–22:00. Instante `2026-11-23T01:30:00Z`: Manaus (`UTC−4`) está em 21:30 do dia 22 e **aberta**; São Paulo (`UTC−3`) está em 22:30 do dia 22 e **fechada**.
2. Virada do dia. Pedido `created_at = 2026-11-23T03:30:00Z`: `business_date` em Manaus é `2026-11-22` (23:30); em São Paulo é `2026-11-23` (00:30). O dashboard do dia 23 inclui o pedido só na filial de São Paulo.
3. Criar filial sem `timezone` grava `America/Sao_Paulo`. `GMT-4` e `Brasilia` voltam `422`.

---

## 7. Estratégia de Paginação: 100% Cursor Opaco + Keyset Pagination

> **Decisão fechada:** **100% dos endpoints paginados da API usam Cursor Opaco (Base64url) sobre Keyset Pagination no PostgreSQL 19.0**, eliminando `OFFSET` e `COUNT(*)` pesado em toda a plataforma.

### Cursor vs Keyset vs Offset (Trade-offs)
- **Keyset Pagination (Seek Method)** é a técnica SQL (`WHERE id < $1` ou `WHERE (sort_col, id) < ($1, $2)`): complexidade **$O(\log(\text{rows}) + N)$** constante em qualquer profundidade e **imune a *row drift*** (novos pedidos chegando no topo não duplicam nem pulam linhas durante a navegação).
- **Cursor Pagination** é o contrato HTTP por cima do Keyset: serializa a tupla da última linha em um token opaco Base64url (`?cursor=...&limit=20`), desacoplando clientes do schema interno do banco.
- **Por que NÃO usar `OFFSET`:** `LIMIT N OFFSET M` exige $O(M + N)$ (PostgreSQL lê e descarta $M$ linhas), sofre de *row drift* em dados vivos e força `COUNT(*)` custoso para montar botões numerados.

### Regras Obrigatórias de Implementação (PostgreSQL 19 + Go + TanStack Table v9)
1. **Contrato HTTP único de listagem:**
   - Query params: `?cursor=<base64url>&limit=<int>` (`limit` default `20`, máximo absoluto `100` — rejeitar `limit <= 0` ou `limit > 100` com `400`).
   - Envelope JSON:
     ```json
     {
       "data": [],
       "page_info": {
         "next_cursor": "eyJ2IjoxLCJpZCI6IjAxOWEuLi4ifQ",
         "has_next_page": true,
         "page_size": 20
       }
     }
     ```
2. **Detecção de próxima página sem `COUNT(*)`:** consultar sempre `LIMIT page_size + 1`. Se retornar `page_size + 1` linhas, descartar a última, marcar `has_next_page: true` e gerar `next_cursor` a partir do último item efetivamente retornado.
3. **Separar query da Página 1 da query com Cursor:** **nunca** escrever `WHERE $1 IS NULL OR (col, id) < ($1, $2)` em prepared statements (`pgx`/`sqlc`), pois o planner do PostgreSQL muda para *generic plan* após ~5 execuções e degrada o Index Seek para Index Scan filtrado.
4. **Direção uniforme na comparação de tuplas (`row-value comparison`):** `(sort_col, id) < ($1, $2)` só vira B-tree Seek no PostgreSQL se ambas as colunas compartilharem a mesma direção (`DESC, DESC` ou `ASC, ASC`) alinhada ao índice composto `(branch_id, sort_col DESC, id DESC)`.
5. **Colunas ordenáveis `NOT NULL` + tipos exatos:** `ROW(...) < ROW(...)` com `NULL` avalia para `UNKNOWN` e descarta linhas silenciosamente; timestamps (quando usados fora do `UUIDv7`) são serializados com precisão de microssegundos (`::text`) dentro do cursor para evitar truncamento em JS.
6. **Validação estrita do cursor:** o backend valida estrutura/schema do Base64url decodificado e garante compatibilidade com a ordenação solicitada (`sort_by`, `sort_dir`); cursor corrompido ou adulterado retorna `400 Bad Request` (`INVALID_CURSOR`).
7. **Integração nos frontends (TanStack Table v9 + TanStack Query v5):** usar `useTable` com `manualPagination: true`, `pageCount: -1` e `useInfiniteQuery` mantendo a cadeia de cursores em cache para navegação instantânea **Anterior / Próxima** (resetando `pageIndex` para `0` sempre que filtro, busca ou ordenação mudar).

---

## 8. Segurança de Sessão, Cookies `HttpOnly` e Proibição de `localStorage`

### Regra Inegociável
> **Nenhuma credencial, token de autenticação, session ID ou JWT pode ser armazenado em `localStorage` ou `sessionStorage`.** Qualquer vulnerabilidade XSS permitiria exfiltração imediata de credenciais armazenadas em Web Storage.

### Por que Sessão Opaca no Redis (`HttpOnly` Cookie) em vez de JWT Stateless?
No Menu Chef, um usuário troca de **empresa e filial** frequentemente e pode ter permissões alteradas ou revogadas no meio do expediente.
- **JWT stateless** sofre de *permission drift* (permissão antiga segue válida até o `exp`) e exige re-emissão a cada troca de filial ou uma denylist no Redis (que anula o benefício stateless).
- **Sessão opaca no Redis (`alexedwards/scs`)** em cookie `HttpOnly` entrega:
  - **Revogação instantânea** (suspensão de empresa, demissão de funcionário ou logout global apaga a chave no Redis);
  - **Zero permission drift:** a sessão guarda apenas `{ user_id, active_company_id, active_branch_id, authenticated_at }`; papéis/permissões vêm de cache curto no Redis invalidado imediatamente em mudanças de vínculo;
  - **Rotação de Session ID** obrigatória no login e em toda troca de empresa/filial (`RenewToken`), prevenindo *session fixation*.

### Atributos Obrigatórios do Cookie de Sessão
- Nome: `__Host-mc_session` (prefixo `__Host-` em HTTPS garante `Secure`, `Path=/` e proíbe atributo `Domain`, impedindo vazamento entre subdomínios). Em desenvolvimento HTTP local: `mc_session`.
- Flags: `HttpOnly; Secure; SameSite=Lax; Path=/`.
- **Proteção CSRF nativa (Go 1.27):** todos os endpoints mutáveis são protegidos por `http.NewCrossOriginProtection()` da standard library (inspeciona `Sec-Fetch-Site` e fallback `Origin` vs `Host`, rejeitando requisições cross-site com `403 Forbidden` sem precisar de tokens CSRF manuais).
- **Senhas:** hash obrigatório com **Argon2id** (`golang.org/x/crypto/argon2`) com parâmetros recomendados pela OWASP (memória >= 64 MiB, iterações >= 3, paralelismo 1–4, salt aleatório de 16 bytes).

---

## 9. Defesa em Profundidade contra SQL Injection e OWASP Top 10

### Camadas de Proteção contra SQL Injection (SQLi)
1. **Zero concatenação de SQL:** é proibido construir queries com `fmt.Sprintf`, `+` ou strings interpoladas. Lint/review bloqueia qualquer SQL dinâmico não tipado.
2. **Queries estáticas (85–90%) via `sqlc` + `pgx/v5`:** compiladas contra o schema real das migrations; todos os argumentos trafegam via *bind parameters* do protocolo binário do PostgreSQL (`$1, $2...`), impossibilitando injeção na árvore sintática SQL.
3. **Queries dinâmicas (10–15%) via `go-jet`:** composição por AST Go tipada gerada do schema. Qualquer campo de ordenação (`sort_by`, `sort_dir`) ou filtro dinâmico passa por **allowlist fechada** (`switch` sobre constantes conhecidas) antes de montar a expressão.
4. **Full-Text Search seguro:** busca de produtos usa exclusivamente `websearch_to_tsquery('public.pt_unaccent', $1)` (que trata operadores inválidos e caracteres especiais com segurança, ao contrário de `to_tsquery`), com limite máximo de 120 caracteres no termo.
5. **Menor privilégio no PostgreSQL:**
   - Role `menuchef_migrator`: usada apenas no job de migrations (`goose`) com permissão de DDL.
   - Role `menuchef_app`: usada pela API em runtime com permissão restrita a `SELECT, INSERT, UPDATE, DELETE` nas tabelas da aplicação (sem `CREATE/DROP/ALTER`, sem superuser).

### Demais Controles OWASP Obrigatórios
- **Broken Access Control (Multi-tenant IDOR):** todo comando/query de filial exige `branch_id` resolvido e validado pelo middleware de autorização contra o vínculo do usuário. Query de tabela multi-tenant **sempre** traz predicado `branch_id` ou `company_id` (nunca buscar pedido, produto, cupom ou pagamento só por `id`). O gate de CI percorre os arquivos `sqlc` e quebra o build se o predicado faltar fora da allowlist. Teste de integração semeia **duas empresas** e confirma que UUID da empresa B, chamado com sessão da empresa A, não devolve corpo da B.
- **RLS do PostgreSQL fica fora do MVP.** O isolamento deste ciclo é aplicação + gate + teste de IDOR. Não criar `ENABLE ROW LEVEL SECURITY` nem policy nas migrations. Reavaliar depois, com o gate já verde. Não é parte do desenho atual.
- **Vínculo único:** `memberships` usa `UNIQUE NULLS NOT DISTINCT (user_id, company_id, branch_id)` (PostgreSQL 15+; o projeto está no 19). `company_owner` grava `branch_id` NULL; os outros papéis gravam `branch_id` NOT NULL, reforçado por `CHECK`. Sem `NULLS NOT DISTINCT`, dois owners da mesma pessoa na mesma empresa passariam, porque `NULL` não colide em `UNIQUE` comum. Índices parciais equivalentes, se um gerador não emitir a cláusula: `UNIQUE (user_id, company_id) WHERE branch_id IS NULL` e `UNIQUE (user_id, company_id, branch_id) WHERE branch_id IS NOT NULL`. Revogação é `UPDATE` de `status` na mesma linha.
- **Validação de Payload & Limites:** limite de tamanho de body (`http.MaxBytesReader`, ex.: 1 MiB para JSON, 5 MiB para upload de imagem), validação estrita de schema (Go validator / Zod nos frontends) e rejeição de campos desconhecidos em payloads críticos.
- **Rate Limiting:** no Traefik v3 (global por IP) e na mesma instância Redis da API (prefixo `mc:rl:`, TTL igual à janela) para rotas sensíveis (`/auth/login`, `/public/leads`, `/public/orders`, `/auth/invite/accept`).
- **Upload Seguro de Imagens:** validar *magic bytes* reais do arquivo (nunca confiar apenas na extensão ou `Content-Type`), decodificar/re-encodar para **WebP** removendo metadados EXIF, e salvar com chave gerada pelo servidor (`UUIDv7` + `.webp`).
- **Segredos:** 100% via variáveis de ambiente / Docker secrets; nunca versionados no Git.

---

## 10. Redis — uma instância (sessão, Pub/Sub, rate-limit e cache)

> **Decisão fechada:** uma única instância Redis 8. O dono do produto dispensou a separação `redis-state` + `redis-cache`. O resto da stack permanece.

Sessão opaca, Pub/Sub do SSE, rate-limit e cache de leitura convivem no mesmo processo. A separação é por **prefixo de chave** (e por canal, no Pub/Sub), não por instância. Há um único `*redis.Client` em `internal/platform/redis`. Dois clients apontando para o mesmo servidor não recriam o isolamento antigo.

### Por que `volatile-ttl` e sessão com TTL de 24h

`maxmemory-policy volatile-ttl`. Essa política só despeja chave que tem TTL, e escolhe a que expira primeiro.

- `allkeys-lru` e `allkeys-lfu` podem apagar a sessão da cozinha no pico. Ficam de fora.
- `noeviction` não apaga nada e passa a recusar `SET` quando a memória enche. Login novo e cache caem juntos. Fica de fora.
- `volatile-lru` também só olha chave com TTL, mas escolhe a menos usada. No sábado o cardápio é tão quente quanto a sessão do balcão, e a sessão entra no sorteio. Fica de fora.
- Sessão sem TTL ficaria imune ao despejo e também à limpeza: um logout perdido vira chave eterna. Quando a memória só tiver chave sem expire, o Redis recusa escrita. Fica de fora.

Com `volatile-ttl`, rate-limit (TTL da janela, dezenas de segundos) e cache (60s a 5 min) saem antes de uma sessão que ainda tem horas. A sessão usa o lifetime absoluto de 24 horas do `alexedwards/scs`, no prefixo `mc:sess:`. Um turno a partir do login cabe nesse prazo. Logout, revogação de vínculo e suspensão da empresa fazem `UNLINK` na hora, sem esperar as 24h.

Efeito aceito: se a memória estourar, a janela de rate-limit da API pode ser esquecida cedo. O limite por IP do Traefik continua na frente. O cardápio recompõe pelo Postgres no próximo miss.

### Convivência na mesma instância

| Uso | Chave ou canal | TTL |
| :--- | :--- | :--- |
| Sessão opaca `scs` | `mc:sess:{id}` | 24h (lifetime absoluto) |
| Cache de cardápio | `mc:cache:menu:branch:{branch_id}:v1` | 5 min ± 15% |
| Cache de meta, horário e "aberto agora" | `mc:cache:branch:slug:{slug}:v1` | 60 s ± 15% |
| Cache de RBAC | `mc:cache:rbac:user:{user_id}:v1` | 2 min ± 15% |
| Rate limit | `mc:rl:{rota}:{sujeito}` | duração da janela |
| Pub/Sub da fila | canal `mc:ps:branch:{branchId}:orders` | não é chave |
| Pub/Sub do pedido | canal `mc:ps:order:{trackingToken}` | não é chave |

Pub/Sub não grava a mensagem em chave, então não entra no `maxmemory` de dados. O buffer de cliente fica em `client-output-buffer-limit pubsub 32mb 8mb 60`.

O helper de cache recusa `SET` sem TTL (erro de programação, com teste de unidade). Nenhuma chave da aplicação nasce sem expire.

### Persistência e teto

`appendonly yes`, `appendfsync everysec`, `save ""` (sem snapshot RDB, para evitar o fork no pico). O AOF também registra escrita de cache. No volume do MVP esse custo é aceito. Depois de um restart, chave de cache pode voltar até o TTL acabar; o `UNLINK` de invalidação que já ocorreu também está no AOF.

`maxmemory 256mb` no MVP, via `REDIS_MAXMEMORY`. É um teto só, compartilhado pelos quatro usos.

### Falha

Não existe mais o modo "o cache caiu e a sessão ficou". Se o Redis não responde, `GET /api/v1/health/ready` falha. Leitura de cardápio público cai no Postgres. Login, sessão, SSE e rate-limit da API ficam indisponíveis até o Redis voltar.

### Regras de Cache-Aside & Invalidação
1. **O que entra no cache (prefixo `mc:cache:`, TTL obrigatório + jitter de ±15% contra *thundering herd*):** cardápio público, meta/horário/"aberto agora" da filial e mapa de RBAC, com as chaves da tabela acima.
2. **O que nunca é cacheado no fluxo de escrita (sempre lido do PostgreSQL 19 em transação):** disponibilidade e preço no checkout, faixas de CEP no fechamento, saldos e limites de cupom, máquina de estados do pedido e fila operacional da cozinha.
3. **Invalidação explícita pós-commit (`UNLINK` não-bloqueante):** todo Command que altera catálogo, horário, pausa, fuso ou vínculo RBAC executa `UNLINK` das chaves afetadas **somente após `tx.Commit()`** (nunca antes, para não repopular o cache com dado antigo).
4. **Proteção contra *cache stampede*:** leituras cacheadas passam por `golang.org/x/sync/singleflight`.
5. **Degradação da leitura de cache:** erro pontual de `GET` no prefixo `mc:cache:` registra log estruturado + métrica e faz fallback ao PostgreSQL. Isso não mantém a sessão de pé se o processo Redis inteiro caiu.

---

## 11. Idempotência (`Idempotency-Key`) & Concorrência Atômica de Cupons

### 11.1 Idempotência de Mutações Críticas (IETF `draft-ietf-httpapi-idempotency-key-header-09`)
Cliques duplos em rede 4G instável ou retentativas de checkout **nunca** podem gerar pedidos duplicados.
- **Rotas obrigatórias:** `POST /api/v1/public/branches/{slug}/orders`, `POST /api/v1/public/orders/{tracking_token}/payments/mock-confirm` e futuras mutações financeiras.
- **Header obrigatório:** `Idempotency-Key: <UUIDv4>` gerado pelo cliente a cada intenção de envio (renovado quando o usuário altera itens/endereço/pagamento). Ausência ou formato inválido retorna `400 Bad Request` (`MISSING_IDEMPOTENCY_KEY`).
- **Armazenamento transacional no PostgreSQL (`idempotency_keys`):** o registro de idempotência vive na **mesma transação SQL** da criação do pedido (garantia ACID sem janela de falha entre Postgres e Redis), com escopo `(scope_key, idempotency_key)` e TTL lógico de 24h.
- **Ciclo de vida em 3 estados:**
  1. **`absent`:** `INSERT INTO idempotency_keys (..., status='in_progress', request_hash=$hash, locked_until=now()+interval '30 seconds') ON CONFLICT DO NOTHING`. Se inseriu, processa o pedido, grava `response_status`/`response_body` e marca `status='completed'` na mesma transação.
  2. **`in_progress` (requisição paralela em voo):** se o lease `locked_until` ainda está ativo, retorna **`409 Conflict`** (`IDEMPOTENCY_IN_PROGRESS`) com `Retry-After: 2`.
  3. **`completed`:** compara o SHA-256 canônico do payload atual com `request_hash`:
     - Hash igual → retorna exatamente o `response_status` e `response_body` armazenados (replay `201 Created`, sem recriar pedido).
     - Hash diferente (mesma chave reutilizada com outro payload) → retorna **`422 Unprocessable Entity`** (`IDEMPOTENCY_PAYLOAD_MISMATCH`).

### 11.2 Concorrência Atômica de Cupons & Estorno Automático
- **Proibido o anti-padrão `SELECT used_count` → checa em Go → `UPDATE`:** sob concorrência real no PostgreSQL (`READ COMMITTED`), duas requisições simultâneas leriam o mesmo saldo e estourariam o limite (*over-redemption*).
- **Padrão obrigatório (dentro da transação de `CreateOrder`):**
  1. Executar **`UPDATE` condicional atômico** no registro do cupom:
     ```sql
     UPDATE coupons
     SET used_count = used_count + 1
     WHERE id = $1
       AND branch_id = $2
       AND is_active = true
       AND (starts_at IS NULL OR starts_at <= now())
       AND (expires_at IS NULL OR expires_at >= now())
       AND (max_uses IS NULL OR used_count < max_uses)
     RETURNING id, discount_type, discount_value, min_order_cents, max_uses_per_customer;
     ```
     Zero linhas retornadas → aborta transação com `409 Conflict` (`COUPON_UNAVAILABLE_OR_EXHAUSTED`).
  2. **Limite por cliente (decisão do dono em 23/11/2026, opção b).** Só entra se `max_uses_per_customer` não for nulo.
     - O identificador é `customer_user_id` da conta logada. Não se usa hash de telefone. A coluna `customer_identifier_hash` **não existe**: ela só serviria para marcar convidado, e essa opção foi recusada. Manter a coluna reabriria a opção (a).
     - Convidado (pedido sem `customer_user_id`) recebe `422 Unprocessable Entity` (`COUPON_REQUIRES_ACCOUNT`), detalhe *"Este cupom tem limite por cliente. Entre na sua conta para usá-lo."* A transação não incrementa `used_count` e não insere resgate. A sacola permanece.
     - Cliente logado: `pg_advisory_xact_lock` na chave `(LockNSCouponCustomer, hash(coupon_id, customer_user_id))`. O lock evita *phantom read* em `READ COMMITTED` sem serializar outros clientes do mesmo cupom. Conta as linhas em `coupon_redemptions` com aquele par. Se a contagem já atingiu `max_uses_per_customer`, aborta com `409 Conflict` (`COUPON_CUSTOMER_LIMIT_REACHED`).
     - O resgate grava `customer_user_id` e `UNIQUE (coupon_id, order_id)`. Não há `UNIQUE (coupon_id, customer_user_id)`: o limite pode ser maior que 1, e o lock mais a contagem é que impõem o teto.
     - Cupom com `max_uses_per_customer` nulo segue disponível para convidado e para conta logada. O resgate de convidado deixa `customer_user_id` nulo. O de conta logada preenche o id, sem aplicar teto por pessoa.
     - Verificação de telefone por código, via **WhatsApp Business API oficial**, fica fora do MVP. Não substitui a conta logada neste ciclo.
- **Estorno automático de cupom (regra de negócio fechada):** sempre que um pedido com cupom transicionar para `canceled_by_customer` (antes de `confirmed`), `rejected_by_store` ou expiração de Pix mocado, a mesma transação de mudança de status executa:
  - `DELETE FROM coupon_redemptions WHERE order_id = $1 RETURNING coupon_id;`
  - Somente se uma linha foi deletada (idempotente contra estorno duplo), executa `UPDATE coupons SET used_count = GREATEST(used_count - 1, 0) WHERE id = $1;`.

---

## 12. Estratégia de Transactions, Locks & Advisory Locks (PostgreSQL 19 + `pgxpool`)

> **Decisão fechada:** **Matriz Híbrida sobre `READ COMMITTED`** — priorizar **constraints declarativas** e **operações atômicas / Optimistic Locking** antes de locks pessimistas, reservando **`pg_advisory_xact_lock`** apenas para casos cirúrgicos sem linha única travável ou coordenação de jobs singleton.

### 12.1 Matriz de Concorrência por Cenário
| Cenário | Mecanismo Obrigatório | Motivo |
| :--- | :--- | :--- |
| **Faixas de CEP sem sobreposição** | `EXCLUDE USING gist (branch_id WITH =, int4range(cep_start::int, cep_end::int, '[]') WITH &&) WHERE (is_active)` | Garantida no schema; erro `SQLSTATE 23P01` → `409 CEP_RANGE_OVERLAP` |
| **Transições de estado do pedido** (ex.: cliente cancela vs cozinha confirma) | **Optimistic Locking** (`version INT NOT NULL DEFAULT 1`) + `UPDATE orders SET status = $next, version = version + 1 WHERE id = $1 AND branch_id = $2 AND status = $expected AND version = $v RETURNING ...` | Zero lock wait para leitores, zero deadlock, retorna `409 Conflict` determinístico se outro ator venceu |
| **Cupom global (`max_uses`)** | **Atomic Conditional `UPDATE`** (`WHERE ... AND (max_uses IS NULL OR used_count < max_uses)`) | 1 único statement atômico; dispensa `SELECT FOR UPDATE` |
| **Número curto da filial (`display_number`)** | Upsert atômico em `branch_order_counters` executado **por último** na transação do pedido | Segura o row lock da filial por < 1–2 ms, minimizando contenção no pico |
| **Workers de fila / expiração em lote de Pix** | **`SELECT ... FOR UPDATE SKIP LOCKED`** | Múltiplas réplicas processam lotes em paralelo sem bloqueio |
| **Cupom por cliente (`max_uses_per_customer`)** | Conta logada obrigatória. **`pg_advisory_xact_lock`** em `(LockNSCouponCustomer, hash(coupon_id, customer_user_id))` e contagem em `coupon_redemptions` | Decisão de 23/11/2026. Convidado recebe `422 COUPON_REQUIRES_ACCOUNT`. Sem hash de telefone |
| **Jobs cron singleton** (retenção LGPD, limpeza `idempotency_keys`/leads) | **`pg_try_advisory_xact_lock(LockNSCronJob, job_id)`** (non-blocking) | Garante execução única entre múltiplas réplicas Go sem depender de lock distribuído no Redis |
| **Reordenação em lote de cardápio / CEP da filial** | **`pg_advisory_xact_lock(LockNSBranchCatalog, branch_hash)`** | Evita *interleaving* em edições administrativas multi-linha simultâneas |

### 12.2 Regras Inegociáveis de Pool, Timeouts & Deadlocks (`pgxpool` + PG 19)
1. **Proibido `pg_advisory_lock` (session-level):** `pgxpool` não executa `DISCARD ALL` ao devolver conexões ao pool — locks de sessão vazariam para requisições não relacionadas. Usar **exclusivamente** `pg_advisory_xact_lock` / `pg_try_advisory_xact_lock` (liberados automaticamente no `COMMIT`/`ROLLBACK`), com tabela centralizada de namespaces `int32` em `internal/platform/db/locks.go`.
2. **Hierarquia estrita de Timeouts configurada em `ConnConfig.RuntimeParams`:**
   $$\text{lock\_timeout}\ (2\text{s}) < \text{idle\_in\_transaction\_session\_timeout}\ (4\text{s}) < \text{statement\_timeout}\ (5\text{s}) < \text{transaction\_timeout}\ (15\text{s}) < \text{HTTP timeout}\ (20\text{s})$$
   - `lock_timeout` (`55P03`) e `statement_timeout` (`57014`) emitem `ERROR` (conexão volta saudável ao `pgxpool` após rollback). `transaction_timeout` (`25P04`) emite `FATAL` e derruba a conexão TCP — usado apenas como rede de segurança extrema.
   - Overrides pontuais dentro de uma transação usam **sempre `SET LOCAL`** (pipelineado em `pgx.Batch` com `pg_advisory_xact_lock` em 1 round-trip); **nunca `SET` sem `LOCAL`** no `pgxpool`.
3. **Split-Transaction — ZERO I/O externo dentro de `BEGIN ... COMMIT`:** jamais chamar `PaymentGatewayPort`, Garage S3 ou Web Push com transação/lock aberto. Para pagamento online: **Tx 1** (curta) persiste `order` (`awaiting_payment`) + `payment` (`pending`) e faz `COMMIT` → chama `PaymentGatewayPort` fora da tx → **Tx 2** (curta) persiste resultado (ou estorna cupom em recusa) e faz `COMMIT`.
4. **Publicação SSE pós-commit (YAGNI no MVP):** eventos de pedido são publicados no Pub/Sub da instância Redis única (canais `mc:ps:`) **somente após `tx.Commit()`** bem-sucedido, complementados por **snapshot inicial do banco** na conexão/reconexão SSE (sem tabela extra de Outbox no MVP; evolução para Transactional Outbox com `SKIP LOCKED` reservada para integrações/webhooks pós-MVP).
5. **Prevenção de Deadlocks & Observabilidade PG 19:** adquirir múltiplos locks sempre em ordem determinística (chaves ordenadas / `ORDER BY id`), preferir `FOR NO KEY UPDATE` quando não alterar PK/UNIQUE, aproveitar `log_lock_waits = on` (default no PG 19) e monitorar a view nativa **`pg_stat_lock`** (incluindo `locktype = 'advisory'`) e erros `55P03`/`40P01`.

---

## 13. Versionamento de API (`/api/v1`) & Contratos OpenAPI 3.1

> **Decisão fechada:** **`Huma v2`** sobre **`chi` v5** (`humachi`) como fonte única de verdade (Code-First com contratos tipados) gerando **OpenAPI 3.1** + **RFC 9457 Problem Details**, consumido nos 3 frontends via **`openapi-typescript` + `openapi-fetch`**.

### Diretrizes de Contrato & Evolução
1. **Prefixo explícito `/api/v1/`:** toda evolução dentro de `v1` deve ser **aditiva e retrocompatível** (novos campos opcionais, novos endpoints, novos valores de enum documentados). Remoções ou mudanças incompatíveis exigem `v2` ou período de transição com headers RFC 8594 (`Deprecation` e `Sunset`).
2. **Isolamento de `Huma v2` na camada `presentation/http`:** structs de input/output HTTP do Huma ficam exclusivamente em `presentation/http` e mapeiam para DTOs/Commands/Queries da camada `application` — mantendo `domain` e `application` 100% livres de tags/dependências HTTP.
3. **Rotas SSE permanecem em `chi` puro:** streams `text/event-stream` (`/api/v1/*/stream`) são registrados diretamente no router `chi` (fora do wrapper JSON do Huma).
4. **Atenção técnica (`chi` + `Huma`):** **não** habilitar `middleware.URLFormat` do `chi` nas rotas do Huma, pois remove sufixos `.json`/`.yaml` e causa `404` em `/api/v1/openapi.json`.
5. **Geração sem banco no CI (Zero Drift):** subcomando `go run ./cmd/api openapi > openapi.json` exporta a especificação OpenAPI 3.1 sem subir PostgreSQL/Redis; o pipeline gera tipos TypeScript via `openapi-typescript` e falha o build se houver divergência entre Go e frontends.

---

## 14. LGPD & Privacidade by Design (Modelo B2B2C)

- **Papéis legais explícitos (Lei nº 13.709/2018):**
  - **Menu Chef = Controlador (Controller):** dados de leads comerciais, contas de usuários da plataforma/empresas, logs de segurança e auditoria.
  - **Empresa / Filial = Controladora; Menu Chef = Operador (Processor):** dados pessoais dos clientes finais coletados em pedidos da filial (nome, telefone, endereço de entrega).
- **Documentos legais versionados em SSG:** `/termos-de-uso` e `/politica-de-privacidade` publicados na `apps/landing` com versão semântica (`terms_v1.0`, `privacy_v1.0`), data de vigência e canal de contato do Encarregado (DPO). Zero pixels/trackers publicitários de terceiros no MVP.
- **Ledger de aceite (`legal_acceptances`):** registro imutável (`actor_type`, `actor_id`, `document_type`, `document_version`, `accepted_at`, `ip_hash`, `user_agent`) gravado no envio de lead, aceite de convite da empresa e criação de conta opcional do cliente.
- **Direitos do titular no MVP (Cliente autenticado):**
  - **Portabilidade / Acesso:** `GET /api/v1/customer/privacy/export` retorna JSON estruturado com perfil, endereços salvos, aceites legais e histórico de pedidos do titular.
  - **Eliminação / Anonimização:** `POST /api/v1/customer/privacy/delete-account` revoga todas as sessões ativas no Redis (`UNLINK` em `mc:sess:`), remove endereços salvos, anonimiza PII do perfil e pedidos históricos (`"Cliente Anonimizado"`, telefone/e-mail substituídos por hash irreversível), **preservando intactos os itens, valores e métricas financeiras da filial**. A exclusão de conta não estorna cupom: a linha em `coupon_redemptions` continua apontando para o `customer_user_id` anonimizado. Uma conta nova é outro id e, no MVP, não herda esse consumo. Fechar esse desvio é papel da verificação de telefone via WhatsApp Business API oficial, fora do MVP.
- **Retenção & Minimização:** `idempotency_keys` purgadas após 24h; tokens de convite expirados purgados após 7 dias; leads inativos após 24 meses; logs `slog` sempre com redaction de PII sensível.

---

## 15. Critérios de Aceite & Verificação

- [ ] Estrutura de camadas (`domain`, `application`, `infrastructure`, `presentation`) verificável por análise estática (`depguard` no `golangci-lint`) garantindo que `domain` e `application` não importam `chi`, `huma`, `pgx`, `redis` ou `net/http`.
- [ ] Migrations criam todas as PKs de domínio como `uuid DEFAULT uuidv7()`, `version` em `orders`, `EXCLUDE USING gist` em `cep_shipping_ranges` e `display_number` sequencial por filial.
- [ ] 100% dos endpoints de listagem usam **Cursor Opaco + Keyset Pagination** (`LIMIT page_size + 1`, zero `OFFSET`, zero `COUNT(*)`), com testes validando ausência de *row drift* e rejeição de cursor inválido (`400 INVALID_CURSOR`).
- [ ] Nenhum arquivo de frontend referencia `localStorage` ou `sessionStorage` para tokens/sessões.
- [ ] `pgxpool` configurado com `RuntimeParams` (`lock_timeout < idle_in_transaction_session_timeout < statement_timeout < transaction_timeout`), zero uso de `pg_advisory_lock` session-level e zero I/O externo dentro de transações SQL (Split-Tx).
- [ ] Uma instância Redis com AOF, `maxmemory-policy volatile-ttl`, prefixos `mc:sess:` / `mc:cache:` / `mc:rl:`, canais `mc:ps:`, helper de cache que recusa `SET` sem TTL, `singleflight` e `UNLINK` pós-commit. Teste de pressão com `maxmemory` baixo despeja cache de TTL curto e preserva sessão com TTL de 24h.
- [ ] Testes de integração validam: idempotência em 3 estados (`409`/`422`/`201`); concorrência de cupom global (50 goroutines); limite por cliente só com `customer_user_id` (convidado → `422 COUPON_REQUIRES_ACCOUNT`; segundo uso além do teto → `409 COUPON_CUSTOMER_LIMIT_REACHED`; cupom sem teto por cliente segue para convidado); estorno automático; corrida de transição de pedido (cancelar vs confirmar → exatamente 1 vence e o outro recebe `409`); sobreposição de CEP (`23P01` → `409`); fuso de Manaus na virada do dia; segundo `company_owner` com `branch_id` nulo rejeitado (`23505`); direitos LGPD (exportação JSON + anonimização preservando faturamento da filial).
- [ ] Gate de CI valida **zero drift** entre `go run ./cmd/api openapi`, `openapi-typescript` e `tsc --noEmit` nos 3 frontends.
- [ ] Gate de CI percorre as queries `sqlc` de tabelas multi-tenant e falha se faltar predicado `branch_id` ou `company_id` fora de `apps/api/queries/tenant_scope_allowlist.txt` (motivo obrigatório em cada linha).
- [ ] Testes automatizados de segurança validam: tentativa de SQLi em busca FTS e filtros dinâmicos; IDOR entre **duas empresas** (UUID da outra não vaza corpo; id inexistente e id de outro tenant no mesmo escopo devolvem o mesmo status); requisição cross-origin sem `Sec-Fetch-Site: same-origin` rejeitada com `403`.
- [ ] Migrations, OpenAPI e extrato não têm coluna `platform_fee_cents` nem `customer_identifier_hash`. RLS do Postgres não é ligado neste ciclo.
