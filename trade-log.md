# Trade Log & Weekly Review Reference

## Log Storage: printed in chat (revised 9/22/26)
Google Drive isn't reliably available in this session, and Jeff would
rather manage his own copy anyway. Nothing is auto-written to any file or
doc. Instead, the run prints the day's entry (below) as a chat message at
the end of every run — trade day or no-trade day — and Jeff copies it into
his own log at the end of each day himself. This is not optional: every
run still ends with an entry printed, even on a no-trade day, so there's a
plain-text record Jeff can paste, even if he doesn't do it every single
day.

## What Gets Logged

**Every trading day gets an entry** — not just days with a trade — so the
weekly review can compute real trade-frequency stats (how often a
breakout even confirms vs. how many days go by with nothing).

### No-trade day (one line)
```
## [YYYY-MM-DD] — NO TRADE
Opening Range: $[OR_low]-$[OR_high] | Final status: [NO BREAKOUT all day / WATCHING seen but never confirmed at HH:MM]
```

### Trade day (full entry) — print this at the end of the run, right after
the entry (and stop order) are confirmed filled/resting. There's no later
run that day to record an exit outcome (see § Log Storage above), so this
entry covers entry only — Jeff appends the exit himself when he closes the
position:

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

**Stop order placed:** SELL [N] ct STOP @ $[stop_price] (-30% of entry). No take-profit order.

**Exit:** [Jeff to fill in when he closes the position — price, time, P&L, and whether the stop triggered or he exited manually]

**Post-trade analysis (entry only — Jeff adds exit commentary himself):**
[Honest assessment of the entry — did the 2-bar confirmation help (avoided
a fakeout) or hurt (gave up entry price waiting for confirmation) this
specific trade? Was the SPY-vs-SPX budget choice the right call in
hindsight? Did the projected target ($[projected_target]) look realistic
given where price already was at entry? Anything about this setup worth
flagging for the weekly review.]
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
Since nothing auto-writes to Drive anymore (see § Log Storage), this reads
whatever entries Jeff has pasted into this chat for the review period (the
prior Mon–Fri) — not a Doc. If nothing is available for that week, say so
and stop — don't fabricate a review from nothing. Exit outcomes (P&L, hold
time, whether the stop triggered or Jeff exited manually) are only as
complete as what Jeff filled in on each entry's `**Exit:**` line — flag
any entry missing that.

### What It Computes
```
- Total trading days in the week vs. days with a confirmed trade
  (trade frequency — is the setup rare or common?)
- Stop-out rate: how often the -30% stop triggered vs. Jeff exiting
  manually vs. the position still being open when reviewed
- CALL vs PUT split
- SPY vs SPX split (how often did SPX actually get used, if ever)
- Average P&L per trade and total P&L for the week (only from entries with
  an exit filled in)
- Average time held per trade (only from entries with an exit filled in)
- How many WATCHING events occurred but never confirmed (fakeout rate) —
  this is the key metric for whether the 2-bar confirmation is earning
  its keep or costing too much entry-price slippage
- Projection accuracy: how often did SPY actually reach the projected_target
  logged at entry, and which method (measured move vs. IV expected move)
  tends to be the more conservative/accurate one in practice — this tells
  us whether the strike-bounding logic is calibrated correctly or too
  tight/loose
- Any pattern across the "post-trade analysis" notes worth flagging
  (e.g., "the -30% stop triggers disproportionately on PUT trades")
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
  target, stop-loss percentage, opening range window length) with the
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
Stop-outs: [N] | Manual exits: [N] | Still open / no exit logged: [N]
CALL/PUT split: [N]/[N] | SPY/SPX split: [N]/[N]
Total P&L: $[amount] | Avg per trade: $[amount] | Avg hold time: [X] min
Fakeout rate (WATCHING → never confirmed): [N] of [N] watching events ([X]%)

Recommendation: [keep as-is / specific change / not enough data yet] — [reasoning]
```

This weekly review is a SEPARATE run from the daily 10:10 AM ET check —
see `references/schedule-setup.md` for its own trigger.
