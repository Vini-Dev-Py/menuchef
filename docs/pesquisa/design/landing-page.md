# Landing page do SaaS

Ideias para a página em que o restaurante decide conhecer o Menu Chef. Não é o cardápio do cliente. Não é decisão de layout.

## O que a página precisa contar

O visitante é dono de restaurante. Ele já compara comissão de marketplace e mensalidade de cardápio próprio, como está em [concorrentes.md](../concorrentes.md). A página tem de dizer, cedo, que o pedido fica no painel da casa e não só no WhatsApp, e que dá para ter mais de uma filial. Preço e o que está incluso importam mais do que um slogan, porque cobrança surpresa é a reclamação mais repetida na pesquisa.

Mais links de galeria, com o que abriu e o que não abriu, estão em [referencias-visuais.md](referencias-visuais.md).

## Referências

| Exemplo | O que dá para observar | Fonte |
| --- | --- | --- |
| Saipos | Abre com gestão do restaurante e integração, não com foto de hambúrguer. O preço “a partir de” aparece na conversa de planos, não necessariamente no primeiro bloco | [saipos.com/sistema/restaurante](https://saipos.com/sistema/restaurante), [planos](https://saipos.com/planos-e-precos) |
| Consumer | Promete cardápio sem taxa por pedido e um caminho curto: cliente pede no celular, pedido cai no computador | [consumer.com.br/cardapio-digital](https://consumer.com.br/cardapio-digital) |
| Anota AI | O herói é o robô no WhatsApp. É o oposto do que o Menu Chef quer enfatizar no pico, mas mostra como um único benefício ocupa a primeira tela | [anota.ai](https://anota.ai/) |
| Goomer | A página de planos compara o que cada faixa inclui. Isso é mais útil do que três cards iguais de “benefício” | [goomer.com.br/planos](https://goomer.com.br/planos) |
| Cardapex | A home vende o pedido caindo no WhatsApp. Serve de contraste: a landing do Menu Chef pode mostrar o painel, não o aplicativo de conversa | [cardapex.com.br](https://www.cardapex.com.br/) |

Padrão visto nessas homes e na skill [frontend-design](https://github.com/anthropics/skills/blob/HEAD/skills/frontend-design/SKILL.md): a primeira tela mostra a coisa característica do produto. Para o Menu Chef, um candidato é o fluxo com nome (recebido, em produção, pronto), não um número grande de “restaurantes atendidos”. A mesma skill trata fade em toda seção como visual genérico.

A skill [Web Interface Guidelines](https://github.com/vercel-labs/web-interface-guidelines) pede foco visível, zoom liberado e alvo grande. Numa landing, o botão “ver o cardápio de exemplo” precisa ser alcançável por teclado.

## Ideias, sem fechar

- Uma coluna no celular, duas no desktop: texto e um recorte do painel ou do cardápio na paleta Brasa.
- Três passos só se forem de fato uma sequência: cadastra o cardápio, o cliente pede, a cozinha confirma. A skill de frontend-design diz para não numerar o que não é sequência.
- Tabela curta do que está na mensalidade e do que não está. Goomer e Cardápio Web já sofrem quando isso fica escondido.
- Sem carrossel de depoimento automático. Movimento que ninguém pediu atrapalha, e o reduced motion teria de desligar.

O protótipo [prototipos/landing/](prototipos/landing/) agora segue um roteiro de quatro cenas, com GSAP e Lenis só quando o sistema não pede menos movimento. A peça herói é a fila, não um modelo 3D. A leitura dos dois sites que inspiraram o ritmo, sem copiar o visual, está em [referencias-proprias.md](referencias-proprias.md).
