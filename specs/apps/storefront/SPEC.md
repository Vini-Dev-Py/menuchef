# Spec de Aplicação — Cardápio Digital (`apps/storefront`)

> **Documento autocontido.** Especifica o aplicativo web público do cliente final (`pedir.menuchef.com.br/{branchSlug}`), projetado **Mobile-First** para funcionar no navegador do celular sem instalar aplicativo. Cobre navegação do cardápio da filial, busca via Full-Text Search, adicionais obrigatórios, sacola persistida, checkout em 3 passos (entrega/retirada + validação prévia de CEP + cupom + pagamento mocado Pix/Cartão ou na hora), acompanhamento de status em tempo real via SSE, cancelamento antes da confirmação e conta opcional com histórico/"Pedir de novo".

---

## 1. Propósito e Experiência do Cliente

- **Acesso imediato via link/QR:** cliente clica no link do WhatsApp, Instagram ou embalagem (`pedir.menuchef.com.br/{branchSlug}`) e vê o cardápio da **filial específica** com a identidade da loja (logo e cor primária da filial, ou Paleta Brasa `#C2410C` como fallback).
- **Zero atrito de cadastro:** é obrigatório permitir concluir pedidos como **convidado** informando apenas nome e telefone. Criar conta é **opcional** (para histórico de pedidos e atalho "Pedir de novo").
- **Nome de produto:** o termo na interface é sempre **Sacola** (nunca "Cart" ou "Carrinho").

---

## 2. Stack Tecnológica (`apps/storefront`)

| Camada | Tecnologia | Uso Específico |
| :--- | :--- | :--- |
| **Core** | **React 19 + TypeScript (`strict: true`) + Vite** | Build leve otimizado para redes móveis 4G |
| **Roteamento** | **TanStack Router** (file-based) | Rotas tipadas (`/$branchSlug`, `/$branchSlug/checkout`, `/$branchSlug/pedido/$trackingToken`, `/minha-conta/pedidos`, `/minha-conta/privacidade`) com busca/categoria na URL |
| **Server State & HTTP Tipado** | **TanStack Query v5 + `openapi-fetch`** | Cache do cardápio, busca FTS com debounce, invalidação limpa após checkout/cancelamento e cliente HTTP 100% tipado gerado do OpenAPI 3.1 da API Go (`openapi-typescript`) |
| **Formulários & Validação** | **TanStack Form + Zod** | Checkout passo a passo, validação de CEP, telefone BR, troco e conta opcional |
| **UI & Estilo** | **Tailwind CSS v4 + shadcn/ui v4 (`@base-ui/react`)** | Bottom Sheet (Drawer Base UI) para sacola e seleção de adicionais; contraste WCAG 2.2 AA |
| **Animações** | **Motion (`motion/react`)** | Envolvido por `<MotionConfig reducedMotion="user">`: feedback discreto ao adicionar item na sacola e abertura da folha; sem animação de tela inteira deslizando no checkout |
| **Autenticação Cliente** | Cookie `HttpOnly; Secure; SameSite=Lax` (mesma origem `/api`) | **Zero credenciais em `localStorage`**. Apenas itens da sacola (dados não sensíveis, escopados por `branchSlug`) são persistidos localmente para sobreviver se o cliente sair do navegador |

---

## 3. Fluxos e Requisitos Funcionais

### 3.1 Cabeçalho da Filial, Horário e Pausa
- Exibe logo da filial, nome, status em tempo real (**Aberta**, **Fechada — abre às HH:MM**, ou **Pausada temporariamente** com motivo da loja), tempo médio estimado e modalidades ativas (Entrega / Retirada).
- **Regra crítica:** quando a loja está **fechada ou pausada**, o cliente ainda pode ver o cardápio para consultar preços, mas os botões "Adicionar" e o checkout ficam **desabilitados** com explicação explícita do horário — evitando pagamento à toa.
- Se a filial definir cor primária customizada, a UI calcula luminância WCAG e escolhe automaticamente texto `#FFFFFF` ou `#1C1917` sobre o botão primário (garantindo `>= 4,5:1`).

### 3.2 Navegação do Cardápio & Busca Full-Text Search
- **Barra de categorias sticky:** fixa logo abaixo do cabeçalho com chips horizontais (nunca carrossel escondendo itens) e *scroll spy* marcando a categoria visível (`scroll-margin-top` compensando a barra fixa).
- **Busca FTS (Full-Text Search):**
  - Input acessível (`font-size >= 16px` no iOS) com debounce de 250ms chamando `GET /api/v1/public/branches/{branchSlug}/products/search?q=...`.
  - Tolera ausência de acentos ("acai" → "Açaí") e pequenos erros de digitação ("x-salda" → "X-Salada") providos pelo PostgreSQL 19 (`pt_unaccent` + `pg_trgm`).
  - Estado vazio orientativo ("Nenhum prato encontrado para '...'. Limpar busca").
- **Card de produto (Mobile-First):**
  - Foto em **WebP** (`loading="lazy"`, `width`/`height` fixos), nome, descrição curta (máx. 2 linhas), badge de promoção (`#F59E0B` com texto `#1C1917`), preço em `BRL` (de/por se promocional).
  - Item esgotado aparece claramente marcado como **"Indisponível"** (com ícone + texto) e não pode ser adicionado.
- **Skeleton sem flash:** aparece somente se a carga ultrapassar 200ms.

### 3.3 Detalhe do Produto, Adicionais Obrigatórios & Observação
- Ao tocar em um produto, abre folha inferior (mobile) ou dialog (desktop) usando **Base UI Drawer/Dialog**:
  - Exibe todos os grupos de variações/adicionais com regra explícita (`Obrigatório · Escolha 1`, `Opcional · Até 3`).
  - **Bloqueio preventivo:** o botão "Adicionar à sacola · R\$ XX,XX" só habilita após todos os grupos obrigatórios atingirem `min_select`, mostrando aviso claro do que falta escolher (evitando pedidos incompletos).
  - Campo livre de **Observação** (`ex.: "Sem cebola, ponto da carne bem passado"`, máx. 200 chars).
  - Seletor de quantidade com alvos `>= 44x44px` e cálculo dinâmico em centavos inteiros.

### 3.4 Sacola Persistida (Bottom Sheet)
- Botão flutuante fixo na base (mobile) ou botão no cabeçalho (desktop) com quantidade total e subtotal parcial.
- Persiste os itens da sacola por `branchSlug` (com versão de schema e TTL de 24h) para que o cliente não perca o pedido se alternar para o WhatsApp e voltar.
- Permite alterar quantidade, remover item, revisar adicionais/observações e limpar sacola.
- Deixa explícito que o valor exibido é **subtotal parcial** (frete e cupom entram no checkout).
- Fecha por alça/botão, clique no backdrop ou tecla `Escape`, devolvendo o foco ao botão que abriu.

### 3.5 Checkout em 3 Passos (Mobile-First)
No mobile, exibe **1 passo por vez** com indicador textual (`Passo 1 de 3`, não só cor); no desktop, mantém o **Resumo do Pedido sticky** na coluna lateral. **Voltar um passo nunca apaga os dados já digitados.**

1. **Passo 1 — Entrega ou Retirada:**
   - Dois botões grandes e claros (controle segmentado): **Entrega** vs **Retirada** (exibe apenas modalidades habilitadas pela filial).
   - Se **Retirada**: exibe endereço da filial e zera frete (`R\$ 0,00`).
   - Se **Entrega**: pede CEP (8 dígitos) e chama imediatamente `POST /api/v1/public/branches/{branchSlug}/shipping/quote`.
     - **CEP fora da área:** exibe alerta claro (`#B91C1C` + ícone + texto: *"Não entregamos neste CEP"*) e **bloqueia o avanço antes de pedir pagamento**, oferecendo atalho para trocar para Retirada (se disponível).
     - **CEP válido:** exibe valor do frete e tempo estimado, abrindo campos de endereço (rua, número, complemento, bairro, referência).
2. **Passo 2 — Identificação (Convidado ou Conta Opcional):**
   - Se já logado via cookie `HttpOnly`: pré-preenche nome e telefone com opção de editar.
   - Se convidado: pede apenas **Nome** e **Telefone/WhatsApp** (sem obrigar senha) + botão discreto "Já tenho conta / Entrar" e aviso claro com links para `menuchef.com.br/termos-de-uso` e `menuchef.com.br/politica-de-privacidade`.
   - Ao optar por criar conta opcional, exige aceite versionado de Termos e Política de Privacidade (`accepted_terms_version`, `accepted_privacy_version`).
3. **Passo 3 — Cupom, Pagamento e Resumo Final:**
   - Campo de **Cupom** com validação imediata (`POST .../coupons/validate`), exibindo desconto ou motivo exato de invalidez inline. Caso o cupom esgote no exato instante do envio (`409 COUPON_UNAVAILABLE_OR_EXHAUSTED`), exibe alerta claro permitindo remover o cupom e concluir sem perder os dados.
   - Seleção clara entre:
     - **Pagar agora (Online — Modo Mocado no MVP):**
       - **Pix (Mocado):** cria pedido em `awaiting_payment`, gera tela de QR Code + botão "Copiar código Pix" + controles de demonstração/teste do MVP ("Simular aprovação", "Simular recusa", "Simular expiração") quando `VITE_MOCK_PAYMENT_UI=true`.
       - **Cartão (Mocado):** formulário de cartão de teste (sem armazenar PAN/CVV), com indicação dos números de teste de aprovação/recusa no ambiente MVP.
     - **Pagar na entrega / retirada** (só exibido se a filial habilitou): **Dinheiro** (campo opcional "Precisa de troco para quanto?"), **Pix na hora** ou **Maquininha da loja** — deixando claro que o pagamento ocorre na porta.
   - **Resumo final completo:** itens + adicionais + observações, subtotal, frete, desconto, total final, endereço ou retirada e forma de pagamento.
   - **Proteção de Idempotência (`Idempotency-Key`):** o cliente gera um UUIDv4 por intenção de checkout (mantido em retentativas de rede e renovado somente se o usuário alterar itens/endereço/pagamento/cupom). Trata `409 IDEMPOTENCY_IN_PROGRESS` (aguarda curto intervalo respeitando `Retry-After` sem criar duplicata) e `422 IDEMPOTENCY_PAYLOAD_MISMATCH` (regenera chave e informa o usuário), com botão em estado de loading preservando o rótulo.

### 3.6 Acompanhamento em Tempo Real (SSE) & Cancelamento Pré-Confirmação
- Tela `/$branchSlug/pedido/$trackingToken` acessível imediatamente após fechar o pedido (e compartilhável/favoritável sem login):
  - Exibe **Número curto em destaque** (`Pedido #042`), comprovante completo e linha do tempo com os mesmos nomes da cozinha:
    `Recebido → Confirmado → Em produção → Pronto → Aguardando retirada / Saiu para entrega → Concluído`.
  - Conecta via `EventSource` em `/api/v1/public/orders/{trackingToken}/stream` atualizando a linha do tempo em tempo real (com reconexão automática e fallback polling se a rede bloquear SSE).
  - **Botão "Cancelar pedido":** visível e ativo **somente** enquanto o status for `awaiting_payment` ou `received`. Exige Dialog de confirmação (e informa que eventual cupom utilizado será estornado automaticamente). Assim que a cozinha confirma (`confirmed`), o botão some e a tela explica que o preparo já começou (exibindo contato da loja).
  - Se o cliente fechou como convidado, oferece convite leve pós-pedido: *"Quer salvar seu histórico para pedir de novo depois? Crie sua senha em 1 passo."* (com aceite versionado de Termos/Privacidade).

### 3.7 Conta Opcional do Cliente, "Pedir de Novo" & Autoatendimento LGPD
- **Rota `/minha-conta/pedidos`** (autenticada por cookie `HttpOnly`):
  - Lista paginada via **Cursor Opaco + Keyset (`UUIDv7`)** (`GET /api/v1/customer/orders?cursor=&limit=20` com `useInfiniteQuery` e botão **"Carregar mais"** — sem `OFFSET`, imune a duplicação quando um novo pedido entra) exibindo loja, data, número curto, itens e status final.
  - Botão **"Pedir de novo"**: chama `POST /api/v1/customer/orders/{orderId}/reorder-preview`, recarrega a sacola daquela filial com os itens disponíveis e avisa claramente caso algum item esteja esgotado ou tenha mudado de preço antes do checkout.
- **Rota `/minha-conta/privacidade`** (direitos do titular LGPD no MVP):
  - Links para `/termos-de-uso` e `/politica-de-privacidade` (com versão aceita pelo usuário).
  - **"Baixar meus dados (JSON)":** dispara `GET /api/v1/customer/privacy/export` e baixa arquivo `.json` com perfil, endereços, aceites legais e histórico de pedidos.
  - **"Excluir minha conta":** abre Dialog de confirmação explícita informando que os dados pessoais serão anonimizados imediatamente, preservando apenas totais financeiros/operacionais das lojas (obrigação legal), dispara `POST /api/v1/customer/privacy/delete-account` e encerra a sessão.

---

## 4. Responsividade (Mobile, Tablet e Desktop)

- **Mobile (`375px`–`639px`):** 1 coluna, barra sticky de categorias, botão de sacola fixo no rodapé (safe-area iOS `env(safe-area-inset-bottom)`), Bottom Sheet para produto e sacola, checkout 1 passo por tela.
- **Tablet (`768px`–`1023px`):** grade de 2 colunas para produtos, drawer lateral (420px) para sacola.
- **Desktop (`>= 1024px`):** catálogo em grade 2–3 colunas com largura máxima confortável (`max-w-5xl`), checkout em 2 colunas (formulário à esquerda + resumo sticky à direita).
- **Rodapé discreto em todas as telas:** links diretos para Termos de Uso e Política de Privacidade do Menu Chef.

---

## 5. Contratos de API Consumidos (`same-origin` via `openapi-fetch`)

| Ação | Endpoint |
| :--- | :--- |
| Carregar filial + cardápio | `GET /api/v1/public/branches/{branchSlug}/menu` |
| Busca FTS | `GET /api/v1/public/branches/{branchSlug}/products/search?q={term}` |
| Cotar frete por CEP | `POST /api/v1/public/branches/{branchSlug}/shipping/quote` |
| Validar cupom | `POST /api/v1/public/branches/{branchSlug}/coupons/validate` |
| Criar pedido (idempotente) | `POST /api/v1/public/branches/{branchSlug}/orders` (`Idempotency-Key`) |
| Detalhe / SSE / Cancelar | `GET /api/v1/public/orders/{trackingToken}` · `.../stream` · `POST .../cancel` |
| Simular pagamento mocado (MVP) | `POST /api/v1/public/orders/{trackingToken}/mock-payment/simulate` |
| Auth & Histórico do cliente | `POST /api/v1/auth/register-customer` · `POST /api/v1/auth/login` · `GET /api/v1/customer/orders` · `POST .../reorder-preview` |
| Autoatendimento LGPD | `GET /api/v1/customer/privacy/export` · `POST /api/v1/customer/privacy/delete-account` |

---

## 6. Critérios de Aceite & Plano de Testes (`apps/storefront`)

- [ ] **Unitários (Vitest + RTL + MSW):**
  - Cálculos de subtotal, adicionais, desconto e total em centavos inteiros;
  - Validação de grupos de adicionais obrigatórios bloqueando "Adicionar";
  - Persistência/restauração da sacola por `branchSlug` sem misturar duas filiais e sem gravar credenciais;
  - Geração, reutilização em retry e renovação de `Idempotency-Key` ao alterar payload do checkout, mais tratamento de `409`/`422`;
  - Comportamento de loja pausada/fechada bloqueando adição;
  - Validação Zod do checkout (CEP, telefone, troco menor que total rejeitado, aceite legal obrigatório ao criar conta).
- [ ] **E2E (Cypress — viewports `375x812`, `768x1024`, `1440x900`):**
  - Busca FTS sem acento e com typo;
  - Montagem de sacola com adicional obrigatório + observação + reload mantendo sacola;
  - CEP fora da área bloqueando avanço antes do pagamento;
  - Checkout completo como convidado com **Pix mocado** (incluindo proteção contra clique duplo via `Idempotency-Key`, aprovação + comprovante `#001`) e com **Cartão mocado** (recusa → retentativa aprovada);
  - Checkout **Retirada** (frete zero) + pagamento na retirada;
  - Cancelamento autônomo antes da confirmação (com estorno de cupom) e bloqueio após confirmação;
  - Fluxo de conta opcional (com aceite legal) + "Pedir de novo" + **exportação JSON e exclusão/anonimização LGPD** + checagem `cypress-axe` (WCAG 2.2 AA).
