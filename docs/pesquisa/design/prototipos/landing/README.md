# Protótipo da landing

Demo de validação, não página de produção. Sobe o servidor na pasta `docs/pesquisa/design`:

```bash
python3 -m http.server 8765
```

Abra `http://127.0.0.1:8765/prototipos/landing/`.

`?cena=passos` e `?cena=fecho` posicionam o scroll para os prints. GSAP e Lenis vêm de CDN e só rodam quando o sistema não pede menos movimento.

## Roteiro

Logline: o dono sente o pedido sair da conversa e entrar numa fila com nome.

1. Abertura, plano aberto, sem pin. Entra a frase e o botão. A fila já está visível para poder tocar. Corte seco para a cena 2.
2. Close na fila, peça herói. No desktop a seção fica pinada cerca de `+=120%` e os cartões entram em fusão. No celular não há pin: a fila cabe na primeira tela.
3. Três passos, plano médio. No celular só o passo atual aparece, com uma barra. Duração curta, sem pin longo.
4. Fecho. Uma frase sobre filial e um link de volta. Sem shader, sem 3D e sem vídeo.

Paleta Brasa clara: `#C2410C`, `#FFFBF5`, `#1C1917`, apoio `#44403C`, acento `#F59E0B`. Display Fraunces, texto Source Sans 3.

A peça herói é a fila. Não há segunda peça pesada.
