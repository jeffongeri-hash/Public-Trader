---
name: public-trader
description: >
  Live Opening Range Breakout (ORB) 0DTE options strategy on SPY, executing
  through the Public.com connector, account 5OI27877 (CASH). v2 (10/7/26):
  runs as ONE long-lived session per trading day, started at 9:55 AM ET —
  it scans every 5-minute bar close for a 2-bar confirmed break of the
  9:30-10:00 ET opening range until 11:30 ET, applies macro/VIX/VWAP/
  range-width/key-level filters, buys a same-day CALL (break up) or PUT
  (break down) sized to a $200 budget, then stays alive watching the
  position every 30 seconds: take-profit, breakeven ratchet, structural
  stop (SPY back inside the range), 60-minute time stop and 3:45 PM ET
  forced close all execute by atomically replacing the resting stop via
  cancel_and_replace_order. A resting -40% STOP_LIMIT protects the position
  if the session dies. One trade per day, enforced from today's fills.
  Every day is logged to the "ORB Trade Log" Google Doc; weekly and monthly
  reviews read that log. ALWAYS trigger on: "run the ORB breakout check",
  "run the ORB session", "check opening range", "SPY breakout", "opening
  range breakout", "run the breakout scan", "check for a breakout", "run
  the weekly review", "run the monthly review", "ORB weekly review", "how's
  the breakout strategy doing", or any request about this SPY/SPX 0DTE
  strategy.
---

# Public Trader Skill — Opening Range Breakout (0DTE), v2

Account: **5OI27877 (CASH)**. Account `5LG81210` (MARGIN) runs a separate,
unrelated playbook — never touch it from this skill.

Reference files (in this repo they sit next to this SKILL.md; older notes
call them `references/<name>`):
- `config.md` — **every parameter** (windows, filters, budget, exits, lock). Read first.
- `signal-logic.md` — opening range, bar scanning, signal/MISSED logic, filters
- `public-submission.md` — strike selection, entry, resting stop, watched exits
- `trade-log.md` — log format, weekly and monthly review
- `schedule-setup.md` — how the routine is triggered

## Why v2 exists (read once)
Sep 18 – Oct 5 lost $500 of $1,000. The trigger was fine when entered on
time; the losses came from (1) hourly runs entering late or missing signals
outright — on 10/6 the call signal confirmed at 10:15 ET and the 10:22 run
never saw it; (2) a one-trade gate that only looked at open positions, so
the strategy re-entered after stop-outs (9/25's re-entry lost $329); (3)
small capped wins vs large losses. v2 fixes all three.

## Strategy in One Paragraph
Record SPY's 9:30–10:00 ET high/low. From 10:05 to 11:30 ET, check every
completed 5-minute bar; the first time two consecutive bars close beyond
the same boundary is the day's only signal. Enter only if the signal is
fresh (≤ 1 bar old, price within 0.10% of the signal close) and every
filter passes. Buy a 0DTE CALL or PUT, put a −40% disaster stop on it, then
watch it live and exit on take-profit, breakeven stop, a SPY close back
inside the range, a 60-minute time stop, or 3:45 PM ET — whichever first.

**Run label:** always open output with `🎯 ORB SESSION — [date] [time] CT`.

---

## Session Flow

```
Phase 0 — Pre-flight (once, before 10:00 ET)
Phase 1 — Entry scan (each 5-min bar close, 10:05 → 11:30 ET, until a decision)
Phase 2 — Position watch (every 30 s while a position is open, until flat or 15:45 ET)
Phase 3 — Log + summary (once, when the day's outcome is final)
```

The session ends after Phase 3. It does not keep running once the day's
outcome is final (no trade possible, or position closed).

### Waiting between checks
Never use a foreground `sleep`. Wait with a background timer — Bash with
`run_in_background: true` running `sleep <seconds>`; you are re-invoked when
it exits — or the Monitor tool with an until-loop. Time each wait to land
~20 s after the next 5-min bar close in Phase 1 (e.g. 10:10:20, 10:15:20),
and `POLL_SECONDS` (30 s) in Phase 2. Refresh the session lock at least
every 5 minutes (config.md § Session Lock).

---

## Phase 0 — Pre-flight

1. **Holiday / weekend:** if the market is closed today, log nothing and stop.
2. **Session lock:** per config.md § Session Lock. Locked → stop immediately.
3. **One-trade gate:** `get_history(account_id="5OI27877", start=<today 00:00 ET>)`,
   `get_portfolio`, `get_orders`. A SPY/SPX option BUY fill today → the
   day's trade is used: if a position is still open, go straight to
   Phase 2 to manage it (recover its resting stop id from `get_orders`);
   otherwise go to Phase 3.
4. **Macro calendar:** search today's US economic calendar (WebSearch, e.g.
   "economic calendar [date] CPI FOMC") plus config.md § Macro Event Rules
   (FOMC dates are listed there). Classify the day `NORMAL`, `DATA DAY`,
   or `NO TRADE` (with the reason). A `NO TRADE` day still waits for the
   opening range so the log has it, then goes to Phase 3.
5. **Context:** prior-day high/low and 63-day high from daily bars; VIX
   quote; `optionsBuyingPower` → `BUDGET = min($200, buying power)`.
6. **Wait** until 10:00:20 ET.

## Phase 1 — Entry Scan

At each wake: fetch today's 5-min SPY bars, drop the forming bar
(config.md § completed-bar rule).

1. First wake only: compute `OR_high`, `OR_low`, `OR_width_pct` from the six
   09:30–09:55 bars. Keep them for the rest of the session.
2. Scan **all** completed bars from 10:00 on (signal-logic.md § Step 4).
   - No confirmed pair yet → status `NO BREAKOUT` or `WATCHING` (newest
     bar alone beyond a boundary). If it's past 11:30 ET → final status
     `NO BREAKOUT all day` → Phase 3. Else wait for the next bar.
   - Signal found → check freshness (config.md § No-Chase Rule). Stale →
     final status `MISSED` → Phase 3.
3. Fresh signal → run every filter in config.md § Entry Filters. Any fail →
   final status `SKIPPED — <filter + values>` → Phase 3.
4. All pass → re-check the one-trade gate, then strike selection, entry
   and the resting disaster stop per public-submission.md §§ Option
   Selection and Entry. Confirm the stop rests as `NEW`. → Phase 2.

## Phase 2 — Position Watch

Every 30 s per open lot: option quote (use the **bid**), SPY quote; at each
5-min bar close also the newest completed SPY bar. Evaluate in this order
and act on the first that applies (public-submission.md § Watched Exits):

1. Stop already filled (`get_order` shows FILLED) → lot closed by its stop.
2. EOD: time ≥ 15:45 ET → close all.
3. STRUCTURAL: newest completed bar closed back inside the range → close all.
4. TIME_STOP: ≥ 60 min since entry and bid < entry × 1.10 → close all.
5. TP: bid ≥ the lot's target → close that lot (LIMIT at bid).
6. BREAKEVEN: bid ≥ entry × 1.15 and stop still below entry → raise the stop.

Every exit is a `cancel_and_replace_order` on the lot's resting stop —
never cancel-then-place, never a second resting order. Confirm with
`get_order` after each. When every lot is closed → Phase 3.

If the session is about to end for any reason while a lot is open, leave
its resting stop in place, say so in the summary, and send a notification
— the stop and Public's own fill notifications cover it.

## Phase 3 — Log and Summary

Append the day to the `ORB Trade Log` Google Doc (trade-log.md format —
full entry for a trade, one line for NO BREAKOUT / MISSED / SKIPPED /
NO TRADE), release the session lock, then print:

```
🎯 ORB SESSION — [date] [time] CT   [LIVE/PAPER]
Macro: [NORMAL / DATA DAY: CPI / NO TRADE: FOMC] | VIX [x] ([+/-y]%)
SPY: $[price] | VWAP $[vwap]
Opening Range: $[OR_low] - $[OR_high] (width [w]%)
Prior day: $[prev_low] - $[prev_high] | 63-day high $[h]
Signal: [none / CALL|PUT confirmed at HH:MM close $x]
Status: [NO BREAKOUT all day / MISSED — reason / SKIPPED — filter / TRADED / NO TRADE — event]

[If traded:]
📈 Bought [N] [SPY/SPX] $[strike] [call/put] 0DTE @ $[fill] (Δ[delta]) — cost $[total], entry HH:MM
   Lot A: [qty] ct → [TP / BREAKEVEN / STRUCTURAL / TIME / DISASTER SL / EOD] @ $[px] HH:MM ($[pnl])
   Lot B: [qty] ct → ...                                                       (omit if N=1)
   Day P&L: $[total] ([pct]%)
```

---

## Weekly and Monthly Reviews
Separate runs. Weekly: first trading day of each week. Monthly: first
trading day of each month, or on request. Both read the `ORB Trade Log`
Google Doc — see trade-log.md for what each computes and outputs. If the
log has no entries for the period, say so plainly and stop — never
fabricate a review.

---

## Error Handling

| Situation | Action |
|---|---|
| Public connector not linked | Don't trade. Report the signal as a text card, tell Jeff to connect it |
| SPY bars/quote fetch fails | Retry once; if still failing, skip this check (Phase 1) or the poll (Phase 2) and try the next one |
| Session lock held by a live session | Stop immediately, no orders |
| Today already has an option BUY fill | No new entry; manage the open position if any |
| Neither SPY nor SPX fits `BUDGET` | `SKIPPED — budget`, log both costs |
| No 0DTE expiration today | `NO TRADE — no 0DTE expiry`, log it |
| `place_order` returns an error | Don't retry silently — report, no trade today |
| Resting stop comes back `FILLED` right after placement | Stop placing orders. Report fill price/qty; that lot is closed |
| `cancel_and_replace_order` errors | Fallback in public-submission.md § Watched Exits — never leave a lot both without a stop and without an exit order |
| Resting close order rejected for exceeding "available to close" | Account has no OCO: only one close order per lot may exist. Report, don't retry with other quantities |
| Session must end with a lot open | Leave the resting stop, report it, notify Jeff |
| Google Drive not linked | Report the outcome in the summary, flag that logging and the session lock failed |
| `ORB Trade Log` / `ORB Session Lock` doc missing | Create it on first write |
