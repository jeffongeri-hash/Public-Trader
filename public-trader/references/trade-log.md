# Trade Log & Weekly Review Reference

## Log Storage: `orb-trade-log.md` in this repo
The log is a markdown file at the repo root, `orb-trade-log.md` (newest
entry last). A cloud routine has no persistent filesystem, so the file is
committed and pushed. (The old Google Doc approach was dropped: the Drive
connector can only rename/move files, not write content.)

**Write pattern (2:45 PM CT run, once the day's outcome is final):**
1. Read `orb-trade-log.md` (create it with a `# ORB Trade Log` heading if
   missing).
2. Append the day's entry (templates below).
3. `git add orb-trade-log.md && git commit -m "Log ORB result [YYYY-MM-DD]"`
4. `git push -u origin <current branch>` — retry up to 4 times (2s, 4s, 8s,
   16s) on network errors. Never create a PR for this.

Logging is not optional — every trading day gets an entry before that
run's summary is displayed. If the push fails, say so in the summary.

## What Gets Logged

**Every trading day gets an entry** — not just days with a trade — so the
weekly review can compute real trade-frequency stats (how often a
breakout even confirms vs. how many days go by with nothing).

### No-trade day (one line)
```
## [YYYY-MM-DD] — NO TRADE
Opening Range: $[OR_low]-$[OR_high] | Final status: [NO BREAKOUT all day / WATCHING seen but never confirmed at HH:MM]
```

### Trade day (full entry) — write this the moment the position is CLOSED
(both lots resolved, or forced EOD close), not at entry, so the log always
contains the complete outcome, not a half-finished one:

```
## [YYYY-MM-DD] — [BULLISH BREAKOUT / BEARISH BREAKDOWN]

**Opening Range:** $[OR_low] - $[OR_high]
**Confirmation bars:** [HH:MM] close $[x] and [HH:MM] close $[y], both [above/below] the range
**Prior day range (context):** $[prev_low] - $[prev_high]

**Entry decision:**
- Underlying chosen: [SPY/SPX] — [reason: e.g. "SPY fit budget at $X, SPX ATM premium was $Y, over budget"]
- Projected target: $[projected_target] (via [measured move / IV expected move] — [show both numbers and which was more conservative])
- Strike/expiry: $[strike] [call/put], 0DTE exp [date] — [note whether this was bounded by the projected target, or was the natural ATM/delta-0.40 pick anyway]
- Delta at entry: [delta]
- Contracts: [N] (floor($400 / ($[premium] x 100)))
- Entry fill: $[premium] x [N] = $[total cost]
- Rationale: [why this was/wasn't a well-timed entry given where price was relative to the range, any notable context like VIX level or overall day trend if visible from the bars]

**Exit — Lot A ([tier1_qty] ct, target +25%/-30%):**
- Outcome: [TP-A HIT / SL-A HIT / EOD FORCED CLOSE]
- Exit price: $[price] at [HH:MM]
- P&L (Lot A): $[gain/loss] ([+/-X]% on premium)

**Exit — Lot B ([tier2_qty] ct, target +40%/-30% — omit this block if N=1):**
- Outcome: [TP-B HIT / SL-B HIT / EOD FORCED CLOSE]
- Exit price: $[price] at [HH:MM]
- P&L (Lot B): $[gain/loss] ([+/-X]% on premium)

**Blended outcome:**
- Total time held: entry [HH:MM] to final exit [HH:MM]
- Total P&L: $[sum of both lots] ([+/-X]% on total cost)
- Note explicitly if execution deviated from the planned tiered exit (e.g.
  manual override, different split than ceil(N/2), scaled out differently)
  — the weekly review needs to know when a trade doesn't actually test
  the systematic rule.

**Post-trade analysis:**
[Honest assessment — did Lot A's closer +25% target actually get hit, or
did premium fall short/overshoot straight past it? Did the runner (Lot B)
benefit from riding toward +40%, or did giving it more room just give back
gains that Lot A's exit had already locked in? Did the 2-bar confirmation
help (avoided a fakeout) or hurt (gave up entry price waiting for
confirmation) this specific trade? Was the SPY-vs-SPX budget choice the
right call in hindsight? Did the projected target ($[projected_target])
actually get reached, overshot, or fall short — and did bounding the
strike to it help or cost anything relative to what an unbounded pick
would've done? Anything about this trade that should inform the weekly
review.]
```

Log every trade — winners and losers both get the full template. The point
is an honest record, not a highlight reel.

---

## Weekly Review

### When
First trading day of each week (typically Monday, 8:00 AM CT — before the
intraday polling starts), reviewing the just-completed prior week (last
Monday through Friday). If a manual run is requested mid-week, review
whatever partial data exists for the current week and say so explicitly —
don't wait silently for week-end.

### What It Reads
`orb-trade-log.md`, filtered to the review period (the prior
Mon–Fri). If the doc is missing or has no entries for that week, say so
and stop — don't fabricate a review from nothing.

### What It Computes
```
- Total trading days in the week vs. days with a confirmed trade
  (trade frequency — is the setup rare or common?)
- Win rate PER LOT: Lot A TP-A hits vs. SL-A hits vs. EOD closes, and the
  same breakdown for Lot B — since they now have different targets (+25%
  vs +40%), their hit rates should be tracked and reported separately, not
  blended into one "win rate" number
- CALL vs PUT split
- SPY vs SPX split (how often did SPX actually get used, if ever)
- Average P&L per trade (blended across both lots), and total P&L for the week
- Average time held per trade
- How many WATCHING events occurred but never confirmed (fakeout rate) —
  this is the key metric for whether the 2-bar confirmation is earning
  its keep or costing too much entry-price slippage
- How often Lot A's +25% target is reached but Lot B's +40% target is NOT
  (i.e., is the runner actually earning its keep, or is it consistently
  giving back what Lot A already locked in?)
- Projection accuracy: how often did SPY actually reach the projected_target
  logged at entry, and which method (measured move vs. IV expected move)
  tends to be the more conservative/accurate one in practice — this tells
  us whether the strike-bounding logic is calibrated correctly or too
  tight/loose
- Any pattern across the "post-trade analysis" notes worth flagging
  (e.g., "SL hit disproportionately on PUT trades", "Lot B rarely reaches
  +40% and usually round-trips to its stop instead")
```
A single week is a small sample (at most 5 trading days) — say so plainly
rather than drawing strong conclusions from 1-2 trades. Treat early weekly
reviews as directional signal, not a verdict; patterns that hold across
several consecutive weekly reviews are worth acting on, a single odd week
usually isn't.

### What It Recommends
End with an explicit, opinionated recommendation — not just numbers:
- **Keep as-is**, or
- **Adjust a specific parameter** (confirmation bar count, budget, delta
  target, TP/SL percentages, opening range window length) with the
  reasoning tied directly to what the week's data showed, or
- **Something structural is off** (e.g., trade frequency too low to be
  worth running, or SPX is never actually usable given the budget so that
  branch of the logic is dead weight)
- **Not enough data yet** — if fewer than ~3 trades have occurred across
  the tracked history so far, say that explicitly rather than forcing a
  confident recommendation out of a tiny sample

Write this as a clear, standalone summary Jeff can act on without having
to re-derive the stats himself.

### Output Format
```
📊 WEEKLY REVIEW — Week of [Mon date] - [Fri date]
Trading days: [N] | Trades taken: [N] ([X]% of days)
Win rate: [N] TP / [N] SL / [N] EOD close ([X]% win rate)
CALL/PUT split: [N]/[N] | SPY/SPX split: [N]/[N]
Total P&L: $[amount] | Avg per trade: $[amount] | Avg hold time: [X] min
Fakeout rate (WATCHING → never confirmed): [N] of [N] watching events ([X]%)

Recommendation: [keep as-is / specific change / not enough data yet] — [reasoning]
```

---

## Monthly Review

### When
First trading day of each month (or on demand), reviewing the prior
calendar month from `orb-trade-log.md`. If the log is missing or has no
entries for that month, say so and stop — never fabricate a review.

### What It Computes
Everything in the weekly review (trade frequency, per-lot TP/SL/EOD
outcomes, CALL/PUT and SPY/SPX splits, total and average P&L, average hold
time, fakeout rate, runner-vs-Lot-A, projection accuracy), over the whole
month, plus:
```
- Week-by-week P&L (so one bad week is visible against the month)
- Win rate and average win vs. average loss (payoff ratio) — and expectancy
  per trade = win% x avg win - loss% x avg loss
- Max drawdown across the month's trades (running P&L peak to trough)
- Entry-time buckets (before 10:30 ET / 10:30-12:00 / after 12:00): which
  window produced the winners and the stop-outs
- Rule adherence: how many trades followed the tiered exit and the delayed
  (10-minute) stop exactly vs. deviated (manual or early exits)
- Fees as % of gross P&L
- Month P&L as % of account value at start of month
```
Flag sample size honestly: under ~20 trades is directional only.

### What It Recommends
Same as the weekly review: keep as-is, adjust one specific parameter tied
to the data, flag something structural, or "not enough data yet". Weigh
patterns that persisted across multiple weeks above single-week noise.

### Output Format
```
📅 MONTHLY REVIEW — [Month YYYY]
Trading days: [N] | Trades taken: [N] ([X]% of days)
Per-lot: A [N] TP / [N] SL / [N] EOD | B [N] TP / [N] SL / [N] EOD
Win rate: [X]% | Avg win $[x] | Avg loss $[y] | Expectancy $[z]/trade
CALL/PUT: [N]/[N] | SPY/SPX: [N]/[N]
Total P&L: $[amount] ([X]% of starting balance) | Max drawdown: $[amount]
Weekly P&L: W1 $[a] | W2 $[b] | W3 $[c] | W4 $[d]
Entry-time buckets: [summary]
Rule adherence: [N] of [N] trades followed the rules
Fakeout rate: [N] of [N] watching events ([X]%)

Recommendation: [keep / specific change / not enough data] — [reasoning]
```

---

This weekly review is a SEPARATE run from the 5-minute intraday polling —
see `references/schedule-setup.md` for its own trigger.
