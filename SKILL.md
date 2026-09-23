---
name: public-trader
description: >
  Live Opening Range Breakout (ORB) 0DTE options strategy on SPY, executing
  through the Public.com connector, account 5OI27877 (CASH). Runs every 5
  minutes on weekdays from 9:00 AM to 2:45 PM CT (10:00 AM-3:45 PM ET) to
  align with 5-minute bar closes. Captures the first-30-minute SPY high/low
  as the opening range; requires 2 CONSECUTIVE completed 5-minute bars to
  close beyond that range before triggering (no single-bar fakeouts) — a
  confirmed break above triggers a live BUY of a same-day (0DTE) CALL, a
  confirmed break below triggers a live BUY of a same-day PUT — on
  whichever of SPY/SPX fits a $400 budget (SPY by default). Previous day's
  high/low is shown as context only, not part of the trigger. One trade
  per day; tiered exit — Lot A (~half the contracts) at +25%/-30%, Lot B
  (runner, remainder) at +40%/-30% — forced end-of-day close
  at 2:45 PM CT to avoid holding into physical-settlement expiration.
  Every trading day (trade or no trade) gets logged to a Google Doc named
  "ORB Trade Log" (via the Google Drive connector — read-modify-write, no
  local filesystem dependency) with full entry/exit rationale and
  post-trade analysis; a separate WEEKLY REVIEW run (first trading day of
  each week) reads that log and recommends whether to keep the strategy
  as-is or adjust it. This is NOT the old 16-ticker 5-strategy swing
  matrix — that content was removed from this skill entirely (it belonged
  to a different context). ALWAYS trigger on: "run the ORB breakout
  check", "check opening range", "SPY breakout", "opening range
  breakout", "run the breakout scan", "check for a breakout", "run the
  weekly review", "ORB weekly review", "how's the breakout strategy
  doing", or any request about this SPY/SPX 0DTE strategy. Runs on a
  5-minute cadence during market hours on weekdays, plus one weekly
  review run.
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
rarely fits). One trade per day. Exit is tiered: roughly half the
contracts (Lot A) target +25%, the rest (Lot B, the runner) target +40% —
both lots stop out at -30% — or force-close
by 2:45 PM CT if neither has hit. Previous day's high/low is shown for
context only — it does not affect the trigger.

## Schedule

Runs **every 5 minutes, weekdays, 9:00 AM–2:45 PM CT** (10:00 AM–3:45 PM
ET), skipping US market holidays — aligned to each 5-min bar close since
the 2-bar confirmation rule needs to see every bar. See
`references/schedule-setup.md` for the full launchd setup (70 runs/day;
this replaces any prior schedule entirely — it doesn't run alongside one).

**Run label:** Always open output with:
`🎯 ORB CHECK — [date] [time] CT`

---

## Overview

```
Step 1 — (First run of the day only, 9:00 AM CT) Capture the opening range:
         SPY 9:30-10:00 AM ET high/low
Step 2 — Fetch previous day's SPY high/low (context only, every run)
Step 3 — Check for an existing position/order from today (one-trade-per-day gate)
Step 4 — Fetch the last 2 completed 5-min SPY bars, check for 2-bar
         confirmed breakout beyond the opening range
Step 5 — If breakout: project a realistic SPY target for the rest of the
         day (measured move vs IV expected move, more conservative wins),
         select SPY vs SPX (budget fit), pick 0DTE strike bounded by that
         target (delta >= 0.40), size contracts to $400 budget
Step 6 — Submit BUY (live), split into Lot A/Lot B, submit each lot's own
         stop-loss (-30%) as a resting SELL STOP_LIMIT DAY order — take-
         profit (Lot A +25%, Lot B +40%) is tracked as a target to watch,
         not a second resting order (the account has no OCO support and
         rejects resting CLOSE quantity beyond what's held)
Step 6.5 — (Every run while a lot is open) Check the lot's current bid
         against its TP target; if reached, cancel that lot's resting SL
         and submit a SELL LIMIT to close it
Step 7 — (Final run of the day, 2:45 PM CT only) Force-close any position
         still open
Step 8 — Log the day's outcome to the "ORB Trade Log" Google Doc via the
         Google Drive connector (full entry if a trade happened,
         one-liner if not) — see references/trade-log.md
Step 9 — Display run summary
```

Read `references/config.md` for the exact parameters (window, budget, delta
target, exit percentages, schedule).
Read `references/signal-logic.md` for the full breakout-detection logic.
Read `references/public-submission.md` for connector tool usage, OSI symbol
construction, and the SPY-vs-SPX selection sequence.
Read `references/trade-log.md` for the logging format and the weekly
review process (see § Weekly Review below for the review workflow itself).

---

## Step 1 — Capture Opening Range (9:00 AM CT run only)

Fetch SPY 5-min bars for 9:30–10:00 AM ET via `Twelve Data:get_time_series`.
OR_high = max high, OR_low = min low across that window. Store for reuse by
every later run that day — don't recompute.

## Step 2 — Previous Day Context

Fetch SPY's prior session daily high/low. Display alongside the opening
range every run. **Not part of the trigger** — informational only.

## Step 3 — One-Trade-Per-Day Gate

```
Public:get_portfolio(account_id="5OI27877")
Public:get_orders(account_id="5OI27877")
```
If a same-day SPY/SPX option position or pending order from this strategy
already exists, skip Steps 4-6 for this run (still do Step 7 if it's the
2:45 PM CT run).

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

BUY the option, split into Lot A (majority) and Lot B (runner), then
submit each lot's own stop-loss (-30%) as a resting **SELL STOP_LIMIT**
DAY order — never `order_type="LIMIT"` for a stop, since a limit priced
below the market fills immediately instead of waiting (this happened live
on 9/23/26 and caused an unintended loss). Take-profit (+25% for Lot A,
+40% for Lot B) is recorded as a target and watched every subsequent run
(§ Step 6.5), not submitted as a second resting order — the account
rejects resting CLOSE quantity beyond what's currently held, so a lot's
own TP+SL pair would already exceed capacity and block the other lot's
exits entirely. See `references/public-submission.md` § ORB Submission
Sequence and § TP Monitoring. Every call here is a real fill — no approval
step. After each SL is placed, confirm via `get_order` that it rests as
`NEW`, not `FILLED`. No custom alert needed beyond that: Public's own
order-fill notifications cover every leg automatically (see § Exit Alerts
in that file).

## Step 6.5 — Take-Profit Monitoring (every run while a lot is open)

For each lot still open (position exists, its SL hasn't filled), fetch the
option's current quote. If the bid has reached that lot's TP target,
cancel its resting SL, confirm the cancellation, then submit a SELL LIMIT
to close it at the current bid. See `references/public-submission.md` §
TP Monitoring for the exact sequence.

## Step 7 — Forced End-of-Day Close (2:45 PM CT run only)

Check Lot A and Lot B independently. If either is still open (its SL
hasn't filled and Step 6.5 hasn't already closed it on TP), cancel its
resting SL, confirm the cancellation, then MARKET SELL to close that lot's
remaining contracts. See `references/public-submission.md` § End-of-Day
Close.

## Step 8 — Log the Day's Outcome

On the 2:45 PM CT run (once the day's outcome is final — TP hit, SL hit,
EOD close, or no trade at all), read-modify-write the "ORB Trade Log"
Google Doc via the Google Drive connector, appending:
- **Trade day:** full entry — opening range, confirmation bars, entry
  rationale, exit outcome, P&L, and honest post-trade analysis
- **No-trade day:** one-line entry — opening range and final status

See `references/trade-log.md` for the exact template and the Drive
read-modify-write pattern. Every trading day gets an entry, win or loss or
nothing — this is a complete record, not a highlight reel, and it's what
the weekly review runs against.

## Step 9 — Display Summary

```
🎯 ORB CHECK — [date] [time] CT
SPY: $[current_price]
Opening Range (9:30-10:00 ET): $[OR_low] - $[OR_high]
Prior Day Range (context only): $[prev_low] - $[prev_high]
Last 2 completed 5-min bars: $[prior_bar_close] → $[last_bar_close]
Status: [NO BREAKOUT / WATCHING - bullish/bearish, 1 of 2 confirmed / BULLISH BREAKOUT CONFIRMED / BEARISH BREAKDOWN CONFIRMED / ALREADY TRADED TODAY / EOD CLOSE]

[If a trade fired:]
📈 Bought [N] [SPY/SPX] $[strike] [call/put] 0DTE @ $[premium] (Δ[delta])
   Cost: $[total]
   Lot A: [tier1_qty] ct — SL $[sl_a] (resting) / TP $[tp_a] (watched)
   Lot B: [tier2_qty] ct — SL $[sl_b] (resting) / TP $[tp_b] (watched)   (omit if N=1)
```

---

## Weekly Review

**Separate workflow, separate trigger** — runs once on the first trading
day of each week (see `references/schedule-setup.md` for the schedule),
or on demand when Jeff asks "run the weekly review" / "how's the breakout
strategy doing."

1. Read the "ORB Trade Log" Google Doc, filtered to the prior Mon-Fri.
   If it's empty or has no entries for that week, say so plainly and stop.
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
| Existing position/order from today found | Skip new entry, still run EOD close check if applicable |
| place_order returns an error | Do NOT retry silently — report to Jeff and skip |
| A resting SL (STOP_LIMIT) comes back `FILLED` immediately via get_order instead of `NEW` | Stop — do not place further orders this run. Report the fill price/qty to Jeff; that lot is now closed, so treat it as such (no separate SL still to manage for it) |
| A resting-order placement is rejected for exceeding "available to close" quantity | This account has no OCO — resting CLOSE quantity is capped at what's held. Only the SL should ever be resting (see `references/public-submission.md` § ORB Submission Sequence); if this still happens, report to Jeff rather than retrying with a different quantity |
| It's the 2:45 PM CT run and no position is open | Just log "no position to close," no action needed |
| Google Drive connector not linked | Report the day's outcome in the run summary anyway, flag that logging failed, tell Jeff to connect it |
| "ORB Trade Log" doc doesn't exist yet | Create it fresh on first write, no error |
| Weekly review requested but log is empty/missing for the period | Say so plainly, don't fabricate a review |
