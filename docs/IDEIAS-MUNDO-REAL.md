# 5 ideias partindo de problemas do mundo real

Sem jargão. Cada ideia começa por uma situação que qualquer pessoa já viveu.

## O que a Arc é, em uma frase

Uma rede feita só para movimentar dinheiro, onde:

- **A taxa é de centavos e é sempre igual.** Uma operação custa em torno de 2 centavos de
  dólar, hoje e no mês que vem. Em outras redes a taxa varia de centavos a dezenas de dólares
  sem aviso, o que mata qualquer produto que faça muitas transferências pequenas.
- **Cai na hora.** Menos de um segundo para confirmar, de forma definitiva.
- **É dólar ou euro de verdade**, emitidos pela Circle (a empresa do USDC).
- **Dá para esconder valores** e ao mesmo tempo dar acesso de leitura a um auditor.

Toda ideia abaixo se apoia nesses quatro pontos. As duas primeiras características são as que
mais importam: elas permitem produtos que **não existem em nenhuma outra rede** porque lá a
taxa comeria o valor movimentado.

---

## Ideia 1 — Caução de aluguel que ninguém consegue segurar

**A situação.** Você aluga um apartamento e deposita 3 meses de caução. No fim do contrato
começa a briga: o dono diz que tem estrago, você diz que não, e o dinheiro fica com ele
enquanto vocês discutem. Você não tem nenhuma garantia além da palavra dele.

**O problema em uma frase.** Quem guarda a caução é uma das partes interessadas na disputa.

**Como fica.** O dinheiro entra num cofre digital que **nem o inquilino nem o dono conseguem
abrir sozinhos**. O programa que guarda tem regras escritas antes de começar:

- Se os dois concordam → devolve na hora.
- Se ninguém reclama até X dias depois da saída → devolve automático para o inquilino.
- Se há disputa → uma terceira pessoa escolhida pelos dois no início (a imobiliária, um
  advogado) decide, e só pode dividir o valor que já está lá.

Todo mundo vê as regras desde o primeiro dia. O dono não precisa confiar no inquilino nem o
contrário — nenhum dos dois tem a chave.

**Por que na Arc.** Dinheiro parado por 30 meses precisa de um lugar estável: caução em dólar
não derrete. E devolver a caução custa centavos, então não faz sentido "ficar com o resto pra
cobrir a taxa".

**Dificuldade.** Baixa. É a mais fácil de construir e de explicar. Boa primeira escolha se
você quer algo pronto rápido.

**O limite honesto.** Isso não resolve *quem tem razão* na disputa — só garante que ninguém
foge com o dinheiro enquanto se discute. É bem menos do que parece, e ainda assim é a parte
que hoje não existe.

---

## Ideia 2 — Caixinha entre amigos com as regras no piloto automático

**A situação.** Aquele arranjo clássico: dez pessoas, cada uma coloca 100 por mês, e cada mês
uma pessoa leva os 1.000. No fim todo mundo recebeu uma vez. Hoje isso roda no grupo do
WhatsApp, com uma planilha e uma pessoa "de confiança" guardando o dinheiro.

**O problema em uma frase.** Alguém para de pagar depois de já ter recebido, ou o organizador
desaparece com o bolo — e não há nada a fazer.

**Como fica.** As regras entram no programa antes de começar: quantas pessoas, quanto por mês,
qual a ordem dos recebimentos. O dinheiro nunca passa pela mão de ninguém — vai direto de
quem paga para quem recebe naquele mês. Quem entra deposita uma garantia que só volta ao
terminar o ciclo, então quem sumir depois de receber perde a garantia. A ordem pode ser
sorteada de um jeito que ninguém consegue manipular. Todo mundo vê quem pagou e quem não
pagou.

**Por que na Arc.** É o encaixe mais claro de todos. São **muitas transferências pequenas e
repetidas** — 10 pessoas × 10 meses = 100 movimentações. Numa rede de taxa alta e variável,
isso é inviável: você pode pagar 15 dólares de taxa para mover 100. Na Arc a taxa é de
centavos e você sabe de antemão quanto vai ser em todos os 10 meses.

**Dificuldade.** Média. A lógica é simples, mas você tem que pensar direito no caso de
inadimplência.

**Por que é boa candidata.** Centenas de milhões de pessoas no mundo usam esse arranjo de
forma informal, e cripto praticamente ignorou isso enquanto construía a décima corretora
descentralizada. É útil, é original, e é fácil de entender numa demo de dois minutos.

**Cuidado.** Como produto real isso mexe com regra financeira (no Brasil, consórcio é
regulado). Para uma demonstração em testnet é tranquilo, mas não anuncie como produto
financeiro pronto.

---

## Ideia 3 — Pagamento de freelance internacional que não depende de confiança

**A situação.** Você é freelancer e fecha com um cliente dos Estados Unidos. Ele não te conhece
e tem medo de pagar antes. Você não conhece ele e tem medo de trabalhar de graça. No fim
alguém cede, e quando o dinheiro vem embora dias no caminho e chega com 6% a 8% a menos
somando taxa de plataforma, spread de câmbio e tarifas.

**O problema em uma frase.** Nem cliente nem freelancer querem ser o primeiro a confiar, e o
caminho do dinheiro é lento e caro.

**Como fica.** O cliente deposita o valor num cofre no começo — então você **vê** que o
dinheiro existe antes de trabalhar. O trabalho é dividido em etapas. Cada etapa entregue e
aprovada libera aquela fatia na hora, em dólar, direto na sua carteira. Se o cliente
simplesmente ignora você depois da entrega, existe um prazo: passado ele sem resposta, libera
automático. Se há discordância real, um terceiro escolhido pelos dois no início decide.

**Por que na Arc.** Chega em menos de um segundo em vez de dias, e a taxa é de centavos em
vez de percentual. Liberar cinco etapas de um trabalho custa uns 10 centavos de dólar no
total.

**Dificuldade.** Média-baixa.

**Um detalhe que ajuda.** A Arc já tem um padrão pronto para exatamente esse formato
(contratar → travar dinheiro → entregar → avaliar → pagar). Ele foi criado pensando em
programas de inteligência artificial contratando outros programas, mas o encanamento serve
igual para gente. Usar a peça oficial em vez de reinventar é um bom sinal para quem vai
avaliar seu projeto.

---

## Ideia 4 — Gorjeta que chega inteira em quem trabalhou

**A situação.** Você paga 10% de gorjeta no restaurante. Aquilo vai para o caixa, e como é
dividido entre garçom, cozinha e copa ninguém sabe direito. Muita gente da equipe descobre no
fim do mês um número que não consegue conferir. O mesmo vale para entregador, barbeiro,
manobrista.

**O problema em uma frase.** A gorjeta passa por um intermediário e a divisão é opaca para
quem deveria receber.

**Como fica.** O cliente aponta a câmera para um QR code na mesa e paga a gorjeta. O programa
divide na hora, seguindo a regra que a equipe combinou (por exemplo: 60% salão, 30% cozinha,
10% copa, entre quem estava naquele turno). Cada pessoa vê o valor chegar no celular **naquele
minuto**, com a conta aberta. Ninguém segura o dinheiro no meio.

**Por que na Arc.** Aqui o argumento é o mais forte de todas as ideias, e é matemático.
Dividir a gorjeta de um turno entre 8 pessoas custa cerca de **3 centavos de dólar**, num
valor total de, digamos, 200 dólares — ou seja, 0,015%. Em redes de taxa alta, dividir um
valor pequeno entre muita gente custa **mais que o próprio valor**, e por isso ninguém
constrói isso. Não é que seja difícil: é que era economicamente impossível.

**Dificuldade.** Baixa-média. O programa de divisão é simples; o trabalho está na interface
de celular, que precisa ser boa.

**O limite honesto.** O gargalo real não é técnico, é que o cliente precisaria ter uma
carteira digital — o que hoje quase ninguém tem. Como demonstração isso não importa. Como
produto, é o obstáculo principal, e é melhor você mesmo dizer isso na apresentação do que
deixar alguém apontar.

---

## Ideia 5 — Mensalidade automática com um limite que o cliente controla

**A situação.** Academia, escola, streaming, aluguel de equipamento. No mundo normal você
autoriza o débito automático e o valor sai todo mês. Você confia no banco para não deixarem
cobrar 10 vezes mais.

**O problema em uma frase.** Em cripto isso não existe: ou você paga manualmente todo mês, ou
dá uma autorização **sem limite nenhum** para a empresa tirar quanto quiser da sua carteira,
para sempre.

**Como fica.** Você assina uma autorização com limites escritos dentro dela: no máximo 50
dólares por cobrança, no máximo uma cobrança a cada 30 dias, válida por um ano, **cancelável
por você a qualquer momento sem pedir para ninguém**. A empresa cobra quando é devido; se ela
tentar cobrar o dobro ou cobrar duas vezes no mesmo mês, o programa simplesmente recusa. Não é
uma promessa de que ela vai se comportar — é impossível ela passar do limite.

**Por que na Arc.** Cobrar mensalidade de 5 dólares só faz sentido se a taxa for de centavos e
previsível. E a Arc já traz de fábrica o recurso que permite esse tipo de autorização com
regras.

**Dificuldade.** Baixa-média, e é a mais rápida de mostrar funcionando: uma tela do cliente
(autorizações ativas, histórico, botão de cancelar) e uma tela da empresa (cobrar, ver
recebimentos, ver falhas).

**Por que agrada quem avalia.** Cobrança recorrente é o coração do negócio da Circle. E
"autorização com limite verificável e cancelamento na mão do cliente" é a linguagem que um
diretor financeiro e um regulador entendem imediatamente.

---

## Bônus — Vaquinha em que o doador vê o dinheiro sendo usado

**A situação.** Você quer ajudar uma campanha, mas não doa porque não sabe se o dinheiro chega
onde diz que vai.

**Como fica.** Cada gasto sai de um cofre público e fica registrado com destino e valor. O
doador acompanha em tempo real. Para casos sensíveis (ajuda a uma pessoa específica), o valor
pode ficar escondido e ainda assim auditável por quem tem permissão.

Fácil de construir e emocionalmente forte numa demo. É a ideia menos original da lista —
já existe gente nessa — mas se você acrescentar liberação por metas (o dinheiro só destrava
quando a etapa é comprovada) fica bem mais interessante.

---

## Qual escolher

| Ideia | Facilidade | Originalidade | Encaixe na Arc |
|---|---|---|---|
| 1 — Caução de aluguel | Alta | Média | Bom |
| 2 — Caixinha entre amigos | Média | **Alta** | **Excelente** |
| 3 — Freelance internacional | Média | Média | **Excelente** |
| 4 — Divisão de gorjeta | Média | **Alta** | **Excelente** |
| 5 — Mensalidade com limite | Média | Média-alta | Bom |

**Recomendação: Ideia 2 ou Ideia 4.**

As duas têm o mesmo argumento poderoso, e é um argumento que quase nenhuma outra submissão vai
fazer: **"isso não é possível em outra rede, e aqui está o cálculo."** Muitas transferências
pequenas entre muitas pessoas é justamente o que taxa alta e variável impede. Mostrar essa
conta prova que você entendeu *por que a Arc existe*, e não só que sabe programar.

A Ideia 1 é a escolha certa se o que você quer é ter algo funcionando na semana que vem.

**Faça isso em qualquer uma que escolher:** coloque na tela o custo total em dólar de todas as
operações que o seu app fez. É a prova visual da tese, custa pouco para implementar, e
praticamente ninguém vai lembrar de fazer.
