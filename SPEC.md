# Stash Launchpad: Solana specification

A Solana program (Anchor) plus a web UI that lets anyone launch a token whose price comes from its own Hashstash-style bonding-curve reserve, priced in a quote asset the creator picks at launch. Each launch can fund grants from its reserve, and holders decide every grant or setting change with PASS/FAIL futarchy markets.

This document is the build brief. It is self-contained; the original design it ports is Hashstash (local reference copy: `C:\Users\taata\hashstash\Hashstash`, especially `docs/SPECIFICATION.md` sections 2–7, `docs/RESERVE.md`, `src/batch.rs`, `src/math.rs`, `src/consensus.rs`).

---

## 1. Scope

### In scope (decided)

| Area | Decision |
|---|---|
| Core | Per-launch bonding curve and reserve (Hashstash curve: reserve R, supply S, curve mass M) |
| Trading | **Instant** buys and sells against the curve, each with a user-signed minimum output (slippage limit) and a per-period price band |
| Quote asset | Chosen per launch at creation: **native SOL (as wrapped SOL) or any classic SPL Token mint**. Token-2022 mints are refused |
| Grants | Holders can approve grants paid from the reserve; a grant lowers the reserve without changing the price, so the curve gets steeper |
| Governance | Hashstash-style **PASS/FAIL decision markets** per launch, deciding grants and a small set of setting changes, entered through a bid auction |
| Deliverable | Anchor program, full test suite on a local validator, and a web UI |

### Out of scope (decided)

- **Mandatory trading fees.** Revenue comes from winning auction bids, as in Hashstash. An optional `trade_fee_bps` setting (default 0) can add trading-fee revenue; see section 4.5. Revenue refills the reserve after grants, as in Hashstash.
- Batched clearing (Hashstash clears each block's orders together; this port trades instantly).
- Token-2022 quote mints, transfer-fee or transfer-hook tokens.
- Cross-launch features, an order book, limit orders that rest.

### Non-negotiable properties

1. **Conservation.** For every launch and every instruction: real quote held by the program's vaults equals reserve + surplus + grant escrow + bid escrow + conditional-vault backing, exactly, in atoms. Real token supply plus outstanding winning conditional tokens equals the curve supply after settlement.
2. **Rounding always favours the reserve.** Buy outputs and sell payouts round down; nobody can profit from a buy-then-sell round trip.
3. **Determinism.** Integer and fixed-point math only. No floats in the program.
4. **No custody of user keys** in the UI; all signing happens in the user's wallet.

---

## 2. Concepts mapped from Hashstash

| Hashstash | Launchpad |
|---|---|
| HASH, the spendable coin | The launch's **quote asset** (wSOL or a classic SPL mint), chosen at creation |
| Stash, the curve token | The launch's **token**: a new SPL mint per launch; the launch PDA is its only mint authority, with no freeze authority |
| Reserve R, supply S, curve mass M | Fields of the `Launch` account; R is held in the launch's quote vault |
| Share scale q (genesis exchange rate) | `share_scale` chosen at creation: the first deposit of d quote atoms creates q·d token atoms |
| Waiting revenue U | `surplus`: quote held by the launch outside the reserve. Sources: winning auction bids, optional trading fees, and any part of a failed grant's set-aside that could not return to the reserve. It **replenishes the reserve up to M**, returning the curve toward flat after grants, exactly as in Hashstash (section 4.5) |
| Grant escrow E | `grant_escrow` of the active decision |
| PASS and FAIL worlds | Two **conditional markets** per decision, each with its own curve copy, traded with conditional tokens (section 7) |
| Block-based observations | Slot-based **observation periods** of fixed length, evaluated lazily (section 7.5) |
| Auction with bids and lead period | Same structure, measured in slots (section 6) |
| MaxPriceChangeBps per block | `max_price_change_bps` per observation period, per curve (section 4.4) |

---

## 3. Accounts

All program accounts are PDAs with fixed seeds. Use Anchor `#[account]` with explicit `space`, store a `bump`, and check every seed in every instruction.

### 3.1 `Launch` (seeds: `["launch", token_mint]`)

| Field | Type | Meaning |
|---|---|---|
| `creator` | Pubkey | Creator (informational only; no special powers after creation) |
| `token_mint` | Pubkey | The launch token mint (created by `create_launch`) |
| `quote_mint` | Pubkey | Chosen quote mint |
| `quote_vault` | Pubkey | Token account owned by the launch PDA, holding R + surplus + escrows |
| `quote_decimals`, `token_decimals` | u8 | Copied from the mints |
| `share_scale` | u64 | q: token atoms per quote atom at the opening flat price |
| `reserve` (R) | u64 | Reserve backing the curve |
| `supply` (S) | u64 | Curve supply; must equal the token mint's supply outside decisions |
| `mass` (M) | u128 | Curve mass (price state, not a balance); see 4.1 |
| `surplus` | u64 | Quote outside the reserve (section 2) |
| `phase` | enum | `Rest { until_slot }`, `Auction { round, opened_slot, ... }`, `Decision { decision }` |
| `settings` | struct | Governed settings, section 8 |
| `price_band` | struct | Period start slot and opening price (M, S) for the band check, section 4.4 |
| `round` | u64 | Current auction round number |
| `bump`, `vault_bump`, `mint_bump` | u8 | PDA bumps |

Invariants outside decisions: `0 < R <= M` once the curve is live; `S > 0` iff `R > 0`; token mint supply == S.

### 3.2 `Proposal` (seeds: `["proposal", launch, round, proposer, id]`)

| Field | Meaning |
|---|---|
| `launch`, `round`, `proposer` | Binding; the ID commits to the proposer so a copied proposal is a separate proposal (Hashstash F075) |
| `kind` | `Grant { payments: Vec<(recipient: Pubkey, amount: u64)> }` (max 16 payments, sorted by recipient, distinct, positive) or `Parameter { setting, old, new }` |
| `description_hash` | 32-byte hash of off-chain text (stored on IPFS/Arweave; the UI shows it) |
| `total_bid` | Sum of all bids, quote atoms |
| `status` | `Active`, `Cancelled { refundable_from_slot }`, `Won`, `Lost` |
| `rent_deposit` | SOL lamports prepaid by the proposer to cover the decision accounts' rent (section 7.1); refunded if the proposal does not open a decision |

### 3.3 `Bid` (seeds: `["bid", proposal, bidder]`)

Amount of quote atoms a bidder has put behind one proposal, held in the bid escrow token account (`["bid_escrow", launch]`). Refundable when the proposal loses, is cancelled, or its round times out; the winner's bids move to `surplus`.

### 3.4 `Decision` (seeds: `["decision", launch, round]`)

| Field | Meaning |
|---|---|
| `proposal` | The winning proposal |
| `opened_slot`, `period_slots`, `periods` (N) | Frozen schedule: N observation periods of `period_slots` slots |
| `pass_margin_bps` | Frozen margin m |
| `worlds[PASS], worlds[FAIL]` | Each: `{ reserve, supply, mass, price_band }` |
| `grant_escrow` | G set aside from the reserve (grants only) |
| `fail_holdback` | The same shortfall held for FAIL (section 7.2) |
| `observed` | Periods observed so far |
| `pass_leads` | Periods in which PASS led |
| `last_observed_slot` | For lazy catch-up |
| `conditional mints` | `pass_quote`, `fail_quote`, `pass_token`, `fail_token` (PDAs) |
| `token_escrow` | Real launch tokens deposited for splitting |
| `cond_quote_vault` | Real quote deposited for splitting |
| `outcome` | `Pending`, `Passed`, `Failed` |

### 3.5 `GrantClaim` (seeds: `["grant_claim", decision, recipient]`)

Marks a passed grant's payment as claimed, so each recipient is paid exactly once.

---

## 4. Curve mathematics

### 4.1 State and price

For a live curve: reserve R, supply S, mass M with `0 < R <= M`.

```text
reserve ratio  r = R / M          (1 = flat; lower = steeper)
marginal price p = M / S           (quote atoms per token atom)
```

Opening (empty curve, first deposit of d quote atoms):

```text
R = d,  M = d,  S = q·d    =>  r = 1, p = 1/q
```

A trade keeps r constant: mass scales with reserve, `M' = M · R'/R`. Grants lower r without changing the price. Replenishment from surplus raises r back toward 1, also without changing the price (section 4.5).

### 4.2 Trades in closed form

With r fixed during a trade:

```text
Buy with d quote atoms:     R' = R + d
                            S' = S · (R'/R)^r            tokens out x = S' - S   (round down)
Sell y token atoms:         S' = S - y
                            R' = R · (S'/S)^(1/r)        quote out  = R - R'     (round down)
Mass after either:          M' = M · R'/R   (round down if R' >= R, round up if R' < R)
Price after either:         p' = M'/S' = p · (R'/R)^(1 - r)
```

On a flat curve (r = 1) these reduce to exact proportional trades at price M/S: `x = floor(d·S/R)`, `quote out = floor(y·R/S)`. Implement the flat case exactly, without the power function.

These are the same curve as Hashstash's `C(x) = R·((1 + x/S)^(1/r) − 1)`, specialised to one order at a time.

### 4.3 Fixed-point implementation

- Represent amounts as u64 and products/intermediates as u128 or U256 (`ruint` or a vendored `U256`); represent ratios and powers in unsigned fixed point with at least 64 fractional bits.
- Compute `a^b` for `a` in (0, 2] and `b` in [r_min, 1] or [1, 1/r_min] as `exp2(b · log2(a))` with directed rounding. Every intermediate rounds in the direction that makes the user's output smaller: lower bound for buy outputs, lower bound for sell payouts.
- Accuracy target: relative error below 2^-48 on outputs, always conservative. Prove direction with tests against a high-precision reference (section 11.2).
- Compute budget target: each buy or sell instruction at most 200,000 compute units, including the band check and token CPIs. Request a higher unit limit in the UI's transactions as a safety margin.
- Refuse inputs that could overflow: d, y, R, S at most 2^63; M at most 2^126; require `R' >= 1` after sells, `S' >= 1` after sells, unless selling the whole supply (allowed only when `max_price_change_bps` permits, as in Hashstash; see 4.4).

### 4.4 Price band (per observation period)

Each curve (main, or each decision world) records the opening price `(M0, S0)` of the current band period (`settings.period_slots` long, aligned to `opened_slot` or launch creation). A trade must keep the closing price within:

```text
p0 · (10000 - b)/10000  <=  p'  <=  p0 · (10000 + b)/10000      b = max_price_change_bps
```

using integer cross-products (`M'·S0·10000` against `M0·S'·(10000 ± b)`). If a buy would exceed the upper bound, **fill it partially**: the largest reserve the band allows is

```text
R_max = R · (p_max / p)^(1 / (1 - r))       for r < 1
```

so the fillable cash is `R_max − R` (round down), the rest of the input is refunded in the same instruction, and the user's minimum output applies to the filled part. Sells mirror this with `p_min`. On a flat curve the price does not move, so no band applies. Governance bounds b to 1..9,999 (never full redemption through governance).

### 4.5 Revenue and replenishment (as in Hashstash)

- **Revenue sources.** Winning auction bids (section 6) and, if enabled, a trading fee of `trade_fee_bps` (default 0) taken from each buy's input and each sell's payout. Revenue is credited to `surplus`.
- **Replenishment.** Outside a decision, whenever `surplus > 0` and `R < M`, move `refill = min(surplus, M − R)` from surplus into the reserve: `R += refill`, `surplus −= refill`. M and S stay the same, so the price does not change and r moves back toward 1. Apply it lazily at the start of every trade and of `advance`, before anything else in the instruction. This matches Hashstash's per-block step `min(U, M − R)`.
- **Grants are temporary.** A grant lowers r. Later revenue refills the reserve and returns the curve to flat, after which further revenue waits in surplus and funds future grants. How fast that happens depends on revenue: winning bids, plus fees if enabled.
- **During a decision**, revenue waits; no replenishment happens until resolution. The selected outcome then replenishes from its surplus (section 5.3).
- **Floor on r.** Refuse any grant that would push r below `r_min` (a creation-time setting, default 1/8, at least 1/64). This only matters if several grants pass before revenue refills the reserve. Below that floor the power-function range and the band computation get numerically fragile.

---

## 5. Grants

### 5.1 Eligibility

A grant with total G (sum of its payments) can open a decision only if:

```text
G <= surplus + floor(R / 2)
and  the resulting ratio  (R - from_reserve) / M  >=  r_min
```

`from_surplus = min(surplus, G)`, `from_reserve = G − from_surplus`. Mass and supply do not change, so the price stays the same at that moment.

### 5.2 Opening

At decision opening (section 7.1), both worlds start from the same curve:

```text
PASS world:  R_pass = R - from_reserve,  M and S as main;  grant_escrow = G
FAIL world:  R_fail = R - from_reserve,  M and S as main;  fail_holdback = from_reserve
surplus -= from_surplus   (moves into grant_escrow)
```

Both outcomes open with identical curves (as in Hashstash), so neither is structurally easier to move.

### 5.3 Resolution

- **PASS:** each recipient claims its exact amount from the grant escrow once (`claim_grant`, with a `GrantClaim` marker). The PASS world's reserve, supply and mass become the main curve.
- **FAIL:** the FAIL world's state becomes the main curve. The holdback and the `from_surplus` part of the escrow go back to `surplus`, and ordinary replenishment (4.5) then moves `min(surplus, M − R)` into the reserve. If FAIL saw no net sales, this restores the reserve exactly. Otherwise the remainder stays in surplus, as in Hashstash. This caps R at M, keeping `R <= M`.
- **PASS:** after claims, ordinary replenishment from surplus resumes; the curve returns toward flat as revenue arrives.

---

## 6. Proposals and auctions

The phases mirror Hashstash: `Rest → Auction → Decision → Rest`. Time is in slots.

- **Propose** (during `Auction`): create a `Proposal` with an initial positive bid (quote atoms, moved into the bid escrow) and a SOL `rent_deposit` that covers the decision accounts. Grant payments: 1–16 distinct recipients, sorted, positive amounts, total fits u64. Parameter changes: one setting, exact old and new values, new within section 8's bounds.
- **Support:** add to a proposal's bid.
- **Leadership** is evaluated whenever any auction instruction or the permissionless `advance` crank runs, for the current slot:
  - The highest total bid leads. The incumbent keeps a tie; otherwise ties go to the smaller proposal ID.
  - A proposal that could not open its decision right now (an unfundable grant, or a ratio below r_min) never leads. It is cancelled and refundable from the next slot.
  - Keep the record of displaced leaders: when a leader is cancelled, the most recently displaced active proposal becomes the leader again with its earlier lead start (Hashstash F016 fix).
  - A strictly higher bid restarts the lead clock.
  - The leader wins when `slot − lead_since >= lead_slots`, or when it leads at the auction timeout (`slot − opened >= auction_timeout_slots`). Without an eligible leader at the timeout, the round ends with no decision and enters `Rest`.
- **Winning bids** become revenue: they move to `surplus` and replenish the reserve (4.5), even if the decision later fails.
- **Losing, cancelled or timed-out bids** are refundable by their bidders (`refund_bid`); never pushed automatically.

---

## 7. PASS/FAIL decisions on Solana

Hashstash forks every account into two worlds. Solana cannot fork balances, so this port uses **conditional tokens**, in the style of MetaDAO's conditional vaults: during a decision, every unit of the real quote or token can be split into one PASS unit and one FAIL unit. Each world trades its own curve, and at resolution the winning units redeem 1:1 for the real asset while the losing units become worthless.

### 7.1 Opening a decision (inside `advance`, or a dedicated `open_decision`)

1. Freeze `periods` N, `period_slots`, and `pass_margin_bps` m from the current settings.
2. Create the four conditional mints (`pass_quote`, `fail_quote`, `pass_token`, `fail_token`), with the decision PDA as mint authority and the same decimals as the underlying. Also create `token_escrow` and `cond_quote_vault`. Rent comes from the winning proposal's `rent_deposit`; the cranker is reimbursed, and any excess is returned to the proposer.
3. Set aside grant funding (5.2) and copy the main curve into both worlds.
4. **Freeze the main curve:** `buy` and `sell` on the main curve are refused while a decision is active. Real token transfers continue normally, because SPL transfers are outside the program.

### 7.2 Conditional instructions

| Instruction | Effect |
|---|---|
| `split_quote(amount)` | Real quote → `cond_quote_vault`; mint `amount` pass_quote + `amount` fail_quote to the user |
| `split_token(amount)` | Real tokens → `token_escrow`; mint `amount` pass_token + `amount` fail_token |
| `merge_quote(amount)` / `merge_token(amount)` | Burn `amount` of both outcomes; return the real asset (any time before resolution) |
| `trade(world, side, amount_in, min_out)` | Buy or sell on that world's curve with that world's conditional tokens, using section 4's math and band. Buys burn the world's quote units into the world's reserve accounting and mint world-token units; sells do the reverse |
| `redeem(world_asset, amount)` | After resolution: winning-world units redeem 1:1 for the real asset; losing units are refused (they can be closed by their owner to recover rent) |

The UI offers "trade in both worlds" as one transaction (split + two trades) for users who only want exposure to the curve.

### 7.3 Backing and conservation

- **Quote:** the main vault's reserve R backs the opening reserve of **both** worlds at once. One real atom backs one PASS atom plus one FAIL atom, the same way split units do. Outstanding winning `*_quote` units are backed by `cond_quote_vault`, and the winning world's curve reserve is backed by the main vault plus whatever winning-world quote units its curve absorbed. Settlement makes these numbers equal exactly. Prove it with an invariant test after every instruction (section 11).
- **Token:** at resolution the main supply becomes the winning world's supply S_w. Redeeming winning `*_token` units transfers from `token_escrow` first. If the escrow runs out, the launch mints the rest, which happens when the winning curve minted more than it burned. Once every unit is redeemed, any leftover escrow is burned in `finalize_tokens`. The real token supply then equals S_w.
- Write these as explicit invariant functions, used by both the program (debug assertions in tests) and the test harness.

### 7.4 The decision rule

PASS leads an observation period exactly when, using that period's closing prices:

```text
M_pass · S_fail · 10000  >  M_fail · S_pass · (10000 + m)
```

Equality, or an empty world, gives no lead. After N periods, PASS wins only if `pass_leads > N/2` (a strict majority; a tie fails) **and** a grant is still fully funded. Same rule as Hashstash section 7.

### 7.5 Lazy, crank-independent observation

Prices change only when a trade changes them, so the closing price of a period equals the price after the last trade before the period ended. Every decision instruction (trade, split, merge, resolve) and the permissionless `advance` crank first run `catch_up(now_slot)`:

```text
while last_observed_slot + period_slots <= now_slot and observed < N:
    evaluate the lead with the CURRENT world prices      # unchanged since the period's last trade
    observed += 1; last_observed_slot += period_slots
```

Run catch-up **before** applying the instruction's own trade, so a trade only counts from the next period: a last-moment trade cannot change a period that has already closed (Hashstash section 7). This is exact, even if nobody cranks for a while. Bound the loop (for example at most 512 periods per instruction), and let `advance` be called repeatedly to finish.

### 7.6 Resolution (`resolve`, permissionless, after N observations)

Set the outcome; copy the winning world's (R, S, M) into the launch; apply 5.3 for grants; schedule parameter changes to activate after rest (section 8); move the phase to `Rest`. Then unfreeze the main curve.

---

## 8. Governed settings

Each launch has these settings, set at creation within the bounds below. Each can be changed only by a passed decision, and a passed change activates after the following rest.

| Setting | Meaning | Bounds (creation and governance) |
|---|---|---|
| `max_price_change_bps` | Price band per period per curve | 1..=9,999 (creation may allow 10,000 only for full-redemption launches; governance never) |
| `pass_margin_bps` | Margin m | 0..=`(ceil(N/2) − 1) · max_price_change_bps − 1`, so PASS can still win |
| `decision_periods` N | Observation periods per decision | 3..=10,000 |
| `period_slots` | Slots per observation and band period | 150..=216,000 (about 1 minute to 1 day) |
| `lead_slots` | Lead needed to win an auction | at most `(auction_timeout_slots − period_slots)/2` |
| `auction_timeout_slots` | Auction length | ≥ `lead_slots + period_slots`, at most about 30 days |
| `rest_slots` | Rest between decisions | at most about 48 days (as Hashstash's 2^22 s) |
| `r_min` | Ratio floor for grants | creation only; default 1/8, at least 1/64 |
| `min_bid` | Minimum opening bid | ≥ 1 atom; spam control |
| `trade_fee_bps` | Optional trading fee credited to surplus (revenue) | 0..=500 (default 0); a governed change moves it by at most a factor of 2 or by 25 bps, whichever is larger |

As in Hashstash, a governed change can only be undone by another decision. So validation must make sure the change cannot make the next decision impossible: check every bound at proposal time and again at activation (Hashstash R01/R05 fixes).

---

## 9. Instructions

All instructions validate accounts with Anchor constraints. A token program account must be `spl_token::ID` (never Token-2022). The quote vault, escrows and conditional vaults must be the PDA token accounts recorded in the launch or decision.

| Instruction | Signer | Key checks |
|---|---|---|
| `create_launch(params)` | creator | Quote mint owned by the classic Token program (or is the native mint); refuse mints with transfer-fee or hook behaviour by refusing Token-2022 entirely. Record whether the quote mint has a freeze authority (USDC does); the UI warns about it. Create the token mint (decimals, launch PDA as mint authority, no freeze authority), the quote vault and the optional Metaplex metadata. Require the creator's opening deposit d > 0 (4.1). Validate settings (section 8) |
| `buy(amount_in, min_out)` | trader | No active decision; band (4.4); partial fill with refund; output ≥ min_out; mint tokens to the trader |
| `sell(amount_in, min_out)` | trader | No active decision; band; burn the trader's tokens; pay quote ≥ min_out |
| `propose(kind, description_hash, bid)` | proposer | Auction phase; kind validation; bid ≥ min_bid; rent_deposit transferred |
| `support(amount)` | bidder | Auction phase; proposal active |
| `refund_bid` | bidder | Proposal lost, cancelled or timed out |
| `advance` | anyone | Phase transitions: Rest → Auction, leadership evaluation, opening decisions, catch_up, resolution, activation of passed settings. Idempotent; bounded work per call |
| `split_*`, `merge_*`, `trade`, `redeem`, `finalize_tokens` | user / anyone | Section 7 |
| `claim_grant` | recipient | Decision passed; payment listed; `GrantClaim` not present |
| `close_*` | owner | Reclaim rent from finished bids, losing conditional token accounts and finished decision accounts, once their balances are zero |

Emit an Anchor event for every state change: launch created, trade with in/out/price, bid, leadership change, decision opened, observation, resolution, claim. That gives indexers and the UI a complete history.

---

## 10. Security requirements

- **Account validation.** Every PDA is checked by its seeds and bump. Every token account by mint, owner and token program. Every mint by token program and stored key. No account is trusted because of its position in the instruction.
- **Signers and authorities.** Only the launch PDA mints or burns the launch token, and only the decision PDA mints or burns conditional tokens. There is no admin key over launches. If a global config exists, it holds no power over funds. Prefer an immutable program after audit, or at least a documented upgrade authority held by a multisig.
- **Math.** Build with `overflow-checks = true` in release. Use checked arithmetic everywhere, round in the documented directions, and property-test against a high-precision reference.
- **Slippage and the band.** `min_out` is mandatory and nonzero in the UI. The per-period band bounds how much one slot's traders can move the price. Document that the band bounds sandwich damage per period, but does not remove it.
- **Decision manipulation.** Keep Hashstash's protections: lazy observation runs before the trade, the band applies per world, equal opening curves, a strict majority, no lead from an empty world, and the ratio floor.
- **Denial of service.** Proposals cost a minimum bid plus a rent deposit. Bound the payments per grant at 16, and the work per `advance` and per catch-up. Never iterate over unbounded account lists inside one instruction.
- **Quote-mint risks.** A freeze authority on the quote mint can freeze the vault. Record it, show it in the UI and require the creator to acknowledge it. Wrapped SOL must be unwrapped and wrapped through the native mint flows.
- **Clock.** Use `Clock::slot` for all schedules, never `unix_timestamp`, which validators can skew.
- **Rent and closing.** Close accounts only when their balances are zero. Return rent to the payer recorded at creation.
- **Audit.** This program holds user funds. Get a professional audit before mainnet, and launch on devnet first.

---

## 11. Tests

### 11.1 Harness

- `anchor test` on a local `solana-test-validator`, plus fast unit and scenario tests with LiteSVM or solana-bankrun.
- A shared invariant checker runs after every instruction in every scenario test. It covers conservation (section 1, property 1), `R <= M`, `S > 0` iff `R > 0`, `mint supply == S` outside decisions, and the decision backing equations (7.3).

### 11.2 Math

- Property tests (proptest) for buy, sell, partial fills and band solving, across ratios from r_min to 1 and magnitudes from 1 atom to 2^63. Check them against a high-precision reference: a Python `mpmath` script that writes JSON vectors, committed under `tests/vectors/`. Optionally cross-check against Hashstash's `src/math.rs`.
- **Round trip:** a buy followed by selling the bought tokens never returns more than the input. The same holds for a sell followed by a buy.
- **Monotonicity:** a larger input never gives a smaller output.
- **Band:** a partial fill lands exactly at or inside the bound, and a one-atom-larger fill would break it.
- **Compute:** measure the worst-case CU per instruction and assert it stays below 200,000.

### 11.3 Scenarios

1. Creating a launch with a wSOL quote and with a classic SPL quote. Refusing a Token-2022 quote, and fake mint, vault or token-program accounts.
2. Trading on flat and on curved curves: slippage refusals, partial fills at the band, refunds.
3. A full governance cycle for a **passing grant**:
   - auction, leadership and timeout;
   - decision opening, with set-aside and identical curves;
   - split, trade both worlds, merge;
   - lazy observation with gaps of no activity;
   - resolution, claims, redemption and token finalisation;
   - conservation holding throughout.
4. The same cycle for a **failing grant**: the holdback returns through replenishment, capped at `M − R`, and surplus accounting is exact.
5. **Replenishment:** after a passed grant, winning bids (and fees, if enabled) refill R step by step until `R == M`, without moving the price. After that, further revenue stays in surplus and is counted in the next grant's capacity. No replenishment happens during a decision.
6. Parameter-change decisions, activating after rest, with every bound enforced at proposal time and at activation.
7. Adversarial cases:
   - an unfundable leading grant is cancelled and the previous leader resumes;
   - a last-moment trade does not alter a closed period;
   - redeeming a losing token, or claiming a grant twice, is refused;
   - overflow at maximum magnitudes;
   - very long catch-up gaps;
   - the ratio-floor refusal.

---

## 12. Web UI

- **Stack:** Next.js (or Vite + React), `@solana/wallet-adapter`, `@coral-xyz/anchor` client generated from the IDL, and SPL token helpers. No server holds keys. An optional indexer reads the program's events.
- **Pages:**
  1. **Launches:** list with price, reserve, ratio r, quote asset and phase.
  2. **Create launch:** name, symbol, image URI, a quote-asset picker (SOL, USDC, or paste a mint), with validation and a freeze-authority warning; share scale, opening deposit and governance settings, defaulting to sensible presets.
  3. **Trade:** a quote with exact output, price impact, the band limit, and `min_out` defaulting to quote − 1%.
  4. **Governance:** the phase timeline, proposals with bids, propose, support and refund.
  5. **Decision view:** PASS and FAIL prices, leads so far against the needed majority, split, merge, trade per world and "trade both", redeem after resolution.
  6. **Grants:** claim for recipients.
- **User disclosures:** no order is guaranteed a price beyond its `min_out`; trades can be sandwiched within the band; decision outcomes follow market prices, not merit; any freeze authority on the quote asset is shown.

---

## 13. Milestones

1. Math crate (`programs/launchpad/src/math.rs` or a separate crate) with reference vectors and property tests.
2. Launch creation, buy and sell, band and partial fills, plus the invariant checker and tests.
3. Proposals, bids, auctions and `advance`, with tests.
4. Conditional tokens, decisions, lazy observation, resolution, grants and redemption, with a full scenario suite.
5. Settings governance with activation bounds.
6. Web UI on devnet.
7. Hardening: compute profiling, fuzzing of instruction sequences (for example Trident), documentation, then an external audit before mainnet.

---

## 14. Environment notes (this machine)

- WSL Ubuntu already has `solana-cli 4.1.1` (Agave), `solana-test-validator`, `cargo-build-sbf 4.1.0`, Rust 1.97.1, Node 22 and npm. **Anchor (and `avm`) is not installed**; install it before scaffolding (`cargo install --git https://github.com/coral-xyz/anchor avm` then `avm install latest`, or a pinned version).
- Build and test inside WSL. Windows has no linker for this toolchain.
