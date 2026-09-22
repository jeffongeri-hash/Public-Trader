---
name: public-trader
description: >
  Live Opening Range Breakout (ORB) 0DTE options strategy on SPY, executing
  through the Public.com connector, account 5OI27877 (CASH). Runs ONCE per
  day on weekdays at 10:10 AM ET (9:10 AM CT) — a single check, not 5-minute
  polling (revised 9/22/26: the cloud routine's native Schedule trigger has
  a 1-hour minimum interval, so continuous intraday polling isn't possible;
  10:10 AM ET is the earliest time the 2-bar confirmation (using the
  10:00-10:05 and 10:05-10:10 ET bars) can even be evaluated). Captures the
  first-30-minute SPY high/low as the opening range; requires 2 CONSECUTIVE
  completed 5-minute bars to close beyond that range before triggering (no
  single-bar fakeouts) — a confirmed break above triggers a live BUY of a
  same-day (0DTE) CALL, a confirmed break below triggers a live BUY of a
  same-day PUT — on whichever of SPY/SPX fits a $400 budget (SPY by
  default). Previous day's high/low is shown as context only, not part of
  the trigger. One trade per day, single lot (no tiering) — entry BUY plus
  ONE real STOP order at -30%, no take-profit order and no bot-managed
  forced close; Jeff exits manually intraday whenever he chooses (revised
  9/22/26 — see § Exit Rule). Every trading day (trade or no trade) gets
  logged as a chat message in this session's own reply (revised 9/22/26 —
  Google Drive isn't reliably available in this session, so nothing is
  auto-written to any doc; Jeff copies the printed entry into his own log
  at the end of each day). This is NOT the old 16-ticker 5-strategy swing
  matrix — that content was removed from this skill entirely (it belonged
  to a different context). ALWAYS trigger on: "run the ORB breakout
  check", "check opening range", "SPY breakout", "opening range
  breakout", "run the breakout scan", "check for a breakout", "run the
  weekly review", "ORB weekly review", "how's the breakout strategy
  doing", or any request about this SPY/SPX 0DTE strategy. Runs once a
  day at 10:10 AM ET on weekdays, plus one weekly review run.
---

# Public Trader Skill — Opening Range Breakout (0DTE)

> **Rebuild note:** This skill previously ran a 16-ticker, 5-strategy equity
> swing-trading matrix. That entire strategy was a mix-up with a different
> skill/conversation and has been fully removed. This skill is now a
> single, focused strategy: **SPY opening-range breakout → 0DTE options.**
>
> Account: **5OI27877 (CASH)**. Account `5LG81210` (MARGIN) runs a separate,
> unrelated playbook — never touch it from this skill.

## Strategy in One Paragraph

Watch SPY's first 30 minutes of trading (9:30–10:00 AM ET) to set an
opening range (high/low). Require **2 consecutive completed 5-minute bars**
to close above that high (or below that low) before treating it as a
confirmed breakout — a single bar poking beyond the range doesn't count.
On confirmation, buy a same-day CALL (bullish) or PUT (bearish). Trade
whichever of SPY or SPX options fits a $400 budget (SPY by default — SPX
rarely fits). One trade per day, one lot, no tiering. Exit: a single real
STOP order at -30% of entry premium is placed right after entry — no
take-profit order, no bot-managed close. Jeff watches the position and
closes it himself whenever he decides to (target, fading premium, end of
day — his call). Previous day's high/low is shown for context only — it
does not affect the trigger.

## Schedule

Runs **once per day, weekdays, 10:10 AM ET (9:10 AM CT)**, skipping US
market holidays (revised 9/22/26 — this routine's Schedule trigger has a
1-hour minimum interval, so 5-minute intraday polling isn't possible; a
single check right after the earliest the 2-bar confirmation can fire is
the workable design instead). See `references/schedule-setup.md` for the
trigger setup — Jeff needs to set the routine's own Schedule trigger to
10:10 AM ET / 9:10 AM CT himself; this skill can't change that from
inside a run.

**Run label:** Always open output with:
`🎯 ORB CHECK — [date] [time] CT`

---

## Overview

```
Step 1 — Capture the opening range: SPY 9:30-10:00 AM ET high/low
Step 2 — Fetch previous day's SPY high/low (context only)
Step 3 — Check for an existing position/order from today (one-trade-per-day gate)
Step 4 — Fetch the last 2 completed 5-min SPY bars (10:00-10:05 and
         10:05-10:10 ET, at this run time), check for 2-bar confirmed
         breakout beyond the opening range
Step 5 — If breakout: project a realistic SPY target for the rest of the
         day (measured move vs IV expected move, more conservative wins),
         select SPY vs SPX (budget fit), pick 0DTE strike bounded by that
         target (delta >= 0.40), size contracts to $400 budget
Step 6 — Submit BUY (live), then submit ONE real STOP order at -30% of
         entry premium. No take-profit order, no lot splitting. Jeff
         exits manually whenever he chooses.
Step 7 — Print the day's outcome as a chat message Jeff can copy into his
         own log (full entry if a trade happened, one-liner if not) —
         see references/trade-log.md. Nothing is auto-written to Drive.
Step 8 — Display run summary
```

Read `references/config.md` for the exact parameters (window, budget, delta
target, exit percentages, schedule).
Read `references/signal-logic.md` for the full breakout-detection logic.
Read `references/public-submission.md` for connector tool usage, OSI symbol
construction, and the SPY-vs-SPX selection sequence.
Read `references/trade-log.md` for the logging format and the weekly
review process (see § Weekly Review below for the review workflow itself).

---

## Step 1 — Capture Opening Range

Fetch SPY 5-min bars for 9:30–10:00 AM ET. Prefer `Twelve Data:get_time_series`
if that connector is enabled in the session; otherwise use
`Public:get_price_history(symbol="SPY", period="DAY", aggregation="FIVE_MINUTES",
trading_session_toggle="REGULAR_HOURS")` and take the bars in that window —
it returns the same OHLCV shape. OR_high = max high, OR_low = min low
across that window.

## Step 2 — Previous Day Context

Fetch SPY's prior session daily high/low. Display alongside the opening
range every run. **Not part of the trigger** — informational only.

## Step 3 — One-Trade-Per-Day Gate

```
Public:get_portfolio(account_id="5OI27877")
Public:get_orders(account_id="5OI27877")
```
If a same-day SPY/SPX option position or pending order from this strategy
already exists, skip Steps 4-6 for this run — one trade per day, and Jeff
manages the exit himself, so there's nothing else for this run to do.

## Step 4 — Breakout Check (2-bar confirmation)

```
Fetch the two most recently completed 5-min SPY bars.
if both bars close > OR_high → CONFIRMED BULLISH BREAKOUT → CALL
if both bars close < OR_low  → CONFIRMED BEARISH BREAKDOWN → PUT
if only the NEWEST bar is beyond a boundary → WATCHING, no trade, log and stop
   (if only the OLDER bar had breached but the newest is back inside the
   range, that's a reverted fakeout — report NO BREAKOUT, not WATCHING)
else → NO BREAKOUT, log and stop
```

## Step 5 — Price Target Projection, Underlying + Strike Selection

See `references/public-submission.md` § ORB Option Selection. First
projects a realistic SPY target for the rest of the day (measured move
from the opening range vs. IV-implied expected move, whichever is closer
to current price — this bounds candidate strikes so a deep, unrealistic
strike never gets picked just for its delta). Then compares SPY vs SPX
0DTE premium within that bounded range; picks whichever fits the $400
budget (SPY preferred if both fit); selects strike via delta ≥ 0.40 among
the bounded candidates.

## Step 6 — Submit LIVE

BUY the option (single lot, full N contracts), then submit ONE real STOP
order (`order_type="STOP"`, not `LIMIT`) at -30% of the fill price to
close the same quantity. No take-profit order — Jeff exits manually
whenever he chooses. See `references/public-submission.md` § ORB
Submission Sequence. Every call here is a real fill — no approval step.
No custom alert needed here: Public's own order-fill notifications cover
every leg automatically (see § Exit Alerts in that file).

## Step 7 — Log the Day's Outcome (in chat)

Print the entry as a chat message at the end of this run — do NOT write
to Google Drive or any file; Jeff copies it into his own log at the end
of each day:
- **Trade day:** full entry — opening range, confirmation bars, entry
  rationale, the stop order placed, and honest rationale for the pick
- **No-trade day:** one-line entry — opening range and final status

See `references/trade-log.md` for the exact template.

## Step 8 — Display Summary

```
🎯 ORB CHECK — [date] [time] CT
SPY: $[current_price]
Opening Range (9:30-10:00 ET): $[OR_low] - $[OR_high]
Prior Day Range (context only): $[prev_low] - $[prev_high]
Last 2 completed 5-min bars: $[prior_bar_close] → $[last_bar_close]
Status: [NO BREAKOUT / WATCHING - bullish/bearish, 1 of 2 confirmed / BULLISH BREAKOUT CONFIRMED / BEARISH BREAKDOWN CONFIRMED / ALREADY TRADED TODAY]

[If a trade fired:]
📈 Bought [N] [SPY/SPX] $[strike] [call/put] 0DTE @ $[premium] (Δ[delta])
   Cost: $[total]
   Stop order: SELL [N] ct STOP @ $[stop_price] (-30%) — no take-profit order; Jeff exits manually
```

---

## Weekly Review

**Separate workflow, separate trigger** — runs once on the first trading
day of each week (see `references/schedule-setup.md` for the schedule),
or on demand when Jeff asks "run the weekly review" / "how's the breakout
strategy doing."

1. Ask Jeff for (or work from, if already pasted into this chat) the trade
   log entries he's copied down for the prior Mon-Fri — there's no Google
   Doc this reads automatically anymore (see § Step 7 above). If nothing
   is available for that week, say so plainly and stop — don't fabricate.
2. Compute the stats in `references/trade-log.md` § Weekly Review — What
   It Computes (trade frequency, win rate, CALL/PUT and SPY/SPX splits,
   P&L, hold time, fakeout rate on WATCHING events).
3. End with an explicit recommendation: keep as-is, adjust a specific
   parameter, or say plainly that there isn't enough data yet — not just
   a table of numbers with no conclusion, and not a confident verdict off
   1-2 trades either.

See `references/trade-log.md` for the full output format.

---

## Error Handling

| Situation | Action |
|---|---|
| Public connector not linked | Generate the order as a text card, do NOT place, tell Jeff to connect it |
| SPY price fetch fails | Retry once; if still failing, skip this run and log |
| Neither SPY nor SPX 0DTE fits $400 budget | Skip the trade, log both costs |
| No delta ≥ 0.40 strike available | Use closest available, flag in rationale |
| Today has no 0DTE expiration for SPY (holiday-adjacent quirk) | Skip for the day, log why |
| Existing position/order from today found | Skip new entry — nothing else for this run to do |
| place_order returns an error | Do NOT retry silently — report to Jeff and skip |
| STOP order placement fails after the BUY fills | Do NOT leave the position unprotected silently — report it immediately and clearly so Jeff can place the stop or exit manually |
| Weekly review requested but no log entries are available for the period | Say so plainly, don't fabricate a review |
