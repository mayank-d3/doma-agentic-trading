# Doma Automated Trading (prototype)

Pick a strategy, set a spending budget, hit play. An agent runs it on your wallet, on-chain, while you are away, all riding on the limit-order infra we are already building. No code, no separate bot platform.

**Live links** (open directly in the browser, built on the real Doma design system, shareable with anyone):

- **Interactive prototype:** https://mayank-d3.github.io/doma-agentic-trading/ (clickable, every screen)
- **Overview / concept page:** https://mayank-d3.github.io/doma-agentic-trading/concept.html (the same walkthrough as a styled one-page preview)

## The flow

Every strategy runs through the same four steps; only the configure step changes shape.

1. **Choose a strategy.** A gallery of playbooks (Grid trading, Buy the floor discount, Buy low / sell high, DCA), each with a plain-English one-liner and a tag.
2. **Configure, with the orders drawn on the chart.** Each strategy has its own configure screen with a live price chart, validation, and a fee-aware profit line. Grid has Auto (a suggested range from recent volatility) and Manual (every field, with error states when the range inverts, an order falls below the minimum size, or the per-grid profit is below fees). Buy low / sell high collapses to one buy price, one sell price, and an amount. DCA takes an amount, a frequency, and a budget. Floor discount takes a minimum discount to the on-chain floor.
3. **Authorize, by approving a budget.** You approve a spending budget for the agent (works with any wallet). It can only trade Doma pairs, can never spend more than you approve, and you revoke anytime by removing the approval. The approval amount is the hard cap.
4. **Monitor, live.** Realized grid profit sits next to unrealized P&L, with executed fills in the activity feed. Pause or close a strategy (closing offers to sell back to USDC or keep the tokens), read the matched-trades history and parameters, and handle out-of-range and budget-used states. Run several at once from a "My strategies" dashboard.

## The four strategies

- **Grid trading.** A ladder of resting buy/sell limit orders across a range, buying each dip and selling each bounce.
- **Buy the floor discount (Doma exclusive).** Domain tokens carry an on-chain buyout floor (the highest of the token's minimum buyout price and its market-based valuations). This strategy buys only when the market trades below that floor by at least the discount you set. It maps directly onto one conditional buy order on the limit-order infra, and it is the one strategy no generic exchange can offer.
- **Buy low, sell high.** One buy price, one sell price. Technically a one-level grid; the beginner on-ramp.
- **DCA on a timer.** A fixed buy on a schedule. Note that DCA is time-triggered while the limit-order engine is price-triggered, so DCA needs a scheduler extension on top of the order primitive.

## The multi-strategy model

One wallet approval is the umbrella cap. Each strategy allocates a budget under that single approval. If a new strategy fits inside your unallocated approval, activating it needs no wallet signature at all; if it does not, you top the approval up (one signature, the approval goes from its current total to the new total). Revoking the one approval stops every strategy instantly and cancels every resting order, while your tokens stay in your wallet. The wallet chip opens this view (total, allocated, per-strategy) from any screen.

## Why not just provide liquidity?

A narrow Uniswap V3 LP band placed entirely above or below the current price is a range order, so one grid level looks a lot like one LP band. The products still differ in ways that matter:

- **Fills are one-way vs reversible.** A filled limit order stays filled. An LP band converts back if price re-crosses it, so to lock a fill you would need a bot to withdraw the position on the cross.
- **You pay the fee vs earn it.** Trading through the limit-order infra makes you the taker: you pay the pool fee plus slippage on each fill. As an LP you are the maker: you earn that fee. On a 1 percent pool a full grid cycle costs roughly 2 percent as a taker, which is exactly why the configure screen is fee-aware.
- **A limit order can say no.** It has an exact trigger, slippage protection, and a stop loss. An LP band below price buys the falling token unconditionally and cannot express a stop.
- **Coverage.** Conditional orders work on bonding-curve tokens that have no pool yet; LP needs a live pool.

So the feature adds real value today (stop-losses, sticky fills, bonding-curve coverage, a retail-comprehensible model). The compounding future version places each grid level as a narrow LP band and withdraws it on the cross, so the grid earns fees per cycle instead of paying them. That depends on LP-position primitives from the infra team (see open questions).

## What this needs from the infra

The split is about execution actions, not triggers.

**Ready on the limit-order infra as-is (a single pass)**

- **Grid trading (one pass)**, **Buy low, sell high (Run once)**, **Floor discount** (a conditional buy at floor times one minus your discount), and **DCA** as a single scheduled buy (already working in the current POC, modulo the scheduler).

**Needs one more building block**

- **Repeating (grid loop / "repeat" buy-low-sell-high):** to keep cycling it must re-arm the opposite order on each fill. That re-arm-on-fill is a chaining primitive the MVP does not have yet; it belongs in the orders-worker, not the browser.
- **DCA scheduling:** a time trigger, since the engine is price-triggered today.
- **Buy during bonding:** the trigger fits, but a pre-graduation token has no Uniswap V3 pool yet, so it needs a bonding-curve fill path alongside the swap-based fill.
- **Buy and create LP / fee-earning LP-mode grid:** the "create LP" half is a mint, not a swap, so it needs an LP-provisioning action chained after the fill.

## Open questions for the limit-order team

- **Strategy grouping:** orders can be filtered by wallet address, but there is no field to say "these orders belong to this grid." A grid needs a strategy id or group on the order so the UI can show, cancel, and account for a strategy as a unit.
- **Re-arm on fill:** is chaining (place the opposite order when one fills) a first-class primitive in the orders-worker, or does the strategy layer poll and re-place?
- **Per-pair FIFO vs a many-order grid:** one order per pair at a time serializes a grid against itself and against other users on the same pair. How does a multi-order grid coexist with that guard?
- **LP-position primitives:** can the infra mint and withdraw a narrow range position on a trigger? That unlocks the fee-earning LP-mode grid and the buy-and-create-LP strategy.
- Plus fill notifications (push vs poll), realized-vs-unrealized P&L accounting, and batch cancel / expiry / revoke as a kill switch.

## Screens

Choose strategy, Configure grid, Configure buy-low / sell-high, Configure DCA, Configure floor discount, Authorize, Monitor, My strategies.

Built in Claude Design on the Doma design system.
