# Doma Automated Trading Prototype v2 Refinement Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Execution note:** This plan modifies a Claude Design prototype, not application code. "Implementation" for Tasks 2 to 6 means pasting a prompt into the Claude Design editor in the user's Chrome (they are logged in), reviewing the result, and iterating once if items are missing. "Tests" are visual verification checklists. Tasks 2 to 6 CANNOT be done by parallel subagents because they all mutate the same design document; run them sequentially in one browser session. Tasks 1, 7, 8 are normal file/CLI work.

**Goal:** Upgrade the hosted prototype from a single-strategy happy path to a realistic multi-strategy product: per-strategy configure screens, real validation and error states, approval reuse across strategies, a working monitor (pause/close/history/parameters), and a portfolio dashboard that manages many strategies across many tokens.

**Architecture:** The prototype is a Claude Design project ("Doma Automated Trading", D3Inc org) exported as a self-contained standalone HTML, hosted on GitHub Pages from `mayank-d3/doma-agentic-trading` (`index.html`). Changes are made by prompting Claude Design, re-exporting the standalone HTML, replacing `index.html`, and pushing. `README.md` and `concept.html` document the concept and must stay in sync.

**Tech Stack:** Claude Design editor (claude.ai/design), Claude-in-Chrome MCP for driving the browser, GitHub Pages for hosting, `git`/`gh` CLI (authenticated as `mayank-d3`).

## Key resources

- Design editor project: https://claude.ai/design/p/5cdc272e-bebf-4540-b0ab-09920e55f4dc?file=Automated+Trading.dc.html
- Live prototype: https://mayank-d3.github.io/doma-agentic-trading/
- Live overview doc: https://mayank-d3.github.io/doma-agentic-trading/concept.html
- Local repo: `~/Desktop/d3-repos/doma-agentic-trading`
- Backend grounding: `doma-backend/docs/adr/conditional-orders.md` (limit-order ADR: approval-to-operational-EOA funding, `DeFiOrder` with `expiresAt` and `minOutputAmount`, per-pair FIFO, no strategy grouping field yet)

## Global Constraints

- No em-dashes anywhere: prototype copy, README, concept doc. Use plain punctuation.
- Keep the existing Doma dark-glass design language exactly: same palette, glass cards, gradient icon tiles, mono font for all numerals. Never let Claude Design restyle existing screens.
- Keep the 4-step flow stepper (Choose strategy, Configure, Authorize, Monitor) and the "My strategies" nav item.
- All authorization copy must describe the approval model (wallet approval is the hard cap, revocable anytime). Never reintroduce "session key" or "delegated wallet" language.
- Buy strategies never promise guaranteed profit. APR keeps its asterisk footnote.
- Quote currency is USDC everywhere.
- Every state added must be a state a real user of the limit-order infra could reach (grounded in the ADR above).
- Each Claude Design prompt below is final copy: paste verbatim. If pasting collapses into a "Pasted text" attachment chip, that is fine; add one line "Apply the attached changes to the current design." and send.

## Strategy-side scope decisions (feature necessity audit, 2026-07-13)

Decisions from the journey-by-journey audit. Tasks below already reflect them; this section records the why.

**Keep:**
- Auto AND Manual grid modes. Auto is the 90% case; Manual is the power-user path. Requirement: Auto must display its rationale (range derived from 7d volatility) or users will not trust it.
- Buy low sell high as its own card. Technically a 1-grid grid, but it is the beginner on-ramp and costs nothing extra on the infra.
- DCA card, with honesty: the limit-order engine is price-event driven; DCA is time driven and needs a scheduler extension (open question already filed to the infra team). Docs must say so.
- Stop loss in Advanced: one conditional sell order, native to the infra, and the one protection LP structurally cannot provide. Highest value per unit of complexity.
- Trigger price in Advanced: deferred activation, trivial.

**Cut:**
- Arithmetic/Geometric spacing toggle. Geometric only matters on ranges wider than about 2x; on a $1.12 to $1.72 domain-token grid it is imported CEX noise. Remove it from the existing screen.
- USD take-profit. Requires realized-P&L accounting the infra does not have yet. Deferred until the P&L open question is answered.
- "Edit parameters" on Monitor. The infra cannot edit orders in place (cancel and re-place only); replaced with "Duplicate with new settings".

**Add:**
- "Buy the floor discount" as a fourth strategy card with its own configure screen. The only Doma-unique strategy: buy when a fractional token trades below its on-chain buyout floor (buyout = max of MBP, FDV, FDV twap; permissionless redemption). Maps directly onto a conditional buy at floor times (1 minus discount). Grid is a commodity; this is the differentiator.
- Manage-approval modal from the wallet chip (view total/allocated per strategy, revoke with consequences). The Authorize screen promises "revoke anytime"; there must be a place to do it. Promoted from the deferred list.
- Budget top-up modal (destination for the "Increase budget" button in the cap-exhausted state).
- Portfolio banner state for "approval revoked externally, all strategies stopped".
- "Why not just provide liquidity?" section in the docs: taker grid pays pool fee plus slippage per fill while an LP range order earns the fee but is reversible, cannot stop-loss, and cannot cover bonding-curve tokens. Future direction (new open question to infra team): LP-position primitives so grid levels can be placed as fee-earning narrow LP bands, withdrawn on cross (also covers Inder's buy-and-create-LP strategy and the ADR's volume-based LP provisioning line).

## Current prototype inventory (verified live 2026-07-13, by clicking through every control)

Screens: Choose strategy, Configure grid trading, Authorize, Monitor, My strategies (portfolio).

Verified defects and gaps:

1. All three strategy cards (Grid, Buy low sell high, DCA) navigate to the same "Configure grid trading" screen. The other two strategies have no configure screens.
2. Monitor: `Pause` and `Close` buttons have no click handlers at all (dead controls). No confirmation modal, no paused state, no close semantics.
3. Monitor: `History` and `Parameters` tabs both render the same placeholder line ("12 matched trades, see Activity below"). Stubs.
4. Configure: `Advanced · trigger, stop-loss, take-profit` row does not expand. Dead control.
5. Portfolio: only the DROPLINE card is clickable; VAULT and NEXUS cards are dead. No aggregate stats beyond "Total value $418", no status chips beyond a green dot, no spend meters, no pause/resume, no empty state.
6. Configure has zero validation or error states. No wallet balance shown, no Max button, no fee awareness, no expiry field, no sell-side inventory explanation.
7. Authorize always shows a fresh $200 approval. Creating a second strategy would re-ask for $200 with no concept of reusing or topping up the existing wallet approval.
8. Chart on Configure is static: Manual price inputs do not move the B/S lines, and the range is not draggable.
9. No error/edge states anywhere: out-of-range has only a tiny "idle · out of range" chart label, no cap-exhausted state, no approval-revoked state, no failed-fill state.

---

### Task 1: Verify working copy and tooling

**Files:**
- Verify: `~/Desktop/d3-repos/doma-agentic-trading/index.html` (2.6 MB standalone export)
- Verify: `~/Desktop/d3-repos/doma-agentic-trading/README.md`, `concept.html`

**Interfaces:**
- Produces: a clean local clone on `main`, `gh` authenticated, Chrome connected via Claude-in-Chrome with the user logged into claude.ai.

- [ ] **Step 1: Verify the clone is present and clean**

Run: `cd ~/Desktop/d3-repos/doma-agentic-trading && git status --short && git log --oneline -1`
Expected: no output from status (clean), one commit line. If the directory is missing: `git clone https://github.com/mayank-d3/doma-agentic-trading ~/Desktop/d3-repos/doma-agentic-trading`

- [ ] **Step 2: Verify gh auth**

Run: `gh auth status`
Expected: `Logged in to github.com account mayank-d3`

- [ ] **Step 3: Verify browser access to the design editor**

Load Claude-in-Chrome tools via ToolSearch, call `tabs_context_mcp`, then `navigate` to `https://claude.ai/design/p/5cdc272e-bebf-4540-b0ab-09920e55f4dc?file=Automated+Trading.dc.html`. Screenshot.
Expected: the Doma Automated Trading editor loads with the chat/prompt input visible on the left and the canvas on the right. If it shows a login wall, stop and ask the user to log in; never enter credentials.

---

### Task 2: Configure screen v2 (validation, funding clarity, working Advanced)

**Files:**
- Modify: Claude Design document "Automated Trading.dc.html" (Configure screen)

**Interfaces:**
- Consumes: existing Configure screen (Auto/Manual toggle, token grid, range inputs, grid stepper, investment amount).
- Produces: Configure screen with wallet balance, Max button, funding split note, expiry select, three visible validation states, fee-aware profit line, expanded Advanced section. Later tasks (4, 6) reference the same $500 balance and a $400 total approval; keep them consistent.

- [ ] **Step 1: Paste this prompt into the Claude Design chat and send**

```
Refine ONLY the "Configure grid trading" screen. Do not restyle anything else, keep the exact same dark glass design language, colors, and mono numerals. No em-dashes anywhere, use plain punctuation.

1. Under "Investment amount", add a muted helper row: "Wallet balance: 500.00 USDC" on the left and a small "Max" pill button on the right end of the input.

2. Below the investment input add a funding split note in a subtle info card: "Buys are funded from USDC. Sell levels arm as your buy levels fill. To arm sells immediately, the agent buys 42.1 DROPLINE (about $60) at market when you activate."

3. Add a compact "Expires" field after the investment amount: a segmented control with options "7 days", "30 days", "Never", default 30 days, with helper text "Open orders are cancelled automatically when the strategy expires."

4. Rename the "Advanced · trigger, stop-loss, take-profit" row to "Advanced · trigger, stop-loss" and make it actually expand on click. Inside, two optional inputs, each with a leading toggle, both default off: "Trigger price" with helper "Start placing orders only when price reaches this." and "Stop loss" with helper "If price falls to this level, cancel all orders and sell holdings back to USDC." Do NOT include a take-profit field.

5. Add validation states, visible in Manual mode when values are bad, styled as small red text under the offending field with a red border on the input:
   a. If lower price >= upper price: "Lower price must be below upper price."
   b. If per-grid order size would fall below $5: "Each grid order would be $3.20. Minimum is $5. Reduce grids or add investment."
   c. Under the profit line, when spacing is too tight, show an amber warning: "Profit per grid (0.4%) is below estimated fees and slippage (0.7%). This grid would lose money on each cycle. Widen the range or reduce grids."

6. Change the profit summary line to be fee aware: "Profit per grid ~1.8% (~1.1% after fees) · 6 orders · ~$33 each".

7. When the range does not include the current price, show an info note under the range inputs: "Range is entirely below current price. This grid will only place buy orders until price enters the range." (or "above ... only sell orders").

8. In Manual mode, the price inputs must update the chart lines: editing lower or upper price repositions the B and S levels on the chart. Also show current values pre-filled: lower 1.12, upper 1.72.

9. Token cards: under each token name add its 24h change in small green or red text (DROPLINE +2.4%, VAULT -1.1%, NEXUS +0.8%, ORBIT +5.2%), and on the DROPLINE card add a tiny cyan dot with tooltip-style label "1 strategy running" to hint that a strategy already exists on this token.

10. REMOVE the "Arithmetic / Geometric" segmented control from Manual mode entirely. Grid spacing is always even.

11. In Auto mode, extend the "Suggested Range & grids" card with one rationale line under the numbers: "Based on 7d volatility of ±18% around the current price." Keep the "Switch to Manual to fine-tune." line.

12. Keep the "Search domain tokens" input visible in Manual mode too (today it only shows in Auto).
```

- [ ] **Step 2: Wait for the generation to finish, then verify in the editor preview**

Checklist (walk the Configure screen in preview/canvas):
- Wallet balance row and Max pill visible under investment amount
- Funding split info card present with the exact buy-upfront copy
- Expires segmented control with 7 days / 30 days / Never
- Advanced row expands and shows Trigger price and Stop loss only (no take-profit)
- Arithmetic/Geometric toggle is gone from Manual mode
- Auto mode shows the volatility rationale line; Manual mode shows the token search input
- Switching to Manual and breaking a value shows the red validation copy (at minimum the states render somewhere visibly as designed states)
- Profit line shows the after-fees number
- DROPLINE token card shows "1 strategy running" marker

- [ ] **Step 3: If any checklist item is missing, send ONE follow-up prompt listing only the missing items**

Format: "These items from my last request are missing or wrong, fix only these, change nothing else:" followed by the numbered missing items copied verbatim from the prompt above.

---

### Task 3: Per-strategy configure screens (Buy low sell high, DCA, Floor discount)

**Files:**
- Modify: Claude Design document (Choose strategy routing, three new screens, one new card)

**Interfaces:**
- Consumes: Choose strategy cards, existing Configure grid screen patterns (token grid, investment amount, wallet balance from Task 2).
- Produces: "Configure buy low, sell high", "Configure DCA", and "Configure floor discount" screens; a fourth "Buy the floor discount" card; all four cards route to their own screen; all four continue to Authorize.

- [ ] **Step 1: Paste this prompt and send**

```
Right now all three strategy cards open "Configure grid trading". Fix the routing and add two new configure screens. Reuse the exact layout system of the grid configure screen: chart or visual on the left, controls on the right, same token selector cards, same investment amount block with wallet balance and Max from the grid screen, same "Activate" button style, same "Back to strategies" link. Keep the flow stepper on "Configure". No em-dashes.

Screen A: "Configure buy low, sell high" (opens from the Buy low, sell high card)
- Left: same price chart but with exactly two lines: a green "Buy at $1.30" line below current price and a red "Sell at $1.60" line above it.
- Right: token selector, then "Buy price" and "Sell price" dollar inputs, then "Amount per cycle" USDC input, then a "Repeat" toggle row: when on, helper text "Restart the cycle after each sell fills. Runs until you stop it or the budget is used."; when off, "Run once, then stop."
- Summary line: "Profit per cycle ~18% (~17% after fees) if both fills execute at your prices."
- Validation state: if buy price >= sell price, red text "Buy price must be below sell price."
- Same Expires control as the grid screen.
- Button: "Activate strategy", goes to the Authorize screen.

Screen B: "Configure DCA" (opens from the DCA on a timer card)
- Left: chart showing a flat dotted accumulation line with small buy dots at regular intervals.
- Right: token selector, then "Buy amount" USDC input (default 25), then "Frequency" segmented control: Daily, Weekly, Monthly (default Weekly), then "Total budget" USDC input (default 200) with helper "The strategy pauses when the budget is spent."
- Summary line: "8 buys of $25 over 8 weeks at current settings."
- Info note: "DCA buys on a schedule, not on price moves. Each buy executes at market price at the scheduled time, with slippage protection."
- Same Expires control.
- Button: "Activate strategy", goes to the Authorize screen.

Screen C: "Configure floor discount" plus a NEW fourth card on the Choose strategy screen.
- New card on Choose strategy, placed second (after Grid trading), with a shield-style gradient icon: title "Buy the floor discount", body "Domain tokens carry an on-chain buyout floor. This agent buys only when the market trades below that floor, an entry unique to Doma.", tag pill "Doma exclusive".
- Screen C layout, same system as the others. Left: price chart with a solid amber horizontal line labeled "Buyout floor $1.55" above the current price line "$1.42", and the gap between them shaded subtly. Right: token selector where each token card also shows its floor: DROPLINE "floor $1.55", VAULT "floor $2.80", NEXUS "no floor", ORBIT "floor $1.90". The NEXUS card is disabled at reduced opacity with the label "No buyout floor set".
- Controls: "Minimum discount" stepper in percent (default 8%) with helper "Buy only when price is at least this far below the floor.", "Per-buy size" USDC input (default 25), "Total budget" USDC input (default 200).
- Live summary card in mono: "Floor $1.55 · Market $1.42 · Discount 8.4% · Buying now qualifies". Also design its waiting variant, shown when the minimum discount is raised above the current discount (for example 12%): "Floor $1.55 · Market $1.42 · Discount 8.4% · Below your 12% minimum, order rests until it qualifies" in muted amber.
- Info note: "The floor is the token's on-chain buyout value (the highest of its minimum buyout price and market-based valuations). Trading below it is a discount to redeemable value, not a guarantee of profit."
- Same Expires control. Button: "Activate strategy", goes to Authorize.

Routing: Grid trading card keeps opening Configure grid trading. Buy low, sell high card opens Screen A. DCA on a timer card opens Screen B. Buy the floor discount card opens Screen C. Keep all four cards on one row or a 2x2 grid, whichever fits the existing card width.
```

- [ ] **Step 2: Verify in preview**

Checklist:
- Choose strategy now shows four cards; clicking each opens a different configure screen
- Screen A shows exactly two price lines and the Repeat toggle with both copy variants
- Screen B shows amount, frequency, total budget, and the schedule-not-price info note
- Screen C shows the floor line on the chart, per-token floors, the disabled no-floor NEXUS card, the discount stepper, and the "Buying now qualifies" summary
- All new screens continue to Authorize via their Activate buttons

- [ ] **Step 3: One follow-up prompt for missing items only (same format as Task 2 Step 3)**

---

### Task 4: Authorize v2 (approval reuse and top-up)

**Files:**
- Modify: Claude Design document (Authorize screen)

**Interfaces:**
- Consumes: Authorize screen; $500 wallet balance from Task 2.
- Produces: Authorize screen showing existing approval, allocation math, and a top-up variant. Consistent story across screens: the wallet already holds a $400 total approval with $350 allocated to the four running strategies shown on the portfolio (Task 6); this screen demonstrates adding a fifth strategy that needs $200, bumping the approval to $550.

- [ ] **Step 1: Paste this prompt and send**

```
Refine ONLY the "Authorize the trading agent" screen. Keep every existing element (wallet card, four guarantee rows, cap explanation). No em-dashes.

1. Add an "Existing approval" summary card above the approval amount: "Current approval: $400 · Allocated to running strategies: $350 · Available for new strategies: $50" with the three numbers in mono font.

2. Make the approval amount an editable-looking input (mono, dollar prefix) instead of static text, pre-filled with 200.

3. Below it, show context-aware helper copy: "This strategy needs $200. You have $50 unallocated. Approving will increase your total approval from $400 to $550." Show a small before/after row: "$400 -> $550" in mono.

4. Rename the primary button to "Increase approval to $550". Keep Cancel.

5. Add a smaller variant note under the button: "If a strategy fits inside your unallocated approval, activation needs no wallet signature at all."

6. Add one muted footer line: "One approval covers all your strategies. Revoking it stops every strategy instantly."

7. Make the wallet chip in the top nav (0x8f...c41a) open a small dropdown card on click, available on every screen: title "Wallet approval", rows in mono: "Total approval $400", "Allocated $350", then one line per strategy ("Grid · DROPLINE $200", "Buy low sell high · VAULT $50", "DCA · NEXUS $50", "Grid #2 · DROPLINE $50"), then a red ghost button "Revoke approval" with helper text "Stops all strategies instantly and cancels every resting order. Your tokens stay in your wallet." Clicking Revoke shows a confirm step inside the dropdown: "Revoke $400 approval?" with buttons "Revoke" (red) and "Keep".
```

- [ ] **Step 2: Verify in preview**

Checklist:
- Existing approval card with 400 / 350 / 50 in mono
- Editable approval input prefilled 200
- Before/after row $400 -> $550
- Button reads "Increase approval to $550"
- Footer line about one approval covering all strategies
- Wallet chip opens the approval dropdown with per-strategy allocations and the two-step Revoke

- [ ] **Step 3: One follow-up prompt for missing items only**

---

### Task 5: Monitor v2 (working controls, real tabs, edge states)

**Files:**
- Modify: Claude Design document (Monitor screen)

**Interfaces:**
- Consumes: Monitor screen (tiles, spend meter, tabs, activity feed).
- Produces: working Pause/Resume and Close with confirm modal, filled History and Parameters tabs, out-of-range banner, cap-exhausted paused state. Task 6 links every portfolio card here.

- [ ] **Step 1: Paste this prompt and send**

```
Refine ONLY the Monitor screen (Grid · DROPLINE.ai / USDC). Keep all existing tiles, the spend meter, the chart, and the activity feed. No em-dashes.

1. Make Pause work: clicking Pause switches the strategy to a paused state: status pill changes from green "Active" to amber "Paused", the button becomes "Resume", and an amber banner appears under the header: "Paused. Resting orders stay open but no new orders will be placed. Resume anytime." Clicking Resume returns to Active.

2. Make Close work: clicking Close opens a confirmation modal (dark glass, centered) titled "Close this strategy?" with body "This cancels all 4 resting orders." and a radio choice: "Sell 84.2 DROPLINE back to USDC at market (about $118 after slippage)" (default) or "Keep my DROPLINE tokens". Buttons: "Close strategy" (red) and "Keep running" (ghost). Confirming shows a closed state: grey "Closed" pill, buttons replaced by a single "New strategy" button, banner "Strategy closed. Final result +$18.40 (+9.2%)."

3. Fill the History tab with a real matched-trades table: columns Cycle, Bought, Sold, Profit. Six rows of realistic data like "B1 -> S1, 12.4 DROPLINE at $1.32, sold at $1.52, +$2.48". Greens for profit. Last row shows an open cycle: "B2 -> waiting, bought at $1.22, resting sell at $1.42, unrealized +$0.90".

4. Fill the Parameters tab: a two-column definition list showing Range $1.12 to $1.72, Grids 6, Mode Auto, Investment $200, Per-grid size about $33, Slippage protection 1%, Expires in 26 days, Created 2h 14m ago. Add a ghost "Duplicate with new settings" button with helper text "Running orders cannot be edited in place. This closes the strategy and opens Configure pre-filled with these values."

5. Replace the tiny "idle · out of range" chart label with a proper amber banner above the chart when price is out of range: "Price ($1.85) is above your range ($1.12 to $1.72). All buy levels are resting, no sells remain. Options: wait for price to return, or close and recreate the grid around the new price." with two inline ghost buttons: "Recreate grid here" and "Dismiss".

6. Add a cap-exhausted example state to the spend meter: when spend reaches the cap, the meter turns amber and the helper reads "Budget used. The strategy is paused automatically. Increase the budget or close the strategy." with a small "Increase budget" ghost button. Clicking it opens a small modal: "Increase strategy budget", a dollar input prefilled 100, a mono before/after row "$200 -> $300", helper "Fits inside your unallocated approval, no wallet signature needed.", buttons "Increase" and "Cancel".

7. In the Activity feed, add one failed-fill entry between the existing rows, with a red icon: "Buy at B2 skipped: slippage above your 1% protection. Will retry on the next price cross." and a muted timestamp.
```

- [ ] **Step 2: Verify in preview**

Checklist:
- Pause toggles to amber Paused with banner; Resume restores Active
- Close opens the modal with the sell-back vs keep-tokens radio; confirming shows the Closed state
- History tab shows the matched-pairs table with an open cycle row
- Parameters tab shows the definition list and the Duplicate with new settings affordance
- Increase budget opens the modal with the before/after row
- Out-of-range banner with the two ghost buttons
- Cap-exhausted amber meter state exists (any reachable demo of it is fine)
- One red failed-fill row in Activity

- [ ] **Step 3: One follow-up prompt for missing items only**

---

### Task 6: Portfolio v2 (multi-strategy dashboard)

**Files:**
- Modify: Claude Design document (My strategies screen)

**Interfaces:**
- Consumes: My strategies screen; approval numbers consistent with Task 4 ($400 total, $350 allocated across the four cards: DROPLINE grid $200, VAULT $50, NEXUS $50, Grid #2 $50); Monitor states from Task 5.
- Produces: aggregate header, rich strategy cards all clickable, empty state, same-pair warning.

- [ ] **Step 1: Paste this prompt and send**

```
Refine ONLY the "My strategies" screen into a real multi-strategy dashboard. Keep the dark glass cards and sparklines. No em-dashes.

1. Replace the single "Total value" line with an aggregate header row of four mono metric tiles: "Total value $418", "Total profit +$23.20 (+5.9%)", "Approval allocated $350 of $400" with a thin progress bar, "Active strategies 2 of 4".

2. Upgrade every strategy card to include: strategy type icon and pair title (keep), a status chip top right (green "Active", amber "Paused", amber "Out of range", grey "Closed"), the P/L line (keep), a thin per-strategy spend meter line (DROPLINE grid "$120 of $200", VAULT "$28 of $50", NEXUS "$30 of $50", Grid #2 "$12 of $50"), runtime like "2h 14m", and a small pause/resume icon button that works on the card without opening it.

3. Show four cards to demonstrate all states: DROPLINE grid (Active, the existing one), VAULT buy low sell high (Paused), NEXUS DCA (Active, accumulating), and a second DROPLINE grid labeled "Grid #2" (Out of range). Every card is clickable and opens the Monitor screen.

4. On the "Grid #2" card add a small info row with an amber dot: "Second strategy on DROPLINE/USDC. Strategies on the same pair share order throughput and may fill more slowly."

5. Add sorting/filter chips above the grid: "All", "Active", "Paused", "Needs attention". "Needs attention" highlights the out-of-range card.

6. Add an empty-state variant of this screen (shown when there are no strategies): centered gradient icon tile, title "No strategies yet", body "Pick a playbook and let the agent trade inside a budget you approve.", primary button "Create your first strategy". Reach it via the filter chip "Closed" showing zero results is fine, or as a separate demo state.

7. Make "My strategies" in the top nav show a small count badge (4).

8. Add a demo state for external revocation: a dismissible red banner across the top of the dashboard, "Wallet approval was revoked. All strategies are stopped and resting orders cancelled. Re-approve to resume." with a "Re-approve" ghost button linking to the Authorize screen. It is fine to expose this state via a small hidden demo toggle or a variant frame.
```

- [ ] **Step 2: Verify in preview**

Checklist:
- Four aggregate tiles including the approval-used progress bar
- Four strategy cards, each with status chip, spend meter, runtime, pause/resume icon
- All four cards clickable through to Monitor
- Same-pair warning row on Grid #2
- Filter chips present; "Needs attention" highlights the out-of-range card
- Empty state reachable or demonstrated
- Nav badge on My strategies
- Revoked-approval red banner state exists (any reachable demo of it is fine)

- [ ] **Step 3: One follow-up prompt for missing items only**

---

### Task 7: Export, deploy, verify live

**Files:**
- Modify: `~/Desktop/d3-repos/doma-agentic-trading/index.html` (replace with new export)

**Interfaces:**
- Consumes: finished design from Tasks 2 to 6.
- Produces: updated live prototype at https://mayank-d3.github.io/doma-agentic-trading/

- [ ] **Step 1: Export standalone HTML from Claude Design**

In the editor: Export menu > Standalone HTML. The file downloads to `~/Downloads/` as `Doma Automated Trading*.html` (pick the newest by mtime).

- [ ] **Step 2: Replace index.html and sanity-check the export**

Run:
```bash
cd ~/Desktop/d3-repos/doma-agentic-trading
NEWEST=$(ls -t ~/Downloads/Doma\ Automated\ Trading*.html | head -1)
cp "$NEWEST" index.html
grep -c "Increase approval" index.html      # expect >= 1
grep -c "Needs attention" index.html        # expect >= 1
grep -c "Close this strategy" index.html    # expect >= 1
grep -c "floor discount" index.html         # expect >= 1
grep -c "Revoke approval" index.html        # expect >= 1
grep -c $'—' index.html             # em-dash count in copy; investigate if large and in UI strings (font data can false-positive, check context with grep -o '.\{30\}—.\{30\}')
```
Expected: the three feature greps nonzero.

- [ ] **Step 3: Commit and push**

```bash
git add index.html docs/superpowers/plans/2026-07-13-prototype-v2-refinement.md
git commit -m "prototype v2: per-strategy configure, approval top-up, working monitor controls, multi-strategy portfolio"
git push origin main
```

- [ ] **Step 4: Verify live (real clicks, not element.click)**

Wait ~2 minutes for Pages. Open https://mayank-d3.github.io/doma-agentic-trading/ in the browser pane. IMPORTANT: the export's onclick attributes are stubs; `element.click()` does nothing. Navigate with real mouse clicks at coordinates, or dispatch full PointerEvent+MouseEvent sequences (pointerdown, mousedown, pointerup, mouseup, click) on the element. Verify: all three strategy cards open distinct configure screens; Pause/Resume and Close work on Monitor; all four portfolio cards navigate; the aggregate header renders.

---

### Task 8: Sync README and concept doc

**Files:**
- Modify: `~/Desktop/d3-repos/doma-agentic-trading/README.md`
- Modify: `~/Desktop/d3-repos/doma-agentic-trading/concept.html`

**Interfaces:**
- Consumes: shipped v2 feature set from Tasks 2 to 6.
- Produces: docs that describe v2 accurately; both live links still first.

- [ ] **Step 1: Update README.md**

Read the current README first. Keep its structure (live links at top, strategy feasibility section). Make exactly these content updates:
- In the walkthrough/features section, replace the single-flow description with the v2 description: four strategies (grid, buy low sell high, DCA, floor discount) each with their own configure screen with validation and fee-aware profit lines, approval reuse and top-up on Authorize, wallet-chip approval management with revoke, working pause/close with close options on Monitor, matched-trades history and parameters tabs, out-of-range, cap-exhausted, and approval-revoked states, and the multi-strategy portfolio dashboard (aggregate tiles, status chips, per-card spend meters, same-pair warning, filters, empty state).
- Describe "Buy the floor discount" as the Doma-exclusive strategy: buys only when a fractional token trades below its on-chain buyout floor (the highest of minimum buyout price and market valuations); maps directly to one conditional buy order on the limit-order infra.
- Add a short "Multi-strategy model" paragraph: one wallet approval is the umbrella cap; each strategy allocates a budget under it; activation inside unallocated approval needs no new signature; topping up increases the single approval; revoking stops everything.
- Add a "Why not just provide liquidity?" section, roughly this content: a narrow Uniswap V3 LP band above or below price is a range order, so one grid level looks like one LP band, but the products differ. LP fills are reversible (price re-crossing un-fills you unless a bot withdraws the position), LP earns the pool fee as maker while these strategies pay fee plus slippage as taker, LP cannot express a stop loss or refuse to buy a falling token, and LP requires a live pool so it cannot cover bonding-curve tokens. Conditional orders trade fee income for exact triggers, sticky fills, protection, and a retail-comprehensible model. Note the future direction: placing grid levels as narrow LP bands that are withdrawn on cross would earn fees per cycle instead of paying them, and is an open question to the infra team (LP-position primitives, which would also cover buy-and-provide-LP strategies).
- Be honest about DCA: it is time-triggered, the limit-order engine is price-triggered, so DCA needs a scheduler extension.
- Keep the "open questions for the limit-order team" pointer if present; it still applies (strategy grouping, re-arm on fill, per-pair FIFO), and add the LP-position-primitives question to it.
- No em-dashes.

- [ ] **Step 2: Update concept.html**

Read it first. Mirror the same content updates in its existing HTML structure (it is a small hand-written file, not an export). Update any screen-by-screen walkthrough section to the v2 screens. Keep both CTA buttons pointing at the live prototype. No em-dashes.

- [ ] **Step 3: Verify and push**

```bash
cd ~/Desktop/d3-repos/doma-agentic-trading
grep -c $'—' README.md concept.html   # expect 0 for both
git add README.md concept.html
git commit -m "docs: sync README and concept doc with prototype v2"
git push origin main
```
Then fetch https://mayank-d3.github.io/doma-agentic-trading/concept.html and confirm it renders the v2 content.

---

## Deferred (explicitly out of scope for v2)

- Draggable range handles on the Configure chart (Bybit-style). High effort in Claude Design for low prototype value; the live-updating price inputs from Task 2 item 8 cover the interaction story.
- Notifications/alerts settings (fill alerts, out-of-range alerts). Worth a v3 pass once the team confirms push-vs-poll for fill events (open question already filed to the infra team).
- USD take-profit on strategies. Requires realized-P&L accounting the infra does not have; revisit when the P&L open question is answered.
- Fee-earning LP-mode grid (grid levels placed as narrow LP bands, withdrawn on cross). Depends on LP-position primitives from the infra team; documented as an open question in Task 8, not designed.
- Mobile layout pass.

## Edge-case coverage map (where each lives after v2)

| Edge case | Covered by |
|---|---|
| Lower >= upper price | Task 2 item 5a |
| Per-grid order below minimum | Task 2 item 5b |
| Grid profit below fees | Task 2 items 5c, 6 |
| Range entirely above/below current price | Task 2 item 7 |
| Sell-side inventory bootstrap (two-sided grid needs base tokens) | Task 2 item 2 |
| Investment vs wallet balance | Task 2 item 1 |
| Strategy expiry | Task 2 item 3 (ADR `expiresAt`) |
| Buy low sell high: buy >= sell | Task 3 Screen A validation |
| DCA budget exhaustion | Task 3 Screen B helper |
| Second strategy without new signature / approval top-up | Task 4 |
| Approval revocation kills everything | Task 4 item 6 |
| Pause vs close semantics, close with sell-back choice | Task 5 items 1, 2 |
| Price exits range | Task 5 item 5 |
| Per-strategy cap exhausted, auto-pause | Task 5 item 6 |
| Fill skipped by slippage protection | Task 5 item 7 |
| Two strategies on the same pair (per-pair FIFO contention) | Task 6 item 4 |
| First-time user, zero strategies | Task 6 item 6 |
| Token with no buyout floor (MBP = 0) | Task 3 Screen C disabled card |
| Floor discount not met yet (resting, waiting) | Task 3 Screen C summary states |
| No way to exit the product entirely (revoke) | Task 4 item 7 |
| Approval revoked externally while strategies run | Task 6 item 8 |
| In-place edit of a running strategy is impossible on the infra | Task 5 item 4 (Duplicate with new settings) |
| DCA is time-triggered on a price-triggered engine | Task 3 Screen B info note, Task 8 docs |
