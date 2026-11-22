# Cardápio digital do cliente

Ideias para o cardápio no celular, sem aplicativo instalado. É a tela que o cliente abre pelo link.

## O que já se sabe do mercado

Consumer, Goomer, Anota AI, Cardápio Web e Cardapex entregam o cardápio no navegador. A Goomer refez o visual em 2022 para dar relevância ao produto, e em 2023 passou a guardar o carrinho de quem sai e volta ([novidades](https://ajuda.goomer.com.br/novidades-do-produto)). Busca no cardápio também aparece nessa história da Goomer.

O cliente come com o polegar. A Web Interface Guidelines pede alvo de pelo menos 44 px no celular e campo com fonte de 16 px para o iOS não ampliar sozinho ([guidelines](https://github.com/vercel-labs/web-interface-guidelines)).

Pins, Mobbin e a gravação do Deliveroo estão em [referencias-visuais.md](referencias-visuais.md).

## Referências

| Exemplo | Padrão | Fonte |
| --- | --- | --- |
| iFood e Chipotle, no Mobbin | Lista de produto, detalhe, adicionar à sacola, resumo. O iFood aparece no fluxo “adding to cart” | [Mobbin, adding to cart](https://mobbin.com/explore/mobile/flows/adding-to-cart-bag) |
| Goomer | Pagamento e cardápio no mesmo ambiente, sem mandar o cliente para outro site. Pix, cartão, NuPay e Google Pay no delivery | [pagamento](https://goomer.com.br/blog/funcionalidades-pagamento-goomer) |
| Cardapex | O cardápio existe, mas o pedido segue para o WhatsApp. O Menu Chef pode manter o cliente na página | [cardapex.com.br](https://www.cardapex.com.br/) |
| 60fps.design | Ícone de sacola flutuante e folha que sobe. O exemplo é de loja, não de restaurante, e o gesto é o mesmo | [Shop Cart Sheet](https://60fps.design/shots/shop-cart-sheet) |

Categoria em carrossel (Embla, [repositório](https://github.com/davidjerleke/embla-carousel)) esconde o que não está na tela. Para um cardápio, a barra de chips que gruda no topo e acompanha a rolagem mostra todas as categorias e ainda diz onde a pessoa está. É o padrão testado no protótipo.

## Ideias de layout

- Uma coluna. Foto, nome, descrição curta, preço, botão “adicionar” com área grande.
- Adicional obrigatório abre antes de confirmar, na própria página ou numa folha. Sem isso o pedido chega incompleto, que é a queixa de item errado nos marketplaces.
- Barra de categorias grudada abaixo do topo. O chip da seção visível fica marcado. Clicar no chip rola até a seção (`scroll-margin` para a barra não cobrir o título).
- Loja fechada ou pausada explica o horário e não deixa adicionar.
- Skeleton no lugar do card enquanto a lista não chegou: retângulo da foto, duas linhas, um bloco de preço.
- Busca simples por nome, como a Goomer acrescentou. Não é o primeiro gesto, mas o cardápio longo pede.

O protótipo [prototipos/cardapio/](prototipos/cardapio/) testa card, barra grudada com scroll spy e skeleton. A paleta é a Brasa clara. Não usa biblioteca.
