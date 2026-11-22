# Sacola

Ideias para a sacola no celular. O nome usado no produto é sacola, não “cart”.

## O que a sacola precisa deixar claro

Itens, quantidade, adicional, observação e total parcial. Editar antes de pagar. O total ainda não é o total final: frete e cupom entram no checkout. A Goomer guarda o carrinho de quem sai e volta ([08/03/2023](https://ajuda.goomer.com.br/novidades-do-produto)). Isso evita a pessoa perder o pedido porque abriu outra conversa.

A folha do 60fps e os fluxos de sacola do Mobbin estão em [referencias-visuais.md](referencias-visuais.md).

## Referências

| Exemplo | Padrão | Fonte |
| --- | --- | --- |
| 60fps.design | Toque no ícone, folha sobe de baixo, fundo escurece, itens e botão de seguir. A transição que eles descrevem usa mola | [Shop Cart Sheet](https://60fps.design/shots/shop-cart-sheet) |
| Mobbin | Fluxos de iFood, Chipotle e Swiggy: adicionar, ver a sacola, seguir para pagamento. Vários usam bottom sheet | [adding to cart & bag](https://mobbin.com/explore/mobile/flows/adding-to-cart-bag) |
| iFood, no mesmo fluxo | O Mobbin lista “Adding to cart on iFood” com resumo do pedido | [Mobbin](https://mobbin.com/explore/mobile/flows/adding-to-cart-bag) |

Bottom sheet é folha presa na base da tela. Em HTML dá para fazer com `<dialog>` ou um painel com `aria-modal`. A Web Interface Guidelines pede foco preso, foco que volta ao botão que abriu, e alvo de 44 px ([guidelines](https://github.com/vercel-labs/web-interface-guidelines)). O drawer do shadcn, se a tela for React, hoje compõe o popup do Base UI ([doc](https://ui.shadcn.com/docs/components/base/drawer)). Vaul faz o gesto de arrastar, mas o shadcn deixou de usá-lo e o repositório teve o último push em 03/10/2025 ([vaul](https://github.com/emilkowalski/vaul)).

## Microinteração de adicionar

A ideia testada: um pontinho sai do botão e vai até o ícone da sacola, e o número sobe. É confirmação de que o toque valeu, sem abrir a sacola inteira. Com `prefers-reduced-motion: reduce`, o pontinho não viaja; o número sobe na hora. O Motion só faria isso sozinho se `reducedMotion` estiver em `"user"`, e o padrão da biblioteca é `"never"` ([MotionConfig](https://motion.dev/docs/react-motion-config)). O protótipo usa a Web Animations API e a media query, sem pacote.

AutoAnimate não desenha essa trajetória. Ele mexe quando um filho entra ou sai da lista ([doc](https://auto-animate.formkit.com/)). Serviria para a lista dentro da sacola, não para o voo.

## Ideias

- Ícone da sacola fixo, com o número de itens.
- Folha com alça, lista, alteração de quantidade e botão “seguir para pagamento”.
- Fechar pela alça, pelo fundo e pela tecla Escape.
- Sacola vazia explica o que fazer, em vez de uma folha em branco.

O protótipo [prototipos/sacola/](prototipos/sacola/) testa o voo e a folha. Não persiste a sacola entre visitas; a ideia da Goomer fica só citada.
