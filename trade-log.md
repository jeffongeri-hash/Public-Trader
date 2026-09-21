# Trade Log & Weekly Review Reference

## Log Storage: Google Drive (not a local file)
The trade log lives in a **Google Doc**, not `~/orb-trade-log.md` on any
particular machine — this strategy may run from a local Mac (launchd) or
from a Claude Code cloud Routine with no persistent local filesystem
between runs, so the log needs to live somewhere both can reach.

**Doc name:** `ORB Trade Log` (create it once via `Google_Drive__create_file`
if it doesn't already exist; search for it by name first via
`Google_Drive__search_files` before assuming it's missing).

**Write pattern (read-modify-write, since Drive has no true append):**
1. `Google_Drive__read_file_content` on the doc to get current contents
2. Append the new entry's markdown text to the end
3. `Google_Drive__update_file` with the full updated content

This happens automatically as part of every day's 12:00 PM CT run — logging
is not optional or a "remember to do this later" step. Every trading day
gets an entry written to the Doc before that run's summary is displayed.

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
(both lots resolved, or forced close), not at entry, so the log always
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
- Outcome: [TP-A HIT / SL-A HIT / FORCED CLOSE]
- Exit price: $[price] at [HH:MM]
- P&L (Lot A): $[gain/loss] ([+/-X]% on premium)

**Exit — Lot B ([tier2_qty] ct, target +40%/-30% — omit this block if N=1):**
- Outcome: [TP-B HIT / SL-B HIT / FORCED CLOSE]
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
The `ORB Trade Log` Google Doc, filtered to the review period (the prior
Mon–Fri). If the doc is missing or has no entries for that week, say so
and stop — don't fabricate a review from nothing.

### What It Computes
```
- Total trading days in the week vs. days with a confirmed trade
  (trade frequency — is the setup rare or common?)
- Win rate PER LOT: Lot A TP-A hits vs. SL-A hits vs. forced closes, and
  the same breakdown for Lot B — since they now have different targets
  (+25% vs +40%), their hit rates should be tracked and reported
  separately, not blended into one "win rate" number
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
Win rate: [N] TP / [N] SL / [N] forced close ([X]% win rate)
CALL/PUT split: [N]/[N] | SPY/SPX split: [N]/[N]
Total P&L: $[amount] | Avg per trade: $[amount] | Avg hold time: [X] min
Fakeout rate (WATCHING → never confirmed): [N] of [N] watching events ([X]%)

Recommendation: [keep as-is / specific change / not enough data yet] — [reasoning]
```

This weekly review is a SEPARATE run from the 5-minute intraday polling —
see `references/schedule-setup.md` for its own trigger.
