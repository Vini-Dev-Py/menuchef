# Recursos planejados do Menu Chef

Inventário do que o produto precisa ter, na linguagem de quem usa. Não há especificação técnica, prazo nem marco. A classificação vem do fluxo que o dono descreveu e do que a pesquisa em [concorrentes.md](concorrentes.md) e [lacunas-e-oportunidades.md](lacunas-e-oportunidades.md) mostrou como pedido ou como falha repetida.

## Classificação

| Marca | Critério |
| --- | --- |
| Essencial | Sem isso o fluxo quebra, ou o dono já pediu |
| Desejável | O mercado já vende e o restaurante sente falta, mas o pedido anda sem isso |
| Diferencial | Pouca gente faz bem, ou só no plano caro; separa o Menu Chef |

Perfil de quem usa:

- **Empresa** é o dono ou gestor da loja.
- **Funcionário** é quem confirma, produz, cobra ou despacha, com permissão menor que a do dono.
- **Cliente** é quem pede.
- **Administrador** é quem opera a plataforma Menu Chef, não o restaurante.

## 1. A empresa contrata o sistema

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Cadastro da empresa com nome, documento e dados da loja | Empresa, Administrador | Essencial | Porta de entrada do fluxo |
| Uma loja pronta para cadastrar cardápio, sem implantação obrigatória | Empresa | Essencial | Cardapex e Consumer mostram que a loja quer abrir e usar. Implantação assistida pode existir; não pode ser a única porta |
| Assinatura com preço, o que está incluso e o que é módulo, escritos antes de pagar | Empresa, Administrador | Essencial | A reclamação mais cara do mercado é cobrança diferente do combinado (Anota AI, Cardápio Web, Delivery Much) |
| Cancelar a renovação por um caminho visível | Empresa | Essencial | “Não consigo cancelar” é categoria grande na Anota AI, na Saipos e na Goomer |
| Teste ou começo sem fidelidade longa | Empresa | Desejável | Goomer, Consumer e InstaDelivery usam isso para a loja experimentar. O formato (dias ou teto) fica para a oferta, não para este inventário |
| Mais de uma loja na mesma conta | Empresa, Administrador | Diferencial | Neemo e Saipos atendem rede. Não é o primeiro restaurante do fluxo |

## 2. A empresa cadastra o cardápio

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Categorias | Empresa | Essencial | Pedido do dono. Todo concorrente de cardápio tem |
| Produtos com nome, preço, descrição e foto | Empresa | Essencial | Pedido do dono. Foto é o que Goomer, Consumer e Anota AI tratam como padrão |
| Adicionais e variações, com preço e obrigatoriedade (escolher um, vários, nenhum) | Empresa | Essencial | Pedido do dono. É onde o pedido chega errado quando o sistema é fraco |
| Observação livre no item (“sem cebola”) | Empresa, Cliente | Essencial | A impressão do Consumer destaca observação. O cliente de marketplace reclama de item errado |
| Produto indisponível, sem apagar o cadastro | Empresa | Essencial | A Goomer mostra “indisponível” no cardápio de tablet. Esgotar no meio do jantar é rotina |
| Promoção no produto ou na categoria (preço menor, combo, destaque) | Empresa | Essencial | Pedido do dono |
| Cupom com regra simples: valor, percentual, frete grátis, validade, uso limitado | Empresa | Essencial | Pedido do dono. Consumer, Anota AI, Goomer e Cardapex já fazem |
| Horário de funcionamento da loja | Empresa | Essencial | Fora do horário o cliente vê a loja fechada e não paga à toa |
| Pausa da loja | Empresa, Funcionário | Essencial | Chuva, forno quebrado, fila impossível. Quase não é anunciado; a operação pede todo dia |
| Horário de um produto (só no almoço, só na sexta) | Empresa | Desejável | Goomer vende isso no plano Automatizar |
| Meio a meio e preço de dois sabores | Empresa | Desejável | Pizzaria. Goomer deixa no plano alto de mesa; Consumer anuncia tamanho e pergunta |
| Rótulo alimentar (vegetariano, sem glúten, picante) | Empresa | Desejável | Goomer no cardápio de tablet. Evita erro e pergunta no WhatsApp |
| Sugestão de acompanhamento | Empresa | Desejável | Cardapex e Goomer. Sobe o ticket dentro do cardápio |
| Pedido mínimo | Empresa | Desejável | iFood no Plano Básico e Cardapex |
| Fiado para cliente conhecido | Empresa | Desejável | Cardápio Web inclui em todos os planos. É hábito de balcão, não de marketplace |
| Domínio com a marca do restaurante | Empresa | Diferencial | MenuDino Completo e Neemo Pro. O link da plataforma resolve o começo |

## 3. O cliente vê o cardápio

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Cardápio no navegador do celular, sem instalar aplicativo | Cliente | Essencial | Padrão de Consumer, Goomer, Anota AI, Cardápio Web, InstaDelivery e OlaClick |
| Link para enviar no WhatsApp, Instagram e embalagem | Empresa, Cliente | Essencial | É assim que o canal próprio chega na pessoa |
| Cardápio com a cara da loja (nome, logo, cores) | Empresa, Cliente | Essencial | O marketplace impõe a marca dele. O canal próprio existe para não fazer isso |
| Loja fechada ou pausada explica o horário e não abre a sacola | Cliente | Essencial | Liga com a pausa e o horário |
| Busca e navegação por categoria | Cliente | Essencial | Cardápio longo sem categoria vira PDF no celular |
| QR Code de mesa | Empresa, Cliente | Desejável | O fluxo do dono é entrega e retirada. Mesa é o segundo salão, e a Goomer cobra à parte. Vale ter, sem misturar com o frete |
| Cardápio só de leitura, sem pedido | Empresa | Diferencial | Goomer oferece no QR Code. Útil para quem ainda anota no papel |

## 4. O cliente monta a sacola

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Sacola com quantidade, adicional, observação e total parcial | Cliente | Essencial | O fluxo do dono |
| Editar a sacola antes de pagar | Cliente | Essencial | Sem isso o cliente desiste ou pede errado |
| Aviso do que falta escolher no produto (adicional obrigatório) | Cliente | Essencial | Evita o pedido incompleto que vira reclamação |
| Comprar com nome e telefone, sem criar conta | Cliente | Essencial | O MenuDino anuncia. Cadastro obrigatório derruba pedido |
| Conta do cliente, para quem quiser | Cliente | Essencial | O dono pediu contas de cliente |
| Guardar sacola de quem não terminou | Empresa | Diferencial | A Anota AI chama de recuperador de vendas. É comercial, não operacional |

## 5. Checkout

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Escolher entrega ou retirada | Cliente | Essencial | Fluxo do dono. Retirada não cobra frete |
| Frete fixo | Empresa, Cliente | Essencial | Pedido do dono |
| Frete por faixa de CEP | Empresa, Cliente | Essencial | Pedido do dono. Quase só o MenuDino escreve isso com todas as letras |
| CEP fora da área: avisar antes de pagar | Cliente | Essencial | Delivery Much tem reclamação de loja que “não entrega nesse endereço” depois do pedido |
| Pagar agora, online | Cliente | Essencial | Fluxo do dono |
| Pix com valor, confirmação e prazo de repasse visíveis para a loja | Empresa, Cliente | Essencial | Anota AI publica 0,99%. Aiqfome, Anota AI e Cardápio Web acumulam reclamação de repasse opaco |
| Cartão online | Cliente | Desejável | Todo marketplace e a maior parte dos cardápios oferecem. Pode entrar depois do Pix |
| Pagar na entrega ou na retirada (dinheiro, Pix na hora ou maquininha da loja) | Cliente, Funcionário | Essencial | Fluxo do dono. iFood deixa a loja escolher no Plano Básico. Os cardápios próprios falam pouco disso |
| Se o pagamento não passa pela plataforma, a plataforma não fica com percentual | Empresa | Essencial | Pedido explícito no relato do Aiqfome sobre taxa em dinheiro e maquininha da casa |
| Troco para pagamento em dinheiro | Cliente | Desejável | Operação de entrega. Não apareceu como destaque nas homes; a cozinha e o motoboy pedem |
| Cupom no checkout | Cliente | Essencial | Liga com o cupom que a empresa cadastra |
| Resumo final antes de fechar: itens, frete, desconto, total, pagamento e endereço ou retirada | Cliente | Essencial | Última chance de não errar o pedido |

## 6. O cliente fecha o pedido

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Número do pedido e comprovante na tela | Cliente | Essencial | Sem isso o cliente não tem o que mostrar na retirada |
| Histórico de pedidos na conta do cliente | Cliente | Essencial | Pedido do dono |
| Pedir de novo a partir de um pedido antigo | Cliente | Desejável | Atalho em cima do histórico. Não foi uma reclamação; é o uso natural da conta |
| Acompanhar o status com os mesmos nomes da cozinha | Cliente | Essencial | Recebido, confirmado, em produção, pronto, aguardando retirada ou saiu para entrega |
| Cancelar enquanto o restaurante não confirmou | Cliente | Essencial | Regra que o iFood publica e que os relatos mostram falhar quando o botão some |
| Aviso quando o status muda | Cliente | Desejável | Consumer avisa pelo WhatsApp. Pode ser tela primeiro e WhatsApp em seguida |
| Avaliação curta depois de concluído, inclusive nota baixa | Cliente, Empresa | Desejável | Há relato de quem não conseguiu avaliar no iFood e no Aiqfome |

## 7. O restaurante recebe, confirma e produz

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Painel da empresa com a fila do dia | Empresa, Funcionário | Essencial | Pedido do dono. O pedido não pode existir só no WhatsApp |
| Alerta de pedido novo | Funcionário | Essencial | InstaDelivery anuncia alarme. Sem aviso, “receber” depende de alguém olhar a tela |
| Confirmar o pedido | Funcionário | Essencial | Fluxo do dono. Separa “vi” de “vou fazer” |
| Recusar com motivo, antes de produzir | Funcionário | Essencial | Item acabou, fora de área que passou, loja pausando |
| Iniciar produção | Funcionário | Essencial | Fluxo do dono |
| Comanda com item, adicional, observação e pagamento (pago ou a receber) | Funcionário | Essencial | É o que falta quando a Saipos não imprime e quando o pedido mora no WhatsApp |
| Impressão da comanda | Funcionário | Essencial | Consumer, Cardápio Web e InstaDelivery tratam isso como o produto, não como extra |
| Impressão por estação (cozinha, bar, copa) | Funcionário | Desejável | Goomer e InstaDelivery anunciam. Casa pequena vive com uma impressora |
| Tela de cozinha no lugar do papel | Funcionário | Desejável | KDS é Saipos, Anota AI, Goomer no plano alto e Consumer no módulo mobile |
| Modo escuro no painel da operação | Funcionário | Desejável | Goomer cita no tablet. Balcão à noite |
| Permissões: quem cadastra cardápio, quem só opera pedido, quem vê financeiro | Empresa, Funcionário | Essencial | Pedido do dono |
| Dashboard do dia e do período: pedidos, faturamento, ticket, entrega contra retirada, produtos mais pedidos | Empresa | Essencial | Pedido do dono. Saipos vai além (CMV, DRE). O essencial aqui é a operação, não o contábil |
| Extrato do que foi pago online e do que falta repassar | Empresa | Essencial | Resposta direta às reclamações de repasse |
| WhatsApp como aviso, não como comanda | Empresa, Cliente | Desejável | Manda link, status e “está pronto”. O pedido mora no painel |
| Bot que só encaminha o cardápio e o status | Empresa | Diferencial | Anota AI automatiza a conversa inteira. O relato do Consumer mostra o robô que responde errado. Começar restrito é o diferencial |

## 8. Pronto, retirada ou entrega

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Marcar pronto e avisar o cliente | Funcionário, Cliente | Essencial | Fluxo do dono |
| Na retirada, o pedido fica aguardando o cliente, com o nome e o número visíveis | Funcionário | Essencial | Fluxo do dono |
| Na entrega, despachar para um entregador da casa | Funcionário | Essencial | Fluxo do dono |
| Cadastro simples de entregadores | Empresa | Desejável | Anota AI, Cardápio Web e InstaDelivery cadastram. Roteirização automática é outro patamar |
| Marcar entregue | Funcionário | Essencial | Fecha o fluxo e libera a avaliação |
| Tempo estimado informado ao cliente | Empresa, Cliente | Desejável | O iFood vive disso, e também das reclamações quando o tempo mente. Mostrar um prazo editável pela loja já ajuda |
| Mapa ou rastreio do entregador | Cliente | Diferencial | iFood no Plano Entrega, InstaDelivery e o app do Consumer. Exige operação que a casa pequena nem sempre tem |
| Integração com entregador de aplicativo (Entrega Sob Demanda e similares) | Empresa | Diferencial | Goomer deixa no plano Integrar |

## 9. Conta, fidelidade e canais que vêm depois

| Recurso | Perfil | Classe | Nota |
| --- | --- | --- | --- |
| Programa de fidelidade ou cashback | Empresa, Cliente | Desejável | Anota AI inclui no plano; Goomer e MenuDino também, em faixas pagas. Vem depois do histórico |
| Campanha para quem já pediu | Empresa | Diferencial | Disparo em massa é plano alto ou módulo (Consumer Alta Performance, Cardápio Web, Saipos Bot pago). Também é onde mora reclamação de mensagem indevida |
| Nota fiscal ao consumidor | Empresa | Desejável | Consumer e Saipos incluem; Cardápio Web cobra módulo. É obrigação de muita loja e não é o fluxo do pedido. Fica desejável, não essencial deste inventário |
| Estoque que baixa sozinho | Empresa | Desejável | Saipos e Consumer vendem ficha técnica. Útil, e é outro ofício (compra e receita), não o fechamento do pedido |
| Pedidos do iFood e de outros apps na mesma fila | Empresa | Diferencial | Saipos é a referência. Cardápio Web e Goomer cobram ou empurram para o plano de cima. O canal próprio precisa andar sozinho antes |
| Totem e autoatendimento de salão | Empresa | Diferencial | Consumer Alta Performance, Cardápio Web e Goomer. Fora do fluxo de delivery descrito |
| Vários caixas, TEF e frente de caixa completa | Empresa | Diferencial | Território do Consumer e da Saipos. O balcão simples (lançar um pedido de quem está no balcão) é desejável; o PDV fiscal completo não é o núcleo |

Balcão para lançar pedido de quem pede no local, sem ser um PDV fiscal, fica **desejável**: Cardápio Web inclui módulo de balcão em todos os planos e o fluxo do dono não cobre o salão, mas a casa que tem retirada quase sempre anota pedido de quem chegou.

## 10. Administrador da plataforma

Quem cuida do Menu Chef, não do restaurante.

| Recurso | Classe | Nota |
| --- | --- | --- |
| Ver empresas cadastradas e o estado de cada uma (ativa, pausada, inadimplente) | Essencial | Sem isso não há SaaS de várias empresas |
| Suspender e reativar uma empresa | Essencial | Inadimplência e abuso. A Anota AI é criticada por bloquear e sumir; o administrador precisa do botão e a empresa precisa entender o motivo |
| Planos, preço e fatura da assinatura | Essencial | É o outro lado da transparência pedida na contratação |
| Ver a fila de uma loja só para suporte, com registro de quem acessou | Essencial | O suporte do mercado é a maior reclamação. Acesso sem rastro vira outro problema |
| Aviso de manutenção visível para as lojas | Desejável | Cardápio Web chegou a publicar instabilidade do iFood no Reclame Aqui. O administrador avisa dentro do produto |
| Métricas da plataforma (lojas ativas, pedidos do dia, falhas de aviso) | Desejável | Operação da plataforma, não dashboard do restaurante |
| Impersonar a loja sem deixar marca | Não entra | Contradiz o rastro de suporte. Não é recurso planejado |

## Leitura por perfil

**Empresa.** Contrata, monta cardápio, define frete, cupom, promoção, horário e pausa, vê dashboard, extrato e histórico agregado, define o que o funcionário pode fazer.

**Funcionário.** Vive da etapa 7 e da etapa 8: alerta, confirmar, produzir, imprimir, avisar pronto, separar retirada e despacho. Não mexe em preço nem em assinatura, salvo permissão.

**Cliente.** Vê o cardápio, monta a sacola, escolhe entrega ou retirada e como pagar, fecha, acompanha, cancela antes da confirmação, consulta o histórico e pode avaliar.

**Administrador.** Cuida de empresas, planos e suporte. Não opera o pedido do restaurante no lugar da cozinha.

## O que este inventário não é

Não escolhe fornecedor de pagamento, não desenha tela, não define banco e não ordena sprints. A ordem implícita é só a do fluxo: sem cardápio não há sacola; sem confirmação não há produção; sem aviso de pronto a retirada e a entrega não existem. Fidelidade, fiscal, estoque, marketplace e totem ficam de fora desse núcleo de propósito.
