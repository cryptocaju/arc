# Arc Network — Pesquisa técnica e ideias de projeto

Documento de trabalho para submissão ao programa de builders da Arc (Office Hours / cargo
Builder no Discord). Contém o levantamento técnico da rede e 6 ideias de projeto avaliadas
por viabilidade real, não por "hype".

---

## 1. O que a Arc é, tecnicamente

Arc é uma L1 da Circle (emissora do USDC), EVM-compatível, construída para "stablecoin
finance". Está em **testnet pública** desde outubro de 2025; mainnet prevista para 2026.

| Item | Valor |
|---|---|
| Rede | Arc Testnet |
| Chain ID | `5042002` (`0x4CEF52`) |
| RPC | `https://rpc.testnet.arc.network` |
| WebSocket | `wss://rpc.testnet.arc.network` |
| Explorer | `https://testnet.arcscan.app` |
| Faucet | `https://faucet.circle.com` |
| Domínio CCTP | `26` |
| USDC | `0x3600000000000000000000000000000000000000` (ERC-20, 6 decimais) |
| EURC | `0x89B50855Aa3bE2F677cD6303Cec089B5F319D72a` (6 decimais) |
| Gateway Wallet (testnet) | `0x0077777d7EBA4688BDeF3E311b846F25870A19B9` |
| Gateway Minter (testnet) | `0x0022222ABE238Cc2C7Bb1f21003F0a260052475B` |

Fonte: skill oficial `circlefin/skills` → `plugins/circle/skills/use-arc/SKILL.md`.

### 1.1 A pegadinha do USDC nativo

Na Arc **o gas token nativo é o próprio USDC**. Não existe "token nativo + token USDC":
é um único pool de fundos exposto por duas interfaces.

- **View nativa**: 18 decimais. Só para gas e `msg.value`. `useBalance` do wagmi devolve isso.
- **View ERC-20**: 6 decimais, em `0x3600…0000`. Use para saldo, transfer, approve, display.

Regras que quebram apps de gente que vem de outras chains:

- Nunca somar as duas views (dobra o saldo).
- USDC → nativo **não é swap**, é o mesmo ativo. Rejeitar essa rota antes da lógica de fees.
- Nunca chamar `decimals()` em sentinela nativa (`0xEeee…eEEeE`, `0x0…0`) — reverte.
- Fator entre as views: `1e18` nativo = `1e6` ERC-20.

### 1.2 Diferenciais da rede que dão narrativa a um projeto

1. **Taxa estável e denominada em dólar.** O custo de uma transação não oscila com o preço
   de um token volátil. Isso muda a economia de qualquer produto que faça **muitas
   transações pequenas** — estratégias de rebalanceamento, streaming de pagamento,
   micropagamento de agente. Em L1/L2 normais essas estratégias morrem quando o gas dispara.
2. **Finalidade determinística sub-segundo** (consenso Malachite). Permite UX de pagamento
   síncrono: confirmar na hora, sem "aguardando confirmações".
3. **Account abstraction nativa**: ERC-4337 e EIP-7702, com paymaster/patrocínio de gas,
   transações em lote e smart accounts.
4. **Privacidade opt-in**: blindar seletivamente detalhes sensíveis de transação preservando
   auditabilidade. *(Superfície de API a confirmar na doc — ver §4.)*
5. **StableFX**: engine de FX on-chain, execução por RFQ com múltiplos provedores de
   liquidez e settlement atômico para pares de stablecoin (USDC/EURC etc.). **Atenção: é
   permissionado** — só instituições com KYB/AML aprovado. Não dá para construir em cima
   de forma permissionless hoje.
6. **Primitivas de agente de IA**: registro de agentes e mercados de trabalho **ERC-8183**
   (job definition → escrow → submissão de entregável → atestação de avaliador).
7. **Circle Gateway**: saldo USDC unificado entre chains, transferência crosschain em
   <500ms via burn/mint.

### 1.3 Restrições reais para escopo de MVP

- Testnet apenas. Nada de mainnet.
- Somente USDC e EURC disponíveis. Ideias "multi-moeda" ficam limitadas a 2 moedas por ora.
- Liquidez de DEX na testnet é essencialmente inexistente → se o projeto precisa de
  mercado, você provavelmente vai ter que **deployar o mercado também**.
- StableFX é permissionado → para FX permissionless, use oráculo + seu próprio pool.

---

## 2. Sobre a ideia do bot de grid

Vale corrigir a premissa: **não é verdade que "só existe stablecoin, logo não há trade"**.
Existe USDC/EURC, ou seja, existe **câmbio EUR/USD on-chain**. EUR/USD tem faixa diária de
~0,3–0,8% e é um par historicamente *mean-reverting* em janelas curtas — que é exatamente
o regime onde grid funciona melhor. Grid em par volátil (ETH) quebra em tendência forte;
grid em par de FX é a aplicação clássica da estratégia no mercado tradicional.

E o encaixe na Arc é forte por um motivo específico e defensável: **grid é uma estratégia
que faz muitas ordens pequenas, então ela é dominada pelo custo de transação.** Numa chain
com gas volátil, o P&L do grid é imprevisível porque o custo por rebalanceamento é
imprevisível. Na Arc, o custo por rebalanceamento é um valor conhecido em dólar. Isso é
uma tese de produto, não um chavão.

O problema honesto: o retorno absoluto de grid em USDC/EURC é **baixo** (captura de spread
em um par de baixa volatilidade). Não vende como "trading bot de lucro". Vende como
**market making passivo / gestão de tesouraria em FX** — que é exatamente o público
institucional da Arc. Está na lista abaixo como Ideia 1, com esse enquadramento.

---

## 3. As 6 ideias

Cada uma com: problema, por que Arc, arquitetura, escopo de MVP, esforço solo e risco.

### Ideia 1 — StableFX Ladder: market maker passivo para pares de stablecoin

**Problema.** Uma tesouraria que carrega USDC e EURC precisa converter nos dois sentidos ao
longo do tempo e paga spread para um balcão em cada conversão. Ninguém oferece a ela o lado
*passivo*: ganhar o spread em vez de pagá-lo.

**Por que Arc.** Requer muitos reposicionamentos de ordem por dia → só fecha a conta com
taxa estável em dólar. E USDC/EURC é o par nativo da rede.

**Arquitetura.**
- `LadderVault.sol` — cofre ERC-4626 que aceita depósito nos dois lados do par.
- `StableCLOB.sol` — livro de ordens limitadas minimalista para um único par, com ticks em
  bps (não precisa de AMM genérico: a faixa de preço é estreita e conhecida).
- `LadderStrategy.sol` — coloca N ordens de compra abaixo e N de venda acima do mid,
  espaçadas em X bps; ao ser preenchida, recoloca o lado oposto. Sem oráculo no caminho
  crítico: o mid vem do próprio livro.
- Keeper em TypeScript (viem) que dispara o recolocamento; ou torne o recolocamento
  `permissionless` com recompensa, para não depender de você estar online.
- Frontend: faixa configurada, ordens vivas, spread capturado, custo de gas acumulado,
  P&L líquido. **Mostrar o gas em dólar como linha de primeira classe** — é a prova da tese.

**MVP.** Um par, uma faixa, recolocamento permissionless, dashboard com P&L líquido de gas.

**Esforço.** 3–4 semanas solo. O CLOB é a parte grande; dá para começar com um livro de
ordens em array ordenado e ticks fixos, sem otimização.

**Diferencial.** Não existe market making passivo pensado para pares de stablecoin — todo
mundo copia Uniswap v3 e sofre com IL num par que não precisa dessa máquina toda.

**Risco.** Sem contraparte na testnet, o livro fica vazio. Mitigação: escreva um "taker
bot" adversarial que negocia contra o livro seguindo o EUR/USD real via oráculo, e use isso
como ambiente de simulação. Isso *fortalece* a demo — vira backtest ao vivo.

---

### Ideia 2 — Camada de avaliadores e reputação para ERC-8183

**Problema.** ERC-8183 define job, escrow, entregável e **atestação de um avaliador**. O
padrão não resolve quem é o avaliador nem por que confiar nele. Hoje é um endereço único e
confiável — o elo fraco de todo o fluxo agêntico.

**Por que Arc.** A Arc declarou primitivas de agente e ERC-8183 como direção de produto.
Isso é infraestrutura faltante *dentro* da direção deles, não um app paralelo. Escrow e
pagamento de avaliador em USDC, com custo por atestação previsível.

**Arquitetura.**
- `EvaluatorRegistry.sol` — avaliadores fazem stake em USDC; stake define elegibilidade.
- `EvaluationRound.sol` — seleção de comitê de K avaliadores por job (VRF ou hash de bloco
  para o MVP), scoring por **commit-reveal**, mediana decide, quem fica fora do consenso é
  slashado.
- `ReputationSPB.sol` — reputação derivada só do histórico on-chain (jobs, taxa de acerto,
  volume). Sem admin, sem score arbitrário.
- Adapter que expõe a interface de avaliador que o ERC-8183 espera, para plugar em jobs
  existentes sem fork do padrão.
- Demo: 2 agentes (um contrata, um entrega) + 3 avaliadores, rodando o ciclo completo.

**MVP.** Registry + commit-reveal + slashing + um job real end-to-end.

**Esforço.** 3–4 semanas. Contratos são o núcleo; a UI pode ser mínima.

**Diferencial.** Alto. Todo mundo está construindo *agentes*; quase ninguém está
construindo o **julgamento** que faz o escrow do agente ser confiável.

**Risco.** Você precisa dos endereços do registry ERC-8183 na testnet da Arc — confirmar na
doc (§4). Se não houver deploy oficial, deploye a referência do padrão você mesmo.

---

### Ideia 3 — Mandatos de débito recorrente (o "débito automático" que crypto não tem)

**Problema.** Crypto tem *push* payments. O mundo real roda em *pull*: débito automático,
assinatura, cobrança recorrente. Hoje a única forma é `approve` infinito para um contrato
de terceiro — sem limite por período, sem revogação granular, sem trilha de auditoria.

**Por que Arc.** EIP-7702 + ERC-4337 nativos permitem delegação com política; Gateway
permite puxar de saldo unificado entre chains; taxa estável torna a cobrança de US$ 4,99
economicamente sã.

**Arquitetura.**
- `MandateRegistry.sol` — mandato assinado (EIP-712) com: beneficiário, teto por cobrança,
  teto por período, intervalo mínimo, validade, revogável unilateralmente a qualquer momento.
- `MandateExecutor.sol` — o beneficiário apresenta o mandato e puxa; o contrato aplica
  rate-limit e teto on-chain. Falha por saldo insuficiente é um evento tipado, não um revert
  opaco (o merchant precisa distinguir "sem fundos" de "revogado").
- Módulo de smart account (7702) para o pagador, para não depender de `approve` infinito.
- SDK TS mínimo: `createMandate`, `charge`, `revoke`, `listMandates`.
- Dois frontends de demo: painel do assinante (mandatos ativos, histórico, botão revogar) e
  painel do merchant (cobrar, MRR, falhas).

**MVP.** Registry + executor + SDK + os dois painéis. Sem Gateway na v1.

**Esforço.** 2–3 semanas. É a mais rápida de mostrar funcionando.

**Diferencial.** Bom. Existem protocolos de assinatura, mas quase nenhum trata o mandato
como **objeto de primeira classe com limites verificáveis on-chain e revogação soberana** —
que é o que um regulador ou um CFO pede.

**Risco.** Baixo. Só EVM padrão. Cuidado com replay de assinatura (nonce + chainId).

---

### Ideia 4 — Antecipação de recebíveis (invoice factoring) em stablecoin

**Problema.** PME emite fatura de US$ 50k com prazo de 30 dias e fica sem capital de giro.
Factoring tradicional cobra caro e leva dias. On-chain, a fatura é um fluxo de pagamento
verificável — logo, colateral.

**Por que Arc.** É literalmente o público da Arc (pagamentos B2B institucionais).
Privacidade opt-in resolve o bloqueio real de adoção: **nenhuma empresa quer publicar seus
termos comerciais e sua lista de clientes num explorer público.**

**Arquitetura.**
- `Invoice.sol` — ERC-721 representando a fatura: sacado, valor, vencimento, hash do
  documento off-chain. Valor e sacado blindados pela camada de privacidade, com divulgação
  seletiva para o financiador e para um auditor.
- `FactoringPool.sol` — cofre ERC-4626 de financiadores; compra faturas com desconto por
  leilão holandês (o desconto abre com o tempo até alguém aceitar → descoberta de preço sem
  oráculo).
- `SettlementEscrow.sol` — o sacado paga em USDC direto ao escrow; distribui principal +
  desconto ao pool. Se atrasar, a perda é do pool, com waterfall explícito.
- Reputação de pagador on-chain a partir de histórico de liquidação.

**MVP.** Fatura + leilão holandês + escrow de liquidação, sem tranches e sem privacidade
(privacidade entra na v2, depois de validar a superfície de API).

**Esforço.** 4–5 semanas. A mais ambiciosa da lista.

**Diferencial.** Alto no formato "simples e permissionless". Centrifuge e Goldfinch atacam
crédito estruturado grande; ninguém atende bem antecipação de fatura pequena em stablecoin.

**Risco.** Modelagem de crédito e default é onde essas coisas afundam. Seja explícito no
pitch: a v1 é sobre o **encanamento** (tokenizar, precificar, liquidar), não sobre
underwriting.

---

### Ideia 5 — Forward de FX colateralizado para tesouraria (USDC/EURC)

**Problema.** Uma empresa fatura em USDC e paga folha em EUR em 90 dias. Ela está exposta a
EUR/USD e hoje só tem duas opções on-chain: comer o risco, ou converter tudo agora e perder
o rendimento do caixa. Falta o instrumento óbvio: **contrato a termo**.

**Por que Arc.** EURC nativo + StableFX + settlement atômico. É o caso de uso que a Circle
está anunciando; construir o lado *permissionless* dele é complementar, não competitivo.

**Arquitetura.**
- `ForwardMarket.sol` — duas pontas travam colateral em USDC/EURC, com taxa a termo e data
  de liquidação acordadas; liquidação por oráculo de EUR/USD no vencimento (cash-settled,
  bem mais simples que entrega física).
- Colateral parcial com margem e liquidação por chamada de margem — ou, para o MVP,
  **totalmente colateralizado** (zero risco de crédito, zero liquidação, muito mais fácil de
  defender numa demo).
- `HedgeVault.sol` — do outro lado, um pool que vende forwards e ganha o prêmio,
  para haver contraparte.
- UI: "devo €X em N dias" → cotação → travar hedge → ver marcação a mercado.

**MVP.** Forward totalmente colateralizado, cash-settled, um par, um oráculo.

**Esforço.** 3 semanas para a versão totalmente colateralizada. Margem parcial dobra isso.

**Diferencial.** Muito alto. Praticamente ninguém constrói derivativos de câmbio entre
stablecoins — todo mundo faz perp de BTC/ETH.

**Risco.** Depende de oráculo de EUR/USD na testnet da Arc. Confirmar quais provedores estão
disponíveis (§4); se nenhum, um oráculo assinado próprio serve para a demo, declarado como
tal.

---

### Ideia 6 — Recibos verificáveis com divulgação seletiva

**Problema.** "Mostre ao auditor que este pagamento de US$ 2M aconteceu, sem revelar as
outras 400 transações desta carteira." Explorer público é tudo-ou-nada. Isso é um bloqueio
concreto de compliance, não teoria.

**Por que Arc.** Este projeto **só existe** por causa da privacidade opt-in com
auditabilidade preservada. É o uso mais direto do diferencial menos explorado da rede.

**Arquitetura.**
- `ReceiptAnchor.sol` — âncora o compromisso de um pagamento (Merkle/Pedersen) no momento da
  liquidação.
- Gerador de recibo: produz um documento assinado que revela *apenas* aquele pagamento e
  prova que ele pertence ao commitment ancorado.
- Verificador: página web onde o auditor cola o recibo e obtém verde/vermelho, sem RPC
  privilegiado e sem ver mais nada.
- Exportador para os formatos que um contador realmente usa (CSV/PDF com hash verificável).

**MVP.** Ancoragem + geração + verificação de um recibo.

**Esforço.** 2–3 semanas — **se** a camada de privacidade tiver API utilizável na testnet.
Sem isso, cai para prova de Merkle sobre eventos públicos, o que ainda é útil mas perde a
graça.

**Diferencial.** Alto e muito alinhado ao discurso institucional da Arc.

**Risco.** O maior da lista: dependência direta de uma feature cuja superfície eu não pude
verificar. **Confirme antes de escolher esta.**

---

## 4. O que confirmar na doc antes de decidir

Não consegui acessar `docs.arc.network` neste ambiente (bloqueio de rede). Verificar:

1. **Privacidade opt-in**: existe precompile/contrato de sistema? Está ativo na testnet?
   Qual a API? → decide se as Ideias 4 (v2) e 6 são viáveis.
2. **ERC-8183**: existe registry/escrow oficial deployado na testnet? Endereços? → Ideia 2.
   Ponto de partida: tutorial "Create your first ERC-8183 job" na doc.
3. **Oráculos**: quais provedores de preço estão na testnet, e existe feed EUR/USD? → Ideia 5.
4. **StableFX**: há alguma interface de leitura (cotações) acessível sem KYB? Se sim, dá
   para usar como referência de preço nas Ideias 1 e 5.
5. **Paymaster**: qual paymaster está disponível na testnet para patrocinar gas? → melhora a
   UX de todas as ideias (usuário sem saldo consegue transacionar).
6. **Sample Applications** na doc: ler antes de começar, para não construir algo que já
   existe como exemplo oficial.

---

## 5. Recomendação

**Escolha a Ideia 3 (mandatos de débito recorrente) ou a Ideia 2 (avaliadores ERC-8183).**

- **Ideia 3** se o objetivo é chegar rápido ao Office Hours com algo rodando: 2–3 semanas,
  zero dependência de feature não verificada, e o problema é imediatamente compreensível
  para qualquer pessoa da Circle — pagamento recorrente é o coração do negócio deles.
- **Ideia 2** se você quer o maior sinal técnico: é infraestrutura faltante numa direção que
  a própria Arc está empurrando, e é o tipo de contribuição que costuma converter em cargo
  de builder, porque outras pessoas passam a construir *em cima* do seu trabalho.

A Ideia 1 (grid/ladder de FX) é uma terceira opção legítima e é o refinamento defensável do
seu instinto original — só precisa ser vendida como market making de tesouraria, não como
bot de lucro.

Sequência sugerida: valide o §4 → escolha uma → contratos com testes Foundry primeiro →
frontend mínimo → deploy na testnet → demo em vídeo de 2 minutos → submeta.

Uma coisa que vale para **qualquer** escolha: instrumente e mostre o custo de gas em dólar
no seu dashboard. É a prova concreta de que você entendeu por que a Arc existe, e quase
nenhuma submissão vai fazer isso.
