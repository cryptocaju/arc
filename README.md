# Arc Network — Pesquisa técnica e ideias de projeto

> **Para implementar:** ver [`docs/ARC-TECH-REFERENCE.md`](docs/ARC-TECH-REFERENCE.md) —
> config de rede, a dualidade do USDC e o pre-flight check de saldo que ela exige, setup de
> viem/wagmi/Foundry, ERC-20, arquitetura de privacidade, ERC-8183, Gateway, custo de gas
> medido, oráculos, e a lista de conflitos entre fontes.

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

**Números (§4.4).** Recolocamento a ~150k de gas = **~0,025 USDC**. Posição de US$ 10k
capturando 5 bps por fill (US$ 5) → gas é **~0,5% da captura**, e é um valor fixo em dólar.
Coloque essa conta no pitch: é a diferença entre "acho que dá" e "aqui está a planilha".

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
- `CommitteeHook.sol` — **entra como o `hook` do `createJob`** (§4.2). É isto que torna o
  projeto plugável: qualquer pessoa passa o endereço do seu hook e ganha avaliação por
  comitê, sem fork do padrão e sem sua permissão.
- Demo: 2 agentes (um contrata, um entrega) + 3 avaliadores, rodando o ciclo completo contra
  a implementação de referência já deployada.

**MVP.** Registry + commit-reveal + slashing + hook + um job real end-to-end.

**Esforço.** 3–4 semanas. Contratos são o núcleo; a UI pode ser mínima.

**Diferencial.** Alto. Todo mundo está construindo *agentes*; quase ninguém está
construindo o **julgamento** que faz o escrow do agente ser confiável.

**Risco.** Baixou bastante depois da §4.2: a referência já está na testnet e o `hook` é um
ponto de extensão oficial. Resta confirmar o endereço no explorer e ler a interface esperada
do hook (quais callbacks, em que estados do job são chamados).

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

**Correção após §4.1.** Eu havia justificado esta ideia com "nenhuma empresa quer publicar
sua lista de clientes". Isso está **errado** para a fase 1 da privacidade da Arc: endereços
de remetente e destinatário **continuam visíveis** por design; só o *valor* é cifrado. Então
o que você consegue esconder é **quanto** foi faturado e com **que desconto** a fatura foi
antecipada — não *de quem*. Ainda é relevante (margem e termos comerciais são sensíveis), mas
é uma alegação bem mais modesta. Não venda privacidade de contraparte nesta ideia.

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

**Risco.** Reduzido pela §4.3: Chainlink, Pyth e RedStone estão documentados na Arc, e Pyth e
RedStone cobrem FX explicitamente. Falta só confirmar o feed id de EUR/USD na testnet — Pyth
em modo *pull* é a aposta mais segura. Se faltar, um oráculo assinado próprio serve para a
demo, declarado como tal.

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

**Esforço.** 2–3 semanas.

**Reavaliação após §4.1 — leia antes de escolher esta.** A privacidade existe e tem API
(precompile APS, `globalPublicKey`, `executePrivateTx`, `PrivateTx` assinada em EIP-712), o
que é bom. Mas **view keys já são um primitivo nativo da rede** — ou seja, a parte
criptográfica da "divulgação seletiva" a Arc já te dá de graça. O que sobra para você
construir é o *workflow*: geração de recibo, UX de verificação para o auditor, exportação
contábil. Isso é trabalho legítimo e útil, mas é **ferramenta em volta de um primitivo
existente**, não um primitivo novo — e portanto um sinal técnico mais fraco que as Ideias 2
ou 5 para efeito de candidatura a builder.

Escolha esta se o seu diferencial for produto e UX de compliance, não profundidade de
protocolo. E o risco continua sendo o maior da lista: é a ideia mais acoplada a uma feature
cuja superfície exata eu não pude ler na doc.

---

## 4. Perguntas abertas — respostas obtidas

Os domínios `docs.arc.io`, `docs.arc.network`, `community.arc.io` e `developers.circle.com`
estão bloqueados pela política de egresso do ambiente onde esta pesquisa foi feita. O que
segue foi recuperado por busca e por repositórios públicos no GitHub. **Confirme os
endereços na doc antes de escrever código contra eles.**

### 4.1 Privacidade opt-in — RESPONDIDO, com uma ressalva que muda o escopo

Arquitetura: contratos falam com o backend criptográfico via **precompiles**. O backend
inicial usa **TEE** (Trusted Execution Environment), com MPC/FHE/ZK plugáveis depois.

Fluxo concreto, pelo whitepaper de privacidade:

1. Cliente pede `globalPublicKey` ao **precompile APS**, que encaminha ao **pEVM** e devolve
   a chave pública global (X-Wing KEM).
2. Cliente cifra a calldata e despacha via `executePrivateTx`.
3. Dentro do enclave: decapsula, decifra com AES-256-GCM, verifica a assinatura EIP-712 da
   struct `PrivateTx` para recuperar o `msg.sender`, e executa o CALL confidencial.
4. **View keys** dão leitura controlada a auditor/regulador.

**A ressalva importante.** A fase 1 entrega **transferências confidenciais: o *valor* é
cifrado, mas endereços de remetente e destinatário permanecem visíveis** (proposital, para
compatibilidade com ferramentas de analytics/monitoramento).

Consequência direta: **a justificativa que eu dei para a Ideia 4 estava errada.** Não dá
para esconder "quem são seus clientes" hoje — só quanto você cobrou de cada um. Ver a
correção na Ideia 4 abaixo.

### 4.2 ERC-8183 — RESPONDIDO, e melhor do que o esperado

A implementação de referência **já está deployada na Arc testnet**, reportada em
`0x0747EEf0706327138c69792bF28Cd525089e4583` *(verificar no explorer)*. A assinatura de
criação de job é:

```solidity
createJob(
    address provider,   // quem executa
    address evaluator,  // quem avalia
    uint256 expiredAt,  // expiração
    string  description,
    address hook        // address(0) = fluxo padrão sem hook
)
```

O job nasce no estado `Open`. O cliente precisa ter USDC de testnet para o escrow do budget.

**O parâmetro `hook` é o achado que destrava a Ideia 2.** A camada de avaliadores entra como
hook — sem fork do padrão, sem pedir permissão a ninguém, plugável em jobs de terceiros. É
o ponto de integração ideal para infraestrutura de terceiros.

### 4.3 Oráculos — RESPONDIDO em nível de provedor

A doc tem uma página `/arc/tools/oracles` listando **Chainlink, Pyth e RedStone** na Arc.
Pyth e RedStone ambos cobrem **FX** explicitamente (Pyth: "crypto, equities, FX, metals",
com modelos pull e push; RedStone: cripto, LSTs, RWAs, fundos tokenizados, FX).

Ou seja, existe caminho para EUR/USD — o que valida as Ideias 1 e 5. **Falta confirmar o
feed id / endereço específico na testnet.** Pyth em modelo pull é o mais provável de estar
disponível sem deploy dedicado por chain.

### 4.4 Custo de gas real — dado concreto

De um deploy público de terceiros na Arc testnet: **3.166.394 de gas ≈ 0,522 USDC**, ou seja
**~1,65 × 10⁻⁷ USDC por unidade de gas** (~0,165 USDC por milhão de gas).

Isso permite calcular a economia da Ideia 1 em vez de torcer por ela. Um recolocamento de
ordem em ~150k de gas custa **~0,025 USDC (dois centavos e meio)**. Numa posição de US$ 10k
capturando 5 bps por fill (US$ 5), o gas é **~0,5% da captura**. A estratégia fecha a conta
com folga confortável — e, principalmente, esse número **não muda** amanhã.

Trate como dado de ordem de grandeza (uma fonte, custo de deploy) e re-meça você mesmo.

### 4.5 Paymaster — PARCIALMENTE respondido

Existe página `/arc/tools/account-abstraction` com plataformas e SDKs para smart accounts,
session keys, integração de paymaster e patrocínio de transação. Paymasters *enshrined* para
EURC e outras stablecoins como gas aparecem como **roadmap**, não como disponível hoje.
Confirmar o que já dá para usar na testnet.

### 4.6 Ainda em aberto

- **StableFX**: se há interface de leitura de cotação sem KYB (serviria de referência de
  preço nas Ideias 1 e 5).
- **Sample Applications** (`/arc/references/sample-applications`): ler antes de começar, para
  não reconstruir um exemplo oficial.
- Endereços/feed ids exatos de oráculo na testnet.

### 4.7 Nota prática de deploy

Um deploy real na testnet precisou da flag `--legacy` no Foundry:

```bash
forge script script/Deploy.s.sol:Deploy \
  --rpc-url https://rpc.testnet.arc.network --broadcast --legacy
```

Provável ausência de suporte a transação tipo-2 (EIP-1559) nesse caminho. Guarde isso — é o
tipo de detalhe que custa uma tarde.

---

## 5. Recomendação

**Depois da rodada de verificação da §4, a Ideia 2 passou a ser a primeira escolha.**

- **Ideia 2 (avaliadores ERC-8183)** — o que era o maior risco virou o maior trunfo: a
  implementação de referência já está deployada na testnet e o `createJob` aceita um `hook`,
  que é exatamente o ponto de extensão que a ideia precisava. Você constrói infraestrutura
  faltante, numa direção que a própria Arc está empurrando, plugável por terceiros sem fork.
  É o perfil de contribuição que converte em cargo de builder, porque outras pessoas passam a
  construir *em cima* do seu trabalho.
- **Ideia 3 (mandatos de débito recorrente)** — continua a melhor escolha se o objetivo é
  velocidade: 2–3 semanas, zero dependência de feature não verificada, e o problema é
  imediatamente compreensível para qualquer pessoa da Circle. Hoje a única forma de cobrança
  recorrente em crypto é `approve` infinito, o que nenhum CFO aceita.
- **Ideia 5 (forward de FX)** subiu: oráculos de FX estão documentados na rede, e derivativo
  de câmbio entre stablecoins é território praticamente vazio.

A Ideia 1 (grid/ladder de FX) é uma terceira opção legítima e é o refinamento defensável do
seu instinto original — só precisa ser vendida como market making de tesouraria, não como
bot de lucro.

Sequência sugerida: valide o §4 → escolha uma → contratos com testes Foundry primeiro →
frontend mínimo → deploy na testnet → demo em vídeo de 2 minutos → submeta.

Uma coisa que vale para **qualquer** escolha: instrumente e mostre o custo de gas em dólar
no seu dashboard. É a prova concreta de que você entendeu por que a Arc existe, e quase
nenhuma submissão vai fazer isso.
