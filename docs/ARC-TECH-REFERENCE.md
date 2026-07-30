# Arc — Referência técnica para implementação

Material suficiente para escrever código contra a Arc testnet hoje. Compilado das skills
oficiais da Circle (`circlefin/skills`), do whitepaper de privacidade da Arc, e de um deploy
público de terceiros na testnet.

## 0. Procedência e nível de confiança

Os domínios `docs.arc.io`, `docs.arc.network`, `community.arc.io` e `developers.circle.com`
estão bloqueados pela política de egresso do ambiente onde isto foi compilado. Nada aqui veio
do site da doc diretamente. Classificação:

| Nível | Significado | O que está aqui |
|---|---|---|
| **A — oficial Circle** | Copiado do repo `circlefin/skills` (material oficial da Circle) | §1, §2, §3, §4, §6, §8, §9 |
| **B — whitepaper Arc** | Whitepaper de privacidade da Arc, via busca | §5 |
| **C — terceiros / a verificar** | Repo público de terceiro, ou snippet de busca | §7, endereço ERC-8183 em §6, §10 |

Trate **C** como pista a confirmar, não como verdade. Onde há conflito entre fontes, está
marcado em §11.

---

## 1. Configuração de rede (nível A)

```
Rede           Arc Testnet
Chain ID       5042002  (0x4CEF52)
RPC            https://rpc.testnet.arc.network
WebSocket      wss://rpc.testnet.arc.network
Explorer       https://testnet.arcscan.app
Faucet         https://faucet.circle.com
Domínio CCTP   26
```

| Token | Endereço | Decimais |
|---|---|---|
| USDC | `0x3600000000000000000000000000000000000000` | 6 (view ERC-20) |
| EURC | `0x89B50855Aa3bE2F677cD6303Cec089B5F319D72a` | 6 |

`arcTestnet` **já existe no viem** — não escreva definição de chain customizada.

Arc é **testnet apenas**. Não existe mainnet para apontar.

---

## 2. A dualidade do USDC — a parte que quebra código (nível A)

Na Arc o ativo de gas nativo **é o próprio USDC**. Não é "token nativo + token USDC": é **um
único pool de fundos exposto por duas interfaces**.

| View | Decimais | Usar para |
|---|---|---|
| Nativa | **18** | somente gas e `msg.value` |
| ERC-20 (`0x3600…0000`) | **6** | saldo, transfer, approve, exibição |

Fator entre as views: `1e18` nativo = `1e6` ERC-20 (10¹²).

Regras (as três primeiras são erro comum de quem vem de outra chain):

- **Nunca somar as duas views** para mostrar saldo — isso dobra o mesmo dinheiro. Exiba um
  único saldo, o da view ERC-20 de 6 decimais.
- **USDC ↔ nativo não é swap nem conversão.** É o mesmo ativo. Detecte e rejeite essa rota
  *antes* da lógica de fee/roteamento.
- **Nunca chame `decimals()` em endereço sentinela nativo** (`0xEeee…eEEeE`, `0x0…0`) — não é
  contrato ERC-20, a chamada reverte.
- Mantenha valores na view de 6 decimais em todo lugar, exceto matemática crua de gas. Seja
  explícito sobre em qual view cada número está.

### 2.1 Consequência prática: o pre-flight check de saldo

Este é o detalhe que mais importa na implementação. Como transferência e gas saem do **mesmo
pool**, você precisa exigir `saldo >= valor + gas`, ambos em unidades nativas de 18 decimais:

```ts
const gasBalance = await publicClient.getBalance({ address: account.address });
const gasEstimate = await publicClient.estimateContractGas({
  address: USDC, abi: erc20Abi, functionName: "transfer",
  args: [recipient, parseUnits(amount, 6)], account,
});
const feeData = await publicClient.estimateFeesPerGas();
const gasCost = gasEstimate * (feeData.maxFeePerGas ?? feeData.gasPrice ?? 0n);

// Derive do metadata da chain, não de chain id hardcoded — generaliza para
// qualquer chain com gas em stablecoin.
const nativeIsUsdc = chain.nativeCurrency.symbol === "USDC";

if (nativeIsUsdc) {
  // Arc: UM pool. gasCost já vem em unidades nativas (18 dec).
  const required = parseUnits(amount, 18) + gasCost;
  if (gasBalance < required) throw new Error("Insufficient USDC (transfer + gas)");
} else {
  // Outras chains: dois saldos independentes, checar cada um.
  if (usdcBalance < parseUnits(amount, 6)) throw new Error("Insufficient USDC");
  if (gasBalance < gasCost) throw new Error("Insufficient gas");
}
```

Se você copiar o padrão de outra chain (checar USDC e gas separadamente), o app vai aceitar
transações que revertem por falta de fundos quando o valor estiver perto do saldo total.

---

## 3. Setup de cliente (nível A)

```ts
import { createPublicClient, createWalletClient, http, erc20Abi,
         parseUnits, formatUnits } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { arcTestnet } from "viem/chains";

const publicClient = createPublicClient({ chain: arcTestnet, transport: http() });

const account = privateKeyToAccount(process.env.PRIVATE_KEY as `0x${string}`);
const walletClient = createWalletClient({ account, chain: arcTestnet, transport: http() });

const USDC = "0x3600000000000000000000000000000000000000";
```

wagmi:

```ts
import { createConfig, http } from "wagmi";
import { arcTestnet } from "viem/chains";

const config = createConfig({
  chains: [arcTestnet],
  transports: { [arcTestnet.id]: http() },
});
```

Cuidado: `useBalance` do wagmi devolve a **view nativa de 18 decimais**, e o `symbol` dela é
`USDC`. Para exibir saldo, leia o ERC-20 em `0x3600…0000` com 6 decimais.

### 3.1 Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash && foundryup

forge script script/Deploy.s.sol:Deploy \
  --rpc-url https://rpc.testnet.arc.network --broadcast --legacy
```

A flag **`--legacy`** apareceu como necessária num deploy real (nível C) — provável ausência
de suporte a transação tipo-2 (EIP-1559) nesse caminho. Se `forge` falhar com erro de tipo de
transação, é isso.

Nunca passe chave privada como flag de CLI fora de teste local. Use `cast wallet import` ou
keystore cifrado.

---

## 4. Operações ERC-20 (nível A)

| Método | Tipo | Assinatura |
|---|---|---|
| `balanceOf` | read | `(owner) → uint256` (bigint cru, 6 dec) |
| `allowance` | read | `(owner, spender) → uint256` |
| `totalSupply` | read | `() → uint256` |
| `transfer` | write | `(to, amount) → bool` |
| `approve` | write | `(spender, amount) → bool` |
| `transferFrom` | write | `(from, to, amount) → bool` |

- `parseUnits(x, 6)` sempre. `parseUnits(x, 18)` num valor de USDC vira **um trilhão de
  dólares**.
- Para depositar em protocolo: `approve()` e depois o método do protocolo. **Nunca
  `transfer()` direto para um contrato** — reverte.
- Verifique `allowance` antes de aprovar: pular isso causa revert silencioso e queima gas.
- Nunca reporte sucesso antes de `waitForTransactionReceipt` e `receipt.status === "success"`.

Ler transferências recebidas:

```ts
const logs = await publicClient.getContractEvents({
  address: USDC, abi: erc20Abi, eventName: "Transfer",
  args: { to: recipient }, fromBlock: 1000000n, toBlock: "latest",
});
```

---

## 5. Privacidade opt-in (nível B)

Arquitetura: contratos falam com o backend criptográfico via **precompiles**. Backend inicial
em **TEE** (Trusted Execution Environment), com MPC/FHE/ZK plugáveis conforme amadurecem.

Fluxo de uma transação privada:

1. Cliente pede `globalPublicKey` ao **precompile APS** → encaminha ao **pEVM** → devolve a
   chave pública global **X-Wing KEM**.
2. Cliente cifra a calldata; o precompile verifica o payload e despacha os bytes do ciphertext
   via **`executePrivateTx`** para o pEVM.
3. No enclave: decapsula o bundle, decifra a calldata com **AES-256-GCM**, verifica a
   assinatura **EIP-712** da struct **`PrivateTx`** para recuperar o `msg.sender`, e executa o
   CALL confidencial.
4. **View keys** dão leitura controlada dos dados confidenciais a auditor/regulador.

### 5.1 O limite da fase 1 — leia antes de planejar em cima disso

A fase 1 entrega **transferências confidenciais: o *valor* é cifrado; os endereços de
remetente e destinatário permanecem visíveis.** Isso é proposital, para manter compatibilidade
com ferramentas de analytics e monitoramento.

Ou seja: **você não pode esconder com quem transacionou, só quanto.** Qualquer projeto cuja
tese seja "esconder a contraparte" não é viável hoje. O whitepaper também menciona estado
privado de contrato e visibilidade governada — confirme o que já está ativo na testnet antes
de depender disso.

---

## 6. ERC-8183 — mercado de trabalho de agentes

Implementação de referência **deployada na Arc testnet** em
`0x0747EEf0706327138c69792bF28Cd525089e4583` *(nível C — confirme no explorer)*.

```solidity
createJob(
    address provider,   // quem executa
    address evaluator,  // quem avalia o entregável
    uint256 expiredAt,
    string  description,
    address hook        // address(0) = fluxo padrão sem hook
)
```

Job nasce em `Open`. O cliente precisa de USDC de testnet para o escrow do budget.

Ciclo do padrão: definição do job → **escrow** dos fundos → submissão do entregável →
**atestação do avaliador** → liquidação.

O parâmetro **`hook`** é o ponto de extensão para infraestrutura de terceiros: você pluga
comportamento no ciclo de vida sem fork do padrão. Confirme na doc quais callbacks o hook
expõe e em que transições de estado são chamados.

---

## 7. Custo de gas medido (nível C)

De um deploy público de terceiros na testnet: **3.166.394 de gas ≈ 0,522 USDC**.

→ **~1,65 × 10⁻⁷ USDC por unidade de gas** (~0,165 USDC por milhão de gas).

Referências derivadas (mesma ordem de grandeza, re-meça você mesmo):

| Operação | Gas aprox. | Custo aprox. |
|---|---|---|
| `transfer` de ERC-20 | ~65k | ~0,011 USDC |
| Interação típica de contrato | ~150k | ~0,025 USDC |
| Deploy de contrato médio | ~1,5M | ~0,25 USDC |

O ponto não é ser barato — é ser **fixo em dólar e conhecido de antemão**. Estratégias com
muitas transações pequenas passam a ter custo modelável.

---

## 8. Circle Gateway — saldo USDC unificado (nível A)

Integração em nível de contrato, **sem SDK**. Depósito no Gateway Wallet numa chain, burn na
origem, mint no destino sem esperar finalidade da origem. Latência alvo <500ms.

**Testnet (todas as chains EVM):**
```
Gateway Wallet   0x0077777d7EBA4688BDeF3E311b846F25870A19B9
Gateway Minter   0x0022222ABE238Cc2C7Bb1f21003F0a260052475B
```

**Mainnet (todas as chains EVM):**
```
Gateway Wallet   0x77777777Dcc4d5A8B6E418Fd04D8997ef11000eE
Gateway Minter   0x2222222d7164433c4C09B0b0D809a9b52C04C205
```

REST de atestação:
```
testnet   https://gateway-api-testnet.circle.com/v1/
mainnet   https://gateway-api.circle.com/v1/
```

Conceitos: *burn intent*, *unified balance*, `gatewayMint`, delegates.

Alternativa: **CCTP** (domínio da Arc = `26`) via Bridge Kit, que faz approve + burn +
atestação + mint numa chamada `kit.bridge()`. Gateway exige depósito prévio mas dá UX melhor;
CCTP não exige depósito mas é mais lento.

---

## 9. Pagamentos de agente — já existe, não reconstrua (nível A)

A Circle **já entrega** a trilha de micropagamento agêntico:

- **Gateway Nanopayments** é o caminho padrão para chamadas sub-centavo, de centavos, ou de
  alta frequência. **x402** faz a negociação do `402 Payment Required`; o Gateway faz as
  autorizações USDC gasless e a liquidação em lote.
- Middleware oficial: **`@circle-fin/x402-batching`** (Express/Node).
- **Agent Marketplace**: `https://agents.circle.com/services`.

A própria skill da Circle marca como *red flag* o instinto genérico de "usar x402 `exact` na
Base por padrão" — o caminho deles é Gateway Nanopayments.

**Implicação para escolha de projeto:** "billing medido por chamada para agentes" está
ocupado pela própria Circle. Construa *em volta* (reputação, avaliação, disputa,
descoberta), não *em cima do mesmo lugar*.

---

## 10. Oráculos (nível C)

A doc tem página `/arc/tools/oracles` listando **Chainlink, Pyth e RedStone** na Arc.

- **Pyth** — cripto, ações, **FX**, metais; modelos pull e push.
- **RedStone** — cripto, LSTs/LRTs, RWAs, fundos tokenizados, **FX**; push, pull e híbrido.
- **Chainlink** — feeds agregados para lending, trading, stablecoins, ativos tokenizados.

Existe caminho para **EUR/USD**. Falta confirmar feed id / endereço na testnet. Pyth em modo
*pull* é a aposta mais provável de funcionar sem deploy dedicado por chain.

---

## 11. Conflitos e lacunas conhecidas

**Conflito de explorer.** A skill `use-arc` diz `https://testnet.arcscan.app`. O guia
`use-usdc/references/evm.md` usa `https://explorer.arc-testnet.io/...` nos exemplos. Ambos são
nível A e se contradizem — confirme qual responde antes de gerar links na sua UI.

**Não verificado:**
- Endereço do ERC-8183 e a interface esperada do `hook`.
- Feed ids de oráculo na testnet.
- O que da camada de privacidade está de fato ativo na testnet, e sua ABI/superfície exata.
- Qual paymaster está utilizável hoje (paymasters *enshrined* para EURC e outras stablecoins
  como gas aparecem como roadmap).
- Se StableFX tem leitura de cotação sem KYB. **StableFX é permissionado** — exige KYB/AML
  aprovado, com execução por RFQ. Não construa em cima dele contando com acesso permissionless.
- Lista de *Sample Applications* oficiais (`/arc/references/sample-applications`) — ler antes
  de começar, para não reconstruir um exemplo que já existe.

---

## 12. Fontes

- `circlefin/skills` → `plugins/circle/skills/{use-arc,use-usdc,use-gateway,accept-agent-payments}`
- Whitepaper de privacidade da Arc (`arc.io/privacy-whitepaper`, e o PDF "Arc Privacy Sector")
- `kenhuangus/arc-ai-agents` → `ARC_TESTNET_DEPLOYMENT_COMPLETE.md` (gas medido, flag `--legacy`)
- Circle, "Introducing Arc"; The Block sobre StableFX; material público sobre ERC-8183
