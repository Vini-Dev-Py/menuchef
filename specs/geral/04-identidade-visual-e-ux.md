# Spec Geral 04 — Identidade Visual, UX Mobile-First, Acessibilidade e UI Stack

> **Documento autocontido.** Consolida a identidade visual escolhida em `docs/pesquisa/` (**Paleta Brasa clara/escura** e marca **Concha com vapor**), regras de responsividade **Mobile-First** (mobile, tablet e desktop), acessibilidade **WCAG 2.2 AA**, convenções da UI stack (**Tailwind CSS v4**, **shadcn/ui v4 + Base UI**, **Motion**, **React Joyride v3.2.0**) e disciplina de animações.

---

## 1. Identidade de Marca: Concha com Vapor

Decisão fechada em 21/11/2026 (`docs/pesquisa/logo.md`):
- **Símbolo:** concha de cozinha preenchida com o miolo na cor do fundo e cabo curto saindo para baixo à esquerda; curvas arredondadas de vapor sobem formando um **`m`** minúsculo (sem parecer chama, coroa, sorriso ou rosto).
- **Lettering:** **"Menu Chef"** em tipografia sem serifa humanista, peso semibold, caixa alta e baixa (com acento em "Menu" quando aplicável na copy institucional ou fiel ao wordmark oficial definido na arte final, sem script/caligrafia).
- **Cores do Logo:**
  - **Modo Claro:** símbolo `#C2410C`, texto `#1C1917` sobre `#FFFBF5` ou `#FFFFFF`.
  - **Modo Escuro:** símbolo `#FB923C` (contraste `8,02:1` sobre `#1C1410` — **nunca usar `#C2410C` sobre fundo escuro**, pois cai para `3,51:1`), texto `#FAF6F1`.
  - **Monocromática (impressão térmica / carimbo):** `#1C1917` sobre claro ou `#FAF6F1` sobre escuro.
  - **Proibido:** amarelo `#F59E0B` no logo, gradientes, 3D, sombras, esticar/deformar.
- **Tamanhos reduzidos (favicon 16px / ícone PWA):** usar apenas o símbolo simplificado (sem texto; em 16px reduzir o vapor para não virar ruído).
- **Separação Marca da Plataforma vs Marca da Loja:**
  - Landing Page e Painel Admin vestem **sempre** a marca Menu Chef + Paleta Brasa (para que "pronto", "atrasado" e "recusado" nunca mudem de significado entre restaurantes).
  - Cardápio Digital (`storefront`) usa a Brasa como tema padrão e permite que cada **filial** sobrescreva seu logo e cor primária (com validação automática de contraste AA para o texto do botão primário).

---

## 2. Sistema de Cores: Paleta Brasa (Claro & Escuro)

Todos os tokens são definidos via CSS-first no **Tailwind CSS v4** (`@theme` e variáveis CSS semânticas).

### Modo Claro (Landing, Storefront padrão e Painel diurno)
| Token Semântico | Hex | Contraste Verificado | Uso Obrigatório |
| :--- | :--- | :--- | :--- |
| `--color-brand-primary` | `#C2410C` | `5,18:1` com `#FFFFFF` / `5,02:1` sobre `#FFFBF5` | CTA principal, ação primária, marca |
| `--color-brand-secondary` | `#44403C` | `9,96:1` sobre `#FFFBF5` | Apoio, navegação secundária, labels fortes |
| `--color-brand-accent` | `#F59E0B` | `8,14:1` com `#1C1917` (**proibido texto branco: `2,15:1`**) | Selos de promoção e destaque, **sempre com texto `#1C1917`** |
| `--color-bg-canvas` | `#FFFBF5` | — | Fundo principal da página (creme quente) |
| `--color-bg-surface` | `#FFFFFF` | — | Cards, sacola, modais, cartões da fila |
| `--color-text-primary` | `#1C1917` | `16,96:1` sobre `#FFFBF5` / `17,49:1` sobre `#FFFFFF` | Texto principal |
| `--color-status-success` | `#15803D` | `5,02:1` com branco | Confirmado, pronto, pago |
| `--color-status-warning` | `#B45309` (texto: `#92400E`) | `5,02:1` com branco / `6,88:1` texto no creme | Atraso, pausa da loja, atenção |
| `--color-status-error` | `#B91C1C` | `6,47:1` com branco | Recusado, fora da área de CEP, erro de pagamento |
| `--color-status-info` | `#1D4ED8` | `6,70:1` com branco | Saiu para entrega, aviso neutro |

### Modo Escuro Operacional (Painel Admin — Cozinha e Balcão à Noite)
| Token Semântico | Hex | Contraste Verificado | Uso |
| :--- | :--- | :--- | :--- |
| `--color-bg-canvas` | `#1C1410` | — | Fundo escuro quente (reduz ofuscamento noturno) |
| `--color-bg-surface` | `#2A211C` | — | Cartões de pedido, sidebars, drawers |
| `--color-text-primary` | `#FAF6F1` | `16,87:1` no fundo / `14,64:1` na superfície | Texto principal |
| `--color-brand-primary` | `#FB923C` | `8,02:1` com texto `#1C1410` | Botões primários no escuro |
| `--color-brand-accent` | `#FDBA74` | `10,76:1` no fundo (texto `#1C1410`) | Destaques e badges |
| Status (texto/ícone no escuro) | Sucesso `#86EFAC` · Alerta `#FDE68A` · Erro `#FECACA` · Info `#BFDBFE` | > `12,5:1` sobre `#1C1410` | Leitura instantânea à distância |

> **Regra obrigatória de status:** cor **nunca** é o único indicador. Como alerta `#B45309` e primária `#C2410C` são da mesma família quente, todo status exibe **cor + ícone + rótulo textual explícito** ("Em produção", "Pronto", "Loja pausada").

---

## 3. Estratégia Mobile-First (Mobile, Tablet e Desktop)

Todo CSS e layout é projetado primeiro para telas estreitas (`360px`–`430px`) e escalado progressivamente pelos breakpoints do Tailwind CSS v4:

| Faixa | Breakpoint | Comportamento por Aplicação |
| :--- | :--- | :--- |
| **Mobile** | Base `< 640px` (`360px`–`430px`, testado a partir de `375px` sem scroll horizontal) | **Storefront:** 1 coluna, barra sticky de categorias, botão de sacola fixo na base, Bottom Sheet (Drawer Base UI), checkout 1 passo por tela. **Admin:** navegação inferior/drawer, Kanban em abas por coluna (1 status por vez com contador), ações grandes de toque. **Landing:** 1 coluna limpa com CTA imediato. |
| **Tablet** | `md` (`768px`) – `lg` (`1024px`) | **Storefront:** 2 colunas de produtos, sacola em drawer lateral. **Admin (caso principal de balcão/cozinha!):** layout otimizado para tablet horizontal (iPad/Android 10"), sidebar colapsável, Kanban com 3–4 colunas visíveis e botões de 48px para toque rápido com mão ocupada. **Landing:** grade 2 colunas. |
| **Desktop** | `xl` (`1280px`) – `2xl` (`1536px`) | **Storefront:** catálogo em grade + resumo lateral no checkout (sticky). **Admin:** sidebar persistente + Kanban completo lado a lado + tabelas densas (TanStack Table v9) para cardápio/CEP/cupons + dashboard completo. **Landing:** layout editorial amplo em 2 colunas com demonstração do fluxo. |

---

## 4. Acessibilidade (WCAG 2.2 Nível AA) & Ergonomia Web

Baseado na *Web Interface Guidelines* e *WCAG 2.2*:
- **Contraste mínimo:** `>= 4,5:1` para texto normal, `>= 3:1` para texto grande e componentes gráficos/bordas (critérios 1.4.3 e 1.4.11).
- **Alvos de toque:** mínimo **`44x44px`** em mobile/tablet (e `48x48px` em ações críticas da cozinha como "Confirmar" e "Pronto").
- **Inputs sem zoom forçado no iOS:** todos os campos de formulário (`input`, `select`, `textarea`) usam `font-size >= 16px` em mobile.
- **Teclado e Foco:** 100% navegável por teclado (`Tab`, `Enter`, `Escape`), `:focus-visible` sempre nítido, *focus trap* em modais/drawers com retorno de foco ao gatilho ao fechar.
- **Kanban acessível:** toda transição de pedido funciona via **botão explícito** ("Confirmar", "Iniciar produção", "Marcar pronto") além de qualquer gesto de arrastar.
- **Ações destrutivas:** excluir item, recusar pedido ou suspender empresa sempre exige confirmação explícita (Dialog) ou mecanismo de desfazer.
- **Estado na URL:** filial ativa, filtros de tabela, abas de status e busca no cardápio sincronizados na URL (TanStack Router type-safe search params) para preservar contexto no refresh e botão voltar.
- **Skeleton sem flash:** exibir skeleton apenas após 150–250ms de latência para evitar piscadas quando a API responde rápido.

---

## 5. UI Stack, Animações e Tours Guiados (Padrão 2026)

| Camada | Ferramenta | Diretriz de Uso |
| :--- | :--- | :--- |
| **Estilização** | **Tailwind CSS v4** | Configuração CSS-first (`@theme`), tokens semânticos da paleta Brasa, zero CSS inline arbitrário |
| **Primitivas Headless** | **shadcn/ui v4 + Base UI (`@base-ui/react`)** | Padrão oficial do shadcn desde jul/2026; componentes copiados para o repositório para controle total de acessibilidade e tokens. Inclui Drawer/Bottom Sheet nativo (substituindo o descontinuado Vaul) |
| **Animações** | **Motion (`motion/react`)** + CSS/WAAPI | Sempre envolver a árvore raiz com **`<MotionConfig reducedMotion="user">`** (o padrão da lib é `"never"`!). Movimento existe **apenas** para responder a interações reais (ex.: feedback ao adicionar na sacola, abertura de Bottom Sheet, entrada de novo card na fila). Zero animação contínua distrativa no painel da cozinha |
| **Tours Guiados** | **React Joyride `3.2.0`** (MIT, React 19 + Floating UI) | Usar named export `import { Joyride } from 'react-joyride'`, hook `useJoyride()` e `onEvent(data, controls)`. Aplicado no **Painel Admin** para onboarding guiado após o primeiro acesso (configurar cardápio, configurar CEP, operar a fila, trocar de filial). Progresso do tour salvo no perfil do usuário na API (com opção "Pular" e "Rever tour" na ajuda) |
| **Mídia** | **WebP** obrigatório | Todas as imagens estáticas da Landing e todas as fotos de produtos/logos enviadas pelas lojas servidas em `.webp` com `width`/`height` explícitos (evitar CLS) e `loading="lazy"` abaixo da dobra (`fetchpriority="high"` apenas na imagem LCP) |

---

## 6. Critérios de Aceite & Verificação

- [ ] Auditoria Lighthouse (Accessibility) >= 95 em Landing, Storefront e Admin (claro e escuro).
- [ ] Nenhum texto branco sobre `#F59E0B` e nenhum ícone `#C2410C` sobre `#1C1410`.
- [ ] Teste E2E Cypress executado em 3 viewports obrigatórias: **Mobile (`375x812`)**, **Tablet (`768x1024`)** e **Desktop (`1440x900`)**, garantindo ausência de overflow horizontal (`document.documentElement.scrollWidth <= window.innerWidth`).
- [ ] Com `prefers-reduced-motion: reduce` ativo, nenhuma animação de deslocamento/voo ocorre e todas as transições funcionam instantaneamente.
- [ ] Tour do React Joyride v3.2.0 no Admin permite avançar, pular via `Escape`/botão e não reaparece automaticamente após concluído.
