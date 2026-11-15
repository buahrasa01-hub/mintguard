# MintGuard — AI Safety Verifier for Agent Free Mints

> Paste a BSC contract address. MintGuard reads the chain **live**, simulates
> the free mint from your wallet, and explains the risks in plain language —
> **before** anyone signs anything. No wallet connection. No key. Nothing is
> executed.

Built for [SLAY HACKS 2026](https://slayy-hacks.devpost.com/) — "Build Bold.
Think Beyond. Create with AI."

**Live demo:** https://buahrasa01-hub.github.io/mintguard/

## The problem

AI agents (for example Binance Agentic Wallet holders) claim free NFT and token
mints on BNB Chain every day. Scams hide inside the contracts themselves:

- a "free" mint that quietly charges a fee on the second claim,
- a free pool that is already empty while the site still advertises it,
- art and metadata that can be re-skinned after the mint,
- a fake Agent-ID gate that points at a registry the project controls,
- an owner who can pause, change prices, or mint the rest of the supply.

An agent cannot read Solidity. A human cannot re-audit every contract by hand
before a five-second mint window closes.

## The solution

MintGuard is a single-file web app. It runs **12 on-chain checks directly from
the browser** — no backend, no key, no wallet connection:

| # | Check | How |
|---|-------|-----|
| 1 | Contract exists | `eth_getCode` — an EOA has no code, nothing to mint |
| 2 | Source verified | Sourcify v2 — exact match means the code you read is the code that runs |
| 3 | Identity | `name()` / `symbol()` / `ownerOf(1)` — NFT or fungible |
| 4 | Metadata frozen | `tokenURI(1)` — on-chain JSON or IPFS, not a re-skinnable server URL |
| 5 | Resale royalty | `royaltyInfo` — ≤ 10% keeps resale viable |
| 6 | Quota | `totalMinted()` / `MAX_SUPPLY()` / `freeRemaining()` — sold out or free pool gone |
| 7 | Mint window | `mintOpen()` |
| 8 | Free price | `priceFor(wallet, 1)` — the price **for your wallet**, not the advertised one |
| 9 | Agent-ID gate | `hasAgentId(wallet)` + `AGENT_ID_REGISTRY()` vs the official Binance registry |
| 10 | Owner | `owner()` — renounced, live admin, or ownerless |
| 11 | Pause | `paused()` |
| 12 | Simulation | `mint(1)` value 0 as a read-only `eth_call` — the real revert reason, with zero gas spent |

The checks feed a **0–100 score** (contract 6 · source 14 · identity 3 ·
metadata 8 · royalty 4 · quota 10 · window 5 · free price 13 · Agent-ID gate 5 ·
registry 5 · owner 7 · not paused 5 · simulation 15) and a verdict:
**GO (85+) · PASS WITH NOTES (70–84) · REVIEW (55–69) · AVOID (<55)**.
A failed simulation or an empty free pool caps the score no matter what else
looks good.

A free keyless model (Pollinations text API) then explains the findings in
plain language. The model only ever explains the on-chain rows above — every
fact comes from the RPC, never from the model. If the model is offline, a
rule-based fallback text is shown instead and the evidence is unaffected.

Two honest edge cases are built in:

- **Already claimed.** If your wallet already used its free mint, the price
  for the next mint is paid and a fresh `mint(1)` simulation reverts. MintGuard
  reads `mintedBy(wallet)` and reports it as proof instead — *your free mint
  worked, a second mint costs X* — instead of crying scam.
- **Flaky networks.** RPC calls retry across four public BSC endpoints, and
  the Sourcify check retries too. A Sourcify outage is reported as
  *unproven* (cap 84), never as *unverified*.

## Try it (30 seconds)

1. Open the [live demo](https://buahrasa01-hub.github.io/mintguard/).
2. Tap **Zephyra ✓** — a real free mint, verified live: 900 / 3,333 minted,
   source exact match, Agent-ID gate passing. Score **94/100 — GO**.
   (The demo wallet already claimed its free Zephyra, so the app proves it
   from the holding and tells you the second mint costs 0.0003 BNB.)
3. Tap **NOVRA ✗** — free pool exhausted (`freeRemaining = 0`), simulation
   reverts with *not enough BNB*. Score **29/100 — AVOID**.
4. Paste any BSC contract address and tap **Verify this mint**.

No wallet connection is ever requested. The wallet field is only used as the
`from` address of read-only calls.

## Stack

- **Single HTML file** (`index.html`, ~26 KB) — HTML + CSS + vanilla JS, no
  build step, no dependencies, no images.
- **BSC public RPC** (`bsc-dataseed1.binance.org`, `bsc-dataseed.binance.org`,
  `1rpc.io/bnb`, `bsc-rpc.publicnode.com`) — `eth_getCode`, `eth_blockNumber`,
  `eth_call`.
- **Sourcify v2** — source verification, no key.
- **Pollinations text API** — keyless, explanation layer only.
- **GitHub Pages** — static hosting.

## Run locally

```sh
python3 -m http.server 8000
# open http://127.0.0.1:8000/
```

Open the file directly (`file://`) also works, but some browsers restrict
`fetch()` from `file://` pages — serve it over HTTP if the checks stay empty.

## Known limits

- The app reads standard getters (`totalMinted`, `MAX_SUPPLY`, `priceFor`,
  `mintOpen`, …). Contracts with custom function names report those checks as
  *unproven* rather than guessing.
- Public RPCs and Sourcify can be slow or briefly unreachable; the app retries,
  but a bad network day means waiting a few seconds, not a wrong verdict.
- The AI explanation can be wrong — it is a paraphrase of the rows above.
  The rows are the source of truth.

## License

MIT. Reference only, not investment advice.
