# Protótipo do cardápio

HTML estático. Não é tela de produção.

Na pasta `docs/pesquisa/design`:

```bash
python3 -m http.server 8765
```

Abra `http://127.0.0.1:8765/prototipos/cardapio/`.

O que dá para ver: card de produto, barra de categorias grudada que marca a seção visível, botão que troca a lista por skeleton. Com `prefers-reduced-motion: reduce`, o skeleton não corre.
