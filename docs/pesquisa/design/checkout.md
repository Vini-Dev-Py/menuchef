# Checkout

Ideias para fechar o pedido: entrega ou retirada, como pagar, Pix, pagar agora ou na hora. Não é integração de pagamento.

## O que o fechamento precisa perguntar

São poucas perguntas, nesta ordem:

1. Entrega ou retirada. Retirada não tem frete.
2. Se for entrega, o CEP já precisa estar numa faixa. Fora da área, a pessoa vê isso antes de pagar. Frete por CEP é o modelo que o dono descreveu; a Goomer publica distância ou bairro, não essa frase.
3. Como pagar: Pix, cartão, ou na entrega e na retirada. A Goomer documenta Pix e cartão dentro do cardápio, sem sair do site ([pagamento](https://goomer.com.br/blog/funcionalidades-pagamento-goomer)).
4. Pagar agora ou na hora. Se o dinheiro não passa pela plataforma, a plataforma não fica com percentual. Isso vem do relato do Aiqfome na pesquisa de concorrentes.
5. Resumo: itens, frete, cupom, total, endereço ou “vou buscar”.

O iFood separa o momento em que a loja ainda não confirmou, e é aí que o cliente cancela com menos conversa. O checkout termina quando o pedido existe com número. A confirmação da cozinha é outra tela, a do painel.

DoorDash, Deliveroo e os arquivos da Figma que citaram pagamento estão em [referencias-visuais.md](referencias-visuais.md).

## Referências

| Exemplo | Padrão | Fonte |
| --- | --- | --- |
| Chipotle, no Mobbin | Checkout de retirada com progresso em passos e controle segmentado (entrega/retirada) | [adding to cart & bag](https://mobbin.com/explore/mobile/flows/adding-to-cart-bag), fluxo “Checking out a product (pickup) on Chipotle” |
| Goomer | Pagamento no próprio cardápio. Carrinho guardado se a pessoa sair | [pagamento](https://goomer.com.br/blog/funcionalidades-pagamento-goomer), [carrinho salvo](https://ajuda.goomer.com.br/novidades-do-produto) |
| Web Interface Guidelines | Botão de enviar mostra espera sem apagar o rótulo. Erro fica no campo. Não bloquear colar. Confirmar ação que não dá para desfazer | [guidelines](https://github.com/vercel-labs/web-interface-guidelines) |

Passo a passo cabe aqui porque o conteúdo é uma sequência. A skill frontend-design alerta para não numerar o que não é sequência ([SKILL.md](https://github.com/anthropics/skills/blob/HEAD/skills/frontend-design/SKILL.md)). Três passos, com o passo atual em texto (“passo 2 de 3”), não só na cor.

Pix precisa de um estado “aguardando pagamento” com o valor e um jeito de copiar o código. Nesta pesquisa não há especificação de provedor. O protótipo só mostra o estado.

## Ideias

- Um passo visível por vez no celular. No desktop, o resumo fica ao lado.
- Entrega e retirada como dois botões grandes, não um select escondido.
- “Pagar na entrega” só aparece se a loja ligou essa opção. O protótipo deixa as duas visíveis para comparar.
- Voltar um passo não apaga o que já foi digitado.
- Reduced motion: a troca de passo é instantânea, sem deslizar a página inteira. Deslizar a tela inteira é o tipo de movimento que a documentação de View Transitions cita como risco ([guia Next](https://nextjs.org/docs/app/guides/view-transitions)).

O protótipo [prototipos/checkout/](prototipos/checkout/) testa o stepper, entrega ou retirada, e pagar agora ou na hora. Não chama banco nem gera Pix de verdade.
