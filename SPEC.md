# Market Supply Token (MST)

A Solana program for launching tokens whose supply is controlled by the market, and whose grants are decided by futarchy.

## What it does

1. **Launch.** Anyone creates a token and picks the **quote asset** it trades against: SOL or any SPL token. The program owns the token's mint authority, so nobody can mint outside the rules below.
2. **Bonding curve.** The token is always bought from and sold to its own curve. Buying pays quote into the token's **reserve** and mints new tokens; selling burns tokens and pays quote out of the reserve. Supply is exactly what the market has bought, and the price follows the curve.
3. **Revenue and grants.** Anyone can pay revenue into a token's reserve. The reserve can pay **grants**, for example to builders or marketing, without changing the token's price.
4. **Auctions.** Anyone can propose a grant or a change to the token's settings. Proposals compete in an **auction**: backers bid quote on the proposals they want decided, and the highest bid wins the right to go to a vote.
5. **Futarchy.** The winning proposal is decided by **PASS/FAIL markets**. Traders trade the token in a "proposal passes" market and a "proposal fails" market. If the token trades higher in the PASS market, the proposal is enacted.

## Bonding curve

The curve has a reserve `R`, a token supply `S` and a **reserve ratio** `r` between 0 and 1. Its price is the Bancor formula:

```text
price = R / (r · S)
buy with d quote:   tokens out = S · ((1 + d/R)^r − 1)
sell y tokens:      quote out  = R · (1 − (1 − y/S)^(1/r))
```

- `r = 1` is a flat curve: buying and selling don't move the price.
- A smaller `r` is a steeper curve: buys push the price up faster, and sells push it down faster.
- The creator funds the opening reserve and picks the opening price. The creator receives the opening tokens.
- Every trade carries a `min_out`, so the user's slippage is bounded.
- Rounding always favours the reserve, so no buy-then-sell sequence makes money off the curve itself.

## Revenue and grants

- **Revenue goes into the reserve.** Anyone can add revenue:
  - any wallet or program can pay quote in with `contribute`, for example a project sharing its income, a partner app routing fees, or a donation;
  - quote sent straight to the token's vault counts too: anything beyond the recorded balances is revenue, and the next instruction or `crank` books it;
  - winning auction bids;
  - an optional trading fee the creator can switch on.

  Adding revenue to the reserve raises `r` toward 1 and does not change the price.
- **Overflow.** `r` can never go above 1. Revenue that arrives while the curve is already flat is held aside as **overflow** instead. It is the only extra balance, and it moves into the reserve whenever a grant has made room again.
- **A grant** is paid from overflow first, and any shortfall from the reserve. Taking reserve does **not** change the price: the program lowers `r` instead, so the curve gets steeper and holders' tokens keep their current price. Later revenue raises `r` back to 1.

## Auctions

- During a proposal window, anyone can **propose**:
  - a grant (recipients and amounts), or
  - a new value for one token setting (fee, window lengths, decision length, margin, and so on).
- Anyone can **bid** quote on a proposal; the total bid is its score.
- When the window closes, the **highest-bid proposal** goes to a futarchy decision. A grant the reserve and overflow can't fund, or a proposal with an invalid setting, is skipped.
- The winner's bids become revenue. Every other bid is refunded to its bidder.

## Futarchy

- **Conditional tokens.** When a decision opens, the program creates PASS and FAIL versions of both the token and the quote asset. Anyone can **split** 1 real unit into 1 PASS unit and 1 FAIL unit, and **merge** them back at any time.
- **Two markets.** The PASS market and the FAIL market each get a copy of the token's curve. Traders buy and sell PASS-token with PASS-quote, and FAIL-token with FAIL-quote. Normal buying and selling of the real token pauses during the decision.
- **The decision rule.** Over the decision period, the program compares the two markets' prices. PASS wins if its time-weighted price beats FAIL's by the configured margin.
  - A simple, crank-free way to measure it: split the decision into fixed slot periods, and score each period at its closing prices.
  - Prices only change when someone trades, so any instruction can score all elapsed periods first, then apply its own trade.
- **Resolution:**
  - The winning market's curve becomes the token's curve.
  - Winning conditional units redeem 1:1 for the real assets; losing units are worthless.
  - A passed grant is paid to its recipients, and a passed setting takes effect.
  - If a grant fails, its funds go back as revenue.

## Program outline (Anchor)

**Accounts**
- `TokenState`: quote mint, the token's single quote vault, `R`, `S`, `r`, overflow, settings and current phase. The vault holds all of the token's quote: reserve, overflow and held bids are balances recorded inside it.
- `Proposal`: kind, recipients or setting, total bid and status.
- `Bid`: one bidder's amount on one proposal.
- `Decision`: the PASS and FAIL curves, conditional mints and vaults, scoring state and outcome.

**Instructions**

| Group | Instructions |
|---|---|
| Launch and trade | `create_token`, `buy`, `sell`, `contribute` |
| Auctions | `propose`, `bid`, `refund_bid` |
| Futarchy | `split`, `merge`, `trade_conditional`, `resolve`, `redeem`, `claim_grant` |
| Upkeep | `crank` (permissionless phase transitions) |

Emit events for every trade, bid and decision step, so a UI or indexer can follow along.

**Must-haves**
- Every account is validated by PDA seeds, mint and owner.
- Only the program's PDAs can mint or burn.
- There is no admin key over funds.
- Math is checked and fixed-point.
- Accounting is conserved: the vault always holds at least the reserve + overflow + held bids, and any excess is revenue.
- The program is tested on a local validator, including a full proposal → auction → futarchy → payout cycle.

## Web UI

- **Launch:** pick a quote asset, the opening price and the deposit.
- **Trade:** a quote with price impact and slippage.
- **Governance:** proposals and bids, a live PASS-versus-FAIL price chart for the current decision, split, merge and trade, then redeem and claim after resolution.
