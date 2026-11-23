# Dois sites que o Vinicius apontou

Abertos em **23 de novembro de 2026**. A leitura é de estrutura e de técnica. O protótipo da landing não repete layout, texto, ilustração nem cor desses sites. A paleta continua a Brasa clara e a marca continua a Concha com vapor.

## mateusemth.dev

Página de portfólio de quem faz ferramenta a partir de uma dor. O HTML público carrega GSAP 3.13 (núcleo, ScrollTrigger e SplitText) pelo cdnjs, Lenis 1.1.13 pelo jsDelivr, e os arquivos próprios `cinema.js`, `peek.js`, `styles.css` e `cinema.css`. As fontes pedidas ao Google são Geist, Geist Mono, Instrument Serif e Chakra Petch.

O ritmo é de filme curto: uma frase de abertura, uma escada de quatro estados (papel, planilha, planilha automática, sistema), casos empilhados com a indicação “role para empilhar”, um roadmap e um contato. `cinema.js` usa pin, scrub, SplitText, Lenis e `matchMedia` com `prefers-reduced-motion`.

O que vale adaptar, sem copiar a página: uma cena por ideia, a fila como objeto que muda enquanto se rola, e o movimento desligado quando a pessoa pede menos animação. O que não entra no Menu Chef: a tipografia dessa página, o tom de portfólio pessoal, os estudos de planilha e os casos de IA. A landing do protótipo conta o caminho do pedido no restaurante, com Fraunces e Source Sans 3, que não são as fontes daquele site.

Fonte: [mateusemth.dev](https://mateusemth.dev). Scripts vistos no HTML: `https://cdnjs.cloudflare.com/ajax/libs/gsap/3.13.0/gsap.min.js`, ScrollTrigger, SplitText e `https://cdn.jsdelivr.net/npm/lenis@1.1.13/dist/lenis.min.js`.

## teamshow.com.br

Landing de produto, feita em Astro (os arquivos públicos estão em `/_astro/` e o runtime é o do Vite). Não há GSAP nem Lenis nos scripts que abri. O movimento é local: `prefers-reduced-motion` é lido no JavaScript, o que entra na tela usa `IntersectionObserver` com opacidade e um deslocamento curto, números sobem com `requestAnimationFrame`, e um gráfico de barras (Chart.js, carregado só quando a seção aparece) desliga a animação se o reduced motion estiver ativo. O menu móvel é `<details>`, fecha com Escape e devolve o foco.

A página é longa e comercial: promessa, grade de recursos, canais, um bloco de voz, um bloco de IA, integrações, clientes, métricas, perguntas e um pedido de demonstração. Funciona porque cada bloco responde uma objeção. Não funciona como modelo de filme: são muitas seções do mesmo peso.

O que vale adaptar: a primeira frase diz o que o produto junta, o botão repete a mesma ação, e o reduced motion está no código e não só no CSS. O que não entra: a grade de dezenas de recursos, os mocks de ligação e de chat, a lista de clientes e a voz visual dessa marca. O Menu Chef, nesta pesquisa, mostra três passos do pedido e uma fila, não um catálogo de canais.

Fonte: [teamshow.com.br](https://teamshow.com.br/). O script `BaseLayout` confirma a leitura de `prefers-reduced-motion` e o observador de entrada. O script do dashboard confirma o Chart.js com `animation` condicionado a essa preferência.

## Como o protótipo fica diferente

Quatro cenas, uma peça (a fila), paleta Brasa, sem pin na abertura e sem pin no celular. GSAP e Lenis entram só nessa landing de estudo, e só quando não há pedido de menos movimento. Cardápio, sacola, checkout e painel continuam sem essas bibliotecas.
