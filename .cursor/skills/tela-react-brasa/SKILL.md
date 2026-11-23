---
name: "tela-react-brasa"
description: "Cria tela ou componente React no padrão do Menu Chef (Tailwind v4, shadcn/Base UI, paleta Brasa, acessibilidade, mobile-first). Use ao mexer em apps/landing, apps/storefront ou apps/admin."
---

# Tela React (Brasa, mobile-first)

Use ao criar página, formulário ou componente em `apps/landing`, `apps/storefront` ou `apps/admin`. A visual está em `specs/geral/04-identidade-visual-e-ux.md`. O comportamento está na spec do app.

## Stack da tela

- React 19, TypeScript `strict`, Tailwind CSS v4, shadcn/ui v4 em cima de Base UI.
- Dados remotos com TanStack Query e `openapi-fetch`. Formulário com TanStack Form e Zod. Erro de campo fica inline, ao lado do campo.
- Motion só com `<MotionConfig reducedMotion="user">`. Sem animação que empurre o checkout inteiro.
- Admin: TanStack Table v9 com paginação por cursor (`manualPagination`, sem "última página").

## Visual e acesso

- Paleta Brasa. Claro: primária `#C2410C`, fundo `#FFFBF5`, texto `#1C1917`. Escuro da cozinha: fundo `#1C1410`, primária `#FB923C`. `#F59E0B` só com texto `#1C1917`. `#C2410C` não vai sobre fundo escuro.
- A landing e o admin vestem a marca Menu Chef. O storefront pode usar a cor e o logo da filial, com contraste de texto do botão de pelo menos 4,5:1.
- Desenhe primeiro em 375px, depois 768px e 1440px. Alvo de toque de pelo menos 44px no storefront e 48px nas ações da fila. Sem overflow horizontal.
- Botão tem nome visível. Status não depende só de cor. Foco volta para quem abriu o sheet. `cypress-axe` no WCAG 2.2 AA.

## Regras que a tela não reinventa

- Preço chega em centavos e só vira `R$` na formatação. A sacola não guarda credencial.
- Sessão é cookie `HttpOnly`. Nada de token em `localStorage`.
- Relógio, "abre às" e "hoje" usam o fuso que a API mandou (`America/Sao_Paulo`, `America/Manaus`). O `is_open` do servidor é a fonte de verdade.
- Copy comercial diz "sem comissão, apenas mensalidade". O extrato não tem coluna de taxa da plataforma.
- Cupom com limite por cliente, no checkout convidado, mostra *"Este cupom tem limite por cliente. Entre na sua conta para usá-lo."* e o botão **Entrar para usar este cupom**. A sacola fica.

A skill `sites-cinematograficos` não vale para estas telas. Ela é de protótipo de estudo.
