# Spec de Aplicação — Painel Administrativo PWA (`apps/admin`)

> **Documento autocontido.** Especifica o **Painel Administrativo PWA** (`app.menuchef.com.br`) que atende, em uma única aplicação protegida por RBAC, tanto as **Empresas/Filiais** (operação de pedidos em tempo real, gestão de cardápio, frete por CEP, cupons, equipe, impressão 1-clique e dashboard) quanto o **Administrador da Plataforma** (onboarding assistido de novas empresas, gestão de planos/status e acesso de suporte com rastro obrigatório de auditoria).

---

## 1. Propósito, Perfis e Separação por RBAC

Um único aplicativo **PWA** (instalável no tablet do balcão, celular do dono ou desktop da cozinha), usando **sempre a Paleta Brasa** (modo claro e **modo escuro operacional para a cozinha à noite**) para que os significados de status nunca mudem entre restaurantes.

| Área / Rota Base | Perfis Autorizados | Responsabilidade |
| :--- | :--- | :--- |
| `/entrar`, `/convite/$token` | Público (não autenticado) | Login único global e aceite de convite gerado pelo Admin da Plataforma ou Dono da Empresa (exigindo aceite versionado dos Termos de Uso e Política de Privacidade gravado em `legal_acceptances`) |
| `/c/$companySlug/b/$branchSlug/*` | `company_owner`, `branch_manager`, `kitchen_operator`, `dispatcher` | Operação diária da filial selecionada (permissões estritas pelo papel naquele vínculo) |
| `/plataforma/*` | `platform_admin` | Gestão SaaS: fila de leads, **onboarding assistido** (criar empresa + 1ª filial + convite), status/planos, consulta de aceites legais e visão de suporte auditada |

---

## 2. Stack Tecnológica (`apps/admin`)

| Camada | Tecnologia | Motivo Técnico (2026) |
| :--- | :--- | :--- |
| **Core** | **React 19 + TypeScript (`strict: true`) + Vite** | SPA autenticado de alta responsividade |
| **PWA** | **`vite-plugin-pwa` v2** (`strategies: 'injectManifest'`, `registerType: 'prompt'`) | Workbox 7 + Service Worker customizado (`src/sw.ts`) para Web Push, precache de assets/áudio de alerta e aviso controlado de atualização (**nunca `autoUpdate`**, evitando reload forçado no meio do rush!) |
| **Roteamento** | **TanStack Router** (file-based) | **Contexto de Empresa e Filial refletido na URL** (`/c/$companySlug/b/$branchSlug/...`) + filtros de pedidos/cardápio em search params tipados |
| **Dados, Tabelas & HTTP Tipado** | **TanStack Query v5 + TanStack Table v9 + `openapi-fetch`** | Table v9 (ago/2026, reativa via TanStack Store, compatível com React Compiler) + cliente HTTP 100% tipado gerado do OpenAPI 3.1 da API Go (`openapi-typescript`) |
| **Formulários** | **TanStack Form + Zod** | Validação granular por campo sem re-renderizar o formulário inteiro; erros sempre inline ao lado do campo |
| **UI & Animações** | **Tailwind CSS v4 + shadcn/ui v4 (`@base-ui/react`) + Motion** | Sidebar + Card + Table + Sheet/Drawer + Dialog; `<MotionConfig reducedMotion="user">`; movimento mínimo e objetivo na fila |
| **Tours Guiados** | **React Joyride `3.2.0`** (MIT, React 19 + Floating UI) | Named export `import { Joyride } from 'react-joyride'`, hook `useJoyride()` e `onEvent(data, controls)` para onboarding guiado sem poluir a tela |
| **Sessão** | Cookie `HttpOnly; Secure; SameSite=Lax` (`__Host-mc_session`) | **Zero uso de `localStorage` para auth/sessão**; toda chamada mutável protegida por `CrossOriginProtection` |

---

## 3. Troca de Contexto Multi-Empresa e Multi-Filial (Diferencial Central)

- O usuário faz **1 único login** e recebe de `GET /api/v1/auth/me` todas as empresas e filiais em que possui vínculo ativo, com o respectivo papel em cada filial.
- **Seletor no topo (Header/Sidebar):**
  - Exibe **Empresa** e **Filial** em texto claro (nunca apenas cor/ícone).
  - Se o usuário só pertence a 1 empresa e 1 filial, o seletor fica oculto/estático (evita controle inútil no primeiro dia).
- **Ao trocar de Empresa/Filial:**
  1. Dispara `POST /api/v1/auth/context` (valida vínculo no servidor e **rotaciona o Session ID** no cookie `HttpOnly`).
  2. Navega para `/c/{novoCompanySlug}/b/{novoBranchSlug}/fila`.
  3. Encerra a conexão SSE anterior, limpa caches TanStack Query da filial antiga e abre imediatamente o stream SSE e permissões da nova filial.
  4. Lembra a última filial usada (salva no perfil no backend) para o próximo login.

### Matriz de Permissões por Papel na Filial
| Capacidade | `company_owner` | `branch_manager` | `kitchen_operator` | `dispatcher` |
| :--- | :---: | :---: | :---: | :---: |
| Ver fila, confirmar, produzir, marcar pronto, imprimir | Sim | Sim | Sim | Leitura + pronto |
| Despachar entrega / marcar entregue / retirada | Sim | Sim | Não | Sim |
| Pausar/retomar loja & marcar produto esgotado | Sim | Sim | Esgotado | Não |
| Editar cardápio, preços, cupons, horários, faixas de CEP | Sim | Sim | Não | Não |
| Ver dashboard operacional e extrato de pagamentos | Sim | Sim | Não | Não |
| Gerenciar equipe da filial | Sim | Sim | Não | Não |
| Criar nova filial e gerenciar dados da empresa | Sim | Não | Não | Não |

---

## 4. Operação da Fila de Pedidos em Tempo Real (Kanban + SSE)

### 4.1 Conexão SSE e Resiliência
- Conecta via `EventSource` em `GET /api/v1/admin/branches/{branchId}/orders/stream`.
- Exibe indicador discreto mas visível no topo: **"Ao vivo"** (verde + ícone) ou **"Reconectando..."** (âmbar + ícone + aviso caso perca internet).
- Ao reconectar, recebe `event: snapshot` completo da fila ativa, garantindo que nenhum pedido se perca se o Wi-Fi da cozinha oscilar.

### 4.2 Colunas e Ações Acessíveis
Colunas espelhando exatamente os nomes que o cliente vê:
1. **Recebidos (`received`)** — destaque de alerta sonoro + cronômetro ("há 2 min"). Ações: **"Confirmar pedido"** (primário `48px`) e **"Recusar"** (abre Dialog exigindo motivo: item acabou, cozinha lotada, outro — e avisa quando houver cupom que o uso será estornado automaticamente).
2. **Confirmados (`confirmed`)** — ação **"Iniciar produção"** + **"Imprimir comanda"**.
3. **Em produção (`in_production`)** — ação **"Marcar pronto"**.
4. **Prontos (`ready` / `awaiting_pickup` / `out_for_delivery`)** — separa visualmente com badge claro **Retirada (aguardando cliente)** vs **Entrega (despachar / em rota)** → ação **"Marcar concluído/entregue"**.

- **Acessibilidade operacional:** todas as transições funcionam via **botão explícito** (navegável por teclado e toque rápido no tablet); arrastar-e-soltar é no máximo opcional, nunca obrigatório.
- **Cartão da fila exibe imediatamente:** número curto grande (`#042`), nome do cliente, **Entrega (bairro)** ou **Retirada**, tempo decorrido, resumo de itens/adicionais/observações em destaque e selo de pagamento (**Pago online (Pix/Cartão mocado)** vs **Cobrar na entrega/retirada — Dinheiro/Troco/Maquininha**).

### 4.3 Impressão de Comanda (1-Clique — MVP)
- Botão **"Imprimir comanda"** no cartão e no detalhe do pedido abre layout dedicado via `window.print()` com folha CSS `@media print` configurável para **58mm** ou **80mm** (preferência da filial).
- Layout térmico monocromático de alto contraste: nome da filial, **`#042`** em fonte grande, data/hora no fuso da filial, modalidade (ENTREGA + endereço completo ou RETIRADA), lista de itens com **adicionais indentados** e **OBSERVAÇÕES EM NEGRITO/CAIXA ALTA**, forma de pagamento (PAGO vs A RECEBER + troco). Sem linha de taxa da plataforma.
- *Impressão silenciosa automática via agente local/WebSerial fica documentada para marco pós-MVP.*

---

## 5. Arquitetura PWA, Alerta Sonoro Híbrido, Wake Lock e Modo Escuro

Baseada nas limitações reais dos navegadores em 2026:
1. **Manifest & Instalação (`vite-plugin-pwa` v2):**
   - `display: "standalone"` (obrigatório para Web Push no iOS/iPadOS), ícones `192x192`, `512x512` e `maskable` com o símbolo Concha com vapor, `theme_color: "#C2410C"`.
   - `registerType: "prompt"`: quando uma nova versão do painel é publicada, exibe banner discreto *"Nova versão disponível — Atualizar agora"* para o operador clicar fora do pico.
   - Precache do app shell e dos arquivos de áudio de alerta (`/sounds/new-order.mp3`).
2. **Alerta Sonoro Híbrido (Limitação Real de PWAs Documentada):**
   - **Em primeiro plano (aba/PWA aberto na cozinha):** botão explícito **"Ativar alerta sonoro"** no cabeçalho (cumprindo política de autoplay de iOS/Chrome, desbloqueando `AudioContext`). Quando chega evento SSE `order_created` (ou mensagem do Service Worker), toca som nítido de campainha/alarme até o pedido ser confirmado ou silenciado.
   - **Tela sempre acesa:** toggle **"Manter tela acesa"** usando **`Screen Wake Lock API`** (suportado em Android, Desktop e iOS 18.4+ Home Screen PWA).
   - **Em segundo plano (Web Push):** Service Worker customizado (`src/sw.ts`) recebe payload compatível com **Declarative Web Push** (iOS 18.4+) e exibe notificação visível (`showNotification` obrigatório no iOS para não revogar permissão) + `navigator.setAppBadge` com a contagem de pedidos aguardando confirmação. *Nota explícita de UX:* em background, o som tocado é o som padrão do sistema operacional (nenhum PWA web garante sirene customizada com tela bloqueada no iOS).
3. **Modo Escuro Operacional:** toggle rápido no cabeçalho (persiste preferência no perfil/CSS) alternando para a paleta Brasa escura (`#1C1410` / `#2A211C` / `#FAF6F1` / primária `#FB923C`).

---

## 6. Módulos de Gestão da Filial & Dashboard

1. **Cardápio & Promoções:** tabela densa (TanStack Table v9) agrupada por categoria com edição rápida de preço, **switch imediato de "Disponível / Esgotado"** (1 clique no meio do jantar), gestão de grupos de adicionais (min/max/obrigatório) e upload de foto (preview + validação de tamanho/formato → backend converte e salva em **WebP**).
2. **Horários, fuso e pausa de emergência:** grade semanal de abertura/fechamento em relógio de parede + seletor de fuso IANA (`America/Sao_Paulo` como default, `America/Manaus` entre as opções; offset cru é rejeitado) + **botão de destaque "Pausar recebimento de pedidos"** no topo do painel com motivo rápido (Chuva, Forno/Equipamento, Fila cheia). O "aberto agora" que a loja vê é o mesmo cálculo do storefront, no fuso da filial.
3. **Frete (Fixo ou Faixas de CEP):** alternância clara entre frete fixo ou tabela de faixas (`CEP inicial`, `CEP final`, `Valor`, `Tempo estimado`, `Ativa`), com validação imediata de faixa inválida ou sobreposta (formulário tabular rápido, sem peso de mapa no MVP).
4. **Cupons:** formulário enxuto (`Código`, `Tipo`: valor fixo / percentual / frete grátis, `Pedido mínimo`, `Validade`, `Limite total de usos`, `Limite por cliente`, `Ativo`), exibindo contagem real de usos (`used_count / max_uses`). O campo **Limite por cliente** traz a ajuda: *"Só quem tem conta logada consegue usar. Convidado vê um pedido para entrar. Vazio = vale para todo mundo."* Decisão do dono em 23/11/2026. O identificador é a conta, não o telefone.
5. **Equipe da Filial:** lista de pessoas com papel na filial, envio de convite por e-mail (se a pessoa já tem conta no Menu Chef em outra loja, apenas vincula à nova filial sem criar nova senha!) e revogação instantânea de acesso.
6. **Dashboard Operacional & Extrato:**
   - Filtro por período (Hoje, 7 dias, 30 dias, personalizado). **Hoje** é a data civil no fuso da filial: um pedido às `2026-11-23T03:30:00Z` entra no dia 23 em São Paulo e no dia 22 em Manaus. Cards: **Pedidos concluídos**, **Faturamento bruto** (soma dos pedidos, sem desconto de comissão — não há comissão), **Ticket médio**, **Entrega vs Retirada** e **Top 10 produtos** (os totais permanecem íntegros mesmo quando clientes finais exercem exclusão/anonimização LGPD).
   - **Extrato de pagamentos online (mocado no MVP):** lista por pedido com valor, método (`Pix mocado` / `Cartão mocado`) e status. Não há coluna de taxa da plataforma. O texto de apoio diz que o Menu Chef cobra só a mensalidade. Quando existir gateway real, taxa do provedor, se mostrada, leva o rótulo de taxa do provedor.
7. **Padrão Único de Paginação nas Tabelas (TanStack Table v9 + Cursor Opaco):**
   - Todas as tabelas do painel (Pedidos históricos, Cardápio, Faixas de CEP, Cupons, Equipe, Extrato, Leads, Empresas, Aceites Legais e Auditoria) usam **Cursor Pagination opaco** (`?cursor=&limit=`) integrado ao **TanStack Table v9** (`useTable` + `manualPagination: true` + `pageCount: -1`) e **TanStack Query v5 (`useInfiniteQuery`)**.
   - Controles de navegação: **"Anterior"** e **"Próxima"** (sem botão "Última página" e sem `COUNT(*)` pesado); páginas já visitadas ficam no cache do `useInfiniteQuery` para retorno instantâneo.
   - Qualquer alteração em filtro, busca ou ordenação reseta `pageIndex` para `0` e inicia nova cadeia de cursores.
8. **Governança Legal & Rodapé:** links permanentes no menu de Ajuda/rodapé para `menuchef.com.br/termos-de-uso` e `menuchef.com.br/politica-de-privacidade`, explicitando o papel de **Controladora** da Empresa/Filial sobre os dados de clientes dos pedidos e de **Operador** do Menu Chef.

---

## 7. Tours Guiados com React Joyride v3.2.0

Implementados com `react-joyride` `3.2.0` (`import { Joyride, useJoyride } from 'react-joyride'`), estilizados com tokens Brasa (claro e escuro) e respeitando `prefers-reduced-motion`:
- **Tour 1 — Primeiro Acesso da Filial (`onboarding_v1`):** guia em 5 passos rápidos: (1) cadastrar 1ª categoria/produto, (2) definir horários e frete por CEP, (3) copiar link do cardápio público, (4) entender o botão de pausa rápida e (5) ativar o som de alerta da fila.
- **Tour 2 — Troca de Contexto (`multi_branch_v1`):** exibido uma única vez quando o usuário passa a ter vínculo com >= 2 empresas/filiais, destacando o seletor no topo.
- Estado de conclusão/pulo salvo via `PATCH /api/v1/auth/me/tours`, com botão **"Rever guia rápido"** sempre disponível no menu de Ajuda.

---

## 8. Área do Administrador da Plataforma (`/plataforma/*`)

Acesso restrito a usuários com `is_platform_admin = true` (verificado no servidor em todas as requisições):
1. **Leads Recebidos (`/plataforma/leads`):** lista de contatos vindos da Landing Page (exibindo versão aceita da Política de Privacidade) com filtros de status (`Novo`, `Em contato`, `Convertido`, `Arquivado`) e ação **"Criar empresa a partir deste lead"**.
2. **Empresas & Onboarding Assistido (`/plataforma/empresas`):**
   - Formulário transacional para cadastrar **Nova Empresa + Primeira Filial + Convite do Dono** (define plano, mensalidade e resumo transparente dos termos; exibe/envia link de convite seguro com expiração).
   - Listagem com estado comercial (`Ativa`, `Pausada`, `Suspensa`), versão de Termos/Privacidade aceita (`GET /api/v1/platform/legal-acceptances`) e ações **Suspender / Reativar** exigindo motivo visível para a empresa (impedindo bloqueios silenciosos).
3. **Suporte Auditado (`/plataforma/suporte`):**
   - Permite selecionar uma empresa/filial e abrir a fila de pedidos em **modo somente-leitura** mediante preenchimento obrigatório do campo **Motivo do acesso**.
   - Exibe banner fixo no topo: *"Modo Suporte (Somente Leitura) — Acesso registrado em auditoria"*.
   - **Proibido impersonar a loja para alterar pedidos ou agir sem rastro.**
   - Tela de consulta da trilha imutável (`support_access_audit_logs`) com quem acessou, quando, qual filial e motivo.

---

## 9. Responsividade (Mobile, Tablet e Desktop)

- **Mobile (`375px`–`639px`):** barra de navegação inferior/drawer, Kanban em **abas por status** (`Recebidos (3)`, `Produção (2)`, `Prontos (1)`) para leitura sem scroll horizontal confuso, botões `>= 48px`.
- **Tablet (`768px`–`1023px` — principal tela de balcão/cozinha):** sidebar colapsável em ícones, Kanban em 3–4 colunas com cartões de alto contraste, modo escuro e Wake Lock a 1 toque.
- **Desktop (`>= 1024px`):** sidebar fixa com seletor Empresa/Filial, Kanban completo, tabelas TanStack Table v9 com busca/ordenação/paginação e dashboard em grade ampla.

---

## 10. Critérios de Aceite & Plano de Testes (`apps/admin`)

- [ ] **Unitários (Vitest + RTL + MSW):**
  - Controle de renderização por papel RBAC (`kitchen_operator` não vê menus financeiros/equipe);
  - Troca de contexto limpa estado da filial anterior e atualiza URL;
  - Validação Zod de faixas de CEP, cupons (`max_uses` e `max_uses_per_customer`, com o texto de ajuda de conta logada), fuso IANA, preços em centavos, aceite legal obrigatório em `/convite/$token` e recusa com motivo obrigatório;
  - "Hoje" do dashboard usa a data civil da filial (virada Manaus × São Paulo em `2026-11-23T03:30:00Z`);
  - Hooks de SSE reconexão, Wake Lock fallback e controle do tour React Joyride v3 (`onEvent`).
- [ ] **Contrato OpenAPI & Type-check:** zero drift entre tipos gerados por `openapi-typescript` (`openapi-fetch`) e `tsc --noEmit`.
- [ ] **PWA Audit:**
  - `manifest.webmanifest` válido, Service Worker `injectManifest` registrado sem erros, estratégia `prompt` exibindo banner de atualização sem reload automático, precache do áudio `/sounds/new-order.mp3`.
- [ ] **E2E (Cypress — `375x812`, `768x1024`, `1440x900`):**
  - Fluxo completo de **Onboarding Assistido** (Admin Plataforma cria empresa + 1ª filial + convite → Dono aceita convite + Termos/Privacidade versionados → completa/pula tour React Joyride);
  - Troca de contexto entre 2 filiais/empresas sem novo login validando alteração de fila e permissões;
  - Operação completa da fila em tempo real via SSE (`received → confirmed → in_production → ready → completed`), recusa com motivo (verificando estorno automático de cupom) e preview de impressão da comanda;
  - Pausa rápida da loja e toggle "esgotado" refletindo imediatamente no Storefront;
  - Modo escuro operacional validado com `cypress-axe` (WCAG 2.2 AA);
  - Acesso de suporte auditado bloqueando ações de escrita e gravando log visível na trilha de auditoria.
