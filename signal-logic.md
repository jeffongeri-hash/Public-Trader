# Signal Logic Reference — Opening Range Breakout (SPY, 0DTE options), v2

Parameters named here (`ENTRY_CUTOFF`, `MAX_CHASE_PCT`, …) are defined in
`config.md`.

---

## Step 1 — Bars

```
Public:get_price_history(symbol="SPY", period="DAY",
  aggregation="FIVE_MINUTES", trading_session_toggle="REGULAR_HOURS")
→ regularMarket.bars

completed = [b for b in bars if b.timestamp + 5 min <= now]
```
The response always includes the forming bar (sometimes with a timestamp
like `10:23` instead of a 5-minute boundary) — it is never part of any
calculation. (A 10/6/26 run mistakenly treated the forming 10:20 bar as
complete.)

## Step 2 — Opening Range (fixed once, at the first Phase 1 check)

```
or_bars  = completed bars stamped 09:30, 09:35, 09:40, 09:45, 09:50, 09:55
OR_high  = max(high), OR_low = min(low), OR_mid = (OR_high + OR_low) / 2
OR_width_pct = (OR_high - OR_low) / OR_mid
```
Don't recompute it later in the day.

## Step 3 — Context (Phase 0, once)

```
Public:get_price_history(symbol="SPY", period="QUARTER", aggregation="ONE_DAY")
prev_high, prev_low = the last bar dated before today
high_63           = max(high) over bars dated before today
Public:get_quotes(symbols=["VIX"], instrument_type="INDEX") → vix_last, vix_prev_close
```

## Step 4 — Signal scan (every Phase 1 check)

```
post = completed bars stamped >= 10:00, in time order
signal = None
for i in 1 .. len(post)-1:
    a, b = post[i-1], post[i]
    if a.close > OR_high and b.close > OR_high: signal = ("CALL", b); break
    if a.close < OR_low  and b.close < OR_low:  signal = ("PUT",  b); break
```
The first confirmed pair is the day's only signal — don't keep looking for
a "better" one later, and never trade the opposite direction afterward.

No signal yet:
```
newest = post[-1]
if newest.close > OR_high → "WATCHING — bullish, 1 of 2"
elif newest.close < OR_low → "WATCHING — bearish, 1 of 2"
else → "NO BREAKOUT"   (an older bar's breach that reverted is NOT watching)
if now > ENTRY_CUTOFF → final: "NO BREAKOUT all day"
                         (or "WATCHING seen but never confirmed at HH:MM")
```
Record every WATCHING bar's time — the reviews use it for the fakeout rate.

## Step 5 — Freshness / no-chase

```
age = index of the signal bar counted back from the newest completed bar
      (0 = newest, 1 = second-newest)
spy = live SPY last
CALL: fresh = age <= 1 and OR_high < spy <= signal_close * (1 + MAX_CHASE_PCT)
PUT:  fresh = age <= 1 and signal_close * (1 - MAX_CHASE_PCT) <= spy < OR_low
also requires now <= ENTRY_CUTOFF
```
Not fresh → final status `MISSED — <signal HH:MM, age N bars, SPY $x vs
signal close $y>`. No trade today. A missed signal is logged, not chased.

Why: when an hourly run found a signal late, it bought after the move
(10/5 confirmed 10:15, filled 10:23 near the local high, −$116).

## Step 6 — Filters

Evaluate in order; the first failure ends the day as `SKIPPED — <name>`,
recording the values that failed.

```
1. Macro:     Phase 0 classification is NO TRADE → skip
2. OR width:  OR_width_pct > 0.40%  (or > 0.30% on a DATA DAY) → skip
3. VWAP:      vwap = Σ((H+L+C)/3 × volume) / Σ volume over today's completed bars
              CALL and spy <= vwap → skip;  PUT and spy >= vwap → skip
4. VIX:       vix_last > 30, or vix_last / vix_prev_close − 1 > 10% → skip
              CALL and vix change > +5% → skip
5. Levels:    CALL: for L in (prev_high, high_63): if spy < L < spy + 1.00 → skip
              PUT:  if spy − 1.00 < prev_low < spy → skip
6. Budget:    handled in option selection (public-submission.md)
```

## Step 7 — Exit signals evaluated in Phase 2

```
STRUCTURAL: newest completed bar (stamped after entry) closes
            CALL: close < OR_high      PUT: close > OR_low
TIME_STOP:  now − entry_time >= 60 min and bid < entry × 1.10
EOD:        now >= 15:45 ET
```
TP, BREAKEVEN and the disaster stop are premium-based — see config.md §
Exit Rules.

## Daily State
Keep in the session: `OR_high/OR_low/OR_width_pct`, context values, the
signal (if any), WATCHING times, and for an open trade: symbol, entry
fill, entry time, and per lot its qty, resting order id, current stop
price and target. If a new session has to pick up an open position
(Phase 0 step 3), rebuild this from `get_history` (entry fill/time),
`get_orders` (resting stop ids and prices) and today's bars (opening
range).
