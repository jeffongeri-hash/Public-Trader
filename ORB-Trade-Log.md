# ORB Trade Log

Actual trade history for the SPY Opening Range Breakout 0DTE strategy,
account 5OI27877. This is the real log referenced by `trade-log.md` — kept
here in the repo (as a GitHub file) rather than the Google Doc, since the
Google Drive connector wasn't enabled for the session that ran this
strategy this week. Entries below Sep 21–25 are backfilled from account
transaction history (`get_history`) after the fact, since no live intraday
log was written for those days — opening range, confirmation-bar, and
projected-target detail that only exists in a live run's context could not
be recovered for days before 9/25, and is noted as unavailable where
missing.

---

## 2026-09-21 — BULLISH BREAKOUT (CALL) — TWO ENTRIES (gate issue, see analysis)

**Opening Range / confirmation bars:** not recoverable (backfilled from trade history only; no live run log exists for this day)

### Trade 1
- Underlying: SPY, 770 strike CALL, 0DTE

**Entry fill:** 5 contracts @ $0.71 = $355.00 at 11:15:29 AM ET
**Exit:** all 5 contracts closed within ~1 minute (11:16:22–11:16:23 AM ET) — 3 @ $0.73, 2 @ $0.72 = $363.25 proceeds
**P&L:** +$8.55 (+2.4% on cost)
**Note:** exit price doesn't correspond to any TP (+25%/+40%) or SL (-30%) level in the current rule — this looks like an ad hoc/manual exit rather than the systematic tiered rule, and predates the 9/23 incident fix that established the current two-lot structure.

### Trade 2 (same day, same contract)
**Entry fill:** 3 contracts @ $0.77 = $231.00 at 11:18:41 AM ET — **3 minutes after Trade 1 fully closed**
**Exit:** 2 @ $0.85 (11:19:45 AM ET), 1 @ $0.81 (11:21:41 AM ET) = $251.15 proceeds
**P&L:** +$20.33 (+8.8% on cost)
**Note:** same issue as Trade 1 — exit doesn't match the tiered TP/SL rule.

**Day total P&L: +$28.88**

**Post-trade analysis:** Two separate entry/exit cycles fired on the same underlying the same day. If both were triggered by the strategy's own breakout check (rather than manual intervention), this is the same one-trade-per-day gate failure documented on 9/25 below — the gate only checks whether a position is *currently* open, so once Trade 1 closed, the next run had a flat portfolio and treated it as a fresh entry opportunity. Both trades happened to be profitable, but the mechanism is unreliable, not evidence the gate works.

---

## 2026-09-22 — NO TRADE

No transactions on the account this day. Opening range / final status not recoverable (no live run log exists for this day).

---

## 2026-09-23 — BEARISH BREAKDOWN (PUT) — the documented incident day

**Opening Range / confirmation bars:** not recoverable (backfilled from trade history only)

**Entry decision:**
- Underlying: SPY 769 strike PUT, 0DTE
- Entry fill: 1 @ $0.90 + 3 @ $0.89 = 4 contracts, $357.00 total, avg ≈ $0.8925, at 10:14:23 AM ET

**Exit — Lot (2 ct):**
- Outcome: SL fired at 10:14:35 AM ET — **12 seconds after entry**
- Exit price: $0.81 (proceeds $162.00) — this is the incident referenced in `public-submission.md`: an SL submitted as a plain `LIMIT` order priced below the live market filled immediately instead of resting, closing this lot for a loss (-9.2% vs. entry) almost instantly instead of waiting for an actual -30% move.
- P&L: -$16.20 (approx., before fees) on this lot

**Exit — Lot (2 ct):**
- Outcome: closed at 10:19:03 AM ET, ~4.5 minutes later
- Exit price: $1.07 (proceeds $214.00), a legitimate gain
- P&L: +$36.10 (approx.) on this lot

**Blended outcome:**
- Total time held: 10:14:23 AM – 10:19:03 AM ET (~4.7 min for the full round trip)
- **Total P&L: +$19.44** (day nets positive despite the mis-handled lot)

**Post-trade analysis:** This is the incident that led to the STOP_LIMIT-only fix in the current skill (`public-submission.md` / `config.md`, revised 9/23/26). The day nets a small gain only because the second lot's exit was strong enough to offset the botched near-instant exit on the first lot — the underlying execution bug was real and costly on its own terms, just not on this day's net P&L.

---

## 2026-09-24 — BULLISH BREAKOUT (CALL)

**Opening Range / confirmation bars:** not recoverable (backfilled from trade history only)

**Entry decision:**
- Underlying: SPY 766 strike CALL, 0DTE
- Entry fill: 3 contracts @ $1.29 = $387.00 at 10:14:39 AM ET

**Exit — Lot A (1 ct):**
- Outcome: SL-A hit
- Exit price: $0.90 at 10:43:43 AM ET (-30.2% vs entry — matches the -30% stop almost exactly)
- P&L: -$0.39 × 1 ×... net -$39 approx (entry $129 vs exit $90)

**Exit — Lot B (2 ct):**
- Outcome: SL-B hit, 9 seconds later
- Exit price: $0.89 at 10:43:52 AM ET (-31.0% vs entry)
- P&L: ≈ -$80 on this lot

**Blended outcome:**
- Total time held: 10:14:39 – 10:43:52 AM ET (~29 min)
- **Total P&L: -$118.67** (-30.7% on total cost)

**Post-trade analysis:** Clean mechanics — both lots hit their -30% stop within seconds of each other, consistent with a sharp reversal against the position. No TP was reached. This is the first day this week where execution matches the current tiered-exit design as documented.

---

## 2026-09-25 — BEARISH BREAKDOWN (PUT) — TWO ENTRIES (one-trade-per-day gate failure, confirmed)

### Trade 1 (legitimate entry)

**Opening Range (9:30-10:00 ET):** $767.75 – $770.31
**Confirmation bars:** 10:10 AM ET close $766.94 and 10:15 AM ET close $766.54, both below the opening range low
**Prior day range (context):** $763.25 – $768.95

**Entry decision:**
- Underlying chosen: SPY — cheap enough to stay far under the $400 budget; SPX not evaluated since SPY premium was well within budget
- Projected target: $765.19 via measured move (range height $2.56 below OR_low $767.75). The IV-implied method was discarded — `get_option_greeks` returned an implied vol of 344.88% on the ATM strike, an obvious data anomaly, so it wasn't used
- Strike/expiry: $767 PUT, 0DTE exp 2026-09-25 — the only bounded candidate (767, 766, 765) whose |delta| cleared the 0.40 floor (766 → -0.39, 765 → -0.31)
- Delta at entry: -0.4953
- Contracts: 2 (floor($400 / ($1.50 limit × 100)))
- Entry fill: $1.29 avg × 2 = $258.00 (limit was $1.50, filled better)
- Rationale: well-timed — both confirmation bars closed solidly below the range, no fakeout

**Exit — Lot A (1 ct, target +25%/-30%):**
- Outcome: SL-A HIT
- Exit price: $0.90 at 10:37:19 AM ET
- P&L (Lot A): -$39.00 (-30.2% on premium)

**Exit — Lot B (1 ct, target +40%/-30% — same strike):**
- Outcome: SL-B HIT, same moment
- Exit price: $0.90 at 10:37:19 AM ET
- P&L (Lot B): -$39.00 (-30.2% on premium)

**Blended outcome:**
- Total time held: 10:21:05 – 10:37:19 AM ET (~16 min)
- **Total P&L: -$77.78** (-30.2% on total cost)
- Execution matched the systematic rule exactly — both lots' STOP_LIMIT orders confirmed resting as `NEW` after placement, both triggered together as SPY continued falling through both stops.

**Post-trade analysis:** Clean, correct mechanics — the breakdown was real (confirmed by both bars), but SPY kept falling past both stops rather than bouncing, so this was a straightforward -30% loss on a valid signal. Nothing to fix about *this* trade's execution.

### Trade 2 (same day, same contract — manual, confirmed by Jeff)

**Entry fill:** 3 contracts @ $1.00 = $300.00 at 10:42:36 AM ET — ~5 minutes after Trade 1's stop-loss closed the position out
**Exit:** 3 contracts @ $0.16 = $48.16 at 12:15:43 PM ET (~93 min later)
**P&L: -$251.66 (-83.9% on cost)** — the worst trade of the week by far

**Post-trade analysis — correction (9/27/26):** this was **not** an automated
one-trade-per-day gate failure — Jeff confirmed he placed this entry
manually himself, outside the strategy. Earlier analysis in this log
attributed it to the gate only checking "is a position currently open"
rather than "did an entry already fire today"; that gap is real and worth
closing regardless (see `config.md` § One Trade Per Day, revised 9/27/26
to also check the trade log), but it wasn't what happened here. Likewise,
the -84% exit (vs. the intended -30% floor) doesn't necessarily reflect a
STOP_LIMIT slippage bug in the algorithm's exit logic, since this position
wasn't opened by the algorithm and may not have carried the same resting
SL structure at all — it's shown here as a real, painful loss on the
account, not as evidence of a system defect. The separately-confirmed
9/23 incident (an automated SL submitted as a marketable `LIMIT`) remains
the actual documented automated bug from this week.

**Day total P&L: -$77.78 + -$251.66 = -$329.44**

---

# Weekly Review — Week of Sep 21 – Sep 25, 2026

📊 **WEEKLY REVIEW — Week of Sep 21 – Sep 25, 2026**

**Trading days:** 5 | **Days with ≥1 confirmed trade:** 4 of 5 (80%) | **Total trade-events:** 6 (two days had a second entry — see notes)

**Win/loss by trade-event:**
| Date | Direction | Contracts | P&L | Outcome type |
|---|---|---|---|---|
| 9/21 (T1) | CALL | 5 | +$8.55 | Non-standard early exit — automated or manual unconfirmed |
| 9/21 (T2) | CALL | 3 | +$20.33 | Non-standard early exit — automated or manual unconfirmed |
| 9/23 | PUT | 4 | +$19.44 | Mixed: 1 lot mis-fired SL (documented automated bug), 1 lot legit gain |
| 9/24 | CALL | 3 | -$118.67 | Both lots SL, clean, automated |
| 9/25 (T1) | PUT | 2 | -$77.78 | Both lots SL, clean, automated |
| 9/25 (T2) | PUT | 3 | -$251.66 | **Manual trade (confirmed by Jeff), not the algorithm** |

**CALL/PUT split:** 3/3 (by trade-event) | **SPY/SPX split:** 6/0 (SPX never used — consistent with it rarely fitting the $400 budget)

**Total P&L: -$399.79** | **Avg per trade-event: -$66.63** | **Avg per active trading day: -$79.96**

**Hold time:** ranged from ~1 min (9/21 T1) to ~93 min (9/25 T2); median ≈ 9.5 min, average ≈ 24 min (skewed by the 93-min anomalous trade). Most exits this week happened far faster than a real TP/SL wait would suggest, which itself is a flag that several of these (9/21, 9/23's mis-fire) weren't governed by the intended rule.

**Fakeout rate (WATCHING → never confirmed):** unavailable — no intraday run log exists for any day this week (Drive wasn't connected, GitHub logging starts with this file).

**Per-lot TP vs. SL hit rate under the *current* rule:** only 9/24 and 9/25-T1 cleanly follow the current two-lot STOP_LIMIT design — both are 4-for-4 SL hits, 0 TP hits. Too small a sample to say anything about whether +25%/+40% targets are realistic; everything else this week either predates the rule (9/21) or was compromised by the 9/23 incident bug.

## Recommendation (revised 9/27/26 after discussion with Jeff)

**Correction to the original review:** the 9/25 second entry was **not** an
automated one-trade-per-day gate failure — Jeff confirmed he placed it
manually. The catastrophic -84% exit on that trade is a real account loss
but isn't confirmed evidence of a STOP_LIMIT bug in the algorithm, since
that position wasn't opened or (necessarily) protected by the strategy's
own exit logic. Whether 9/21's two quick, non-standard-exit trades were
also manual is still unconfirmed — worth checking with Jeff before drawing
conclusions about the algorithm's actual same-day-only discipline.

**Changes made as of 9/27/26, ahead of the coming week:**
1. **One-trade-per-day gate hardened anyway.** Even though this week's
   double-entries were manual, the gate only ever checked "is a position
   currently open" — a real gap regardless of what triggered it this week.
   It now also checks the trade log itself for any entry already logged
   today (including a new "entry fired" marker written the moment the BUY
   fills, not just at close), independent of current position state. See
   `config.md` § One Trade Per Day and `public-submission.md` § Duplicate /
   Exposure Check.
2. **Stop-loss buffer widened from $0.05 to $0.15.** Jeff's call: keep the
   resting order a `STOP_LIMIT` (not a plain market `STOP`) but give it
   three times the room to actually trigger during a fast move. See
   `config.md` § Exit Rule and `public-submission.md` § ORB Submission
   Sequence. This is a precautionary improvement, not a fix for a
   confirmed bug this week — the 9/23 incident (SL submitted as a
   marketable `LIMIT`) remains the one documented automated stop-handling
   bug so far.
3. **Net P&L this week is -$399.79**, including the manual -$251.66 trade.
   Excluding that manual trade, the algorithm's own trades net -$148.13
   across the days it ran cleanly (9/23, 9/24, 9/25-T1) plus the two
   unconfirmed 9/21 trades. Still nowhere near enough sample to judge the
   underlying edge — revisit after a few clean weeks under the hardened
   gate and wider stop buffer.
4. Google Drive logging will be picked up in a fresh chat session (this
   one's tool set was fixed at startup and can't add it mid-session); this
   file stays the source of truth in the meantime.
