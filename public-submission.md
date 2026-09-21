# Public.com Submission Reference — ORB 0DTE Options

## Connector Status

Orders are LIVE FILLS, not drafts. `place_order` executes immediately
against account **5OI27877 (CASH)**. There is no approval queue.
Account 5LG81210 (MARGIN) is a SEPARATE playbook — never touch it here.

---

## Duplicate / Exposure Check (run at the top of every check)

```
positions = Public:get_portfolio(account_id="5OI27877")
open_orders = Public:get_orders(account_id="5OI27877")
```
If a same-day SPY/SPX option position or pending order from this strategy
already exists → skip new entries this run (one trade per day). Still run
the end-of-day forced-close check regardless (see § End-of-Day Close below).

---

## ORB Option Selection (SPY vs SPX, 0DTE)

```
STEP 0 — Project a realistic target price (informs Step 2's candidate
range only — does NOT change TP/SL percentages, see config.md § Price
Target Projection for the full formulas)
  measured_target = project the opening range's height from whichever
    boundary broke (OR_low - range_height for a breakdown, OR_high +
    range_height for a breakout)
  iv_target = current_price ± (current_price × IV × sqrt(time remaining
    today / (252 × 6.5 hrs)))   [use IV from a quick get_option_greeks
    call on a near-ATM strike]
  projected_target = whichever of the two is closer to current_price

STEP 1 — Confirm today has a 0DTE expiration for each underlying
→ Public:get_option_expirations(account_id="5OI27877", symbol="SPY")
→ Public:get_option_expirations(account_id="5OI27877", symbol="SPX")
  Confirm today's date appears in each list. SPY lists daily expirations
  every trading day; SPX does too but double-check — if today isn't listed
  for one of them, that underlying is simply not eligible today.

STEP 2 — Build candidate strikes BOUNDED by projected_target
  SPY strikes: $1 increments, only between current SPY price and
    projected_target (inclusive) — e.g. current $762, projected_target
    $759 on a breakdown → candidates at $762, $761, $760, $759 only,
    never $758 or deeper
  SPX strikes: $5 increments, same bounding logic, current SPX price to
    the SPX-scaled equivalent of projected_target (fetch SPX price via
    Public:get_quotes(symbols=["SPX"], instrument_type="INDEX") if supported,
    else derive an approximate SPX level from SPY × ~10 as a rough guide only
    — always confirm against a real SPX quote before pricing an order)

  OSI symbol format: <TICKER><YYMMDD><C or P><strike_in_thousandths, 8 digits>
  e.g. SPY 0DTE today 2026-09-14, $760 call → "SPY260914C00760000"
  e.g. SPX 0DTE today 2026-09-14, $5700 put → "SPX260914P05700000"

STEP 3 — Delta check within the bounded candidates only
→ Public:get_option_greeks(account_id="5OI27877", osi_symbols=[<candidates>])
  Pick the strike per underlying whose |delta| is closest to but ≥ 0.40,
  among the bounded candidates only. If none reach 0.40 within the bounded
  range, use the closest available within that range — do NOT reach past
  projected_target for a higher-delta strike, since that assumes a move
  the day's own data doesn't support.

STEP 4 — Premium for each underlying's chosen strike
→ Public:get_quotes(account_id="5OI27877", symbols=[<osi_symbol>], instrument_type="OPTION")
  contract_cost = premium × 100 (for either underlying — both are 100-multiplier)

STEP 5 — Pick the underlying
  If both contract_costs ≤ $400 → use SPY (tighter spreads, more liquid)
  If only one ≤ $400 → use that one
  If neither ≤ $400 → SKIP the trade, log both costs in the summary
```

Log `projected_target` and which method (measured move vs. IV) produced it
in the trade log entry — see `references/trade-log.md`.

---

## ORB Submission Sequence — tiered two-lot exit (revised 9/18/26)

```
STEP A — BUY TO OPEN (live fill, full N contracts)
→ Public:place_order(
    account_id="5OI27877",
    instrument_type="OPTION",
    order_side="BUY",
    order_type="LIMIT",
    symbol=<chosen_osi_symbol>,
    limit_price=<ask or last, rounded>,
    quantity=<contracts N>,
    open_close_indicator="OPEN",
    time_in_force="DAY"
  )

Once filled, split into two lots:
  tier1_qty = ceil(N / 2)
  tier2_qty = N - tier1_qty   # 0 if N == 1

STEP B — LOT A EXITS (tier1_qty contracts)
→ TP-A: Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="LIMIT",
    symbol=<chosen_osi_symbol>,
    limit_price=round(entry_premium * 1.25, 2),
    quantity=<tier1_qty>,
    open_close_indicator="CLOSE", time_in_force="DAY"
  )
→ SL-A: Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="LIMIT",
    symbol=<chosen_osi_symbol>,
    limit_price=round(entry_premium * 0.70, 2),
    quantity=<tier1_qty>,
    open_close_indicator="CLOSE", time_in_force="DAY"
  )

STEP C — LOT B EXITS (tier2_qty contracts — SKIP if N == 1, no runner)
→ TP-B: Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="LIMIT",
    symbol=<chosen_osi_symbol>,
    limit_price=round(entry_premium * 1.40, 2),
    quantity=<tier2_qty>,
    open_close_indicator="CLOSE", time_in_force="DAY"
  )
→ SL-B: Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="LIMIT",
    symbol=<chosen_osi_symbol>,
    limit_price=round(entry_premium * 0.70, 2),
    quantity=<tier2_qty>,
    open_close_indicator="CLOSE", time_in_force="DAY"
  )
```

`time_in_force="DAY"` throughout — a 0DTE option ceases to exist after
today, so there's nothing to extend.

**Unlinked legs warning, per lot:** within Lot A, TP-A and SL-A don't
cancel each other — whichever fills first, cancel the other **for that lot
only**. Same for Lot B independently. Lot A and Lot B never interact with
each other's orders; each pair is its own unlinked bracket sized to its
own quantity.

Confirm to Jeff as each fires:
  "🎯 ORB [BULLISH BREAKOUT/BEARISH BREAKDOWN] — bought [N] [SPY/SPX]
   [strike] [call/put] 0DTE @ $[premium] (Δ[delta]). Cost: $[premium×100×N].
   Lot A: [tier1_qty] ct, TP $[tp_a] / SL $[sl_a].
   Lot B: [tier2_qty] ct, TP $[tp_b] / SL $[sl_b]." (omit Lot B line if N==1)

---

## Exit Alerts — Public's own order-fill notifications (revised 9/18/26)

The Public connector exposed to this skill has **no alert-creation tool**
(`create_alert` and equivalents exist on the IBKR connector, not Public's).
Public's own in-app price alerts also wouldn't help here even if set
manually — they only cover stocks/crypto price movement, not individual
option contracts, so there's no way to alert directly on "this put hits
$1.63."

**No custom alert is needed anyway.** Public automatically sends an
order-fill notification (confirmed via email in Jeff's own account,
likely push too if enabled in the Public app's own notification settings)
the instant ANY order on the account fills — this already covers every
leg this skill submits: the entry BUY, and each of TP-A/SL-A/TP-B/SL-B.
Nothing extra to configure in this skill for that. Jeff should just
confirm push notifications for order fills are enabled in Public's own
app (Account Settings → Notifications) if he wants faster-than-email
delivery.

---

## End-of-Day Close (2:45 PM CT / 3:45 PM ET run only)

```
positions = Public:get_portfolio(account_id="5OI27877")
```
Check each lot independently — either or both may still be open:
```
If Lot A still has open contracts (neither TP-A nor SL-A filled):
→ Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="MARKET",
    symbol=<osi_symbol>, quantity=<Lot A remaining qty>,
    open_close_indicator="CLOSE", time_in_force="DAY"
  )
  Cancel whichever of TP-A/SL-A is still resting.

If Lot B still has open contracts (neither TP-B nor SL-B filled):
→ Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="MARKET",
    symbol=<osi_symbol>, quantity=<Lot B remaining qty>,
    open_close_indicator="CLOSE", time_in_force="DAY"
  )
  Cancel whichever of TP-B/SL-B is still resting.
```

Confirm to Jeff:
  "⏰ EOD FORCED CLOSE — [TICKER] [strike] [call/put], [N] ct sold at
   $[price] (P&L: $[gain/loss]). 0DTE contract would have expired
   [ITM/OTM] otherwise." (report per lot if only one lot needed closing)

---

## Pre-Connection Fallback

If the Public connector is not linked:
1. Complete Steps 1–4 of signal-logic.md normally (opening range, prior day
   context, breakout check)
2. Generate the order object without placing it
3. Display as a card, tell Jeff to connect the Public connector and re-run

---

## Actual Tools Used by This Skill

| Tool | Purpose |
|------|---------|
| `get_portfolio` | Check for an existing same-day position (dup/exposure check) |
| `get_orders` | Check for pending orders (dup check) |
| `get_quotes` | Live SPY price for the breakout check; option premiums |
| `get_option_expirations` | Confirm 0DTE availability for SPY/SPX today |
| `get_option_greeks` | Delta check for strike selection |
| `place_order` | Live single-leg equity-quote-read / option order — EXECUTES IMMEDIATELY |

No `place_multileg_order` needed — this strategy is always a single-leg
long call or long put, never a spread.
