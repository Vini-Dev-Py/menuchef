# Painel da empresa

Ideias para a tela de quem opera a casa: fila do dia, cardápio, cupom, frete por CEP e troca de empresa ou filial. Continua sendo referência, não desenho fechado.

## O que a operação pede

O fluxo já descrito em [recursos-planejados.md](../recursos-planejados.md) tem nomes estáveis: recebido, confirmado, em produção, pronto, aguardando retirada ou saiu para entrega. A Saipos vende um kanban e uma tela só para vários canais ([saipos.com/sistema/restaurante](https://saipos.com/sistema/restaurante)). A reclamação recorrente é pedido que não imprime e monitor de cozinha parado, não falta de gráfico. O dashboard do dia importa; o gráfico decorativo, não.

Trocar de filial sem outro login é o diferencial anotado em [lacunas-e-oportunidades.md](../lacunas-e-oportunidades.md). A OlaClick abre outra guia e pede outra senha ([ajuda](https://help.olaclick.com/pt-br/help/gerencie-at%C3%A9-4-filiais-com-mais-controle)). O seletor precisa dizer empresa e filial em texto, não só com cor. A paleta Brasa já separa sucesso, alerta, erro e info em [cores-e-paletas.md](../cores-e-paletas.md), sempre com palavra.

Galerias e arquivos da Figma que abriram nesta rodada estão em [referencias-visuais.md](referencias-visuais.md).

## Referências de layout

| Exemplo | Padrão | Fonte |
| --- | --- | --- |
| Saipos | Kanban da operação, KDS, visão de várias lojas no dashboard de franquias | [restaurante](https://saipos.com/sistema/restaurante), [dashboard de franquias](https://saipos.com/sistema/restaurante/dashboard-de-franquias-para-restaurante) |
| Goomer | Gestor de pedidos e, no plano mais alto, tela de cozinha. A comparação de planos mostra o que o plano de baixo não tem | [goomer.com.br/planos](https://goomer.com.br/planos) |
| shadcn/ui | Sidebar + card + tabela é a composição que a skill oficial sugere para dashboard. Drawer e sheet existem para painel lateral | [skill](https://github.com/shadcn/ui/blob/main/skills/shadcn/SKILL.md), [drawer](https://ui.shadcn.com/docs/components/base/drawer) |
| Web Interface Guidelines | Estado importante na URL, para o botão voltar não perder a filial aberta. Ação destrutiva pede confirmação ou desfazer | [guidelines](https://github.com/vercel-labs/web-interface-guidelines) |

Não achei um print público e estável do kanban da Saipos para embutir. O texto da página descreve o quadro; o desenho fino não está nessa fonte.

## Ideias por pedaço

**Fila.** Colunas com os nomes do fluxo, cartão com número do pedido, bairro ou “retirada”, e há quanto tempo entrou. Mover com botão (“iniciar produção”) além de arrastar. Arrastar sozinho falha no teclado. AutoAnimate animaria o card mudando de coluna se a lista for React ([doc](https://auto-animate.formkit.com/)); o protótipo faz a troca sem biblioteca.

**Cardápio e cupom.** Lista densa, não grade de foto grande. A cozinha edita preço e “esgotado”. Cupom é um formulário curto: valor ou percentual, validade, frete grátis. Erro ao lado do campo, não só um toast. Sonner ([repositório](https://github.com/emilkowalski/sonner)) é um candidato a aviso, não à fila.

**Frete por CEP.** Tabela de faixas: do CEP, até o CEP, valor, ou “não entregamos”. Uma faixa inválida aparece na hora. Isso é formulário, não mapa. Mapa é desejável mais tarde e pesa.

**Troca de contexto.** Um controle no topo: empresa, depois filial. Quem tem uma filial só não vê uma escolha vazia. Ao trocar, a fila inteira muda. Guardar a filial na URL, como pede a guideline da Vercel, evita a pessoa atualizar a página e cair na casa errada.

**Skeleton.** Na primeira carga da fila, blocos cinza no formato do cartão, com um atraso mínimo para não piscar se a resposta for imediata. A guideline da Vercel sugere cerca de 150–300 ms para aparecer e 300–500 ms visível ([texto](https://github.com/vercel-labs/web-interface-guidelines)).

O protótipo [prototipos/painel/](prototipos/painel/) testa o kanban, o seletor de filial e o skeleton. Não testa impressão nem mapa.
