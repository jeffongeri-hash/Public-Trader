# Public.com Submission Reference — ORB 0DTE Options, v2

## Connector Status
Orders are LIVE FILLS, not drafts. `place_order` and
`cancel_and_replace_order` execute immediately against account
**5OI27877 (CASH)**. There is no approval queue. In `MODE = PAPER`
(config.md) none of the order calls below are made — fills are simulated
at the quoted ask (entry) / bid (exit) and logged as `[PAPER]`.
Account 5LG81210 (MARGIN) is a SEPARATE playbook — never touch it here.

---

## One-Trade Gate (Phase 0 and again right before entry)

```
history   = Public:get_history(account_id="5OI27877", start=<today 00:00 ET ISO>)
positions = Public:get_portfolio(account_id="5OI27877")
orders    = Public:get_orders(account_id="5OI27877")
```
Blocked if ANY of: a `TRADE` with `side="BUY"` and a `SPY…`/`SPX…` option
symbol dated today; an open SPY/SPX option position; an open SPY/SPX
option order. Checking only open positions is what allowed the 9/21 and
9/25 re-entries — the fills check is mandatory.

---

## Option Selection (SPY vs SPX, 0DTE) — unchanged from v1

```
STEP 0 — projected_target (config.md § Price Target Projection)
  measured_target = OR boundary that broke ± OR height
  iv_target       = price ± price × IV × sqrt(hours_left / (252 × 6.5))
                    (IV from get_option_greeks on a near-ATM strike)
  projected_target = whichever is closer to price

STEP 1 — Public:get_option_expirations(account_id="5OI27877", symbol="SPY") (and "SPX")
  Today must be listed, else that underlying is ineligible.

STEP 2 — Candidate strikes between current price and projected_target, inclusive
  SPY $1 increments; SPX $5 increments (SPX price via
  get_quotes(symbols=["SPX"], instrument_type="INDEX")).
  OSI: <TICKER><YYMMDD><C|P><strike × 1000, 8 digits>
  e.g. SPY 0DTE 2026-10-06 $780 call → "SPY261006C00780000"

STEP 3 — Public:get_option_greeks(account_id="5OI27877", osi_symbols=[...])
  |delta| closest to but ≥ 0.40 inside the bound; else closest inside the
  bound (flag it). Never reach past projected_target.

STEP 4 — Public:get_quotes(account_id="5OI27877", symbols=[osi], instrument_type="OPTION")
  contract_cost = ask × 100

STEP 5 — both ≤ BUDGET → SPY; one ≤ BUDGET → that one; neither → SKIPPED — budget
```

---

## Entry

```
STEP A — BUY TO OPEN
→ Public:preflight_order(...same args...)        # sanity-check cost / buying power
→ Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="BUY", order_type="LIMIT",
    symbol=<osi>, limit_price=<current ask>,
    quantity=<N>, open_close_indicator="OPEN", time_in_force="DAY")
```
Poll `get_order` every few seconds for up to 60 s. Not filled after 60 s →
`cancel_order`, confirm CANCELLED, and the day ends `MISSED — entry not
filled` (don't chase with a higher limit). Partial fill → cancel the rest
and continue with the filled quantity as N.

`entry` = the actual average fill price. `entry_time` = fill time. Split
into lots: `tier1_qty = ceil(N/2)`, `tier2_qty = N − tier1_qty`.

```
STEP B — DISASTER STOP, one per lot (resting)
→ Public:place_order(
    account_id="5OI27877", instrument_type="OPTION",
    order_side="SELL", order_type="STOP_LIMIT",
    symbol=<osi>,
    stop_price=round(entry × 0.60, 2),
    limit_price=round(entry × 0.60 − 0.05, 2),
    quantity=<lot qty>, open_close_indicator="CLOSE", time_in_force="DAY")
→ Public:get_order(...) → must be status "NEW". FILLED immediately → stop,
  report, that lot is closed.
```
Record each lot's resting `order_id`, its current stop price, and its
target (`entry × 1.25` for Lot A, `entry × 1.40` for Lot B).

**Never** `order_type="LIMIT"` for a stop (9/23/26: a sell limit below the
market filled instantly). Only ever one resting close order per lot — the
account has no OCO and rejects close quantity above what's held.

Confirm:
```
🎯 ORB [CALL/PUT] — bought [N] [SPY] $[strike] [call/put] 0DTE @ $[entry] (Δ[d]), cost $[x].
   Lot A [q] ct: stop $[s] (resting) / TP $[t] (watched)
   Lot B [q] ct: stop $[s] (resting) / TP $[t] (watched)      (omit if N=1)
```

---

## Watched Exits — `cancel_and_replace_order` (Phase 2)

Every 30 s: `get_quotes` on the option (use the **bid**) and on SPY. The
resting stop is the only order on the lot; every exit or stop change
atomically **replaces** it, so there is never a second close order (no
oversell) and never zero protection (no gap between a cancel and a new
order).

**Close a lot at take-profit** (bid ≥ the lot's target):
```
→ Public:cancel_and_replace_order(
    account_id="5OI27877", order_id=<lot's resting stop id>,
    order_type="LIMIT", limit_price=<current bid, 2 dp>,
    quantity=<lot qty>, time_in_force="DAY")
```

**Close a lot immediately** (STRUCTURAL, TIME_STOP, EOD):
```
→ Public:cancel_and_replace_order(
    account_id="5OI27877", order_id=<lot's resting stop id>,
    order_type="MARKET", quantity=<lot qty>, time_in_force="DAY")
```

**Raise the stop** (BREAKEVEN — bid ≥ entry × 1.15, or Lot A's TP filled):
```
→ Public:cancel_and_replace_order(
    account_id="5OI27877", order_id=<lot's resting stop id>,
    order_type="STOP_LIMIT",
    stop_price=round(entry + 0.02, 2),
    limit_price=round(entry + 0.02 − 0.05, 2),
    quantity=<lot qty>, time_in_force="DAY")
```
Only ever raise a stop, never lower it.

**After every replace:**
1. The call returns the replacement order (take its new `order_id` — it
   becomes the lot's tracked order). If it doesn't, find it in
   `get_orders`.
2. `get_order` on it.
   - TP LIMIT: should fill within a few polls. If it hasn't filled after
     60 s and bid has dropped below the limit, replace it again at the new
     bid only if bid is still ≥ entry × 1.15; otherwise replace it back to
     a breakeven STOP_LIMIT (the target was missed — keep the trade
     protected and continue watching).
   - MARKET: should be FILLED; record the fill price.
   - STOP_LIMIT: should be NEW.
3. If the original order turns out to have FILLED before the replace
   landed (the replace errors with "filled" or similar), the lot is
   already closed at its stop — record that, place nothing.

**Fallback if `cancel_and_replace_order` errors for another reason:**
1. `cancel_order` the resting stop.
2. Poll `get_order` until `CANCELLED` (if it shows FILLED instead, the lot
   is closed — stop here).
3. Only then `place_order` the exit (SELL, CLOSE, same qty) — LIMIT at bid
   for TP, MARKET otherwise; for a stop raise, a new STOP_LIMIT.
4. If step 3 fails, immediately re-place the original disaster stop and
   report. A lot must never sit with no close order.

**Lot B after Lot A's TP fills:** raise Lot B's stop to breakeven at once
(if not already), keep watching for TP_B / STRUCTURAL / TIME_STOP / EOD.

Report every exit as it happens:
```
✅ Lot [A/B] closed — [TP / BREAKEVEN STOP / STRUCTURAL / TIME STOP / DISASTER SL / EOD]
   [q] ct @ $[px] at HH:MM CT — P&L $[x] ([y]%)
```

---

## If the session has to end with a lot open
Leave the resting stop exactly as it is (disaster or breakeven), report it,
and send a notification. The next session's Phase 0 detects the open
position and resumes Phase 2, rebuilding state per signal-logic.md § Daily
State. Public itself emails/pushes every fill, so a stop fill is never
silent.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `get_history` | Today's fills — the one-trade gate; rebuild state on resume |
| `get_portfolio` / `get_orders` | Open positions / resting orders |
| `get_price_history` | SPY 5-min bars (signal, VWAP, structural stop), daily bars (levels) |
| `get_quotes` | SPY, VIX (`INDEX`), SPX (`INDEX`), option bid/ask |
| `get_option_expirations` / `get_option_greeks` | 0DTE availability, delta, IV |
| `preflight_order` | Cost check before the entry |
| `place_order` | Entry BUY and the resting disaster stops — EXECUTES IMMEDIATELY |
| `cancel_and_replace_order` | Every watched exit and every stop raise — EXECUTES IMMEDIATELY |
| `get_order` | Confirm every order's status |
| `cancel_order` | Unfilled entry; fallback path only |

No spreads — always a single-leg long call or long put.
