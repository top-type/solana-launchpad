# Market Supply Token (MST): specification

MST is a Solana program that turns any new SPL token into a **market-controlled-supply token**:

- The program is the only mint authority. Tokens are minted only when someone pays the quote asset into the token's reserve, and burned only when someone redeems them from it. Supply is whatever the market has bought.
- Price comes from a Bancor-style bonding curve over that reserve. Each token picks its own **quote asset** (SOL or any classic SPL token) at creation.
- Each token has a **treasury** that can fund grants. A grant draws on the reserve without moving the price, which lowers the curve's reserve ratio; revenue later refills the reserve.
- Holders control grants and setting changes through **PASS/FAIL decision markets** (futarchy): a proposal passes when the market prices the token higher if it passes than if it fails.

Trades are instant: each buy or sell executes on its own against the curve, with a slippage limit. No admin key can move reserve funds; only a passed decision can fund a grant.

---

## 1. Concepts

| Term | Meaning |
|---|---|
| **Quote asset** | The SPL mint a token is priced in (wrapped SOL, USDC, or any classic SPL mint), fixed at creation. Token-2022 mints are not accepted, because transfer fees and hooks would break the reserve accounting |
| **Reserve (R)** | Quote atoms held by the program that back the token supply |
| **Supply (S)** | Token atoms in existence; always equals the mint's supply outside decisions |
| **Virtual reserve (V)** | Pricing state with `V >= R`. Price is `p = V / S` quote atoms per token atom |
| **Reserve ratio (r)** | `r = R / V`, in (0, 1]. At `r = 1` the curve is **flat** (constant price); lower `r` means a steeper curve |
| **Treasury** | Quote held outside the reserve (`treasury` balance). It funds grants, and refills the reserve |
| **Revenue** | Quote flowing into the treasury: winning proposal bids, and an optional trading fee |
| **Decision** | A PASS/FAIL market pair that decides one proposal |

This is the Bancor formula with connector weight `r`. The difference is that r is not fixed: grants lower it and revenue raises it back toward 1, and neither changes the current price.

---

## 2. Curve

### 2.1 Opening

At creation the creator deposits `d0 > 0` quote atoms and picks `tokens_per_quote = q` (the opening price is `1/q`):

```text
R = d0,  V = d0,  S = q · d0        -> r = 1 (flat), price 1/q
```

The creator receives the `q · d0` opening tokens.

### 2.2 Trades (closed form)

The reserve ratio stays constant through trades: V scales with R (`V' = V · R'/R`).

```text
buy  (pay d quote):    R' = R + d,          S' = S · (R'/R)^r,      tokens out = S' − S        (round down)
sell (burn y tokens):  S' = S − y,          R' = R · (S'/S)^(1/r),  quote out  = R − R'        (round down)
after either:          V' = V · R'/R  (round down when R grows, up when R shrinks)
                       p' = V'/S' = p · (R'/R)^(1 − r)
```

- On a flat curve (`r = 1`) these are exact proportional trades: `tokens out = d·S/R`, `quote out = y·R/S`. Implement this case exactly, without the power function.
- An optional trading fee (`trade_fee_bps`, default 0) is taken from the buy input or the sell output and credited to the treasury.

### 2.3 Price band

Per **period** (`period_slots` long), each curve remembers its opening price `p0`. A trade may move the closing price at most `band_bps` above or below `p0`. The check uses integer cross-products: compare `V'·S0·10000` with `V0·S'·(10000 ± band_bps)`.

When a buy would exceed the band, **fill it partially** up to the band edge. With `r < 1`, the largest allowed reserve has a closed form:

```text
R_max = R · (p_max / p)^(1 / (1 − r))      fillable = R_max − R   (round down); refund the rest
```

Sells mirror this with `p_min`. A flat curve's price cannot move, so the band never binds there. The band bounds how far one period's traders, including sandwichers in a Jito bundle, can move the price. Each user's `min_out` bounds their own loss.

### 2.4 Fixed-point implementation

- Amounts are u64. Use u128 or U256 for intermediates, and unsigned fixed point with at least 64 fractional bits for ratios.
- Compute `x^e` as `exp2(e · log2 x)`. Every step rounds in the direction that makes the user's output smaller.
- Accuracy target: relative output error below 2^-48, never in the user's favour.
- Budget: at most 200,000 CU for `buy`/`sell`, including token CPIs. The client requests a compute-unit limit with margin.
- Bounds: R, S and amounts at most 2^63, V at most 2^126, `r >= r_min`. A sell must leave `S' >= 1` and `R' >= 1`, except when burning the whole supply, which the band never allows through governance.

---

## 3. Treasury, revenue and grants

### 3.1 Revenue and refilling

Revenue (winning bids, plus fees if enabled) goes to the treasury. Outside a decision, before any other work, every trade and every `crank` first refills the reserve:

```text
refill = min(treasury, V − R);   R += refill;   treasury −= refill
```

V and S stay the same, so the price does not change while r moves back toward 1. Once `R == V` the curve is flat again and further revenue accumulates in the treasury.

### 3.2 Grants

A grant pays up to 16 recipients fixed quote amounts. With total G:

```text
eligible if  G <= treasury + floor(R / 2)  and  (R − from_reserve) / V >= r_min
from_treasury = min(treasury, G),  from_reserve = G − from_treasury
```

Approving a grant lowers R by `from_reserve` and leaves V and S unchanged. The price stays the same and the curve gets steeper. Revenue then refills it over time (3.1). One grant can at most halve the ratio, and `r_min` (creation setting, default 1/8, at least 1/64) stops consecutive grants from making the curve too steep to trade.

---

## 4. Governance

### 4.1 Lifecycle

`Idle → Proposing → Deciding → Cooldown → Idle`. All times are in slots.

1. **Proposing** (`proposal_window_slots`): anyone may submit a proposal with a bid in the quote asset, plus a SOL deposit covering the decision accounts' rent. Others may add to a proposal's bid.
   - Proposal kinds: a grant (3.2), or a change to one setting (section 5).
   - A proposal's ID is derived from its proposer and contents, so a copy by someone else is a separate proposal.
2. At the end of the window, the **highest-bid eligible proposal** opens a decision. Ties go to the earlier proposal. An ineligible proposal, such as an unfundable grant, is skipped and its bids become refundable.
   - The winner's bids are revenue, whether the proposal later passes or not.
   - Every other bid is refundable by its bidder.
   - With no eligible proposal, the round ends.
3. **Deciding** (section 4.2).
4. **Cooldown** (`cooldown_slots`): passed setting changes activate when it ends.

### 4.2 Decision markets (conditional tokens)

Opening a decision:

1. Create four conditional SPL mints: `PASS-quote`, `FAIL-quote`, `PASS-token` and `FAIL-token`. Each has the decision PDA as mint authority and the decimals of its underlying. Also create two escrow vaults for split deposits.
2. Copy the main curve into a **PASS curve** and a **FAIL curve**. For a grant, both curves open with the grant's `from_reserve` already removed. PASS escrows the grant total; FAIL holds the same amount aside. Both markets therefore open at the same price with the same steepness, and neither is easier to move.
3. Freeze the main curve: `buy` and `sell` are refused until resolution. Ordinary SPL transfers of the token are unaffected.

During the decision:

| Instruction | Effect |
|---|---|
| `split` | Lock 1 real quote atom (or token atom) and mint 1 PASS and 1 FAIL conditional atom of it |
| `merge` | Burn 1 PASS and 1 FAIL atom; unlock 1 real atom |
| `trade_conditional` | Buy or sell on the PASS or FAIL curve with that market's conditional tokens: the same math and band as section 2 |
| `split_and_trade_both` | Convenience: split, then the same trade in both markets |

### 4.3 Observation and the decision rule

The decision runs for N periods. PASS **leads** a period when, at that period's closing prices:

```text
V_pass · S_fail · 10000  >  V_fail · S_pass · (10000 + margin_bps)
```

PASS wins if it leads in **more than N/2 periods** and, for a grant, the grant is still fully funded. Ties fail.

**No crank needed.** Prices change only on trades, so a period's closing price is the price after its last trade. Every decision instruction, and the permissionless `crank`, first runs a catch-up step: it scores every fully elapsed period with the prices as they stand, then applies its own action. A trade therefore only affects periods after its own, and a closed period can never be changed. Catch-up handles at most 512 periods per call; call `crank` again to continue.

### 4.4 Resolution (`resolve`, permissionless, after N periods)

- The winning market's curve becomes the main curve, and its conditional tokens redeem 1:1 for the real assets.
  - Winning token-redemptions are paid from the token escrow first; if the escrow runs out, the program mints the remainder.
  - `finalize` burns any escrow left over, so the mint supply equals the winning curve's supply.
- The losing market's conditional tokens are worthless; owners can close the accounts to recover rent.
- **PASS grant:** each recipient claims its amount once (`claim_grant`).
- **FAIL grant:** the held-aside amount returns to the treasury, and the reserve refills from it as usual (3.1).
- **PASS setting change:** it is scheduled to activate at the end of cooldown.

### 4.5 Conservation

At all times, for each token:

```text
quote vault  =  R + treasury + grant escrow + bid escrow
split vaults =  outstanding conditional atoms of each pair
```

After resolution, the winning market's reserve and supply must match real balances exactly. Implement these as invariant functions that tests check after every instruction.

---

## 5. Settings

Each setting is set at creation, and changeable only by a passed decision. Every bound is enforced at proposal time and again at activation, so a passed change can never make the next decision impossible.

| Setting | Meaning | Bounds |
|---|---|---|
| `band_bps` | Max price move per period, per curve | 1..=9,999 |
| `period_slots` | Length of a period | 150..=216,000 (about 1 minute to 1 day) |
| `decision_periods` N | Periods per decision | 3..=10,000 |
| `margin_bps` | PASS margin | Must stay below `(ceil(N/2) − 1) · band_bps`, so PASS can still win |
| `proposal_window_slots` | Proposing phase length | ≥ `period_slots`, at most about 30 days |
| `cooldown_slots` | Cooldown length | at most about 30 days |
| `min_bid` | Minimum opening bid | ≥ 1 atom |
| `trade_fee_bps` | Trading fee into the treasury | 0..=500; a change moves it by at most ×2 or ±25 bps |
| `r_min` | Reserve-ratio floor for grants | Creation only; default 1/8, at least 1/64 |
| `tokens_per_quote` q | Opening price | Creation only |

---

## 6. Accounts

All PDAs store their bump; every instruction re-derives and checks seeds.

| Account | Seeds | Contents |
|---|---|---|
| `TokenState` | `["mst", mint]` | quote mint and vault, decimals, R, S, V, treasury, settings, phase, band state, round counter, freeze-authority flag of the quote mint |
| `quote_vault` | `["vault", mint]` | Program-owned SPL token account for the quote asset |
| `Proposal` | `["proposal", mint, round, proposer, id]` | kind, description hash (text off-chain on IPFS/Arweave), total bid, status, rent deposit |
| `Bid` | `["bid", proposal, bidder]` | amount (escrowed in `["bid_escrow", mint]`) |
| `Decision` | `["decision", mint, round]` | proposal, schedule, margin, PASS and FAIL curves `{R, S, V, band}`, grant escrow and holdback, periods observed, PASS leads, last observed slot, conditional mints and split vaults, outcome |
| `GrantClaim` | `["claim", decision, recipient]` | Marks a claimed grant payment |

The token mint is a classic SPL mint created by `create_token`. Its mint authority is the `TokenState` PDA, and it has no freeze authority. Metadata uses Metaplex Token Metadata.

---

## 7. Instructions

| Instruction | Signer | Notes |
|---|---|---|
| `create_token` | creator | Validate the quote mint: owned by the classic Token program, or the native mint. Refuse Token-2022. Record any freeze authority. Create the mint, vault and metadata. Take the opening deposit. Validate settings |
| `buy(quote_in, min_tokens_out)` | user | Refill first; band and partial fill; fee; mint tokens |
| `sell(tokens_in, min_quote_out)` | user | Refill first; band and partial fill; fee; burn tokens and pay quote |
| `propose(kind, description_hash, bid)` | proposer | Proposing phase only. Checks the bid ≥ `min_bid` and the rent deposit, and validates the grant or setting |
| `add_bid(amount)` / `refund_bid` | bidder | Refunds open for losing, skipped or cancelled proposals |
| `crank` | anyone | Phase transitions, opening the winning decision, catch-up, resolution, activation. Bounded work; idempotent |
| `split` / `merge` / `trade_conditional` / `split_and_trade_both` | user | During a decision |
| `resolve` / `redeem` / `finalize` / `claim_grant` | anyone or holder | After the decision |
| `close_*` | owner | Reclaim rent once balances are zero |

Every state change emits an Anchor event (trade, bid, decision opened, period scored, resolution, claim), so indexers and the UI can rebuild history.

---

## 8. Security requirements

- **Accounts:** check every PDA by seeds and bump, every token account by mint, owner and token program (`spl_token::ID` only), and every mint against the stored key. Never trust an account because of its position in the instruction.
- **Authority:** only PDAs mint or burn. No admin key controls funds. Ship the program immutable, or with a multisig upgrade authority.
- **Math:** build with `overflow-checks = true`; use checked arithmetic everywhere; round toward the reserve; property-test against a high-precision reference.
- **MEV:** `min_out` is mandatory, and the UI defaults it to quote − 1%. The band bounds per-period price movement. Document that sandwiching inside the band is still possible.
- **Decision integrity:** catch-up runs before the instruction's own trade; markets open identical; empty markets never lead; ties fail; the `r_min` floor applies.
- **DoS and rent:** proposals need `min_bid` plus a rent deposit; at most 16 grant recipients; bounded loops; no unbounded `remaining_accounts` iteration.
- **Quote mints:** a freeze authority on the quote mint (USDC has one) can freeze the vault. Record it, show it in the UI and require the creator to acknowledge it. Wrap and unwrap SOL through the native mint.
- **Clock:** use `Clock::slot`, never `unix_timestamp`.
- **Audit:** do a devnet launch and get an external audit before mainnet.

---

## 9. Tests

- **Framework:** `anchor test` on `solana-test-validator`, with fast tests on LiteSVM or bankrun. One invariant checker (4.5) runs after every instruction in every scenario.
- **Math:**
  - property tests against an `mpmath` reference that writes JSON vectors to `tests/vectors/`;
  - round trips are never profitable, and outputs are monotonic;
  - partial fills land exactly at the band edge;
  - worst-case compute units for each instruction are measured and asserted.
- **Scenarios:**
  1. Create tokens with a wSOL quote and with a USDC-like quote; refuse Token-2022 and fake mint, vault or program accounts.
  2. Buy and sell on flat and steep curves: slippage refusals, partial fills, fees.
  3. **Passing grant:**
     - proposing, then the winner opening;
     - identical PASS and FAIL markets;
     - split, trade both markets, merge;
     - idle gaps between trades, handled by catch-up;
     - resolve, claim grants, redeem, finalize.
  4. **Failing grant:** the holdback returns to the treasury and refills the reserve.
  5. **Refill:** revenue returns `r` to 1 without moving the price; then surplus revenue accumulates in the treasury.
  6. **Setting changes:** bounds checked at proposal and at activation.
  7. **Adversarial:**
     - a last-moment trade cannot change a closed period;
     - a losing redemption or a double grant claim is refused;
     - maximum magnitudes cause no overflow;
     - very long idle gaps catch up correctly;
     - the `r_min` refusal works.

---

## 10. Web UI

Next.js (or Vite + React) with `@solana/wallet-adapter` and an Anchor client generated from the IDL. The server holds no keys.

| Page | Contents |
|---|---|
| Explore | Tokens with price, reserve, ratio, quote asset and phase |
| Create | Name, symbol and image; quote-asset picker (SOL, USDC, or paste a mint) with a freeze-authority warning; opening deposit and price; governance presets |
| Trade | Exact quote, price impact, band limit, `min_out` |
| Governance | Proposals, bids, the PASS and FAIL markets with live prices and leads against the majority, split, merge, trade, redeem, claim |

Users are told:
- the price is guaranteed only down to their `min_out`;
- decisions follow market prices, not merit;
- any freeze authority on the quote asset.

---

## 11. Milestones

1. Math crate with reference vectors and property tests.
2. `create_token`, `buy`, `sell`, band, fees, refill, plus the invariant checker.
3. Proposals, bids and `crank`.
4. Decision markets, catch-up, resolution, grants, redemption.
5. Setting changes.
6. Web UI on devnet.
7. Compute profiling, instruction-sequence fuzzing (Trident), external audit.
