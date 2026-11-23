# Spec Geral 01 — Visão de Produto, Domínio e Marcos Sólidos

> **Documento autocontido.** Consolida todas as decisões de negócio derivadas da pesquisa de mercado (`docs/pesquisa/`), perfis de acesso, modelo operacional de onboarding assistido, limites do MVP e os **Marcos Sólidos (M0–M6)** com Definition of Done (DoD).

---

## 1. Problema e Posicionamento

O mercado brasileiro de pedidos de comida divide-se entre:
1. **Marketplaces (iFood, Aiqfome, Delivery Much):** trazem demanda, mas cobram comissões de 12% a 23% (+ taxas de pagamento e mensalidades), aplicam regras rígidas de marca e acumulam queixas de repasse opaco, taxa sobre dinheiro na porta e suporte automatizado sem saída.
2. **Sistemas de canal próprio (Consumer/MenuDino, Goomer, Cardápio Web, Anota AI, Saipos, Cardapex):** cobram mensalidade fixa, mas falham em pontos operacionais críticos identificados em relatos públicos:
   - Jogar o pedido para o WhatsApp sem fila estruturada (Cardapex).
   - Tratar filial como outro login/outra aba (OlaClick) ou engessar preço único na rede (Saipos).
   - Esconder cobranças modulares, dificultar cancelamento e omitir regras de pausa da loja, frete por faixa de CEP e pagamento na entrega.

### Proposta de Valor do Menu Chef
- **Canal próprio white-label por filial:** o cliente pede no navegador do celular sem instalar aplicativo, vendo a identidade do restaurante.
- **Comanda no painel, nunca perdida no WhatsApp:** fila operacional com estados explícitos e sincronizados entre cozinha e cliente.
- **Multi-empresa e multi-filial com 1 único login:** a mesma pessoa pode operar empresas e filiais diferentes com papéis distintos e trocar de contexto sem deslogar.
- **Transparência comercial:** mensalidade clara, zero comissão sobre entrega do motoboy ou pagamento feito diretamente na porta (dinheiro/maquininha/Pix da loja).

---

## 2. Perfis de Usuário e Escopo de Responsabilidade

| Perfil | Definição | O que faz no MVP |
| :--- | :--- | :--- |
| **Visitante (Dono interessado)** | Lojista que acessa a Landing Page | Conhece proposta, fluxo e transparência de planos e envia solicitação de contato (lead) |
| **Administrador da Plataforma** | Operador do SaaS Menu Chef (não é o restaurante) | Recebe leads, cadastra empresa e primeira filial, define plano, envia convite de primeiro acesso, suspende/reativa empresas e acessa fila de loja **somente para suporte com rastro obrigatório de auditoria** |
| **Empresa (Dono / Gestor)** | Grupo contratante do Menu Chef (1 a N filiais) | Gerencia filiais, marca, equipe e papéis por filial, vê dashboard e extrato de pagamentos da unidade |
| **Funcionário (Operador / Cozinha / Despacho)** | Pessoa vinculada a uma ou mais filiais | Entra com 1 único acesso, alterna contexto (empresa/filial), confirma/recusa pedidos, atualiza produção, imprime comanda, marca pronto e despacha/entrega conforme seu papel |
| **Cliente Final** | Consumidor que acessa o link/QR da filial | Navega e busca no cardápio, monta sacola, fecha pedido como **convidado** (nome + telefone) ou com **conta opcional** (histórico e "pedir de novo"), escolhe entrega/retirada e pagamento, acompanha status em tempo real e pode cancelar antes da confirmação da cozinha |

---

## 3. Regra Crítica de Onboarding (Decisão de Negócio)

> **No estágio atual do produto, empresas NÃO iniciam sua operação sozinhas (sem self-signup aberto).**

1. A empresa interessada preenche o formulário de contato na Landing Page (ou aciona o canal comercial).
2. O lead entra na fila comercial do **Administrador da Plataforma**.
3. Após validação comercial, o **Administrador da Plataforma** cria:
   - Registro da **Empresa** (razão/nome fantasia, documento, status `active`, plano contratado e regras transparentes de cobrança);
   - Registro da **Primeira Filial** (slug público único, endereço base);
   - Vínculo inicial do **Responsável (Owner)** + envio de **convite seguro** (token aleatório com expiração) para definir senha e acessar o Painel PWA.
4. Ao acessar pela primeira vez, a empresa passa por um tour guiado (**React Joyride v3**) no painel e configura cardápio, horários e frete antes de publicar o link.

---

## 4. Hierarquia Multi-Empresa, Filiais e Troca de Contexto

Regra de domínio central: **contrato, marca-mãe e identidade das pessoas ficam na Empresa; tudo que muda na operação de rua fica na Filial.**

| Pertence à Empresa | Pertence à Filial |
| :--- | :--- |
| Contrato, plano, status comercial e fatura | Endereço, telefone, slug público (`pedir.menuchef.com.br/{slug}`) e QR Code |
| Marca-mãe (nome/logo padrão herdável) | Personalização opcional de logo e cor primária do cardápio público |
| Lista de filiais e permissão de criação | Cardápio ativo, preços, variações, adicionais e disponibilidade (esgotado) |
| Quadro de usuários e vínculos (`membership`) com papel específico por filial | Horário de funcionamento e **pausa imediata da loja** (chuva/forno/rush) |
| — | Frete fixo ou por **faixa de CEP** (com bloqueio pré-pagamento fora de área) |
| — | Cupons e promoções da unidade |
| — | Fila de pedidos, numeração curta (`#001`), impressão de comanda, retirada e entrega |
| — | Dashboard operacional e extrato de pagamentos online da unidade |

### Regras da Troca de Contexto
- Um usuário autenticado possui 1 identidade global (`users.id`).
- Cada vínculo (`memberships`) conecta `(user_id, company_id, branch_id, role)`.
- A filial ativa fica refletida na URL do Painel PWA (`/c/{companySlug}/b/{branchSlug}/...`) para resistir a reload e botão voltar.
- Se o usuário só pertence a 1 empresa e 1 filial, o seletor não exibe controles redundantes.
- Ao trocar de filial, a fila SSE, permissões, cardápio, cupons e métricas mudam imediatamente, com **rotação do Session ID** no servidor.

---

## 5. Máquina de Estados do Pedido (Linguagem Única Cozinha ↔ Cliente)

O cliente vê exatamente o mesmo nome de etapa que a cozinha opera:

```text
[awaiting_payment] (se online mocado)
        │ (pagamento aprovado)
        ▼
   [received]  ──(cliente cancela sem conversa)──► [canceled_by_customer]
        │      ──(loja recusa com motivo)────────► [rejected_by_store]
        ▼ (funcionário confirma)
   [confirmed]
        │ (funcionário inicia produção)
        ▼
 [in_production]
        │ (funcionário marca pronto)
        ▼
     [ready]
       ├── (modo retirada) ─► [awaiting_pickup] ──► [completed]
       └── (modo entrega)  ─► [out_for_delivery] ─► [completed]
```

**Invariantes de domínio:**
1. O cliente só pode cancelar autonomamente enquanto o status for `awaiting_payment` ou `received` (antes de `confirmed`). Após `confirmed`, cancelamento exige ação autorizada da loja com motivo registrado.
2. Pedido de **retirada** tem frete obrigatoriamente `0` e não exige endereço de entrega.
3. Pedido de **entrega** exige CEP dentro de uma faixa ativa cadastrada pela filial (ou frete fixo habilitado). CEP fora de área impede o fechamento **antes** de qualquer tela de pagamento.
4. Pagamento **na entrega ou retirada** (dinheiro com troco opcional, Pix na porta ou maquininha da casa) **nunca** incide taxa de plataforma e entra direto em `received`.
5. No MVP, pagamentos online (**Pix** e **Cartão**) passam pelo `MockPaymentAdapter` com transições determinísticas (aprovar, recusar, expirar). Gateways reais serão tratados em marco pós-MVP.
6. **Estorno automático de cupom:** se um pedido com cupom for cancelado pelo cliente antes de `confirmed`, recusado pela loja (`rejected_by_store`) ou tiver o pagamento Pix mocado expirado, o uso do cupom é **estornado atomicamente** (decrementa `used_count` com piso `0` e remove o vínculo em `coupon_redemptions`), liberando o cupom para novo uso.
7. **Governança LGPD (B2B2C):** o Menu Chef atua como **Controlador** para dados de conta da Plataforma/Empresas/Leads e como **Operador (Processor)** para dados de pedidos processados em nome de cada Empresa/Filial (**Controladora**). A exclusão de conta do cliente final **anonimiza** seu PII (nome, telefone, e-mail, endereços) preservando os totais financeiros/operacionais das filiais (base legal: cumprimento de obrigação legal/fiscal e execução contratual).

---

## 6. Escopo do MVP vs Pós-MVP

### Dentro do MVP (Marcos M0–M6)
- Infra completa em Docker Compose + Traefik v3.7 + PostgreSQL 19.0 + Redis isolado (`redis-state` + `redis-cache`) + Garage S3.
- Contratos OpenAPI 3.1 gerados do código Go (`Huma v2` sobre `chi`) com SDK TypeScript gerado automaticamente (`openapi-typescript` / `openapi-fetch`).
- Landing Page SSG com SEO/AEO/GEO, `llms.txt` v2, `sitemap.xml`, `robots.txt`, WebP, páginas legais SSG versionadas (`/termos-de-uso`, `/politica-de-privacidade`) e captação de leads.
- Onboarding assistido pelo Administrador da Plataforma (criar empresa, 1ª filial, convite, aceite legal versionado em `legal_acceptances`, suspender/reativar, acesso de suporte auditado).
- Multi-empresa, multi-filial, RBAC por vínculo e troca de contexto sem novo login.
- Cardápio completo da filial: categorias, produtos com foto WebP, variações/adicionais (obrigatórios/opcionais), observação livre, item indisponível/esgotado, promoções, cupons (com controle atômico de concorrência e estorno automático), horários e pausa da loja.
- Busca de produtos via PostgreSQL Full-Text Search (`pt_unaccent` + `pg_trgm`).
- Frete fixo ou por faixa de CEP + bloqueio pré-pagamento fora de área.
- Sacola persistida, checkout idempotente (`Idempotency-Key` IETF) como convidado (nome + telefone) ou com conta do cliente, entrega ou retirada.
- Pagamento mocado (Pix e Cartão via Adapter) + pagamento na entrega/retirada (dinheiro com troco, maquininha, Pix na hora).
- Fila da cozinha em tempo real (SSE + Redis Pub/Sub), alerta sonoro híbrido + Web Push + Wake Lock, modo escuro, impressão 1-clique (`@media print` 58mm/80mm) e tours com React Joyride v3.
- Histórico de pedidos do cliente ("pedir de novo"), autoatendimento LGPD do cliente (exportação JSON + exclusão/anonimização de conta) e dashboard operacional da filial.
- Testes unitários (Go + Vitest), integração (Testcontainers incluindo concorrência e idempotência) e E2E (Cypress).

### Fora do MVP (Backlog Pós-MVP Estruturado)
- Integração com gateway Pix/Cartão real (Asaas, Efí, Mercado Pago, Pagar.me) — apenas plugar novo Adapter em `PaymentGatewayPort`.
- Impressão automática silenciosa via agente local/WebSerial e impressão por estação (cozinha/bar).
- Bot limitado de WhatsApp (apenas envio de link e status, nunca comanda).
- Catálogo-base compartilhado entre filiais e dashboard consolidado da rede.
- Pizza meio a meio, horário por produto individual, QR Code de mesa, fiado e avaliação pós-entrega.
- Emissão de NFC-e, estoque com ficha técnica, fidelidade/cashback, rastreio GPS de motoboy e integração com marketplaces (iFood).
- Self-signup automatizado de empresas e cobrança recorrente automatizada.

---

## 7. Marcos Sólidos (M0–M6)

Cada marco entrega valor funcional isolável e possui **critérios objetivos de pronto (DoD)**. Nenhum marco seguinte começa com o anterior sem testes verdes.

### M0 — Fundação, Infraestrutura & Contratos
- **Objetivo:** Subir toda a topologia local de desenvolvimento/homologação e estabelecer a base de qualidade e contratos OpenAPI.
- **Entregáveis:**
  - Estrutura de diretórios do monorepo (`apps/api`, `apps/landing`, `apps/storefront`, `apps/admin`, `infra/`).
  - `docker-compose.yml` com Traefik v3.7 + `docker-socket-proxy`, PostgreSQL 19.0 (extensões `unaccent`, `pg_trgm`), `redis-state` + `redis-cache` e Garage v2.5.
  - Setup de migrations (`goose`), `sqlc`, `go-jet`, `Huma v2` (`humachi`) com exportação CLI de `openapi.json` + geração `openapi-typescript`, linter Go (`golangci-lint`) e pipeline de testes (Go unit + Testcontainers, Vitest, Cypress).
  - Endpoints `/api/v1/health/live` e `/api/v1/health/ready` validando Postgres, `redis-state`, `redis-cache` e Garage.
- **DoD:** `docker compose up` sobe sem erros; todos os containers saudáveis; roteamento same-origin via Traefik respondendo nos 3 domínios locais; drift check OpenAPI/TS e smoke tests verdes.

### M1 — Landing Page (SSG, SEO/AEO/GEO), Páginas Legais & Captação de Leads
- **Objetivo:** Publicar a vitrine institucional do Menu Chef otimizada para busca tradicional e motores generativos, com compliance legal LGPD e captação de leads para onboarding assistido.
- **Entregáveis:**
  - `apps/landing` em TanStack Start (SSG/Prerender) com paleta Brasa clara, logo Concha com vapor e imagens 100% WebP.
  - Roteiro de conversão transparente (fluxo da cozinha, multi-filial, sem comissão sobre motoboy/pagamento na porta, tabela clara de planos).
  - Páginas SSG versionadas **`/termos-de-uso`** e **`/politica-de-privacidade`** (separação LGPD Controlador vs Operador B2B2C, canal DPO/encarregado, retenção e zero trackers publicitários de terceiros no MVP).
  - `/llms.txt` (v2), réplicas `.md`, `/sitemap.xml`, `/robots.txt` granular para bots de IA e JSON-LD (`SoftwareApplication`, `FAQPage`, `Organization`).
  - Endpoint `POST /api/v1/public/leads` com validação, sanitização, rate limit e registro da versão aceita da Política de Privacidade.
- **DoD:** `curl` nas páginas da landing (incluindo `/termos-de-uso` e `/politica-de-privacidade`) retorna HTML completo sem precisar de JS; Lighthouse Mobile/Desktop >= 95 (Performance, SEO, Accessibility, Best Practices); teste E2E Cypress enviando formulário de contato e validando persistência.

### M2 — Identidade, Sessão Segura, Multi-tenancy & Onboarding Assistido
- **Objetivo:** Permitir que o Administrador da Plataforma provisione empresas/filiais e que usuários operem múltiplas empresas/filiais com 1 único login.
- **Entregáveis:**
  - Autenticação via sessão opaca no `redis-state` (`alexedwards/scs`) em cookie `HttpOnly; Secure; SameSite=Lax` + `http.CrossOriginProtection` (Go 1.27). Zero uso de `localStorage`.
  - RBAC granular por vínculo `(user_id, company_id, branch_id, role)` com papéis: `platform_admin`, `company_owner`, `branch_manager`, `kitchen_operator`, `dispatcher`.
  - Telas do Administrador da Plataforma no `apps/admin`: lista de leads, criação de empresa + 1ª filial, geração de convite seguro, suspensão/reativação de empresa com motivo.
  - Aceite de convite com registro imutável de versão aceita de Termos e Privacidade (`legal_acceptances`), login unificado e **seletor de Empresa/Filial refletido na URL** com rotação de Session ID.
- **DoD:** Testes unitários e integração cobrindo isolamento multi-tenant (impossível acessar filial sem vínculo); E2E Cypress testando criação de empresa pelo Admin → aceite de convite + aceite legal → troca entre 2 filiais com papéis diferentes sem relogar.

### M3 — Cardápio da Filial, FTS PT-BR, Cupons, CEP & Storefront
- **Objetivo:** Permitir que a filial cadastre sua operação e que o cliente navegue e pesquise produtos pelo celular.
- **Entregáveis:**
  - Gestão no `apps/admin`: categorias, produtos, upload de fotos convertidas/validadas em WebP no Garage S3, grupos de adicionais/variações (min/max/obrigatório), toggle rápido de item indisponível, promoções, cupons, horários de funcionamento, botão de pausa da loja e tabela de frete (fixo ou faixas de CEP).
  - Cache de leitura no `redis-cache` (cardápio público e horários da filial) com `singleflight`, TTL com jitter (±15%) e invalidação explícita pós-commit (`UNLINK`).
  - Busca de produtos no PostgreSQL 19.0 via `tsvector` (`pt_unaccent`) + `pg_trgm` (`word_similarity`).
  - Vitrine `apps/storefront` mobile-first: identidade da loja, barra sticky de categorias com scroll spy, busca FTS com debounce, modal/sheet de adicionais obrigatórios e aviso claro quando loja fechada/pausada.
- **DoD:** Busca encontra termos sem acento e com erros de digitação ("x-salda" → "X-Salada", "acai" → "Açaí"); testes de integração confirmam uso de índice GIN e invalidação de cache; E2E Cypress cobre cadastro no admin + exibição e busca no storefront em viewport mobile (375px) e desktop.

### M4 — Sacola Persistida, Checkout Idempotente, Cupons Atômicos & Pagamento Mocado
- **Objetivo:** Fechar o fluxo completo de compra do cliente com validação de área de entrega, proteção contra cliques duplos/race conditions e pagamento mocado determinístico.
- **Entregáveis:**
  - Sacola persistida localmente (apenas itens do carrinho — nunca credenciais), bottom sheet Base UI, microinteração de adicionar respeitando `prefers-reduced-motion`.
  - Checkout em 3 passos claros: (1) Entrega ou Retirada + validação prévia de CEP, (2) Dados do cliente (convidado com nome/telefone OU conta opcional do cliente com aceite legal versionado), (3) Pagamento (Pix mocado, Cartão mocado via `MockPaymentAdapter`, ou Pagar na entrega/retirada com troco) + aplicação de cupom + resumo final.
  - **Idempotência IETF (`Idempotency-Key`):** 3 estados (`absent` / `in_progress` → `409` / `completed` → replay `201` ou `422` em divergência de payload) na mesma transação PostgreSQL do pedido.
  - **Concorrência atômica de cupons:** `UPDATE` condicional atômico em `coupons` + tabela `coupon_redemptions` com constraint `UNIQUE`, mais estorno automático em cancelamento pré-confirmação, recusa da loja ou expiração de Pix mocado.
  - Geração de pedido com `UUIDv7` (PK), `display_number` sequencial da filial (`#001`) e token público aleatório (`crypto/rand`) para acompanhamento sem login.
- **DoD:** CEP fora de área bloqueia antes do pagamento; teste de integração concorrente (50 goroutines) confirma zero *over-redemption* de cupom e estorno automático correto; retry com mesma `Idempotency-Key` não duplica pedido; pagamento Pix/Cartão mocado aprova, recusa e expira de forma determinística; E2E Cypress valida checkout completo guest e autenticado.

### M5 — Operação da Cozinha em Tempo Real (PWA) & Tours Guiados
- **Objetivo:** Entregar a experiência operacional de pico no sábado à noite, com atualização instantânea, alertas confiáveis e impressão de comanda.
- **Entregáveis:**
  - Stream **SSE + `redis-state` Pub/Sub** para a fila da filial no `apps/admin` e para o rastreamento do pedido no `apps/storefront`, com snapshot inicial na reconexão (`Last-Event-ID`).
  - Kanban responsivo (colunas por status com botões de ação acessíveis por teclado — nunca depender só de drag-and-drop).
  - Cancelamento autônomo pelo cliente antes de `confirmed` (com estorno automático de cupom quando aplicável); recusa com motivo pela loja (idem).
  - PWA completo (`vite-plugin-pwa` v2 `injectManifest` + `registerType: 'prompt'`): modo escuro operacional, `Screen Wake Lock API`, alerta sonoro híbrido (foreground custom sound + Web Push background).
  - Impressão 1-clique formatada em `@media print` para bobinas térmicas 58mm e 80mm destacando adicionais, observações e status de pagamento (pago vs a receber/troco).
  - Tours guiados com **React Joyride v3.2.0** (primeiro acesso da filial, operação da fila e troca de contexto).
- **DoD:** Pedido criado no storefront aparece no Kanban em < 1s; mudança de status reflete no storefront em < 1s; cliente consegue cancelar antes de confirmar (estornando cupom) e é bloqueado após confirmação; impressão gera layout limpo 80mm; E2E Cypress valida o ciclo completo `received → completed`.

### M6 — Histórico do Cliente, LGPD Autoatendimento, Dashboard, Suporte Auditado & Hardening
- **Objetivo:** Consolidar retenção do cliente, direitos do titular LGPD, visão gerencial da filial, governança de suporte da plataforma e auditoria final de segurança/performance.
- **Entregáveis:**
  - Área "Meus Pedidos" para clientes autenticados com atalho **"Pedir de novo"** (validando disponibilidade e preços atuais dos itens).
  - **Autoatendimento LGPD do Cliente:** `GET /api/v1/customer/privacy/export` (download JSON dos próprios dados e pedidos) e `POST /api/v1/customer/privacy/delete-account` (revoga sessões e anonimiza PII preservando totais financeiros das filiais).
  - Dashboard operacional da filial (pedidos do dia/período, faturamento, ticket médio, entrega vs retirada, top produtos) e extrato transparente de pagamentos online mocados.
  - Modo de suporte auditado para o Administrador da Plataforma: visualiza fila de uma filial em modo leitura mediante justificativa obrigatória registrada em tabela imutável `support_access_audit_logs`.
  - Jobs agendados de retenção LGPD (`idempotency_keys` 24h, `leads` inativos 24 meses, convites expirados) + auditoria de segurança (OWASP Top 10, SQLi, CSRF, rate limiting, headers Traefik) e testes de carga básicos na busca FTS e SSE.
- **DoD:** Zero impersonação silenciosa; todo acesso de suporte gera log imutável; exclusão de conta anonimiza 100% do PII sem alterar faturamento da filial; cobertura mínima de testes atingida; suite E2E Cypress rodando em CI contra Docker Compose completo.
