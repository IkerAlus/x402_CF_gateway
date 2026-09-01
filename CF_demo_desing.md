# Monetization Gateway × NEAR Intents 1Click — Demo Design

**Status:** Draft for review · **Date:** 2026-09-01 · **Audience:** integration partner engineering, NEAR Intents engineering

A lightweight demo showing how a Cloudflare's x402 monetization gateway can offer sellers **automatic settlement on any of 35+ chains and 180+ tokens** while buyers keep paying stablecoins over standard x402 on any [Coinbase CDP facilitator network](https://docs.cdp.coinbase.com/x402/seller/facilitator) — with **zero changes to the x402 protocol, the facilitator, or buyer-side clients**.

The mechanism: the x402 `payTo` address is set to a [1Click Swap API](https://docs.near-intents.org/) (1CS) deposit address from a live quote. The facilitator's on-chain settlement transaction *is* the swap deposit. NEAR Intents then delivers the seller's configured token on the seller's configured chain, asynchronously, with no custody hop in between.

---

## Background

**The monetization gateway** gates web pages, datasets, APIs, and MCP tools behind usage-based micropayments at the edge — the partner's proxy layer that terminates buyer requests in front of the seller's origin. Throughout this doc, "the edge" means that component: the monetization gateway itself, which in x402 terms plays the resource-server role. Payments use **x402**: the buyer receives `402 Payment Required` with machine-readable `PaymentRequirements`, signs a stablecoin payment (EIP-3009 / Permit2 on EVM, SPL on Solana), and retries; a **facilitator** (Coinbase CDP) verifies and settles on-chain, and the edge returns the resource. Today the seller accumulates stablecoins on the payment network and redeems them out-of-band.

**NEAR Intents 1Click Swap API** (`https://1click.chaindefuser.com`) abstracts cross-chain swaps into a REST flow: request a quote, receive a unique `depositAddress`, send funds to it, and the swap executes and delivers to the recipient on the destination chain. It currently spans **35 blockchains** (EVM chains, Solana, Bitcoin, NEAR, TON, Tron, Stellar, XRP, Sui, Aptos, Cardano, …) and ~188 tokens.

The demo composes the two: the partner keeps its exact x402 flow on the buyer side; 1CS turns the settled funds into whatever the seller wants, wherever the seller wants it.

## Design at a glance

| Decision | Choice | Why |
|---|---|---|
| Where 1CS plugs in | `payTo` = 1CS `depositAddress` in `PaymentRequirements` | Facilitator settlement doubles as swap deposit; no second transfer, no custody by the partner |
| Quote mode | `EXACT_OUTPUT`, wet quote (`dry: false`), one per payment request per advertised network | Seller price is fixed in seller terms; buyer-side `amount` = quote `amountIn` covers swap cost |
| Buyer assets | Stablecoins only (USDC primary) on facilitator networks | Matches facilitator support and keeps spread negligible (stable→stable) |
| Seller settlement | Chain + token + address from static gateway config | Demo simplicity; production moves this to per-seller rules |
| Buyer completion | Facilitator `settle` response → `200 OK`, unchanged | Buyer latency identical to stock x402; the gateway never monitors the payment chain |
| Seller completion | Asynchronous; gateway tracks 1CS status to `SUCCESS` | Seller leg finishes seconds-to-minutes later depending on origin-chain finality |
| Failed swaps | 1CS `refundTo` = partner-controlled address per network family | Buyer is unknown at quote time; refunds become an ops/reconciliation flow |
| Scheme | `exact` only | `upto` (variable amount) and `batch-settlement` don't fit a fixed per-request quote yet |

The trust model on the buyer side is unchanged (facilitator can only settle the signed amount to the stated `payTo`). What changes is the seller side: delivery of the destination-chain payout relies on NEAR Intents executing the swap — the same trust assumption any 1CS integrator makes, now made by the seller instead of the buyer.

## Architecture

```mermaid
flowchart LR
    subgraph buyer_side ["Buyer side (unchanged x402)"]
        B["Buyer / agent<br/>x402 client + wallet"]
    end

    subgraph pe ["Partner edge — one deployable unit"]
        MG["Monetization gateway<br/>(402 issuance, verify/settle calls,<br/>resource release)"]
        GW["1CS settlement module (new)<br/>quote builder · settlement tracker<br/>seller config · refund ledger"]
    end

    subgraph ext ["External services"]
        FAC["Coinbase CDP Facilitator<br/>verify + settle"]
        ONE["1Click Swap API<br/>quote · deposit/submit · status"]
    end

    ORIG["Origin chain<br/>Base / Polygon / Arbitrum / Solana"]
    DEST["Seller chain + token<br/>any of 35+ chains"]

    B -- "1: GET resource / retry with payment" --> MG
    MG -- "2: buildRequirements" --> GW
    GW -- "3: POST /v0/quote (EXACT_OUTPUT, wet)" --> ONE
    MG -- "4: verify / settle" --> FAC
    FAC -- "5: transfer buyer → depositAddress" --> ORIG
    MG -- "6: recordSettlement {txHash}" --> GW
    GW -- "7: deposit/submit + poll status" --> ONE
    ONE -- "8: swap + withdrawal" --> DEST
```

Money flows exactly once on the buyer side (buyer → deposit address, edge 5) and once on the seller side (NEAR Intents → seller address, edge 8). The settlement module touches control flow only — it never holds funds.

## Buyer / seller flow

### Happy path

```mermaid
sequenceDiagram
    autonumber
    participant B as Buyer (x402 client)
    participant MG as Partner Monetization Gateway
    participant GW as 1CS settlement module
    participant OC as 1Click Swap API
    participant F as CDP Facilitator

    B->>MG: GET /resource
    MG->>GW: buildRequirements(resource, price) — internal call
    loop per advertised network
        GW->>OC: POST /v0/quote {dry:false, swapType:EXACT_OUTPUT,<br/>amount:price, destinationAsset:sellerToken,<br/>recipient:sellerAddr, refundTo:partnerRefundAddr, deadline}
        OC-->>GW: {depositAddress, amountIn, amountOut, deadline}
    end
    GW-->>MG: accepts[] (one PaymentRequirements per network,<br/>payTo = depositAddress)
    MG-->>B: 402 Payment Required + PAYMENT-REQUIRED header
    B->>B: pick network, sign EIP-3009 / SPL payment<br/>(amount = amountIn, to = depositAddress)
    B->>MG: retry GET /resource + PAYMENT-SIGNATURE header
    MG->>F: POST /verify
    F-->>MG: valid
    MG->>F: POST /settle
    F->>F: broadcast transfer buyer → depositAddress, confirm
    F-->>MG: {success, txHash, network}
    MG-->>B: 200 OK + resource  «buyer done, seconds after retry»
    MG->>GW: recordSettlement(depositAddress, txHash) — internal call
    GW->>OC: POST /v0/deposit/submit {depositAddress, txHash}
    loop until terminal status
        GW->>OC: GET /v0/status/{depositAddress}
    end
    OC-->>GW: SUCCESS «seller paid on their chain in their token»
    GW-->>MG: terminal status in ledger (seller monitoring endpoint)
```

Narrative, in the style of the x402 spec flow:

1. **Buyer → edge**: request a gated resource.
2. **Edge → settlement module**: build `PaymentRequirements` for this request — an internal call. The module holds the seller's settlement config (chain, token, address); the price in seller terms comes from the gateway's pricing rules.
3. **Module → 1CS**: one wet `EXACT_OUTPUT` quote per advertised buyer network. `amount` = resource price in the seller's token; 1CS returns `amountIn` (what the buyer must pay, price + swap cost) and a unique `depositAddress` valid until `deadline`.
4. **Edge → buyer**: `402` with one `accepts[]` entry per network. Per entry: `scheme: "exact"`, `network` (CAIP-2), `asset` = origin-chain USDC contract, `amount` = `amountIn`, `payTo` = `depositAddress`, `maxTimeoutSeconds` well inside the quote deadline. Quote metadata (`amountOut`, seller chain/token, quote deadline, correlation id) rides in `extra`.
5. **Buyer**: standard x402 client behavior — choose an entry, sign, retry. Nothing 1CS-specific.
6. **Edge → facilitator**: `verify`, then `settle`. The facilitator broadcasts the buyer's transfer **to the deposit address** and confirms it. This is stock CDP facilitator behavior; it neither knows nor cares that `payTo` is a swap deposit.
7. **Edge → buyer**: `200 OK` with the resource as soon as `settle` succeeds. Buyer-perceived latency is identical to stock x402. The settlement module does not monitor the payment chain at all.
8. **Edge → settlement module**: hand off the settle result (`depositAddress`, `txHash`) — internal, fire-and-forget.
9. **Module → 1CS**: `deposit/submit` with the txHash (accelerates detection; 1CS would also detect the deposit on its own), then poll `status/{depositAddress}`.
10. **1CS**: executes the swap and withdraws to the seller's address on the seller's chain. Status reaches `SUCCESS` — the seller leg is complete, typically seconds to a few minutes after step 7 depending on origin-chain finality.

An example `accepts[]` entry (buyer pays USDC on Base; seller configured for USDC on NEAR, price $0.10):

```json
{
  "scheme": "exact",
  "network": "eip155:8453",
  "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "amount": "100650",
  "payTo": "0x<1cs-deposit-address>",
  "maxTimeoutSeconds": 300,
  "extra": {
    "settlement": "near-intents-1click",
    "amountOut": "100000",
    "destinationAsset": "nep141:17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1",
    "destinationChain": "near",
    "quoteDeadline": "2026-09-01T12:34:56Z",
    "correlationId": "…"
  }
}
```

### Failure paths

| Failure | What happens | Who is affected |
|---|---|---|
| Buyer never pays | Quote expires at `deadline`; deposit address goes inactive. No funds moved. | Nobody. Gateway ledger entry expires. |
| `verify`/`settle` fails | Stock x402 behavior: edge returns 402/error, buyer can retry (fresh 402 → fresh quote). | Buyer retries. No funds moved on failed settle. |
| Settlement lands after quote `deadline` | 1CS refunds the deposit to `refundTo` (partner-controlled). **Buyer has paid and received the resource; seller is not paid.** | Ops: reconcile and pay out the seller from the refund wallet. Mitigated by `maxTimeoutSeconds` ≪ deadline and an edge-side freshness check before `settle`. |
| Swap fails / route unavailable mid-flight | 1CS status `REFUNDED` or `FAILED`; funds to `refundTo`. Same asymmetry as above. | Ops reconciliation. Rare for stable→stable routes. |
| Duplicate payment to same deposit address | Surplus over `amountIn` is refunded to `refundTo`. | Ops. Edge should never reuse an `accepts[]` entry across payments. |
| 1CS quote API unreachable at 402 time | No `PaymentRequirements` → gateway fails open or closed per policy (demo: fail closed with 503). | Buyer sees an error; no funds at risk. |
| Settlement tracker down at handoff time | 1CS still detects the deposit on-chain by itself; swap completes. The tracker back-fills status by polling on restart. | Seller reporting delayed only. |

The one structurally new risk is the third row: buyer-paid-but-seller-unpaid, with funds parked in the refund wallet. It is bounded (per-request amounts), detectable (every such case is a `REFUNDED`/expired ledger entry), and rare (facilitator settles in seconds; deadlines are minutes) — but it is the item to design away on the path to production.

## Network and asset coverage

Buyer side — CDP facilitator networks (x402 v2), checked against the live 1CS token list on 2026-09-01:

| Facilitator network | CAIP-2 | Facilitator schemes | Demo buyer asset | 1CS origin support |
|---|---|---|---|---|
| Base | `eip155:8453` | exact, upto, batch-settlement | USDC `0x8335…2913` | ✅ `nep141:base-0x8335….omft.near` |
| Polygon | `eip155:137` | exact, upto, batch-settlement | USDC `0x3c49…3359` | ✅ (USDT also available) |
| Arbitrum | `eip155:42161` | exact, upto, batch-settlement | USDC `0xaf88…5831` | ✅ (USDT0 also available) |
| World | `eip155:480` | exact, upto, batch-settlement | USDC | ❌ **World Chain not yet supported by 1CS** |
| Solana | `solana:5eykt4UsFv8P…` | exact | USDC `EPjFW…Dt1v` | ✅ (USDT, USD1 also available) |
| Base Sepolia / World Sepolia / Solana Devnet | — | — | — | ❌ 1CS is mainnet-only |

Demo coverage: **4 of 5 facilitator mainnets**, USDC as the advertised buyer asset (USDT variants can be enabled per network by config). World Chain is a 1CS roadmap item (see [Open points](#open-points-and-weak-points)); testnets are unusable end-to-end because NEAR Intents has no testnet, so the demo runs on mainnet with cent-sized prices.

Seller side: any of the 35 chains / ~188 assets on `GET /v0/tokens` — e.g. USDC on NEAR (`nep141:17208628…36133a1`), USDT on Tron (`nep141:tron-d28a…omft.near`), USDC on Stellar, USDT on TON, or any facilitator network itself.

## The 1CS settlement component — spec

A single small TypeScript/Node component (1CS calls via the official `@defuse-protocol/one-click-sdk-typescript` SDK). It does two things — quote issuance and post-settlement tracking — and contains **no verify/settle logic, no payment-chain RPC, and no keys**; payment verification and on-chain settlement stay with the CDP facilitator. Its only secret is the 1CS API JWT (Near One-provisioned; unauthenticated 1CS calls carry a 0.2% fee).

It is **internal logic of the monetization gateway**: same operator, deployed as one unit — in the demo, one process configured for one seller; in production, either one instance per merchant/seller or a shared config-driven deployment (per-seller settings are pure config). The two integration points are in-process function calls — or the same calls against a co-deployed private sidecar bound to localhost where the edge runtime can't embed a Node module. The only exposed surface is the seller-monitoring endpoint.

### Internal interfaces and internal logic

| Interface | Purpose |
|---|---|
| `buildRequirements(resource, priceOut)` | Invoked by the 402-construction path. The seller is the instance's config (a shared multi-tenant deployment adds a seller id). Runs one wet `EXACT_OUTPUT` 1CS quote per advertised network (in parallel) and returns the x402 `accepts[]` array. Latency ≈ one quote round-trip, target < 1 s. |
| `recordSettlement(depositAddress, txHash, network)` | Invoked right after a successful facilitator `settle`. Idempotent (keyed on `depositAddress`, insert-only). Hands off to the settlement tracker. |

**Quote construction** — for each advertised network, `POST /v0/quote` with:

| 1CS parameter | Value |
|---|---|
| `dry` | `false` (wet — must mint a real deposit address) |
| `swapType` | `EXACT_OUTPUT` |
| `amount` | resource price in the seller token's smallest unit (`amountOut` is what the seller receives, exactly) |
| `originAsset` | the network's configured stablecoin, e.g. `nep141:base-0x8335….omft.near` |
| `destinationAsset` | seller config, e.g. NEAR USDC |
| `depositType` / `recipientType` | `ORIGIN_CHAIN` / `DESTINATION_CHAIN` (or `INTENTS` if the seller wants a NEAR Intents balance) |
| `recipient` | seller's settlement address (config) |
| `refundTo` / `refundType` | partner-controlled refund address for the network's address family / `ORIGIN_CHAIN` |
| `deadline` | now + `QUOTE_TTL` (demo default 10 min) |
| `referral` | optional partner referral tag — 1CS supports app-fee sharing per referral |

Response → x402 mapping: `amountIn` → `amount`, origin token contract → `asset`, `depositAddress` → `payTo`, `maxTimeoutSeconds` = `min(QUOTE_TTL − buffer, 300 s)`; everything else → `extra`. The edge must treat 402 bodies as uncacheable — every payment request needs fresh deposit addresses.

**Ledger** — one insert-only table keyed by `depositAddress`, opened at quote time: quote snapshot → payment txHash → 1CS terminal status (`SUCCESS` / `REFUNDED` / `FAILED` / expired). Demo persistence: JSON file snapshot (the source of truth for any individual swap remains the 1CS status endpoint, so the ledger is reconstructible). Entries whose quotes expire unpaid are pruned; `REFUNDED`/`FAILED` entries are never pruned — they are the ops reconciliation queue.

**Settlement tracker** — a long-lived background task. On handoff, calls `deposit/submit` best-effort (a failure is non-fatal — 1CS detects the deposit on-chain regardless), then polls `GET /v0/status/{depositAddress}` every ~5 s with a per-call timeout until a terminal status. An entry still non-terminal after 30 minutes is flagged `STALE` for ops and drops to low-frequency polling. On restart, the tracker re-polls every open ledger entry, so no swap is lost to a crash.

### Exposed interface / API

`GET /v1/settlements/{depositAddress}` — the component's only public endpoint, for seller dashboards and ops tooling. Returns the correlated view of one payment: quote parameters, origin-chain payment `txHash`, live 1CS status, and a refund flag.

### Configuration

```jsonc
{
  "seller": {                       // demo: single static seller; prod: per-seller rules from the partner
    "destinationAsset": "nep141:17208628…36133a1",   // USDC on NEAR
    "recipient": "seller.near",
    "recipientType": "DESTINATION_CHAIN"
  },
  "advertisedNetworks": ["eip155:8453", "eip155:137", "eip155:42161", "solana:5eykt…"],
  "buyerAssets": { "eip155:8453": "USDC", "...": "USDC" },   // stablecoins only
  "refundAddresses": { "evm": "0x<partner-ops>", "solana": "<partner-ops>" },
  "quoteTtlSeconds": 600,
  "oneClick": { "jwt": "…", "baseUrl": "https://1click.chaindefuser.com" }
}
```

### Non-goals for the demo

- No `verify`/`settle` implementation, no payment-chain RPC, no wallets in the service.
- No refund automation (refund ledger + ops runbook only).
- No `upto` or `batch-settlement` schemes; `exact` only.
- No multi-seller tenancy, dashboards, or fiat off-ramp — 1CS `SUCCESS` on the seller's chain is the endpoint of the demo.
- No World Chain, no testnets (upstream gaps, not gateway gaps).

## Demo scope and components

Since the monetization gateway itself is partner-internal, the demo stands in for the edge with a minimal resource server wired exactly the way the real edge would be:

1. **Monetization-gateway demo app** — one process: a paywalled `GET /article` behind stock x402 middleware pointed at the CDP facilitator for verify/settle, with the 1CS settlement module (specced above) embedded at the two integration points — 402 construction and post-settle handoff — plus the exposed monitoring endpoint. The x402/facilitator wiring is ~200 lines; its only purpose is to mark exactly what the partner's edge would implement.
2. **Buyer clients** — scripts using the standard x402 client SDKs, one EVM wallet (Base/Polygon/Arbitrum USDC) and one Solana wallet, funded with a few dollars.
3. **Demo script** — one run per origin network: buyer pays USDC on Base → seller receives USDC on NEAR (and one flashier route, e.g. Solana USDC → Tron USDT), showing the buyer's sub-5-second 200 OK and the seller's on-chain payout minutes later, plus one forced-failure run showing the refund ledger.

Everything runs on mainnet with cent-sized prices (the CDP facilitator's free tier of 1,000 settlements/month more than covers a demo).

## Estimated effort

Single engineer, assuming existing 1CS partner JWT and CDP API keys:

| Work item | Estimate |
|---|---|
| 1CS settlement module: requirements builder, multi-network quote fan-out (official 1CS TypeScript SDK), settlement tracker, ledger, config, monitoring endpoint | 5–6 days |
| Demo app around it: x402 middleware + CDP facilitator wiring | 2 days |
| Buyer clients (EVM + Solana) and wallet funding | 1–2 days |
| Mainnet end-to-end runs, TTL/timeout calibration, forced-failure path, demo script + README | 2 days |
| **Total** | **≈ 2 engineer-weeks** |

Optional polish: a one-page status UI over `GET /v1/settlements` (+2–3 days).

## Open points and weak points

1. **Expiry race (buyer-paid, seller-unpaid).** If facilitator settlement confirms after the quote deadline, 1CS refunds to the partner wallet while the buyer already has the resource. Bounded and rare, but the design's one real asymmetry. Demo mitigations: `maxTimeoutSeconds` ≪ quote TTL, edge freshness check before `settle`, forced re-quote (fresh 402) near expiry. Production needs a defined reconciliation flow — e.g. automatic seller payout from the refund wallet.
2. **N wet quotes per 402.** Every payment request mints one deposit address per advertised network (4 today), most never used. Fine at demo scale; at production scale this is quote/deposit-address load on 1CS and a DoS surface (unauthenticated 402s are free to trigger). Mitigations: cap advertised networks per seller, partner-side bot mitigation in front, and a 1CS-side ask for a bulk/lightweight quote path.
3. **World Chain gap.** Facilitator supports it; 1CS doesn't. Either exclude World (demo choice) or NEAR Intents prioritizes adding it — a concrete coverage ask this project surfaces.
4. **No testnet story.** NEAR Intents is mainnet-only, so CDP sandbox networks can't demo the full loop. Mainnet-with-cents is acceptable for a demo but complicates CI-style integration testing for the partner.
5. **Buyer pays the spread.** `amount` (buyer) = `amountOut` (seller) + swap cost. Stable→stable this is small (tens of bps), but the 402 shows a number slightly above the seller's list price. Disclosure via `extra.amountOut`; whether the partner wants to absorb or surface it is a product question.
6. **Seller-leg trust and finality.** The seller's payout depends on NEAR Intents execution and is asynchronous. Sellers used to "settled = money on my address" get "settled = swap in flight" for a few minutes. Reporting must distinguish facilitator-settled from 1CS-`SUCCESS`.
7. **Status transport.** 1CS status is poll-only today; the tracker polls per open swap. Fine at demo volume; a webhook/push channel is a 1CS product ask for production scale.
8. **`extra` field conventions.** The metadata shape used here (`settlement: "near-intents-1click"`, `amountOut`, …) is ad hoc. Worth formalizing as an x402 extension profile through the x402 Foundation so clients/tooling can render it.

## Path to production

Phased, each phase independently shippable:

**Phase 1 — Hardened single-tenant service** (from the demo)
- Durable store (SQL) for the settlement ledger; idempotent notification handling; restart-safe poller back-fill.
- Real observability: metrics (quote latency, settle→SUCCESS lag, refund rate), alerting on `REFUNDED`/`FAILED`/expiry races, reconciliation export.
- Security hardening: 1CS JWT rotation, quote-rate limits per seller; in sidecar packaging, localhost binding / internal service auth for the module's interfaces.
- Runbooks: refund-wallet reconciliation, 1CS incident handling, stuck-swap escalation.

**Phase 2 — Product integration with the partner**
- Move seller settlement config into the monetization gateway's rules (per-seller chain/token/address, advertised networks, price), delivered to the gateway per request or via config sync.
- Decide deployment granularity: per-merchant gateway instances co-deployed with the resource server vs one shared config-driven service for all sellers. The service is small and stateless enough for either; per-seller settings are pure config.
- Wire the settle notification as a first-class edge hook; define fail-open/closed policy when the gateway is unreachable.
- Fee model: partner referral tag on quotes for revenue share; decide who carries the swap spread.

**Phase 3 — Close the structural gaps**
- Expiry-race elimination: verify-time quote freshness contract with the edge, possibly verify-time re-quoting.
- 1CS platform asks, driven by volume: World Chain support, bulk/cheap quote endpoint, status webhooks, longer/configurable deposit deadlines.
- Scheme expansion: `upto` (needs a quote model for variable amounts — e.g. quote at settle time instead of 402 time), `batch-settlement` compatibility confirmation with Coinbase.
- Formalize the `extra` profile as an x402 Foundation extension spec.

**Phase 4 — Scale-out**
- Multi-region gateway, deposit-address pre-warming if quote latency at the edge matters, seller dashboard, EURC and additional stablecoins, additional facilitator networks as CDP adds them.

## Questions for the partner

1. **Integration surface.** Can the monetization gateway construct `PaymentRequirements` dynamically per request (fresh `payTo`/`amount` from a 1CS quote via the embedded module), rather than from static rule config? What is the latency budget for 402 construction at the edge?
2. **Settle hook.** Can the 402/settle path invoke the settlement module with the facilitator settle result (`network`, `txHash`, `payTo`) immediately after settlement? (Without it the flow still works via 1CS's own on-chain deposit detection, just seconds slower on the seller leg.)
3. **Caching.** Confirm 402 responses on monetized routes are never cached — every response carries single-use deposit addresses.
4. **Refund custody.** Is the partner comfortable operating per-network refund wallets and the reconciliation flow?
5. **Seller config.** Where should seller settlement preferences live (dashboard rules?), and how are they delivered to the settlement service — per-request or synced?
6. **Deployment granularity and packaging.** The settlement component runs inside the monetization gateway, with Near One supplying the software and 1CS credentials. One instance per merchant/seller, or a single shared config-driven deployment? Embedded in-process (Node), or a co-deployed private sidecar where the edge runtime can't host it? (Same code either way; this determines config delivery and SLA design.)
7. **Network and asset priorities.** How important is World Chain for launch (it gates a 1CS roadmap item)? USDC-only on the buyer side, or USDT variants too?
8. **Fees.** Should the partner take a revenue share via the 1CS referral mechanism? Who absorbs the buyer-side swap spread — surfaced to the buyer, or netted from the seller?
9. **Volume expectations.** Rough 402-issuance and paid-conversion volumes, to size quote-generation rate limits and the 1CS capacity ask.
10. **Scheme roadmap.** Timeline for `upto` and `batch-settlement` on monetized routes, so quote-model work can be sequenced.

## References

- Monetization-gateway announcement (example x402 gateway product) — https://blog.cloudflare.com/monetization-gateway/
- Coinbase CDP x402 facilitator (networks, schemes) — https://docs.cdp.coinbase.com/x402/seller/facilitator
- x402 specification — https://github.com/coinbase/x402/blob/main/specs/x402-specification.md
- 1Click Swap API docs — https://docs.near-intents.org/
- 1Click token list (live) — https://1click.chaindefuser.com/v0/tokens
- NEAR Intents explorer — https://explorer.near-intents.org/
- x402-1CS gateway — prior implementation of the 1CS quote→`PaymentRequirements` mapping (`src/payment/quote-engine.ts`), settlement poller (`src/payment/settler.ts`), and swap ledger (`src/storage/store.ts`) — https://github.com/IkerAlus/x402_1CS_gateway

*Coverage tables reflect the live 1CS token list and CDP facilitator docs as of 2026-09-01.*
