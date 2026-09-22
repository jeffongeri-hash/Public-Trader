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
already exists → skip new entries this run (one trade per day, and there's
no bot-managed close to run either — see § End-of-Day Close below).

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

## ORB Submission Sequence — single lot, stop-loss only (revised 9/22/26)

Retired the tiered two-lot TP/SL bracket after the first live trade: this
CASH account won't let separate unlinked SELL orders on the same contracts
total more than the quantity held (a TP leg and an SL leg together exceed
that), and critically, a SELL **LIMIT** priced below the live market fills
immediately instead of resting as protection — it is NOT a stop. A real
stop-loss requires `order_type="STOP"` with `stop_price`, triggering a
market sell only once the price actually falls there.

```
STEP A — BUY TO OPEN (live fill, full N contracts, single lot)
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
→ Confirm the fill via get_order before sizing the stop — use the actual
  average fill price as entry_premium, not the limit price submitted
  (they can differ if price moves between submission and fill).

STEP B — STOP-LOSS (quantity N, the whole position)
→ Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="STOP",
    symbol=<chosen_osi_symbol>,
    stop_price=round(entry_premium * 0.70, 2),
    quantity=<N>,
    open_close_indicator="CLOSE", time_in_force="DAY"
  )
```

No take-profit order. `time_in_force="DAY"` — a 0DTE option ceases to
exist after today, so there's nothing to extend. Jeff exits the position
himself, whenever he chooses, for anything other than the stop triggering.

Confirm to Jeff:
  "🎯 ORB [BULLISH BREAKOUT/BEARISH BREAKDOWN] — bought [N] [SPY/SPX]
   [strike] [call/put] 0DTE @ $[premium] (Δ[delta]). Cost: $[premium×100×N].
   Stop order: SELL [N] ct STOP @ $[stop_price] (-30%). No take-profit
   order — exit manually whenever you're ready."

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
leg this skill submits: the entry BUY and the single STOP order.
Nothing extra to configure in this skill for that. Jeff should just
confirm push notifications for order fills are enabled in Public's own
app (Account Settings → Notifications) if he wants faster-than-email
delivery.

---

## End-of-Day Close — retired (revised 9/22/26)

No bot-managed forced close, even though polling continues every 5
minutes through 2:45 PM CT. Jeff exits the position himself, whenever he
chooses; the STOP order from § ORB Submission Sequence is the only
automated protection. Each mid-day run's one-trade-per-day gate (§
Duplicate / Exposure Check) will keep finding the open position and skip
straight to reporting status — check with
`Public:get_portfolio(account_id="5OI27877")` and report plainly. Don't
place any close order unless Jeff asks you to. On the final (2:45 PM CT)
run, still just report status as-is — a position open into the close is
Jeff's to manage, not something this skill force-closes.

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
| `get_price_history` | Fallback for SPY 5-min bars when Twelve Data isn't enabled in the session (period="DAY", aggregation="FIVE_MINUTES") |
| `place_order` | Live BUY and the single STOP order — EXECUTES IMMEDIATELY |
| `get_order` | Confirm the BUY's actual fill price before sizing the stop |

No `place_multileg_order` needed — this strategy is always a single-leg
long call or long put, never a spread.
