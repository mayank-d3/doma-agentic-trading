# Doma Automated Trading (prototype)

Pick a strategy, set a spending budget, hit play. An agent runs it on your wallet, on-chain, while you are away, all riding on the limit-order infra we are already building. No code, no separate bot platform.

**Live links** (open directly in the browser, built on the real Doma design system, share either with anyone):

- **Interactive prototype:** https://mayank-d3.github.io/doma-agentic-trading/ (clickable, all six screens)
- **Overview / concept page:** https://mayank-d3.github.io/doma-agentic-trading/concept.html (the same walkthrough as a styled one-page preview)

## The flow

Grid trading and buy-low / sell-high both run through the same four steps; only the configure step changes shape.

1. **Choose a strategy.** A small gallery of playbooks (Grid trading, Buy low / sell high, DCA), each with a plain-English one-liner and a "best for" tag.
2. **Configure, with the orders drawn on the chart.** Set the range, grid count, and amount. A live price chart shows exactly where each buy sits below price and each sell above it. Auto shows a suggested range; Manual exposes every field. Buy-low / sell-high collapses this to one buy price, one sell price, and an amount.
3. **Authorize, by approving a budget.** You approve a spending budget for the agent (works with any wallet). It can only trade Doma pairs, can never spend more than you approve, and you revoke anytime by removing the approval. The approval amount is the hard cap.
4. **Monitor, live.** Realized grid profit sits next to unrealized P&L, with executed fills plotted on the grid. Run several at once from a "My strategies" view.

## What this needs from the infra

The split is clean, and it is about execution actions, not triggers.

**Ready on the limit-order infra as-is (a single pass)**

- **Grid trading**: a ladder of resting buy/sell limit orders placed across your range.
- **Buy low, sell high (Run once)**: buy at your low, sell at your high, then stop.
- **DCA on a timer**: a scheduled buy (already working in the current POC).

**Needs one more building block**

- **Repeating (grid loop / "repeat" buy-low-sell-high)**: to keep cycling, it must re-arm the opposite order on each fill. That re-arm-on-fill is a chaining primitive the MVP does not have yet; it belongs in the orders-worker, not the browser. In the prototype this is the "Repeat / Needs re-arm (roadmap)" state.
- **Buy during bonding**: the trigger fits, but a pre-graduation token has no Uniswap V3 pool yet, so it needs a bonding-curve fill path alongside the swap-based fill.
- **Buy and create LP**: the buy is a normal limit order; the "create LP" half is a mint, not a swap, so it needs an LP-provisioning action chained after the fill.

## Under the hood

- **Trigger:** the limit-order backend already watches Uniswap swaps and fills resting orders. A limit order is a keeper swap when price crosses your level, with slippage protection (minimum received).
- **Where it runs:** the hot loop lives on the server (orders-worker), so it keeps running when the tab is closed. The browser only configures and displays.
- **Custody:** approval model. You approve a budget for the router; the approval amount is the hard cap, and it works with any wallet (MetaMask included). No delegated key.
- **Fairness:** a grid's orders spread across many price levels, so the "who fills first" problem mostly dissolves. v1 is first-come-first-serve with disclosures.

## Screens

Choose strategy, Configure grid, Configure buy-low / sell-high, Authorize, Monitor, My strategies.

Built in Claude Design on the Doma design system.
