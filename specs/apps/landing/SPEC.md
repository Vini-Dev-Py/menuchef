# Spec de Aplicação — Landing Page Institucional (`apps/landing`)

> **Documento autocontido.** Especifica a Landing Page comercial do Menu Chef voltada para donos de restaurantes que comparam marketplaces e cardápios próprios. Detalha a arquitetura **SSG / Prerender em build-time** com **TanStack Start**, estratégia completa de **SEO, AEO e GEO**, especificação **`llms.txt` v2** + réplicas `.md`, `sitemap.xml`, `robots.txt` granular para bots de IA, pipeline de imagens **WebP**, responsividade Mobile-First, roteiro de conversão transparente e integração de captação de leads (onboarding assistido).

---

## 1. Objetivo de Produto e Público

- **Público:** donos e gestores de restaurantes, hamburguerias, pizzarias e pequenas redes com 1 ou mais filiais que desejam sair da dependência de comissões de marketplace (12%–23%) sem cair no caos de receber pedidos soltos no WhatsApp.
- **Regra comercial (Onboarding Assistido):** a landing **não oferece criação instantânea de conta**. O CTA principal direciona para o formulário de contato / agendamento de ativação assistida (enviando o lead para o Administrador da Plataforma via `POST /api/v1/public/leads`).
- **Diferenciais que a página precisa provar logo acima da dobra:**
  1. Pedido nasce no cardápio com a marca da loja e cai direto na **fila do painel** (nunca perdido no WhatsApp).
  2. Status claros com o mesmo nome na cozinha e no celular do cliente (`Recebido → Confirmado → Em produção → Pronto → Retirada/Entrega`).
  3. **Multi-filial de verdade com 1 único login** e preço/frete por CEP independentes por unidade.
  4. **Zero comissão sobre taxa do motoboy ou pagamento feito na porta** (dinheiro, maquininha ou Pix da casa) e tabela transparente do que está incluso no plano.

---

## 2. Stack Tecnológica & Decisão de Renderização (Por que SSG/Prerender?)

| Camada | Escolha | Motivo Técnico (Evidência 2026) |
| :--- | :--- | :--- |
| **Framework & Build** | **TanStack Start (modo SSG / Prerender)** sobre **React 19 + Vite + TypeScript** | Um Vite SPA puro entrega `<div id="root"></div>` vazio. Estudos de logs (2025–2026) comprovam que **bots de IA (`OAI-SearchBot`, `GPTBot`, `ClaudeBot`, `PerplexityBot`, `ChatGPT-User`) não executam JavaScript**. O prerender em build-time entrega 100% do HTML, meta tags e JSON-LD no primeiro byte mantendo a mesma stack React/Vite/TanStack do restante do projeto |
| **Estilização & UI** | **Tailwind CSS v4** + **shadcn/ui v4 (Base UI)** | Paleta **Brasa clara** (`#C2410C`, `#FFFBF5`, `#1C1917`, destaque `#F59E0B` só com texto escuro) + logo **Concha com vapor** |
| **Animações** | **Motion (`motion/react`)** com `<MotionConfig reducedMotion="user">` | Animações contidas e significativas na peça herói (demonstração viva da fila de pedidos); sem fade genérico em toda seção e sem carrossel automático |
| **Formulários & HTTP Tipado** | **TanStack Form + Zod + `openapi-fetch`** | Validação client-side instantânea com mensagens inline por campo; cliente HTTP tipado gerado automaticamente do OpenAPI 3.1 da API Go (`openapi-typescript`) |
| **Runtime Container** | `nginxinc/nginx-unprivileged` (non-root) | Serve arquivos estáticos prerenderizados com **HTTP `404` real** (`try_files $uri $uri/ $uri.html =404;`) — nunca Soft 404 |

> **Cuidado técnico obrigatório no SSG React:** usar `hydrateRoot` no cliente (nunca `createRoot` sobre HTML prerenderizado, para não destruir o DOM e degradar LCP).

---

## 3. Arquitetura de SEO, AEO (Answer Engine Optimization) e GEO (Generative Engine Optimization)

### 3.1 HTML Semântico, Metadados e Structured Data (JSON-LD)
Toda rota prerenderizada (`/`, `/funcionalidades`, `/planos`, `/multi-filial`, `/perguntas-frequentes`, `/contato`, `/termos-de-uso`, `/politica-de-privacidade`) deve conter no HTML cru (`curl -s`):
- Hierarquia semântica estrita: um único `<h1>` claro e factual, `<header>`, `<main>`, `<section>`, `<article>`, `<footer>`.
- `<title>` único (50–60 chars), `<meta name="description">` (140–160 chars), `<link rel="canonical">`, `hreflang="pt-BR"`.
- **Open Graph & Twitter Cards:** `og:title`, `og:description`, `og:type`, `og:url`, `og:image` (1200x630 otimizada), `og:locale="pt_BR"`.
- **JSON-LD embutido no `<head>`:**
  - `Organization` (Menu Chef, logo, canais de contato).
  - `SoftwareApplication` (`applicationCategory: "BusinessApplication"`, `operatingSystem: "Web, PWA"`, `inLanguage: "pt-BR"`, `offers` transparentes sem preços escondidos).
  - `FAQPage` (perguntas diretas que donos de restaurante fazem a assistentes de IA: *"Como funciona frete por CEP?"*, *"O Menu Chef cobra comissão sobre pagamento na entrega?"*, *"Dá para gerenciar várias filiais no mesmo login?"*).
  - `BreadcrumbList` nas páginas internas.

### 3.2 Redação Otimizada para AEO e GEO
- **Respostas diretas e citáveis:** abrir cada tópico/FAQ com uma resposta factual de 40–60 palavras em linguagem direta antes de detalhar.
- **Tabelas comparativas HTML reais (`<table>`):** comparação clara do que está incluso na assinatura, fluxo de status e modelo de cobrança (motores generativos extraem tabelas semânticas com alta precisão).
- **Zero jargão vazio:** usar os termos reais do domínio brasileiro de food service (*cardápio digital sem comissão*, *frete por faixa de CEP*, *gestor de pedidos*, *multifilial*, *pagamento na entrega*, *comanda térmica*).

### 3.3 Especificação `llms.txt` v2 (Agosto/2026) + Réplicas `.md`
Seguir a revisão **v2 da especificação `llms.txt`**:
1. **`/llms.txt`** na raiz (`text/markdown; charset=utf-8`):
   - `# Menu Chef` (H1 obrigatório)
   - `> Resumo conciso da plataforma SaaS de cardápio digital e delivery próprio multi-filial no Brasil.` (blockquote)
   - Parágrafo orientando sobre o modelo comercial (sem comissão por pedido, onboarding assistido via contato) e links estruturados em seções H2 (`## Produto`, `## Planos e Transparência`, `## Perguntas Frequentes`, `## Legal e Privacidade`, `## Contato`) apontando para as réplicas `.md`.
2. **Réplicas Markdown limpas (`.md`):** cada página pública principal expõe sua versão limpa em Markdown (ex.: `/index.md`, `/funcionalidades.md`, `/planos.md`, `/perguntas-frequentes.md`, `/termos-de-uso.md`, `/politica-de-privacidade.md`) geradas no build a partir da mesma fonte de conteúdo (evitando divergência entre HTML e Markdown).
3. **Descoberta formal v2:** cada página HTML inclui no `<head>` e/ou header HTTP `Link`:
   - `<link rel="alternate" type="text/markdown" href="/funcionalidades.md" />`
   - `<link rel="describedby" href="/llms.txt" />`
4. **`/llms-full.txt`** opcional gerado no build concatenando todas as páginas públicas para agentes de leitura em lote.

### 3.4 `sitemap.xml` e `robots.txt` Granular
- **`/sitemap.xml`:** gerado automaticamente em build-time listando apenas rotas públicas canônicas (`200 OK`, incluindo `/termos-de-uso` e `/politica-de-privacidade`) com `<lastmod>` (ISO 8601), sem páginas utilitárias ou redirecionamentos.
- **`/robots.txt`:** configurado de forma explícita separando busca/citação de treinamento, garantindo visibilidade em AEO/GEO:
  - Permitir crawlers de busca tradicionais (`Googlebot`, `Bingbot`, `Applebot`).
  - Permitir crawlers de busca e fetch sob demanda de IA (`OAI-SearchBot`, `ChatGPT-User`, `Claude-SearchBot`, `Claude-User`, `PerplexityBot`, `Perplexity-User`).
  - Política explícita (documentada e configurável) para crawlers de treinamento puro (`GPTBot`, `ClaudeBot`, `CCBot`, `Bytespider`).
  - Referenciar `Sitemap: https://menuchef.com.br/sitemap.xml`.

---

## 4. Pipeline de Imagens **WebP** & Core Web Vitals

- **100% das imagens fotográficas e ilustrações raster em `.webp`** (com versões responsivas `srcset` / `sizes` para 400w, 800w, 1200w geradas em build-time via pipeline Vite/`vite-imagetools` ou `sharp`).
- Ícones e logo Concha com vapor em **SVG otimizado** inline/asset.
- Atributos obrigatórios em toda `<img>`: `width`, `height`, `alt` descritivo (ou `alt=""` se puramente decorativa), `decoding="async"`, `loading="lazy"` abaixo da dobra e `fetchpriority="high"` na imagem/hero LCP.
- **Metas de Core Web Vitals:**
  - `LCP <= 2.0s` (mobile 4G simulado)
  - `CLS <= 0.05`
  - `INP <= 150ms`
  - Lighthouse Mobile & Desktop `>= 95` em todas as 4 categorias.

---

## 5. Estrutura de Conteúdo, Páginas Legais (LGPD) e Responsividade Mobile-First

| Seção | Propósito | Layout Mobile (`375px`) → Tablet (`768px`) → Desktop (`1440px`) |
| :--- | :--- | :--- |
| **1. Header** | Logo Concha com vapor + links âncora/rotas + CTA "Falar com a equipe" + link "Já sou cliente (Entrar)" apontando para `app.menuchef.com.br` | Mobile: menu drawer Base UI (alvo >= 44px); Desktop: barra limpa 1 linha |
| **2. Hero (Peça Central)** | Proposta direta ("Seu cardápio próprio, com a comanda no painel — não perdida no WhatsApp") + prévia interativa/estática da **fila de pedidos na paleta Brasa** + CTA primário | Mobile: 1 coluna, texto + CTA acima da dobra; Desktop: 2 colunas assimétricas |
| **3. Fluxo em 3 Passos (Sequência real)** | 1) Ativação assistida e cardápio no ar → 2) Cliente pede no celular sem app → 3) Cozinha confirma, produz e avisa | Mobile: pilha vertical; Tablet/Desktop: 3 colunas conectadas |
| **4. Multi-filial & Operação** | Troca de empresa/filial no mesmo login, frete por faixa de CEP, pausa rápida, modo escuro de cozinha e PWA | Grade responsiva 1 → 2 colunas com capturas em WebP |
| **5. Transparência de Planos** | Tabela clara do que **está incluso** vs o que **fica de fora / módulo futuro** + garantia de zero taxa sobre entrega ou pagamento na porta | Cards empilhados no mobile; tabela semântica `<table>` no desktop |
| **6. FAQ (AEO/GEO)** | 6–8 perguntas objetivas usando Accordion Base UI (conteúdo presente no HTML prerenderizado para crawlers) | 1 coluna legível (largura máx. 70ch) |
| **7. Formulário de Contato (Onboarding Assistido)** | Captura: Nome, Nome do Restaurante, WhatsApp/Telefone, E-mail, Quantidade de Filiais, Mensagem opcional + checkbox explícito de aceite da Política de Privacidade (`privacy_v1.0`) | Inputs com `font-size >= 16px`, máscara leve de telefone BR, validação Zod inline |
| **8. Footer** | Links institucionais, `/termos-de-uso`, `/politica-de-privacidade`, link discreto para `/llms.txt` e canal comercial/DPO | 1 coluna → 4 colunas |

### 5.1 Páginas Legais SSG (`/termos-de-uso` e `/politica-de-privacidade`)
- **Versionamento explícito no topo de cada documento:** exibe versão semântica (ex.: `Versão 1.0 — Vigência a partir de DD/MM/AAAA`), índice navegável (`max-w-3xl`, tipografia legível) e link de acesso direto consumido também pelo Storefront e pelo Admin PWA.
- **Papéis LGPD (Lei nº 13.709/2018) em linguagem clara (B2B2C):**
  - Explica que o **Menu Chef** é **Controlador** para dados de leads, contas da plataforma e logs de segurança.
  - Explica que cada **Restaurante (Empresa/Filial)** é **Controlador** dos dados de pedidos dos clientes finais e o **Menu Chef** atua como **Operador (Processor)**.
- **Transparência de Cookies & Zero Trackers no MVP:** a Landing, o Storefront e o Admin usam **apenas cookies estritamente necessários** (sessão `HttpOnly` `__Host-mc_session`) e **zero pixels/trackers publicitários de terceiros** no MVP — dispensando banner intrusivo de consentimento de anúncios e declarando essa política de forma transparente.
- **Direitos do titular & DPO:** seção explícita com canal de contato do Encarregado (DPO), prazos de retenção (`idempotency_keys` 24h, leads inativos 24 meses) e instruções de autoatendimento (exportação JSON e exclusão/anonimização de conta no Storefront).

---

## 6. Contrato de Integração com a API (`same-origin` via `openapi-fetch`)

A Landing só depende de **1 único endpoint** da API (e permanece 100% navegável e testável com mock caso a API esteja isolada):

- **`POST /api/v1/public/leads`**
  - **Request JSON:**
    ```json
    {
      "contact_name": "Maria Oliveira",
      "company_name": "Brasa & Pão Burguer",
      "phone": "11987654321",
      "email": "maria@brasaepao.com.br",
      "branch_count": 2,
      "notes": "Operamos delivery e retirada, queremos migrar do WhatsApp.",
      "accepted_privacy_version": "privacy_v1.0",
      "honeypot": ""
    }
    ```
  - **Response `201 Created`:** `{ "id": "019a...", "status": "received", "message": "Recebemos seu contato. Nossa equipe retornará em breve." }`
  - **Erros esperados (RFC 9457):** `400/422` (erros por campo exibidos inline), `429` (rate limit excedido com mensagem amigável).
  - **Anti-spam:** campo honeypot invisível + rate limit por IP no Traefik/`redis-state`.

---

## 7. Critérios de Aceite & Plano de Testes (`apps/landing`)

- [ ] **Verificação de Prerender (AEO/GEO & Legal):** `curl -s http://menuchef.localhost/`, `.../termos-de-uso` e `.../politica-de-privacidade` retornam `<h1>`, conteúdo completo, versão legal e JSON-LD no HTML bruto sem executar JS.
- [ ] **Verificação de Arquivos Técnicos:**
  - `GET /llms.txt` → `200 OK`, `text/markdown`, válido conforme spec v2;
  - `GET /index.md`, `/termos-de-uso.md`, `/politica-de-privacidade.md` e demais réplicas `.md` → `200 OK`, consistentes com as páginas HTML;
  - `GET /sitemap.xml` → `200 OK`, XML válido com todas as rotas canônicas;
  - `GET /robots.txt` → `200 OK`, regras explícitas para bots de busca e IA + diretiva `Sitemap:`;
  - `GET /pagina-que-nao-existe` → `404 Not Found` real (não `200 OK`).
- [ ] **Verificação de Mídia & Privacidade:** zero imagens `.png`/`.jpg` pesadas (100% `.webp` ou `.svg`) e zero scripts de tracking/anúncios de terceiros carregados.
- [ ] **Unitários (Vitest + RTL):** validação Zod do formulário de lead (incluindo obrigatoriedade do aceite da Política de Privacidade), máscara de telefone, estados de envio/erro/rate-limit e `<MotionConfig reducedMotion="user">`.
- [ ] **E2E (Cypress):** testa carregamento em `375px`, `768px` e `1440px` sem scroll horizontal, leitura de `/termos-de-uso` e `/politica-de-privacidade`, envio bem-sucedido de lead com aceite legal, exibição de erro inline em campos inválidos e auditoria `cypress-axe` (WCAG 2.2 AA).
