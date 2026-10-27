# Market Supply Token (MST)

A Solana program for launching tokens whose supply is controlled by the market, and whose grants are decided by futarchy.

## What it does

1. **Launch.** Anyone creates a token and picks the **quote asset** it trades against: SOL or a classic SPL token. The program owns the token's mint authority, so nobody can mint outside the rules below. Every token opens on a **flat curve**: one fixed price.
2. **Bonding curve.** The token is always bought from and sold to its own curve. Buying pays quote into the token's **reserve** and mints new tokens; selling burns tokens and pays quote out of the reserve. Supply is exactly what the market has bought.
3. **Revenue and grants.** Anyone can pay revenue into the reserve. A **grant** pays quote from the reserve to one recipient: a wallet, a multisig, or a program that does anything with it. Neither changes the price: a grant makes the curve steeper, and revenue flattens it again.
4. **Auctions.** Anyone can propose a grant or a change to one of the token's settings. Proposals compete in an **auction**: backers bid quote on the proposals they want decided, and the first proposal to **hold the lead for long enough** goes to a decision.
5. **Futarchy.** The decision splits the token into two worlds, PASS and FAIL, each with its own market. It is a race too: the first side whose price holds the lead for long enough wins. **Only the winning world is real**: the other is discarded as if it never happened.

## Bonding curve

The curve has a reserve `R`, a token supply `S` and a **reserve ratio** `r` between 0 and 1. Its price is the Bancor formula:

```text
price = R / (r · S)
buy with d quote:   tokens out = S · ((1 + d/R)^r − 1)
sell y tokens:      quote out  = R · (1 − (1 − y/S)^(1/r))
```

- `r = R / (price · S)`: it is the share of the token's market cap held in the reserve.
- `r = 1` is a flat curve: buying and selling don't move the price, and the reserve backs every token at that price.
- A smaller `r` is a steeper curve: buys push the price up faster, and sells push it down faster.
- **Trades never change `r`.** Only grants lower it and only revenue raises it.
- Selling every token always returns exactly `R`, so the curve can't run out of reserve.
- **Launch:** the creator deposits the opening reserve `R₀` and picks the opening price `p₀`. The curve starts flat (`r = 1`) and the creator receives the opening supply `S₀ = R₀ / p₀`. While the curve is flat, buys and sells all happen at `p₀`; the first grant from the reserve is what bends it.
- `r` never goes below a tiny program constant `MIN_R = 0.0001`, only so the fixed-point math stays precise. There is no other floor.
- Every trade carries a `min_out`, so the user's slippage is bounded.
- Buying and then selling the same tokens returns exactly what was paid, less rounding. Rounding always favours the reserve, so no sequence of trades makes money off the curve itself.

## Revenue

- **Sources.**
  - Anyone can pay quote in with `contribute`: a project sharing its income, a partner app routing fees, a donation.
  - Quote sent straight to the token's vault counts too: anything beyond the recorded balances is revenue, and the next instruction or `crank` books it.
  - Winning auction bids.
- **Revenue drips in.** New revenue goes to an **incoming** balance and enters the reserve evenly over `drip_slots`. New revenue restarts the drip for the whole incoming balance. Without the drip, someone could buy just before a large payment and sell right after, taking part of it.
- **Entering the reserve raises `r`, not the price:** adding `c` gives `r' = r · (R + c) / R`.
- **Overflow.** `r` can't go above 1. Revenue that arrives once the curve is flat is held aside as **overflow**. Overflow only exists while `r = 1`.
- The drip pauses from the moment an auction is won until its decision resolves. Revenue that arrives then waits in incoming.

## Grants

A grant is an **amount of quote** and a **recipient**: any address, such as a wallet, a multisig or a program. To make a grant do something (a buyback, a stream, a split among many people), send it to a program that does that.

**Funding**
- A grant is paid from overflow first, and any rest from the reserve.
- Taking `x` from the reserve leaves the price unchanged: `R' = R − x` and `r' = r · R' / R`. The curve gets steeper, so holders' tokens keep their current price but the reserve backs less of the market cap. Later revenue raises `r` back toward 1.
- The most a grant can take is `available = overflow + R · (1 − MIN_R / r)`.
- Example: `R = 1,000`, `r = 1`, no overflow. A 400 grant leaves `R = 600` and `r = 0.6`; the price is unchanged and the reserve now backs 60% of the market cap. Revenue of 400 later brings `r` back to 1.
- A grant is only ever paid in the PASS world (see Futarchy). If its proposal fails, nothing was taken out, so there is nothing to return.

**Paying it**
- If the grant passes, anyone can call `pay_grant` to send the amount to the recipient's associated token account for the quote asset. It creates that account if it doesn't exist yet, and the caller pays its rent. The recipient never signs anything.
- The recipient isn't called when paid. A program that should act on the money needs its own way to be triggered, for example an instruction anyone can call.
- If the payment can't go through, for example because the account is frozen, the grant stays owed and anyone can retry.
- The recipient is public for the whole auction and decision, so the markets can judge what it will do with the money. If it's an upgradeable program, whoever controls its upgrades decides that.

## Auctions

- **The cycle.** An **auction** runs while the token trades normally. As soon as a proposal wins it, normal trading pauses and the proposal goes to a decision. When the decision resolves, the next auction starts.
- During an auction, anyone can **propose**:
  - a grant: an amount and a recipient (see Grants). The amount must be at most `available` when proposed; or
  - a new value for one setting.
- Anyone can **bid** quote on a proposal. Bids can't be withdrawn until the auction ends.
- At any moment, the proposal with the highest total bid is the **leader**. A proposal takes the lead only by bidding past the leader's total. Until someone bids, there is no leader.
- **The auction is a race: the first proposal to lead for `auction_lead_slots` in total wins.** Only the leader's clock runs. Outbidding the leader stops its clock, and the new leader has to build up its own lead time, so there is no last moment to snipe.
- On-chain this is O(1):
  - every `bid` (and `crank`) first credits the slots since the last update to the current leader, then applies the new bid;
  - the slot at which the current leader would win is always known, so any instruction can tell whether the auction is over. From that slot on, bids, buys and sells are refused until `crank` closes the auction.
- **When the auction ends:**
  - The winner's bids become revenue, and every other bid is refunded to its bidder with `refund_bid`.
  - Exception: if the winner is a grant that is now larger than `available` (sells during the auction shrank the reserve), there is no decision, every bid is refunded, and the next auction starts.

## Futarchy

- **Two worlds.** When a decision opens, the token splits into a PASS world, where the proposal is enacted, and a FAIL world, where it isn't. When the decision ends, the winning world becomes the real token and the other is discarded.
- **Conditional tokens.** The program creates PASS and FAIL versions of both the token and the quote asset. During the decision, anyone can **split** 1 real unit into 1 PASS unit and 1 FAIL unit, and **merge** them back.
- **Two markets.** Traders buy and sell PASS-token with PASS-quote on the PASS curve, and FAIL-token with FAIL-quote on the FAIL curve.
  - The FAIL curve is a copy of the token's curve.
  - The PASS curve is a copy with the grant already taken out, from overflow first and then the reserve, which lowers its `r`. Both curves start at the same price.
  - The program holds the grant until it's paid. Nobody can trade it; otherwise money that only exists if PASS wins could be spent pushing PASS up.
  - A setting change doesn't touch either curve. It only takes effect if PASS wins.
- **Normal trading pauses** from the moment the auction is won until the decision resolves. `contribute` still works, and its revenue waits in incoming.
- **What this means for the markets.**
  - When the grant comes partly from the reserve, the PASS curve is steeper, so the same buy moves PASS's price more than FAIL's. Net buying during a decision favours PASS, and net selling favours FAIL.
  - A flat curve's price can't move. While the token is flat (`r = 1`), the FAIL price is fixed at the current price. A proposal that leaves both curves flat (a setting change, or a grant fully covered by overflow) can't pass, because PASS can never beat FAIL.
- **The decision is a race too: the first side to lead for `decision_lead_slots` in total wins.**
  - At any moment, PASS is the **leader** when `PASS price > FAIL price × (1 + pass_margin_bps / 10,000)`; otherwise FAIL is. With no trading, FAIL leads from the start and wins after `decision_lead_slots`.
  - Every slot is credited to whichever side was leading at the prices left by the last trade before it.
  - Pushing the price up for one slot wins one slot. To win, a side has to hold the lead.
  - A decision ends as soon as either side reaches the target, so it lasts at most `2 × decision_lead_slots − 1` slots.
  - On-chain this is O(1) and needs no crank to score: prices only change when someone trades, so every instruction first credits the slots since the last update to the current leader, then applies its own trade and re-checks who leads. The slot at which the current leader would win is always known; from that slot on, trades are refused and `crank` resolves the decision.
- **Resolution**, by `crank` once the decision has ended:
  - The winning world's curve (`R`, `S`, `r`, overflow) becomes the token's curve.
  - Winning conditional units redeem 1:1 for the real assets at any time after resolution; losing units are worthless. Real tokens held through the decision are simply tokens in the winning world.
  - The program mints or burns real tokens in its escrow so that the escrow matches the winning conditional tokens outstanding. Real supply then equals the winning curve's `S`.
  - If PASS wins, a setting change takes effect from the next auction, and the grant can be paid with `pay_grant`.
  - If FAIL wins, nothing happened: no grant, no setting change.
  - The drip resumes and the next auction starts.

## Settings

The creator picks each setting at launch. After that, only a passed proposal can change one, and the change applies from the next auction. The program sets no limits on the values; the markets decide whether a change is a good idea. A passed change can even stop governance for good (a huge `auction_lead_slots` or margin means nothing passes again) or freeze normal trading for good (a huge `decision_lead_slots` means the next decision never ends).

| Setting | Meaning |
|---|---|
| `auction_lead_slots` | Lead time a proposal needs to win an auction |
| `decision_lead_slots` | Lead time PASS or FAIL needs to win a decision |
| `pass_margin_bps` | How far PASS's price must beat FAIL's for PASS to lead |
| `drip_slots` | How long revenue takes to enter the reserve; 0 means at once |

A slot is about 400 ms.

## Assets

- **Quote:** SOL, held as wrapped SOL, or any classic SPL Token mint. Token-2022 mints are refused, since transfer fees and transfer hooks would break the vault's accounting.
- **The token:** a classic SPL mint with 6 decimals and Metaplex metadata (name, symbol, URI) set at launch. A program PDA holds the mint authority, and there is no freeze authority.
- **Conditional tokens:** classic SPL mints with the same decimals as the asset they represent, minted and burned only by program PDAs.

## Program outline (Anchor)

**Accounts**
- `TokenState`: the mint, quote mint and quote vault; `R`, `S`, `r`, overflow, incoming revenue and drip state; settings; current phase; and the current auction's leader, its lead slots and the slot it would win at.
- `Proposal`: its auction, kind, grant amount and recipient or setting and value, total bid, lead slots and status.
- `Bid`: one bidder's amount on one proposal.
- `Decision`: the PASS and FAIL curves, the four conditional mints, the real-token escrow, the held grant, lead state (current leader, PASS and FAIL lead slots, last update) and outcome.

The quote vault holds all of the token's quote as balances recorded inside it: reserve, overflow, incoming revenue, held bids, refunds owed, quote backing conditional units, and grants held or owed.

**Instructions**

| Group | Instructions |
|---|---|
| Launch and trade | `create_token`, `buy`, `sell`, `contribute` |
| Auctions | `propose`, `bid`, `refund_bid` |
| Futarchy | `split`, `merge`, `trade_conditional`, `redeem`, `pay_grant` |
| Upkeep | `crank`: books revenue, advances the drip, closes auctions, opens decisions, resolves them |

- Anyone can call `crank`. Whoever's call opens a decision pays the rent for its accounts.
- Proposal and bid accounts can be closed after their auction to return their rent.
- Emit events for every trade, revenue booking, bid, lead change, phase change and resolution, so a UI or indexer can follow along.

**Must-haves**
- Every account is validated by PDA seeds, mint and owner.
- Only the program's PDAs can mint or burn.
- No admin key, and the creator has no special powers after launch.
- Math is checked and fixed-point, and `r` never goes below `MIN_R`.
- The program never calls any program other than SPL Token, Associated Token, System and Metaplex metadata.
- Accounting is conserved: the vault always holds at least the sum of its recorded balances, and any excess is revenue. During a decision, each world's claims on real assets add up to exactly what the program holds for them.
- The program is tested on a local validator, including full proposal → auction → decision → payout cycles where PASS wins, where FAIL wins, and where the grant no longer fits. Property tests cover conservation and show that no trade sequence profits off the curve.

## Web UI

- **Launch:** pick a quote asset, the opening price, the deposit, the settings (prefilled with defaults), and the name, symbol and image.
- **Trade:** a quote with price impact and slippage, plus the current `r`. Shows when trading is paused for a decision.
- **Governance:** proposals with their total bids and lead time, and bidding. Each grant shows its recipient, and says when the recipient is a program. During a decision: a live chart of PASS and FAIL prices with each side's lead time, plus split, merge and trade. After it: redeem.
