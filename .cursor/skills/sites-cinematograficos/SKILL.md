---
name: "sites-cinematograficos"
description: "Monta sites cinematográficos (scroll narrativo, cenas pinadas, texto revelado, 3D e shaders) com GSAP, Lenis, R3F, Paper Shaders e React Bits/Magic UI/Aceternity."
---

# Sites cinematográficos

Use esta skill quando o pedido for um site, landing page ou hero "cinematográfico", "de premiação", "estilo Awwwards", com scroll narrativo, parallax, cenas pinadas, texto animado, 3D ou fundos animados.

O objetivo é uma página que se comporta como um filme: cenas com começo, meio e fim, guiadas pelo scroll, com movimento intencional. Nada de efeito espalhado sem motivo.

## 1. Antes de codar: roteiro

Escreva (na resposta, curto) o roteiro da página antes de qualquer código:

1. **Logline**: uma frase com o que a página precisa fazer o visitante sentir/fazer.
2. **Lista de cenas** (4 a 7), cada uma com:
   - plano (aberto, médio, close),
   - o que entra, o que sai, o que fica pinado,
   - duração em scroll (ex.: `+=150%`),
   - transição para a próxima (corte seco, fusão, wipe, zoom).
3. **Paleta** (1 dominante, 1 apoio, 1 acento) e **tipografia** (1 display com personalidade + 1 texto legível).
4. **Uma peça "herói"** por página: OU 3D (R3F), OU shader, OU vídeo/imagem forte. Nunca as três brigando.

Se o usuário não deu marca/conteúdo, invente um conteúdo plausível e diga isso numa linha.

## 2. Stack padrão

Padrão: **Next.js (App Router) ou Vite + React + TypeScript + Tailwind**.

| Camada | Biblioteca | Quando |
|---|---|---|
| Motor de animação | `gsap` + `@gsap/react` (ScrollTrigger, SplitText, Flip) | Sempre |
| Scroll suave | `lenis` | Sempre |
| 3D | `three` + `@react-three/fiber` + `@react-three/drei` | Só se um objeto 3D conta a história |
| Fundos GPU | `@paper-design/shaders-react` | Heros e transições no lugar de vídeo pesado |
| Componentes animados | React Bits, Magic UI, Aceternity UI (via shadcn CLI) | Detalhes pontuais |

Regras de escolha:
- Use **no máximo uma** biblioteca de componentes por projeto (React Bits OU Magic UI OU Aceternity). Magic UI se o projeto já usa shadcn/ui.
- Prefira GSAP para timelines e scroll. Use Motion (framer) só se o projeto já usa.
- Sem jQuery, sem AOS, sem locomotive-scroll.

Instalação base:
```bash
npm i gsap @gsap/react lenis
# opcionais
npm i three @react-three/fiber @react-three/drei
npm i @paper-design/shaders-react@<versão exata>   # fixe a versão: 0.0.x tem breaking changes
```
GSAP é 100% gratuito desde 2025, inclusive SplitText, MorphSVG e ScrollSmoother, no pacote público `gsap`. Nunca use `gsap-trial` nem registry privado.

Para componentes de React Bits, Magic UI e Aceternity, use o comando shadcn/jsrepo mostrado na página de cada componente (reactbits.dev, magicui.design, ui.aceternity.com). Não invente a URL do registry; se não souber, abra a página do componente ou peça ao usuário.

Skills complementares recomendadas (sugira instalar se não estiverem presentes):
- `/plugin marketplace add greensock/gsap-skills` (oficial GSAP)
- `frontend-design` (Anthropic) para direção de arte

## 3. Padrões de código

### 3.1 Lenis + GSAP (uma vez, no layout raiz)

```tsx
'use client'
import { ReactLenis, type LenisRef } from 'lenis/react'
import 'lenis/dist/lenis.css'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { useEffect, useRef } from 'react'

gsap.registerPlugin(ScrollTrigger)

export function SmoothScroll({ children }: { children: React.ReactNode }) {
  const lenisRef = useRef<LenisRef>(null)
  useEffect(() => {
    const update = (time: number) => lenisRef.current?.lenis?.raf(time * 1000)
    gsap.ticker.add(update)
    gsap.ticker.lagSmoothing(0)
    const lenis = lenisRef.current?.lenis
    lenis?.on('scroll', ScrollTrigger.update)
    return () => gsap.ticker.remove(update)
  }, [])
  return <ReactLenis root options={{ autoRaf: false }} ref={lenisRef}>{children}</ReactLenis>
}
```
Modais e áreas com scroll próprio recebem `data-lenis-prevent`.

### 3.2 Cena pinada com scrub (a base de tudo)

```tsx
'use client'
import { useRef } from 'react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { useGSAP } from '@gsap/react'
gsap.registerPlugin(ScrollTrigger, useGSAP)

export function CenaParallax() {
  const root = useRef<HTMLDivElement>(null)
  useGSAP(() => {
    const mm = gsap.matchMedia()
    mm.add('(prefers-reduced-motion: no-preference)', () => {
      const tl = gsap.timeline({
        scrollTrigger: { trigger: root.current, start: 'top top', end: '+=200%', pin: true, scrub: 1 },
      })
      tl.to('.camada-fundo', { yPercent: -10, ease: 'none' }, 0)
        .to('.camada-meio', { yPercent: -25, ease: 'none' }, 0)
        .to('.camada-frente', { yPercent: -45, ease: 'none' }, 0)
        .fromTo('.titulo-cena', { scale: 3, autoAlpha: 0 }, { scale: 1, autoAlpha: 1 }, 0.2)
    })
  }, { scope: root })
  return <section ref={root} className="relative h-screen overflow-hidden">{/* camadas */}</section>
}
```
Regras: sempre `useGSAP` com `scope` (limpa sozinho no unmount); `ease: 'none'` em parallax com scrub; `autoAlpha` em vez de `opacity`; anime só `transform`/`opacity`/`filter`, nunca `top/left/width/height`.

### 3.3 Texto revelado (SplitText)

```tsx
useGSAP(() => {
  SplitText.create('.titulo', {
    type: 'lines,words',
    mask: 'lines',
    autoSplit: true,               // refaz ao redimensionar / trocar fonte
    onSplit: (self) => gsap.from(self.words, {
      yPercent: 110, autoAlpha: 0, stagger: 0.05, ease: 'power3.out',
      scrollTrigger: { trigger: '.titulo', start: 'top 85%', end: 'top 40%', scrub: 1 },
    }),
  })
}, { scope: root })
```
Rode depois de `document.fonts.ready` se a fonte for web font sem `autoSplit`. Mantenha o texto original acessível (SplitText já preserva `aria-label`).

### 3.4 3D guiado por scroll (R3F + drei)

```tsx
import { Canvas, useFrame } from '@react-three/fiber'
import { ScrollControls, Scroll, useScroll, Environment } from '@react-three/drei'

function Objeto() {
  const scroll = useScroll()
  useFrame((state) => {
    const t = scroll.offset            // 0..1
    state.camera.position.z = 8 - t * 4
    state.camera.lookAt(0, 0, 0)
  })
  return <mesh><torusKnotGeometry args={[1.2, 0.4, 200, 32]} /><meshStandardMaterial metalness={0.6} roughness={0.25} /></mesh>
}

export function Hero3D() {
  return (
    <Canvas dpr={[1, 1.5]} camera={{ fov: 40 }}>
      <ScrollControls pages={3} damping={0.2}>
        <Objeto />
        <Environment preset="city" />
        <Scroll html>{/* textos sobre a cena */}</Scroll>
      </ScrollControls>
    </Canvas>
  )
}
```
Regras: `dpr` limitado a 1.5; modelos em GLB comprimido (Draco/Meshopt); carregar com `dynamic(() => import(...), { ssr: false })` no Next; fallback estático (imagem) em celular fraco ou `prefers-reduced-motion`.

### 3.5 Fundo shader (Paper Shaders)

```tsx
import { MeshGradient } from '@paper-design/shaders-react'

<MeshGradient
  className="absolute inset-0 -z-10"
  colors={['#2b1d2a', '#ff6a3d', '#f2c6a0', '#2a6f73']}
  distortion={0.8}
  swirl={0.6}
  speed={0.2}
/>
```
Cores vêm da paleta do roteiro. `speed` baixo (0.1–0.3) parece cinema; alto parece screensaver. Com `prefers-reduced-motion`, `speed={0}`.

### 3.6 Componentes de biblioteca

Bons usos por cena:
- Abertura: texto (React Bits BlurText/SplitText, ou o SplitText do GSAP).
- Prova social: Marquee (Magic UI).
- Oferta/preço: BorderBeam, ShimmerButton (Magic UI) ou Spotlight (Aceternity).
- Números: NumberTicker (Magic UI).
- Galeria: 3D Card / parallax (Aceternity).

Depois de instalar um componente, **leia o código gerado** e ajuste cores/tipografia ao roteiro. Não deixe o visual padrão da biblioteca.

## 4. Linguagem cinematográfica → técnica

| No roteiro | Na página |
|---|---|
| Plano aberto → close | `scale` da imagem 1 → 1.6 com scrub, pinado |
| Travelling lateral | seção horizontal: pin + `x: -(largura - viewport)` |
| Fusão | duas camadas sobrepostas, `autoAlpha` cruzado |
| Corte seco | troca de cena sem easing, no fim do pin |
| Profundidade | 3 a 5 camadas com `yPercent` diferentes |
| Créditos de abertura | SplitText em linhas com máscara |
| Luz/atmosfera | shader de fundo + grão (noise) sutil |
| Ritmo | alternar cenas pinadas longas com seções curtas que só rolam |

## 5. Regras não negociáveis

- **Acessibilidade**: todo movimento dentro de `gsap.matchMedia()` com `(prefers-reduced-motion: no-preference)`; versão reduzida mostra o conteúdo final, parado. Contraste AA. Conteúdo legível sem JS.
- **Performance**: 60fps no notebook médio; LCP < 2.5s. Nada de vídeo autoplay acima de ~3 MB no hero. `will-change` só durante a animação. Pause WebGL fora da viewport.
- **Mobile**: testar em 375px. Reduzir pins longos (`end` menor), desligar 3D pesado, manter Lenis (ele não interfere no toque por padrão). Regras detalhadas na seção 5.1.

### 5.1 Mobile (obrigatório, desde a primeira cena)

Projete a versão de 375px junto com a de desktop, não depois. Regras aprendidas na prática:

**Layout e tamanho**
- Cada card/cena pinada deve **caber numa tela** do celular (altura ≤ `100svh`). Se não couber, enxugue no celular: esconda metadados, mostre só o bloco mais importante do texto (com `line-clamp`), reduza figuras (`zoom: .6` ou versão simplificada) e tire legendas. Card "espichado" no celular é defeito.
- Dados que precisam ficar numa linha (e-mail, domínio, número): `white-space: nowrap` + fonte proporcional à largura, ex.: `font-size: min(clamp(...), calc((100vw - 2 * var(--gutter)) / N))`, onde N ≈ número de caracteres × 0,6.
- Mocks e tabelas: nunca com barra de rolagem própria. No celular, esconda colunas secundárias ou troque por versão compacta.
- Grades de cards de hover (vitrines, galerias) viram grade de 2 colunas ou lista no celular; a informação que no desktop só aparece no hover precisa estar visível ou não ser essencial.
- Use `container-type: inline-size` + `@container` quando um card muda de largura (ex.: acordeão), para esconder etiquetas e reduzir o título em vez de cortar texto.

**Scroll horizontal**
- Sempre `body, main { overflow-x: clip; }` (não `hidden`: `clip` não cria contêiner de rolagem e não quebra `position: sticky`).
- Rotações 3D (`rotationX` com perspectiva), `scale` > 1 e magnetismo podem passar da largura da tela por um instante e gerar barra lateral que "aparece e some". Teste rolando a página inteira e medindo `document.documentElement.scrollWidth - innerWidth` em vários pontos: tem que ser 0.

**Pins e emendas**
- No celular, pins mais curtos (ex.: `+=150%` a `+=170%` no lugar de `+=240%`) e nada de pin na abertura: o visitante precisa conseguir ver e tocar o que está na primeira tela (ex.: vitrine de projetos) antes de a próxima cena cobrir.
- Emendas sem tela em branco, mas sem prender o scroll: prefira a próxima cena encostar (borda arredondada) em vez de cortina sobre conteúdo interativo.
- Use `gsap.matchMedia()` com condições (`desk: "(min-width: 720px)"`) para as cenas se reconstruírem ao cruzar o breakpoint, em vez de decidir celular/desktop uma vez só no carregamento.
- Cenas com lista de passos + palco: no celular, palco em cima, e só o passo atual visível (todos no mesmo `grid-area`, alternando opacidade), com barra de progresso horizontal.

**Sticky e pilhas de cards**
- Nenhuma classe utilitária pode sobrescrever `position: sticky` dos cards (ex.: um `position: relative` do BorderBeam). Garanta com `.card.beam-host { position: sticky; }` e confira o `position` computado.
- Card mais alto que a tela: `top = min(base, innerHeight - altura - 16)`, e meça a posição natural com `position: static` (não `relative`, que aplica o `top`).
- Gatilhos de ScrollTrigger em elementos sticky: use posições numéricas calculadas da posição natural, não `trigger: card`.

**Interação**
- Hover não existe no toque: efeitos de hover (cor da marca entrando, magnetismo) só com `(hover: hover)`; no celular o estado padrão já deve ser bonito (ex.: logo na cor da marca).
- Modais: uma coluna no celular, `max-height: calc(100svh - 24px)` com rolagem interna e `data-lenis-prevent`.
- Elementos flutuantes (avatar, botão de WhatsApp) não podem cobrir títulos: no texto, use `float` + `shape-outside` para o título contornar; nas laterais, deixe meio escondido e aparecendo só quando falar.
- **Limpeza**: nunca `gsap.to` solto em `useEffect` sem cleanup; usar `useGSAP`. Chamar `ScrollTrigger.refresh()` depois de carregar imagens/fontes que mudam altura.
- **SEO**: textos reais no HTML (não dentro de canvas), metadados, `alt` nas imagens.
- **Licenças**: React Bits é MIT + Commons Clause (pode usar em projetos, não revender a biblioteca); GSAP tem licença própria gratuita, inclusive comercial; demais MIT. Avise se o usuário for vender um template.

## 6. Fluxo de trabalho

1. Roteiro (seção 1) → mostrar ao usuário em poucas linhas e seguir, salvo se ele pediu para aprovar antes.
2. Scaffold do projeto + `SmoothScroll` no layout.
3. Construir **uma cena por vez**, rodando o dev server e conferindo no navegador (ou com screenshot, se houver ferramenta) **em desktop e em 375px** antes da próxima.
4. Passar pela checklist abaixo.
5. Entregar com: como rodar, lista de cenas, onde trocar cores/textos/assets.

## 7. Checklist final

- [ ] Cada cena tem começo, meio e fim claros no scroll
- [ ] Só uma peça "herói" pesada (3D ou shader ou vídeo)
- [ ] `prefers-reduced-motion` testado (emular no DevTools)
- [ ] 375px sem scroll horizontal e sem pins quebrados (medir `scrollWidth - innerWidth` = 0 rolando a página toda)
- [ ] No celular, cada card/cena cabe numa tela, e-mail/domínio em uma linha, nada essencial só no hover
- [ ] Cards sticky com `position: sticky` computado (nenhuma classe sobrescrevendo)
- [ ] Primeira tela utilizável sem ser coberta pela cena seguinte
- [ ] Sem erro no console; sem ScrollTrigger duplicado após navegação
- [ ] Lighthouse: Performance ≥ 85, Acessibilidade ≥ 95
- [ ] Cores e fontes da biblioteca de componentes ajustadas ao roteiro
- [ ] Textos indexáveis no HTML

## Referências

- GSAP: https://gsap.com/docs/v3/ · skills oficiais: https://github.com/greensock/gsap-skills
- Lenis: https://github.com/darkroomengineering/lenis
- R3F/drei: https://github.com/pmndrs/drei
- Paper Shaders: https://shaders.paper.design/
- React Bits: https://reactbits.dev · Magic UI: https://magicui.design · Aceternity: https://ui.aceternity.com
- 21st MCP (opcional, requer API key): `npx @21st-dev/cli@latest init --client claude`