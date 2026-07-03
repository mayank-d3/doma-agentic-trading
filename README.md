# Doma Automated Trading (prototype)

Pick a strategy, set a spending cap, hit play. An agent runs it on your wallet, on-chain, while you are away, all riding on the limit-order infra we are already building. No code, no separate bot platform.

**Live prototype:** https://mayank-d3.github.io/doma-agentic-trading/

Clickable, all six screens, built on the real Doma design system. Send the link to anyone.

## The flow

Grid trading and buy-low / sell-high both run through the same four steps; only the configure step changes shape.

1. **Choose a strategy.** A small gallery of playbooks (Grid trading, Buy low / sell high, DCA), each with a plain-English one-liner and a "best for" tag.
2. **Configure, with the orders drawn on the chart.** Set the range, grid count, and amount. A live price chart shows exactly where each buy sits below price and each sell above it. Buy-low / sell-high collapses this to one buy price, one sell price, and an amount.
3. **Authorize, once and capped.** One approval lets the agent trade on your behalf, but only on Doma pairs and only up to the cap you set. Pause or revoke anytime.
4. **Monitor, live.** Realized grid profit sits next to unrealized P&L, with executed fills plotted on the grid. Run several at once from a "My strategies" view.

## What this needs from the infra

The split is clean, and it is about execution actions, not triggers.

**Ready on the limit-order infra as-is**

- **Grid trading**: a ladder of resting buy/sell limit orders that re-arm the opposite side on each fill.
- **Buy low, sell high**: a one-level grid: buy at your low, sell at your high, repeat.
- **DCA on a timer**: a scheduled buy (already working in the current POC).

**Needs one more building block**

- **Buy during bonding**: the trigger fits, but a pre-graduation token has no Uniswap V3 pool yet, so it needs a bonding-curve fill path alongside the swap-based fill.
- **Buy and create LP**: the buy is a normal limit order; the "create LP" half is a mint, not a swap, so it needs an LP-provisioning action chained after the fill.

## Under the hood

- **Trigger:** the limit-order backend already watches Uniswap swaps and fills resting orders. A grid is just a set of those orders that re-arms on fill.
- **Where it runs:** the hot loop lives on the server (orders-worker), so it keeps running when the tab is closed. The browser only configures and displays.
- **Extra infra:** none for grid / buy-low, as long as the re-arm-on-fill is a first-class primitive in the orders-worker rather than a browser loop.
- **Fairness:** a grid's orders spread across many price levels, so the "who fills first" problem mostly dissolves. v1 is first-come-first-serve with disclosures.

## Screens

Choose strategy, Configure grid, Configure buy-low / sell-high, Authorize, Monitor, My strategies.

Built in Claude Design on the Doma design system.
