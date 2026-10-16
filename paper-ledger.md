# ORB Paper Ledger  [PAPER]

Paper trading started **2026-10-12** on an assumed **$1,000** account
(`PAPER_START`), after live trading was cancelled (live account 5OI27877 ended
at $149.43). Fill rules: config.md § Mode. One entry per trading day, formats
in trade-log.md. Reviews (weekly/monthly) read this file.

Running equity = $1,000.00 + sum of `Day P&L` below.

| Date | Status | Day P&L | Equity |
|---|---|---|---|
| 2026-10-12 | TRADED — PUT, BREAKEVEN stop both lots | −$0.04 | $999.96 |
| 2026-10-13 | TRADED — PUT, STRUCTURAL both lots | −$44.08 | $955.88 |
| 2026-10-14 | SKIPPED — OR width 0.41% > 0.30% (CPI data day) | $0.00 | $955.88 |
| 2026-10-15 | SKIPPED — OR width 0.39% > 0.30% (PPI/retail sales data day) | $0.00 | $955.88 |
| 2026-10-16 | TRADED — PUT, TP both lots | +$109.96 | $1,065.84 |

## Entries
(newest last)

## 2026-10-12 — BEARISH BREAKDOWN  [PAPER]

**Opening range:** $778.32 - $779.40 (width 0.14%)
**Signal:** 10:05 close $777.76 and 10:10 close $777.79 (both below OR low); seen at 10:15:20 ET, age 0 bars
**Context:** prior day $775.14-$779.42 | 63-day high $781.62 | VWAP $778.77 | VIX 15.36 (+3.5%) | Macro NORMAL (Columbus Day; CPI is Wed 10/14)
**Filters:** all passed — OR width 0.14% <= 0.40%; SPY 777.86 < VWAP 778.77; VIX 15.36 (<30, +3.5% < 10%); prior-day low 775.14 more than $1.00 below price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $777.24 via measured move (OR height 1.08 below 778.32); IV method was farther away
- Strike: $778 put 0DTE (SPY261012P00778000), Δ-0.50 — FLAG: no integer strike lies inside the bound [777.24, 777.86]; took the nearest strike with |Δ| >= 0.40 (777P was Δ-0.32)
- Contracts: 2 (floor($200 / ($0.80 x 100)))
- Fill [PAPER]: $0.80 x 2 = $160.00 at 10:15:44 ET (ask; signal->fill ~ 34 s after bar close)

**Lot A (1 ct, TP +25% = $1.00):** BREAKEVEN STOP @ $0.80 10:28:38 — $0.00 gross (0%) | peak bid while held $0.97 (+21%)
**Lot B (1 ct, TP +40% = $1.12):** BREAKEVEN STOP @ $0.80 10:28:38 — $0.00 gross (0%) | peak bid $0.97

**Day:** held 10:15-10:28 (13 min) | P&L −$0.04 (fees $0.01 x 4), −0.0% of cost
**Stop history:** disaster $0.48 -> breakeven $0.82 at 10:27 (bid 0.97). Stop gap-filled at $0.80 (stop −$0.02 per config) when bid printed $0.81.
**Execution deviations:** none (no orders placed; PAPER).

**Post-trade analysis:** Entry was timely (<1 min after the signal bar closed). The breakeven ratchet fired at +21% bid but the +25% TP at $1.00 was 3 cents away — the trade gave back the move and was stopped flat; the −$0.02 stop slippage convention is what turned a +$0.02 stop into a $0.80 fill. SPY never closed back inside the range (structural stop not hit), so the thesis held; the option decayed/chopped on a very narrow day (range 0.14%, 0DTE theta heavy). Worth watching in reviews: breakeven at entry+$0.02 on a 0.80 premium leaves very little room.

## 2026-10-13 — BEARISH BREAKDOWN  [PAPER]

**Opening range:** $777.16 - $778.75 (width 0.20%)
**Signal:** 10:50 close $776.99 and 10:55 close $776.89 (both below OR low); seen at 11:00:23 ET, age 0 bars. WATCHING seen at 10:50 (1 of 2) before confirming; price sat inside the range 10:05-10:45 (10:30 close exactly on OR low $777.16)
**Context:** prior day $777.05-$779.40 | 63-day high $781.62 | VWAP $777.81 | VIX 15.33 (+0.8%) | Macro NORMAL (bank earnings, existing home sales; CPI is Wed 10/14)
**Filters:** all passed — OR width 0.20% <= 0.40%; SPY 776.94 < VWAP 777.81; VIX 15.33 (<30, +0.8%); prior-day low 777.05 already cleared (above price)

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $775.57 via measured move (OR height 1.59 below 777.16); IV method (IV ~1.8) was far farther
- Strike: $776 put 0DTE (SPY261013P00776000), Δ-0.32 — FLAG: only integer strike inside the bound [775.57, 776.94]; below the 0.40 delta preference (777P, Δ-0.51, sits above price/outside the bound)
- Contracts: 4 (floor($200 / ($0.42 x 100)))
- Fill [PAPER]: $0.42 x 4 = $168.00 at 11:00:45 ET (ask; signal->fill ~ 45 s after bar close)

**Lot A (2 ct, TP +25% = $0.525):** STRUCTURAL @ $0.31 11:15:22 — −$22.00 gross (−26%) | peak bid while held $0.46 (+10%)
**Lot B (2 ct, TP +40% = $0.588):** STRUCTURAL @ $0.31 11:15:22 — −$22.00 gross (−26%) | peak bid $0.46

**Day:** held 11:00-11:15 (15 min) | P&L −$44.08 (gross −$44.00, fees $0.01 x 8), −26.2% of cost
**Stop history:** disaster $0.25 never touched (low bid $0.32); breakeven trigger $0.483 never reached.
**Execution deviations:** none (no orders placed; PAPER). Exit fill = bid $0.32 minus $0.01 MARKET slippage.

**Post-trade analysis:** Entry was timely (45 s after the confirming bar) but the signal was thin: SPY had chopped inside the range for 45 min and only slipped ~0.4 below OR low on the confirming bars, so the put was bought at $0.42 with delta -0.32 and no momentum behind it. The 11:10 bar closed at $777.17, 1 cent above OR low $777.16, and fired the structural stop; the option bid was $0.32 (-24%) at the time, so the stop cost most of the loss rather than saving it — it was already in the red when the thesis broke. The option never got closer than +10% to the +15% breakeven trigger. Worth watching in reviews: entries confirmed 80+ minutes after the open on a narrow range, and a 1-cent structural margin.

## 2026-10-14 — SKIPPED — OR width (CPI data day)  [PAPER]

**Opening range:** $773.20 - $776.35 (width 0.41%)
**Signal:** PUT — 10:05 close $773.01 and 10:10 close $772.90 both below OR low; fresh at the 10:15 check (age 0)
**Context:** prior day $775.95-$778.75 | 63-day high $781.62 | VIX 15.43 at open, 15.90 (+4.1%) at 11:35 | Macro DATA DAY: Sep CPI 8:30 ET (headline fcst +0.6% m/m), Beige Book 2:00 PM
**Filter failed:** OR width 0.41% > 0.30% (data-day limit; also > 0.40% normal limit). No trade, no position.
**Outcome (hypothetical, not traded):** SPY kept sliding, 771.64 at 11:35 (−0.82% on the day), never back inside the range — the skipped PUT would have been a winner. Worth tracking in the monthly review's filter stats: OR-width skips on data days.

## 2026-10-15 — SKIPPED — OR width (PPI / retail sales data day)  [PAPER]

**Opening range:** $764.43 - $767.44 (width 0.39%)
**Signal:** CALL — 10:30 close $767.80 and 10:35 close $768.20 both above OR high; seen at 10:40:16 ET, age 0 bars (fresh, SPY ~$768.14). WATCHING at 10:15 (close $767.53, 1 of 2; 10:20 closed exactly on OR high $767.44, no pair)
**Context:** prior day $766.71-$776.35 | 63-day high $781.62 | VIX 17.67 (+4.7%) at open | Macro DATA DAY: Sep PPI (fcst +0.5% m/m) and retail sales (+0.3%) 8:30 ET; Fed speaker times unconfirmed (none found in 10:00-11:30 ET)
**Filter failed:** OR width 0.39% > 0.30% (data-day limit). No trade, no position. VWAP/level filters not evaluated (first failure ends the day).
**Outcome (hypothetical, not traded):** not computed from option history; SPY was $768.14 at 10:40, price opened -0.4% gap-down and was reclaiming the range. Track in the monthly filter scorecard: OR-width skips on data days (10/14, 10/15).

## 2026-10-16 — BEARISH BREAKDOWN  [PAPER]

**Opening range:** $771.00 - $773.25 (width 0.29%)
**Signal:** 11:00 close $770.99 and 11:05 close $769.74 (both below OR low); seen at 11:10:20 ET, age 0 bars. WATCHING at 10:25 (close $770.91, 1 of 2) did not confirm (10:30 closed back inside at $771.44); price chopped inside the range 10:00-10:55
**Context:** prior day $764.43-$772.87 | 63-day high $781.62 | VWAP $771.67 | VIX 16.16 (-1.8%) | Macro DATA DAY: monthly opex (3rd Friday); no data releases found; Fed Chair Warsh IMF talk 11:30 PM ET Thu 10/15, outside the session
**Filters:** all passed — OR width 0.29% <= 0.30% (data-day limit); SPY 769.48 < VWAP 771.67; VIX 16.16 (<30, -1.8%); prior-day low 764.43 more than $1.00 below price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $768.75 via measured move (OR height 2.25 below 771.00)
- Strike: $769 put 0DTE (SPY261016P00769000), Δ-0.43 — only integer strike inside the bound [768.75, 769.48]
- Contracts: 2 (floor($200 / ($0.98 x 100)))
- Fill [PAPER]: $0.98 x 2 = $196.00 at ~11:10:45 ET (ask; ~25 s after the signal bar closed)

**Lot A (1 ct, TP +25% = $1.225):** TP @ $1.53 11:22:40 — +$55.00 gross (+56%) | low bid while held $0.68 (-31%)
**Lot B (1 ct, TP +40% = $1.372):** TP @ $1.53 11:22:40 — +$55.00 gross (+56%) | peak bid $1.53

**Day:** held 11:10-11:22 (12 min) | P&L +$109.96 (gross +$110.00, fees $0.01 x 4), +56.1% of cost
**Stop history:** disaster $0.588 never touched (low bid $0.68); breakeven trigger $1.127 never fired separately — the bid jumped from $1.10 to $1.53 between two 30 s polls (SPY 769.12 -> 768.47), clearing both TPs at once.
**Execution deviations:** none (no orders placed; PAPER). Both TP fills assume a LIMIT at the polled bid of $1.53 filled in full, per config.md § Mode; the bid moved fast and a real fill could differ.

**Post-trade analysis:** Entry was timely (~25 s after the confirming bar) on a signal that took 70 minutes of chop to form. The option drew down as far as -31% (bid $0.68) within the first 5 minutes while SPY bounced ~$0.50, close to but well above the -40% disaster stop, then rallied +56% in one 30 s step when SPY dropped $0.65. Order-of-magnitude swings in 30 s are normal for a 0DTE put with gamma 0.13 and IV 278%. The 11:15 structural check was far from firing (needs a close above 771.00). Worth watching in reviews: a paper TP that fills both lots at a single polled bid after a gap; the 0.30% data-day OR limit was cleared by only 1 bp (0.29%).
