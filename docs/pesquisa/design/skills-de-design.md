# Skills de design

Ideias para um agente que desenhe telas do Menu Chef. Não é adoção, nem padrão do repositório. Estrelas e licenças mudam; os números abaixo foram lidos em **22 de novembro de 2026**.

Uma skill é um texto de instrução para o agente. Ela não desenha sozinha e não substitui contraste, teclado e teste no celular.

## O que foi lido de verdade

| Skill | O que faz | Licença e alcance | Fonte |
| --- | --- | --- | --- |
| `frontend-design` | Direção visual a partir do assunto, do público e do trabalho da tela. Pede tipografia deliberada, movimento só quando responde a uma ação, e evita o visual genérico de página gerada | O repositório [anthropics/skills](https://github.com/anthropics/skills) tinha 198.959 estrelas e o último push em 16/11/2026. A API do GitHub não devolve uma licença única do repositório. O texto da skill está em [SKILL.md](https://github.com/anthropics/skills/blob/HEAD/skills/frontend-design/SKILL.md). Um guia de setembro de 2026 diz que a pasta da skill tem LICENSE Apache-2.0 próprio ([Yi Cheng](https://yicheng-yvonne.com/en/posts/2026/09/ui-ux-agent-skills-ranking-2026/)) | [SKILL.md](https://github.com/anthropics/skills/blob/HEAD/skills/frontend-design/SKILL.md) |
| `shadcn` | Como compor a biblioteca shadcn/ui: procurar no registro antes de inventar componente, usar variante pronta e cor semântica | Repositório [shadcn-ui/ui](https://github.com/shadcn-ui/ui), 128.761 estrelas, licença MIT, push em 21/11/2026. Release `shadcn@4.23.0` em 27/10/2026 | [SKILL.md](https://github.com/shadcn/ui/blob/main/skills/shadcn/SKILL.md) |
| `ui-ux-pro-max` | Base pesquisável de estilos, paletas, pares de fonte, tipos de gráfico e regras de UX, inclusive texto que não pode cortar e alvo de toque | [nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill/), 152.775 estrelas, LICENSE MIT na raiz. A issue [#481](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill/issues/481) (set/2026) apontou que `cli/README.md` dizia CC-BY-NC-4.0. A issue foi fechada com a recomendação de tratar MIT como licença pretendida. Esta pesquisa não reabriu o README do CLI, então a frase antiga pode ainda estar lá | [repositório](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill/), [LICENSE](https://raw.githubusercontent.com/nextlevelbuilder/ui-ux-pro-max-skill/HEAD/LICENSE) |
| Web Interface Guidelines | Lista de decisões de interface: teclado, foco visível, alvo de 44 px no celular, campo com 16 px para o iOS não dar zoom, estado de URL, confirmação de ação destrutiva, semântica antes de ARIA | [vercel-labs/web-interface-guidelines](https://github.com/vercel-labs/web-interface-guidelines), 1.218 estrelas, MIT. O texto longo está no README. A skill de revisão mora na coleção [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) (34.739 estrelas; a API não devolve licença única) | [guidelines](https://github.com/vercel-labs/web-interface-guidelines), [LICENSE](https://github.com/vercel-labs/web-interface-guidelines/blob/main/LICENSE) |
| `sites-cinematograficos` | Roteiro de cenas, uma peça herói, scroll com GSAP e Lenis, e um checklist de reduced motion, 375 px e Lighthouse. Stack que ela propõe: Next ou Vite + React, GSAP, Lenis, e 3D ou shader só se a história pedir | Arquivo do amigo do Vinicius, copiado sem edição para [`.cursor/skills/sites-cinematograficos/SKILL.md`](../../../.cursor/skills/sites-cinematograficos/SKILL.md). Não está num repositório público com estrelas. A própria skill diz que o GSAP público é gratuito desde 2025, com licença da Webflow, e que React Bits é MIT com Commons Clause | O texto anexado nesta pesquisa |

## Prós e contras

**frontend-design.** Serve para a landing e para o cardápio não parecerem um template roxo. O próprio texto manda abrir com o que é característico do assunto, e o Menu Chef já tem paleta Brasa e logo de concha. O risco é a skill empurrar “risco estético” num painel de cozinha, onde a fila de pedidos precisa ser óbvia. Movimento espalhado em todo card é exatamente o que ela chama de visual genérico.

**shadcn.** Útil se o painel for React e a equipe quiser copiar o componente para o repositório, em vez de depender de um pacote fechado. Não escolhe a cara do Menu Chef. Sem o projeto na biblioteca, a skill não tem o que governar.

**ui-ux-pro-max.** Cobre muita combinação de estilo, fonte e regra de UX. Para o Menu Chef, o valor está nas regras de texto longo, toque e formulário, não em escolher um dos 80 estilos prontos por cima da Brasa. A divergência de licença do CLI precisa ser lida de novo antes de copiar o pacote para um produto comercial.

**Web Interface Guidelines.** É a skill mais próxima de um checklist de uso: foco, teclado, zoom, loading que não pisca, URL com o estado da tela. Encaixa no checkout e na troca de filial. Não desenha a página.

**sites-cinematograficos.** Cabe na landing: roteiro, uma peça só (aqui a fila de pedidos, não 3D), cenas com começo e fim, e o checklist da seção 7. No cardápio, na sacola e no checkout, só o que não pesa: reduced motion, 375 px sem barra lateral, alvo de toque e hierarquia. No painel da cozinha, não: pin, scroll suave e shader atrasam quem precisa ver o pedido. A skill pede GSAP em toda página cinematográfica; nos protótipos de operação o movimento continua em CSS.

## O que o guia de setembro cita e esta pesquisa não leu por inteiro

O artigo [Which UI/UX Agent Skills Are Actually Worth Using?](https://yicheng-yvonne.com/en/posts/2026/09/ui-ux-agent-skills-ranking-2026/) (13/09/2026) também aponta `taste-skill`, `DESIGN.md` (`google-labs-code/design.md`), `frontend-design-review` da Microsoft, `motion-design-skill` e `premium-frontend-ui`. As estrelas desse artigo foram conferidas pelo autor em 13/09/2026 e ele avisa que mudam. Não abri esses repositórios nesta rodada. Tratar a ordem do artigo como ranking fechado seria repetir a opinião dele, não uma leitura nossa.

## Ideia de uso, sem virar regra

Uma leitura possível, só como hipótese: `frontend-design` na landing, com a Brasa e a concha já escolhidas como limite; Web Interface Guidelines no cardápio, na sacola e no checkout; `shadcn` só se o painel nascer em React com essa biblioteca. `ui-ux-pro-max` como consulta de UX, não como gerador de paleta, porque a paleta já está em [cores-e-paletas.md](../cores-e-paletas.md).
