# Base agent starter (TypeScript)

A template for an onchain agent on Base, written in TypeScript. Press **Use this template** at the top of this page to get your own copy with a clean history.

One pass of the agent does four things:

1. **Checks its own dependencies with Sato Hub Preflight before installing them.**
2. **Reads market data from Base**: block, gas price and the Chainlink ETH/USD feed, over a public RPC.
3. **Runs every intent past a spending policy** before acting. Allowed actions are `read` and `quote`. Tokens are limited to USDC and WETH on Base, and a quote is capped at 25 USD.
4. **Asks for a swap quote** (USDC → WETH) and reports it: the venue, the quoted output, the implied price against the feed, the response signature and the fee disclosure word for word.

No API key, no account, no wallet, no secrets. It runs as-is in a clean container, and the CI in this repo runs it on every push.

## Setup

Node 20 or newer. One command:

```sh
npm run setup && npm start
```

`npm run setup` runs Preflight on `package.json` (a zero-dependency script, `scripts/preflight-deps.mjs`) and then `npm install`. It stops if a dependency comes back `no`. `npm start` runs one pass: policy → market → policy checks → quote → report. `npm run typecheck` checks the types.

To make `caution` block the install as well: `node scripts/preflight-deps.mjs --fail-on caution`.

| Variable | Default | |
|---|---|---|
| `BASE_RPC_URL` | `https://mainnet.base.org` | any Base mainnet RPC |
| `AMOUNT_USDC` | `10` | quote size. Above 25 the policy refuses it (`P4-size`) |
| `SATO_USER_AGENT` | `base-ts-agent-starter/0.1` | please set your own |

## What it does NOT do

- **It holds no private key.** The wallet in `src/policy.ts` has no `sign` or `broadcast` method. `sign` is refused by the policy and cannot happen in code either.
- **It never builds, signs or sends a transaction.** The quote request sends no `taker`, so Sato Hub builds no transaction and returns `tx: null`. The code stops if one ever comes back.
- **The quote is unsigned.** It is a price and a venue, not a transaction and not a fill. Prices move between the quote and any trade you make later.
- **It does not trade and does not move funds.**
- **It has not been audited.** This is a starting point, not reviewed code. Read it before you build on it.
- **Preflight is not a security review.** It reports what Sato Hub has on record about a package and when it was checked. `unknown` means no record. That is not a pass and not a fail; general npm packages usually come back `unknown`, because Sato Hub lists agent tooling.
- **The pre-install check does not verify the response signature.** The verifier, `satohub-core`, is one of the packages being checked. The swap quote is verified after install.
- **The policy runs in this process.** Anyone who can edit the process can edit the policy. That is fine for a starter that cannot sign. It is not enough for a wallet that can.

## The fee, stated plainly

The quote comes from Sato Route (`GET https://satohub.ai/api/route/swap`, through `satohub-core`). If you later build and sign a transaction from a Sato Route route, Sato Route takes a fee inside that transaction: 15 bps when either side is a volatile token (as with USDC → WETH), 3 bps stable-to-stable, 25 bps cross-chain. The quoted output already has the fee deducted. This starter never builds that transaction, so running it costs nothing. The full disclosure is printed word for word on every run. If you want a quote with no Sato fee, call a venue's own quote API instead.

## From starter to agent

The parts you would change, in order:

- **Signing.** Swap `PolicyWallet` for a signer that enforces its policy on the server that holds the key, not in this process. Ask Sato Hub's `recommend_stack` for the wallet slot of your own goal, and run Preflight on whatever you pick. A pick is a record of what is open, active and verifiable, not an endorsement.
- **Execution.** Sending a `taker` to Sato Route returns an unsigned transaction. Simulate it, sign it with your signer, and check the fee disclosure again before you sign.
- **A loop.** `src/agent.ts` runs one pass. An agent runs it on a schedule and keeps state between passes.
- **CI.** `.github/workflows/ci.yml` runs setup, typecheck and one pass with no secrets. Set `SATO_USER_AGENT` there to your project's name.

## Sato Hub in your coding agent

The same data this starter reads is available to your coding agent over MCP: search the directory, get a stack for a goal, get install steps, run Preflight.

- Install in your client: https://satohub.ai/install?utm_source=github&utm_medium=template&utm_campaign=starter
- The MCP server and its tools: https://satohub.ai/mcp?utm_source=github&utm_medium=template&utm_campaign=starter (endpoint `https://satohub.ai/api/mcp`)

## Files

| | |
|---|---|
| `scripts/preflight-deps.mjs` | Preflight on `package.json` before install. Zero dependencies |
| `src/policy.ts` | the policy, `evaluate()`, and the key-less `PolicyWallet` |
| `src/market.ts` | read-only Base reads with viem |
| `src/agent.ts` | one pass, start to finish |
| `.github/workflows/ci.yml` | setup + typecheck + start on Node 22, no secrets |

This template is generated from `examples/base-ts-agent-starter` in [sato-hub-integrations](https://github.com/satohubai/sato-hub-integrations).

MIT — see [LICENSE](LICENSE).
