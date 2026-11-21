# Cores e paletas

Paleta do **produto** Menu Chef: painel da loja, site da plataforma e tema padrão do cardápio antes de a casa aplicar a própria marca. O cardápio de cada restaurante continua com logo e cores da casa. Isso é o que Consumer, Goomer e Cardapex já prometem, e é o motivo de o canal próprio existir ao lado do iFood.

As razões de contraste abaixo foram calculadas pela fórmula de luminância relativa do WCAG (canal linearizado, coeficientes 0,2126, 0,7152 e 0,0722). Os números estão arredondados em duas casas. Servem para escolher a paleta, não para um laudo de acessibilidade de uma tela pronta.

## O que a cor faz num delivery

No food service, a convenção de marca é estável, mesmo sem ser uma lei fisiológica:

- **Vermelho, laranja e amarelo** aparecem em comida rápida porque sugerem calor, pressa e apetite. É o território do iFood, do Consumer e de grande parte das redes de lanche.
- **Verde** sugere folha, fresco e natural. Funciona em salada, açaí e saudável. Como cor única de um sistema, também lembra banco e aplicativo de mensagem.
- **Marrom, terracota e creme** sugerem forno, pão, café e grelha. Menos “app de cupom”, mais cozinha.
- **Azul** sugere confiança e ferramenta. É a cor que vários sistemas brasileiros escolheram para o **software**, enquanto a foto do prato faz o trabalho do apetite.
- **Roxo** é a saída deliberada do aiqfome. O fundador contou que o mercado estava tomado de vermelho e amarelo, e que o roxo foi aceito justamente por ser fora do padrão ([Promoview, 28/05/2025](https://www.promoview.com.br/aiqfome-comunicacao-hiperlocal/)).

Cor não substitui palavra. Pessoa com baixa visão de cor não distingue só pelo matiz. Status de pedido precisa de texto (“em produção”, “pronto”) e de ícone, além da cor. O critério de contraste do WCAG trata disso ao calcular luminância e ignorar o matiz: o que segura a leitura é claro contra escuro ([entendendo o 1.4.3](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html)).

## Contraste

WCAG 2.2, critério 1.4.3, nível AA:

- Texto comum: pelo menos **4,5:1**.
- Texto grande (cerca de 18 pt, ou 14 pt em negrito): pelo menos **3:1**.
- Logo e texto decorativo ficam de fora.

Componentes de interface (borda de campo, ícone que significa algo) pedem **3:1** no critério 1.4.11 ([entendendo o 1.4.11](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html)).

Nesta proposta, botão com texto branco só usa cor que passa de 4,5:1. Amarelo claro de destaque não recebe texto branco: o branco em cima de `#F59E0B` fica em **2,15:1**.

## O que os concorrentes usam

Hex extraído do HTML público em 21/11/2026, salvo quando a fonte é outra. É a cor que o site está servindo, não um manual oficial completo.

| Marca | Cores vistas | Leitura |
| --- | --- | --- |
| iFood | `#EA1D2C` e branco. Pantone 185 C numa reprodução pública do guia de identidade ([pdfcoffee](https://pdfcoffee.com/guia-de-identidade-visual-ifood-pdf-free.html)). O site ifood.com.br respondeu 403 para leitura automática | Vermelho de apetite, dono do “pedir comida” no Brasil. Chegar perto desse hex faz o Menu Chef parecer pele do iFood |
| Consumer | No HTML da home: `#F42717`, `#D91B1B`, azul `#0259A3`, creme `#F3E6B8` e `#FFF8E6`, verde `#2ECC71` | Também é vermelho de comida, com creme e um azul de apoio. É a referência de interação do dono, e por isso não deve ser copiada na tinta |
| Saipos | No HTML: roxo `#280E59`, azul `#0842BB` e `#286EFF`, creme `#F0EBE2` e `#FAF5EB`, laranja `#FF4B1E`. Agregador não oficial ([Brandfetch](https://brandfetch.com/saipos.com)) lista `#406EFF`, `#E96312` e `#280E59` | Ferramenta, não molho. O creme esquenta o azul. O laranja aparece como acento, não como marca inteira |
| Anota AI | No HTML atual: azul `#0098FC` e `#0188E0`, fundo `#F6F7F8`. O Brandfetch lista `#BD2426`, que **não** dominou a página lida | O produto mudou para azul de software. O vermelho antigo, se ainda existir no logo, não é a tela |
| Goomer | HTML com `#0E5CFE`. O território visual oficial descreve azul e branco como paleta principal, e uma paleta secundária vibrante para ilustração, sem os hex no texto lido ([brand](https://goomer.brandgym.com.br/index.php/territorio-visual/)) | Azul tecnológico. A comida fica na foto |
| Cardápio Web | `#1863DC` e roxo `#59327A` no HTML | Azul com roxo. A comunicação da marca usa roxo também no texto do Reclame Aqui |
| Aiqfome | Lilás `#C484D9` sobre quase preto no HTML. O roxo é escolha declarada. Brandfetch, não oficial, lista `#FFCC33`, `#D66195` e `#A37540` | O único grande que fugiu do vermelho de propósito, com amarelo de apoio para não ficar frio |
| Delivery Much, InstaDelivery, Cardapex, OlaClick | Hex confiável não saiu do HTML lido | Não dá para afirmar a paleta |

O padrão do software de restaurante no Brasil, em 2026, é azul. O padrão do aplicativo em que o cliente pede é vermelho (iFood) ou roxo (aiqfome). O Consumer, que o dono citou pela interação, está no vermelho.

## Cores de status

Iguais nas quatro paletas, para o pedido não mudar de significado quando a marca muda. No modo claro o texto branco fica em cima do tom; no escuro o tom vira texto claro sobre o fundo escuro.

| Papel | Claro | Branco sobre a cor | Cor sobre creme `#FFFBF5` | Escuro (texto) | Texto claro sobre `#1C1410` |
| --- | --- | --- | --- | --- | --- |
| Sucesso | `#15803D` | 5,02:1 | 4,86:1 | `#86EFAC` | 12,93:1 |
| Alerta | `#B45309` | 5,02:1 | 6,88:1 no tom `#92400E` | `#FDE68A` | 14,58:1 |
| Erro | `#B91C1C` | 6,47:1 | 6,28:1 | `#FECACA` | 12,55:1 |
| Info | `#1D4ED8` | 6,70:1 | 6,50:1 | `#BFDBFE` | 12,77:1 |

`#15803D` sobre o creme passa AA por pouco (4,86:1). Em botão, o caminho folgado é texto branco em cima do próprio verde (5,02:1). Alerta como texto corrido no creme usa `#92400E` (6,88:1), não o amarelo `#F59E0B`.

Status nunca depende só da cor: ícone e rótulo acompanham. “Pronto” não pode ser apenas um ponto verde.

## Paleta 1. Brasa clara

Conceito: chapa, molho e pão. Quente, sem ser o escarlate do iFood (`#EA1D2C` é mais claro e mais puro). O fundo é o creme do miolo do pão, não o branco de hospital.

| Papel | Hex | Uso |
| --- | --- | --- |
| Primária | `#C2410C` | Botão principal, link forte, marca |
| Secundária | `#44403C` | Barra, título de apoio, texto de botão secundário |
| Destaque | `#F59E0B` | Selo de promoção e preço, sempre com texto `#1C1917` |
| Fundo | `#FFFBF5` | Página |
| Superfície | `#FFFFFF` | Cardápio, sacola, cartão de pedido |
| Texto | `#1C1917` | Texto corrido |
| Sucesso | `#15803D` | Confirmado, pronto, pago |
| Alerta | `#B45309` | Atraso, pausa, estoque curto |
| Erro | `#B91C1C` | Recusado, fora da área, pagamento falhou |
| Info | `#1D4ED8` | Saiu para entrega, aviso neutro |

Contraste no modo claro:

| Par | Razão | AA texto comum |
| --- | --- | --- |
| Texto `#1C1917` no fundo `#FFFBF5` | 16,96:1 | Passa |
| Texto `#1C1917` na superfície `#FFFFFF` | 17,49:1 | Passa |
| Primária `#C2410C` no fundo `#FFFBF5` | 5,02:1 | Passa |
| Branco na primária `#C2410C` | 5,18:1 | Passa |
| Secundária `#44403C` no fundo | 9,96:1 | Passa |
| Branco na secundária | 10,27:1 | Passa |
| Texto `#1C1917` no destaque `#F59E0B` | 8,14:1 | Passa |
| Branco no destaque `#F59E0B` | 2,15:1 | Não usar |

Modo escuro, para cozinha e balcão à noite (a Goomer já trata modo escuro como função de operação):

| Papel | Hex |
| --- | --- |
| Fundo | `#1C1410` |
| Superfície | `#2A211C` |
| Texto | `#FAF6F1` |
| Primária | `#FB923C`, com texto `#1C1410` no botão |
| Destaque | `#FDBA74`, com texto `#1C1410` |
| Sucesso, alerta, erro, info | `#86EFAC`, `#FDE68A`, `#FECACA`, `#BFDBFE` |

| Par | Razão |
| --- | --- |
| Texto `#FAF6F1` no fundo `#1C1410` | 16,87:1 |
| Texto `#FAF6F1` na superfície `#2A211C` | 14,64:1 |
| Texto `#1C1410` na primária `#FB923C` | 8,02:1 |
| Destaque `#FDBA74` no fundo | 10,76:1 |

O alerta `#B45309` e a primária `#C2410C` são parentes. Por isso pausa e atraso usam ícone e a palavra, não só o laranja.

## Paleta 2. Horta

Conceito: folha e fruta. Serve se as primeiras casas forem açaí, salada e natural. O amarelo de destaque devolve o apetite que o verde sozinho não dá.

| Papel | Hex |
| --- | --- |
| Primária | `#166534` |
| Secundária | `#3F6212` |
| Destaque | `#EAB308`, texto `#14221A` |
| Fundo | `#F7FBF4` |
| Superfície | `#FFFFFF` |
| Texto | `#14221A` |
| Sucesso | `#047857` |
| Alerta | `#B45309` |
| Erro | `#B91C1C` |
| Info | `#1D4ED8` |

| Par | Razão | AA texto comum |
| --- | --- | --- |
| Primária `#166534` no fundo `#F7FBF4` | 6,81:1 | Passa |
| Branco na primária | 7,13:1 | Passa |
| Texto `#14221A` no fundo | 15,75:1 | Passa |
| Texto `#14221A` no destaque `#EAB308` | 8,60:1 | Passa |
| Branco na secundária `#3F6212` | 7,08:1 | Passa |
| Branco no sucesso `#047857` | 5,48:1 | Passa |

Sucesso e primária são os dois verdes. O sucesso é mais azulado (`#047857` contra `#166534`) e, mesmo assim, o rótulo “pago” ou “pronto” é obrigatório.

Modo escuro: fundo `#052E16`, superfície `#14532D`, texto `#ECFDF5` (8,65:1 sobre a superfície), primária clara `#86EFAC` com texto `#052E16` (10,62:1).

## Paleta 3. Forno

Conceito: pizza, padaria e massa. Terracota em vez de laranja vivo. Mais calma que a Brasa, menos “lanche de aplicativo”.

| Papel | Hex |
| --- | --- |
| Primária | `#9A3412` |
| Secundária | `#44403C` |
| Destaque | `#FDBA74`, texto `#1C1917` |
| Fundo | `#FAF7F2` |
| Superfície | `#FFFFFF` |
| Texto | `#1C1917` |
| Sucesso | `#15803D` |
| Alerta | `#854D0E` |
| Erro | `#B91C1C` |
| Info | `#1D4ED8` |

| Par | Razão | AA texto comum |
| --- | --- | --- |
| Primária no fundo `#FAF7F2` | 6,84:1 | Passa |
| Branco na primária | 7,31:1 | Passa |
| Texto no fundo | 16,37:1 | Passa |
| Secundária no fundo | 9,61:1 | Passa |
| Texto `#1C1917` no destaque `#FDBA74` | 10,37:1 | Passa |
| Branco no alerta `#854D0E` | 6,85:1 | Passa |

O alerta `#854D0E` ainda é marrom, como a primária. A distância é maior do que na Brasa, e continua insuficiente sem ícone.

Modo escuro pode reaproveitar o da Brasa (`#1C1410`, `#2A211C`, `#FAF6F1`). A primária clara do botão fica `#FDBA74` com texto `#1C1410` (10,76:1 no fundo, e o mesmo par texto-escuro sobre o pêssego passa de 10:1).

## Paleta 4. Mesa

Conceito: salão, vinho e toalha. Menos fome imediata, mais restaurante. Longe do iFood e longe do azul dos sistemas.

| Papel | Hex |
| --- | --- |
| Primária | `#881337` |
| Secundária | `#44403C` |
| Destaque | `#FBBF24`, texto `#1C1917` |
| Fundo | `#FFF1F2` |
| Superfície | `#FFFFFF` |
| Texto | `#1C1917` |
| Sucesso | `#047857` |
| Alerta | `#B45309` |
| Erro | `#B91C1C` |
| Info | `#1E3A5F` |

| Par | Razão | AA texto comum |
| --- | --- | --- |
| Primária `#881337` no fundo `#FFF1F2` | 8,71:1 | Passa |
| Branco na primária | 9,57:1 | Passa |
| Texto `#1C1917` no destaque `#FBBF24` | 10,48:1 | Passa |
| Branco na info `#1E3A5F` | 11,50:1 | Passa |
| Branco no sucesso `#047857` | 5,48:1 | Passa |

Erro `#B91C1C` e primária `#881337` são os dois vermelhos. É o ponto fraco desta paleta: “recusado” e “botão de pedir” podem parecer a mesma família. Ícone e rótulo resolvem; mesmo assim é a paleta em que o erro mais compete com a marca.

Modo escuro: fundo `#1C1410` fica quente demais para o vinho. Fundo neutro `#292524`, texto `#FAF6F1` (14,10:1), primária clara com o mesmo vinho só se o texto em cima for branco (9,57:1, o par não depende do fundo da página).

## Recomendação

**Brasa clara** para o Menu Chef.

É a única das quatro que faz as três coisas ao mesmo tempo. Parece comida, não ferramenta azul. Não repete o vermelho `#EA1D2C` do iFood nem o `#F42717` do Consumer, que é a referência de interação e não de tinta. E passa AA nos pares de texto, botão e fundo, com modo escuro já contado para quem opera de noite.

Horta fica de reserva se o primeiro grupo de clientes for comida saudável. Forno fica de reserva se for pizzaria e padaria: o contraste da primária é até mais folgado (7,31:1 com branco), e o alerta mistura mais com a marca. Mesa é a mais “restaurante” e a pior para status de erro, porque a marca já é vermelha.

O amarelo `#F59E0B` não é cor de texto branco. Promoção e preço em destaque usam texto `#1C1917`. Status usa a tabela verde, âmbar, vermelho e azul, com palavra ao lado. A casa que quiser o cardápio nas cores dela troca a primária do cardápio público; o painel da operação pode permanecer na Brasa, para o funcionário não reaprender o significado de “pronto” a cada loja.
