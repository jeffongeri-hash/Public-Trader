# Signal Logic Reference — Opening Range Breakout (SPY, 0DTE options)

This replaces the old 16-ticker/5-strategy swing matrix entirely. That
content belonged to a different skill/conversation and has been removed.

---

## Step 1 — Capture the Opening Range (once per day, first run at 9:10 AM CT)

```
Fetch SPY intraday bars for 9:30–10:00 AM ET (8:30–9:00 AM CT) via
Twelve Data:get_time_series(symbol="SPY", interval="5min",
  start_date=<today 9:30 ET>, end_date=<today 10:00 ET>)

OR_high = max(high) across those bars
OR_low  = min(low) across those bars
```

Store OR_high/OR_low for reuse by every subsequent check that day — don't
recompute mid-day, the range is fixed once the first 30 minutes are over.

## Step 2 — Fetch Previous Day High/Low (context only)

```
Fetch SPY's prior trading day daily bar via Twelve Data:get_time_series
(interval="1day", outputsize=2) → use the second-most-recent row.

prev_day_high, prev_day_low
```
Display these alongside the opening range in every run's output. They do
NOT affect the trigger — informational only, per Jeff's explicit call.

## Step 3 — Check for an Existing Position Today

Before evaluating the breakout condition, check:
```
Public:get_portfolio(account_id="5OI27877")
Public:get_orders(account_id="5OI27877")
```
If either shows an open SPY/SPX option position or pending order opened
today by this strategy → SKIP the rest of this run (one trade per day).
Still print the day's log entry if this is the 2:45 PM CT run (see Step 8
below) — there's no forced-close check anymore, Jeff exits manually.

## Step 4 — Breakout Trigger (2-bar confirmation on the 5-minute chart)

```
Fetch the most recent completed 5-minute SPY bars via
Twelve Data:get_time_series(symbol="SPY", interval="5min", outputsize=3)
— discard the currently-forming bar (if the API includes it), use the two
most recently CLOSED bars.

last_bar, prior_bar = the two most recent completed 5-min closes

if last_bar.close > OR_high AND prior_bar.close > OR_high:
    direction = "BULLISH BREAKOUT CONFIRMED"
    option_type = "CALL"
elif last_bar.close < OR_low AND prior_bar.close < OR_low:
    direction = "BEARISH BREAKDOWN CONFIRMED"
    option_type = "PUT"
elif last_bar.close > OR_high:
    # only the newest bar has breached — needs one more to confirm
    direction = "WATCHING — bullish, 1 of 2 bars confirmed"
    → no trade this run, log status, re-check next run
elif last_bar.close < OR_low:
    # only the newest bar has breached — needs one more to confirm
    direction = "WATCHING — bearish, 1 of 2 bars confirmed"
    → no trade this run, log status, re-check next run
else:
    # last_bar is back inside the range — even if prior_bar had breached,
    # that breach is now stale/reverted. Do NOT report WATCHING here.
    direction = "NO BREAKOUT"
    → no trade this run, log status and stop
```

**Only the MOST RECENT bar's status determines WATCHING vs NO BREAKOUT.**
`WATCHING` means the newest completed bar just breached a boundary and
we're waiting on the next one to confirm — it is not a general "one of the
last two, whichever" check. If the newest bar has already closed back
inside the range, any earlier breach is stale and reverted: report
`NO BREAKOUT`, don't carry it forward as `WATCHING`. (An earlier version of
this logic used a symmetric XOR-of-XOR check that could never fire the
"one bar breached" case correctly — fixed here.)

**A single bar beyond OR_high/OR_low is NOT enough.** Both of the two most
recently completed 5-minute bars must close beyond the same boundary
before a trade fires. This filters out single-bar fakeouts right at the
range edge. Live/intraday tick price is not used for the trigger itself —
only completed 5-min bar closes count.

## Step 5 — Price Target Projection, Underlying Selection, Strike/Delta

Before picking a strike, project a realistic SPY target for the rest of
the day (measured move from the opening range vs. IV-implied expected
move, whichever is closer to current price — see `references/config.md`
§ Price Target Projection for the formulas). This bounds which strikes
are even considered — no reaching for a deep, unrealistic strike just
because its delta looks appealing.

See `references/public-submission.md` § ORB Option Selection for the full
tool-call sequence (target projection, 0DTE expiration lookup, bounded
candidate strikes, delta check via `get_option_greeks`, premium via
`get_quotes`, SPY-vs-SPX budget comparison).

## Step 6 — Sizing

```
contracts = floor($400 / (premium × 100)), minimum 1
```

## Step 7 — Submit (single lot, stop-loss only — revised 9/22/26)

BUY the option (CALL or PUT per Step 4) for the full contract count N, one
lot, no tiering. Once filled, submit ONE `order_type="STOP"` SELL at -30%
of the entry premium for quantity N. No take-profit order. See
`references/public-submission.md` § ORB Submission Sequence.

## Step 8 — Print the Log Entry (once per day), No Forced Close

No bot-managed end-of-day close — Jeff exits manually whenever he
chooses. Print the trade-log entry (see `references/trade-log.md`) as a
chat message exactly once per day: on a trade day, print it on the same
run the trade fires (entry details are freshest right then); on a
no-trade day, print the one-liner only on the final run (2:45 PM CT),
since that's the first point "no breakout all day" is final. Any other
mid-day run just reports its per-run status (Output Format below), no log
entry.

---

## Daily State to Track

Each day is independent — no state carries from one day to the next except
the day's own opening range (fixed once at 9:10 AM CT) and whether a trade
has already fired (checked live via `get_portfolio`/`get_orders`, no
separate state file needed).

## Output Format

Every run should report:
```
🎯 ORB CHECK — [date] [time] CT
SPY: $[current_price]
Opening Range (9:30-10:00 ET): $[OR_low] - $[OR_high]
Prior Day Range (context only): $[prev_day_low] - $[prev_day_high]
Last 2 completed 5-min bars: $[prior_bar.close] → $[last_bar.close]
Status: [NO BREAKOUT / WATCHING - 1 of 2 bars confirmed / BULLISH BREAKOUT CONFIRMED / BEARISH BREAKDOWN CONFIRMED / ALREADY TRADED TODAY]
```
