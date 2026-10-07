# Trade Log, Weekly Review & Monthly Review Reference

## Log Storage: Google Drive (not a local file)
The trade log is a **Google Doc** named `ORB Trade Log` — not
`~/orb-trade-log.md` on any machine. Cloud routines have no persistent
filesystem between runs; Drive persists.

- Find it with `search_files` (`title = 'ORB Trade Log'`) before assuming
  it's missing; create it with `create_file` on the first write.
- Write = read-modify-write (Drive has no append): `read_file_content` →
  append the entry → `update_file` with the full content.
- Phase 3 of every session writes the day's entry before the summary is
  shown. Logging is not optional.

Entries are tagged `[PAPER]` when `MODE = PAPER`. Reviews report paper and
live results separately and never blend them.

## What Gets Logged — every trading day

### No-trade day (one line)
```
## [YYYY-MM-DD] — [NO BREAKOUT / MISSED / SKIPPED / NO TRADE]
OR $[low]-$[high] (w [x]%) | Macro [NORMAL/DATA DAY: x] | VIX [x] ([+/-y]%) | Status: [detail]
```
`[detail]` examples:
- `NO BREAKOUT all day` / `WATCHING at 10:35, 11:10 — never confirmed`
- `MISSED — CALL confirmed 10:15 close $780.03, seen 10:22, SPY back inside range`
- `SKIPPED — OR width 0.46% > 0.40%` / `SKIPPED — VWAP: CALL with SPY $x < VWAP $y`
- `NO TRADE — FOMC decision day`

**Hypothetical outcome for MISSED and SKIPPED days:** add one more line —
what the trade would have done under the exit rules, from the option's
5-min history (`get_price_history(symbol=<osi>, period="DAY",
instrument_type="OPTION", aggregation="FIVE_MINUTES")`), e.g.
`Hypothetical: 780C @1.14 10:15 → STRUCTURAL 10:20 @0.95 (−17%)`. This is
how the reviews tell whether a filter is saving money or costing it.

### Trade day (full entry) — written once every lot is closed
```
## [YYYY-MM-DD] — [BULLISH BREAKOUT / BEARISH BREAKDOWN]

**Opening range:** $[OR_low] - $[OR_high] (width [x]%)
**Signal:** [HH:MM] close $[x] and [HH:MM] close $[y]; seen at [HH:MM:SS], age [0/1] bar
**Context:** prior day $[low]-$[high] | 63-day high $[h] | VWAP $[v] | VIX [x] ([+/-y]%) | Macro [..]
**Filters:** all passed — [one line of the values]

**Entry:**
- Underlying: [SPY/SPX] — [reason]
- Projected target: $[t] via [measured $a / IV $b — which was more conservative]
- Strike: $[k] [call/put] 0DTE — [bounded by target / natural pick], Δ[d]
- Contracts: [N] (floor($[BUDGET] / ($[premium] × 100)))
- Fill: $[entry] × [N] = $[cost] at [HH:MM:SS] (signal→fill [s] seconds)

**Lot A ([q] ct, TP +25%):** [TP / BREAKEVEN STOP / STRUCTURAL / TIME STOP / DISASTER SL / EOD]
  @ $[px] [HH:MM] — $[pnl] ([pct]%) | peak bid while held $[max] ([pct]%)
**Lot B ([q] ct, TP +40%):** ... (omit if N=1)

**Day:** held [HH:MM]–[HH:MM] ([m] min) | P&L $[total] ([pct]% of cost)
**Stop history:** [e.g. disaster $0.68 → breakeven $1.16 at 10:31]
**Execution deviations:** [none / anything not per the rules — manual action, replace errors, fallback path used, session restart]

**Post-trade analysis:** [Honest assessment: was the entry timely? Did the
structural/time/breakeven stops help or cut a winner short (check what the
option did after exit)? Did TP fire near the peak or leave a lot on the
table? Did the projected target get reached? Anything the reviews should
know.]
```
Winners and losers both get the full template — an honest record, not a
highlight reel.

---

## Shared Stats (used by both reviews)
```
- Trading days; days by status: TRADED / NO BREAKOUT / MISSED / SKIPPED (by filter) / NO TRADE (by event)
- Trades: CALL vs PUT, SPY vs SPX
- Per lot: count of each exit type (TP, BREAKEVEN, STRUCTURAL, TIME, DISASTER SL, EOD)
- Win rate (P&L > 0) and average win $ vs average loss $  ← the v1 failure mode was +$31 / −$164
- Expectancy per trade = win_rate × avg_win − loss_rate × avg_loss
- Total and average P&L; average hold time; average signal→fill delay
- Fakeout rate: WATCHING events that never confirmed / all WATCHING events
- MISSED count — should be ~0 now that the session watches every bar; if not, execution is broken
- Filter scorecard: for each filter, days it skipped and the sum of their
  hypothetical P&L (negative sum = the filter is saving money)
- Exit scorecard: after STRUCTURAL / TIME exits, did the option go on to
  reach the TP? (how often each stop cuts a winner)
- Projection accuracy: how often SPY reached projected_target, which method was closer
```

---

## Weekly Review

**When:** first trading day of each week, before the session starts,
covering the prior Mon–Fri. If run mid-week on request, cover the partial
current week and say so.

**Reads:** the `ORB Trade Log` doc, filtered to the week. No entries → say
so and stop.

**Computes:** the shared stats for the week.

**Recommends** (always end with one, explicit):
- **Keep as-is**, or
- **Adjust one specific parameter** (name it, old → new value) with the
  reason tied to the week's numbers, or
- **Something structural is off**, or
- **Not enough data yet** — under ~3 trades across all v2 history, say so
  rather than forcing a conclusion. One week (≤ 5 days) is directional only.

**Output:**
```
📊 WEEKLY REVIEW — Week of [Mon] - [Fri]
Days: [N] | Traded [N] | No breakout [N] | Missed [N] | Skipped [N] ([by filter]) | No trade [N]
Exits (A / B): TP [n/n] | BE [n/n] | Structural [n/n] | Time [n/n] | SL [n/n] | EOD [n/n]
Win rate [x]% | Avg win $[w] / avg loss $[l] | Expectancy $[e]/trade
Total P&L $[x] | Avg hold [m] min | Avg signal→fill [s] s
Fakeouts: [n] of [n] WATCHING events ([x]%)
Filters: [name: skipped n, hypothetical $x] ...

Recommendation: [...] — [reasoning]
```

---

## Monthly Review

**When:** first trading day of each month (scheduled routine) or when Jeff
asks "run the monthly review". Covers the prior calendar month; run
mid-month on request, it covers month-to-date and says so.

**Reads:** the `ORB Trade Log` doc, filtered to the month. Also pulls the
account's actual fills for the month from
`get_history(account_id="5OI27877", start=<month start>, end=<month end>)`
and reconciles: every BUY/SELL of a SPY/SPX option should map to a logged
trade. List any fill with no log entry, and any logged trade with no fill.
If the log has no entries for the month, say so plainly, still show the
account-fills P&L for the month, and stop — don't fabricate a review.

**Computes:**
- The shared stats for the month
- The same stats per week of the month, side by side, so a trend is
  visible (is expectancy improving? are filters skipping more over time?)
- Account P&L from fills vs log P&L (they should match; explain gaps)
- Max drawdown: largest peak-to-trough drop in cumulative daily P&L
- Longest losing streak
- Starting vs ending account value for the period and the % change
- Comparison with prior months (v1 baseline: Sep 18 – Oct 5, 2026 lost
  $500 of $1,000, 5 wins avg +$31 / 4 losses avg −$164, 2 re-entries)
- Every rule change made during the month (from git history of this repo
  or the log's notes) and the stats before vs after it

**Recommends:** the monthly review is the one allowed to recommend
**bigger** changes, since it has a 15–23-day sample:
- Keep as-is
- Change specific parameters (each with old → new and the evidence)
- Change the budget (only up after ≥ 2 consecutive positive-expectancy
  months; down after a > 20% drawdown)
- **Pause live trading → PAPER** if expectancy is negative over ≥ 10
  trades, or drawdown > 30% of the starting account value
- Not enough data — fewer than ~8 trades in the month

**Output:**
```
📅 MONTHLY REVIEW — [Month YYYY]  ([full month / month-to-date through DATE])
Account: $[start] → $[end] ([+/-x]%) | Fills P&L $[x] | Log P&L $[y] | Reconciled: [yes / N gaps — list]
Days: [N] | Traded [N] | No breakout [N] | Missed [N] | Skipped [N] | No trade [N]
Win rate [x]% | Avg win $[w] / avg loss $[l] | Expectancy $[e]/trade
Max drawdown $[d] ([x]%) | Longest losing streak [n]
Exits (A / B): TP [n/n] | BE [n/n] | Structural [n/n] | Time [n/n] | SL [n/n] | EOD [n/n]
Filters: [name: skipped n, hypothetical $x] ...
By week: [W1 exp $x, P&L $y] | [W2 ...] | ...
vs prior month: [...]

Recommendation: [...] — [reasoning]
```
