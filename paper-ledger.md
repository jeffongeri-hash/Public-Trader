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
| 2026-10-19 | TRADED — CALL, DISASTER SL (1 lot) | −$47.02 | $1,018.82 |
| 2026-10-21 | TRADED — PUT, STRUCTURAL (1 lot) | −$62.02 | $956.80 |
| 2026-10-22 | SKIPPED — VIX +14.9% on the day (> 10%) | $0.00 | $956.80 |
| 2026-10-23 | TRADED — CALL, BREAKEVEN stop (1 lot) | −$0.02 | $956.78 |
| 2026-10-26 | TRADED — CALL, DISASTER SL (1 lot) | −$48.02 | $908.76 |
| 2026-10-27 | TRADED — CALL, TP both lots | +$57.94 | $966.70 |
| 2026-10-28 | NO TRADE — FOMC decision day | $0.00 | $966.70 |
| 2026-10-29 | TRADED — CALL, DISASTER SL (1 lot) | −$48.02 | $918.68 |
| 2026-10-30 | SKIPPED — VIX +14.48% on the day (> 10%) | $0.00 | $918.68 |
| 2026-11-02 | TRADED — CALL, TP (1 lot) | +$38.98 | $957.66 |
| 2026-11-03 | TRADED — CALL, BREAKEVEN stop (1 lot) | −$0.02 | $957.64 |
| 2026-11-04 | NO BREAKOUT all day (OR width 0.71% would have failed the filter) | $0.00 | $957.64 |

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

## 2026-10-19 — BULLISH BREAKOUT  [PAPER]

**Opening range:** $766.21 - $768.06 (width 0.24%)
**Signal:** 10:10 close $768.34 and 10:15 close $768.64 (both above OR high); seen at 10:20:20 ET, age 0 bars. Price sat inside the range 10:00-10:05
**Context:** prior day $760.64-$773.25 | 63-day high $781.62 | VWAP $767.17 | VIX 17.67 (−11.6%) | Macro NORMAL (no major releases or Fed speakers found; calendar search was thin, Oct 27-28 FOMC not yet)
**Filters:** all passed — OR width 0.24% <= 0.40%; SPY 768.73 > VWAP 767.17; VIX 17.67 (<30, falling); prior-day high 773.25 and 63-day high 781.62 more than $1.00 above price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $769.91 via measured move (OR height 1.85 above 768.06); IV method (IV ~3.05) was far farther
- Strike: $769 call 0DTE (SPY261019C00769000), Δ0.45 — only integer strike inside the bound [768.73, 769.91]
- Contracts: 1 (floor($200 / ($1.12 x 100)))
- Fill [PAPER]: $1.12 x 1 = $112.00 at ~10:20:45 ET (ask; ~25 s after the signal bar closed)

**Lot A (1 ct, TP +25% = $1.40):** DISASTER SL @ $0.65 10:32:55 (stop $0.67 minus $0.02) — −$47.00 gross (−42%) | peak bid while held $1.13 (+1%)

**Day:** held 10:20-10:33 (12 min) | P&L −$47.02 (gross −$47.00, fees $0.01 x 2), −42.0% of cost
**Stop history:** disaster $0.67 never ratcheted; breakeven trigger $1.288 never reached (peak bid $1.13).
**Execution deviations:** none (no orders placed; PAPER). The disaster stop was seen at a 30 s poll (bid $0.65 after $0.75 33 s earlier, SPY 767.19); the polled bid was already at/below the stop, so the fill is simulated at stop minus $0.02 per config. One poll gap of ~70 s around 10:29-10:30 (waiting for the 10:25 bar close); bid was $0.97-$1.05 either side, so no effect.

**Post-trade analysis:** Entry was timely and filters were clean, but the breakout never extended: the 10:20 and 10:25 bars held 768.4-768.8 (barely 0.3-0.7 above the OR high), then SPY slid ~$1.40 in 7 minutes, with the option going $1.05 -> $0.65 between 10:30:32 and 10:32:55. The -40% disaster stop fired before the structural exit could: the 10:30 bar had not yet closed, and SPY was still ~$1.10 above the OR high-minus-close threshold only on bar closes, so the premium-based stop was the binding exit. The option lost 42% on a 0.67% (≈$1.00) SPY pullback to 767.19 — a delta-0.45, gamma-0.12 0DTE call at IV ~300% is very sensitive; a structural exit one bar earlier would have saved ~$0.30 at best. Worth watching in reviews: fast reversals within 15 min of entry on a signal whose 2 bars closed only 0.3-0.6 beyond the boundary (10/13 and 10/19 both thin).

## 2026-10-21 — BEARISH BREAKDOWN  [PAPER]

**Opening range:** $769.73 - $771.34 (width 0.21%)
**Signal:** 10:05 close $768.78 and 10:10 close $768.76 (both below OR low); seen at 10:15:21 ET, age 0 bars. Price sat inside the range 09:30-10:00 (10:00 close $769.90)
**Context:** prior day $764.25-$772.29 | 63-day high $781.62 | VWAP $770.31 | VIX 16.93 (−2.2%) | Macro NORMAL (calendar search found no releases, Fed speakers or mega-cap earnings; thin result, Oct 27-28 FOMC not yet)
**Filters:** all passed — OR width 0.21% <= 0.40%; SPY 768.92 < VWAP 770.31; VIX 16.93 (<30, −2.2%); prior-day low 764.25 more than $1.00 below price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $768.12 via measured move (OR height 1.61 below 769.73)
- Strike: $769 put 0DTE (SPY261021P00769000), Δ-0.51 — FLAG: no integer strike lies inside the bound [768.12, 768.89]; took the near-side strike (769P, above price). 768P (Δ-0.41, ask $1.21) lay past the target
- Contracts: 1 (floor($200 / ($1.62 x 100)))
- Fill [PAPER]: $1.62 x 1 = $162.00 at ~10:15:45 ET (ask; ~45 s after the signal bar closed)

**Lot A (1 ct, TP +25% = $2.03):** STRUCTURAL @ $1.00 10:35:22 — −$62.00 gross (−38%) | peak bid while held $1.76 (+9%), low bid $1.01

**Day:** held 10:15-10:35 (20 min) | P&L −$62.02 (gross −$62.00, fees $0.01 x 2), −38.3% of cost
**Stop history:** disaster $0.97 never touched (low bid $1.01); breakeven trigger $1.863 never reached (peak bid $1.76).
**Execution deviations:** none (no orders placed; PAPER). Exit fill = bid $1.01 minus $0.01 MARKET slippage. Polls ran roughly every 30 s, with a couple of ~1 min gaps around bar-close checks.

**Post-trade analysis:** Entry was timely (~45 s after the confirming bar) on a filter-clean but shallow signal: both signal bars closed only ~$1.0 below OR low. SPY spent 10:10-10:30 drifting from 768.4 back toward the boundary, and the option bled from +9% to -38% on drift and theta. The 10:30 bar closed $0.20 back above OR low and fired the structural stop; the bid at that moment ($1.01) was one cent above the disaster stop, so the structural exit saved only ~$0.05 versus the disaster stop. SPY kept grinding up after the exit, so the thesis failed. Worth watching in reviews: third thin-signal loss in four trades (10/13, 10/19, 10/21 — confirmed with closes only $1-1.6 beyond the range) and the 769P being ITM/near-ATM with theta ~$1.6/hr on a quiet tape.

## 2026-10-22 — SKIPPED — VIX (up 14.9% on the day)  [PAPER]

**Opening range:** $765.35 - $767.45 (width 0.27%)
**Signal:** PUT — 10:20 close $764.42 and 10:25 close $763.34 both below OR low; seen at 10:30:45 ET, age 0 bars (fresh, SPY ~$762.81, inside the 0.10% chase limit). WATCHING bearish at the 10:25 check (10:20 bar alone). SPY gapped down -0.9% at the open (prior close $773.09) and sat inside the range 09:30-10:15
**Context:** prior day $768.40-$774.44 | 63-day high $781.62 | VWAP $766.34 | VIX 19.16 (+14.87% vs prev close 16.68; 18.32, +9.8% at 09:52) | Macro NORMAL (calendar search returned nothing for 10/22 — macro check unavailable; Oct 27-28 FOMC not yet)
**Filters:** macro pass (unverified); OR width 0.27% <= 0.40% pass; SPY 762.81 < VWAP 766.34 pass; **VIX FAIL** — +14.87% > +10% limit. Levels and budget not evaluated (first failure ends the day).
**Outcome:** no trade, no position. Hypothetical outcome not computed. A volatility-spike gap-down day filtered out by the VIX rule; track in the monthly filter scorecard.

## 2026-10-23 — BULLISH BREAKOUT  [PAPER]

**Opening range:** $759.14 - $761.20 (width 0.27%)
**Signal:** 10:40 close $761.64 and 10:45 close $762.33 (both above OR high); seen at 10:50:20 ET, age 1 bar (signal bar 10:45, newest completed 10:50). SPY gapped down -0.5% at the open (prior close $763.96) and sat inside the range 09:30-10:35 (10:15-10:25 closes $759.20/$759.20/$759.65 hugged the OR low without a pair)
**Context:** prior day $761.16-$767.71 | 63-day high $781.62 | VWAP $760.23 | VIX 19.24 (+1.2%; +5.6% at 09:52, which would have blocked a CALL, but the filter is evaluated at entry) | Macro NORMAL (calendar search thin: only UMich sentiment 10:00; no Fed speakers or mega-cap earnings found)
**Filters:** all passed — OR width 0.27% <= 0.40%; SPY 762.40 > VWAP 760.23; VIX 19.24 (<30, +1.2% < 5%); prior-day high 767.71 and 63-day high 781.62 more than $1.00 above price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $763.26 via measured move (OR height 2.06 above 761.20); IV method (IV 23%) was farther
- Strike: $763 call 0DTE (SPY261023C00763000), Δ0.45 — only integer strike inside the bound [762.37, 763.26] (762C, Δ0.54, sits below price)
- Contracts: 1 (floor($200 / ($1.48 x 100)))
- Fill [PAPER]: $1.48 x 1 = $148.00 at ~10:50:50 ET (ask; ~30 s after the signal was seen)

**Lot A (1 ct, TP +25% = $1.85):** BREAKEVEN STOP @ $1.48 10:58:13 (stop $1.50 minus $0.02) — $0.00 gross (0%) | peak bid while held $1.75 (+18%), low bid $1.37 (-7%)

**Day:** held 10:50-10:58 (8 min) | P&L −$0.02 (fees $0.01 x 2), −0.0% of cost
**Stop history:** disaster $0.89 never touched; breakeven ratchet to $1.50 at 10:55:45 (bid $1.73 crossed the $1.70 trigger); stop hit at 10:58:13 (bid $1.48).
**Execution deviations:** none (no orders placed; PAPER). Polls ran about every 30-35 s. SPY never closed back inside the range during the hold (structural not tested; the 10:55 bar had not closed).

**Post-trade analysis:** Entry was timely (signal age 1 bar, ~30 s after seen) and filters were clean, but the option peaked at $1.75, 10 cents short of the +25% TP at $1.85, and then bled back while SPY slipped from 763.05 to 762.53 on heavy 0DTE theta (~$1.5/hr). The breakeven ratchet at +15% protected the capital, and the -$0.02 stop slippage convention turned a +$0.02 stop into a flat fill, same as 10/12. Second breakeven stop in six trades. SPY was still ~$1.3 above the OR high at exit, so the thesis held; the stop cut a trade that stayed alive. Worth watching in reviews: breakeven at entry+$0.02 on a ~$1.50 premium leaves almost no room (a 0.13 pullback from the peak), and the VIX regime flipped from +5.6% to +1.2% within an hour, so the VIX filter's timing matters.

## 2026-10-26 — BULLISH BREAKOUT  [PAPER]

**Opening range:** $761.63 - $764.30 (width 0.35%)
**Signal:** 10:10 close $764.67 and 10:15 close $765.65 (both above OR high); seen at 10:20:25 ET, age 0 bars. WATCHING bullish at the 10:10 bar (10:05 close $763.59 inside). Price sat inside the range 09:30-10:05
**Context:** prior day $758.68-$763.47 | 63-day high $781.62 | VWAP $763.24 | VIX 18.17 (−6.6%) | Macro NORMAL (macro check unavailable: calendar search returned nothing usable for 10/26; Oct 27-28 FOMC not yet)
**Filters:** all passed — OR width 0.35% <= 0.40%; SPY 765.68 > VWAP 763.24; VIX 18.17 (<30, −6.6%); prior-day high 763.47 already cleared, 63-day high 781.62 more than $1.00 above price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $766.97 via measured move (OR height 2.67 above 764.30)
- Strike: $766 call 0DTE (SPY261026C00766000), Δ0.47 — only integer strike inside the bound [765.69, 766.97] (765C, Δ0.58, sits below price)
- Contracts: 1 (floor($200 / ($1.14 x 100)))
- Fill [PAPER]: $1.14 x 1 = $114.00 at ~10:21 ET (ask; ~35 s after the signal was seen)

**Lot A (1 ct, TP +25% = $1.43):** DISASTER SL @ $0.66 10:30:55 (stop $0.68 minus $0.02) — −$48.00 gross (−42%) | peak bid after fill $1.02 (−11%, never above entry), low bid $0.68

**Day:** held 10:21-10:31 (10 min) | P&L −$48.02 (gross −$48.00, fees $0.01 x 2), −42.1% of cost
**Stop history:** disaster $0.68 never ratcheted; breakeven trigger $1.32 never reached (peak bid after fill $1.02).
**Execution deviations:** none (no orders placed; PAPER). The stop was seen at a ~35 s poll (bid $0.68 after $0.76 at 10:30:18); fill simulated at stop minus $0.02 per config. The bid touched $0.70 at 10:27 and bounced to $0.83 before the final slide.

**Post-trade analysis:** Entry was timely and filters were clean, but the entry ask $1.14 came right after a $1.13 quote at the local high; the option lost 10% within 40 s (bid $1.01 at 10:21:39) as SPY slipped 0.35 from the signal print. SPY then drifted from 765.68 to 764.46, ~$0.16 above the OR high, without ever closing a bar back inside (structural never fired: 10:20 and 10:25 closed 764.87 and 764.76), so the premium-based -40% stop was the binding exit on a ~$1.2 SPY pullback. Fourth thin-signal loss in five trades: both signal bars closed only $0.4-1.4 above the OR high and the entry paid the post-spike ask. Worth watching in reviews: entries made within a minute of a bar-close price spike, and the -40% stop on ~$1 premiums sitting inside normal 0DTE noise (stopped at SPY 764.46 while the structural level was 764.30).

## 2026-10-27 — BULLISH BREAKOUT  [PAPER]

**Opening range:** $765.78 - $768.36 (width 0.34%)
**Signal:** 10:55 close $768.72 and 11:00 close $769.10 (both above OR high); seen at 11:05:20 ET, age 0 bars. The 10:50 close of $768.35 missed the OR high by one cent, so the pair started a bar later. Earlier, a 10:15 close of $765.44 (below OR low) was a bearish 1 of 2 that reverted at 10:20. Price sat inside the range 09:30-10:45
**Context:** prior day $761.63-$767.73 | 63-day high $781.62 | VWAP $766.99 | VIX 17.56 (−4.5%) | Macro NORMAL — FOMC day 1 of 2 (decision Wed 10/28, not today); Conference Board consumer confidence 10:00 ET and durable goods 8:30 ET are not on the config.md data-day list; mega-cap earnings could not be verified (calendar search returned nothing for FOMC or earnings), so macro check partly unavailable
**Filters:** all passed — OR width 0.34% <= 0.40%; SPY 769.09 > VWAP 766.99; VIX 17.56 (<30, −4.5%); prior-day high 767.73 already cleared, 63-day high 781.62 more than $1.00 above price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $770.94 via measured move (OR height 2.58 above 768.36)
- Strike: $770 call 0DTE (SPY261027C00770000), Δ0.35 — FLAG: below the 0.40 delta target; the only integer strike inside the bound [769.09, 770.94] (769C, Δ0.50, sits below price)
- Contracts: 3 (floor($200 / ($0.60 x 100))), Lot A 2 ct, Lot B 1 ct
- Fill [PAPER]: $0.60 x 3 = $180.00 at ~11:05:45 ET (ask; ~25 s after the signal was seen)

**Lot A (2 ct, TP +25% = $0.75):** TP @ $0.77 11:16:53 — +$34.00 gross (+28%) | low bid while held $0.45, peak bid $0.77
**Lot B (1 ct, TP +40% = $0.84):** TP @ $0.84 11:20:24 — +$24.00 gross (+40%) | peak bid $0.84

**Day:** held 11:05-11:20 (15 min) | P&L +$57.94 (gross +$58.00, fees $0.01 x 6), +32.2% of cost
**Stop history:** disaster $0.36 never touched (low bid $0.45 at 11:10:25, −25%); breakeven trigger $0.69 hit at 11:15:03 (bid $0.69), both stops raised to $0.62, never touched (low bid afterward $0.69).
**Execution deviations:** none (no orders placed; PAPER). Both TP fills assume a LIMIT at the polled bid filled in full, per config.md § Mode. Polls ran about every 30-35 s; structural check passed at the 11:05 and 11:10 bar closes ($768.73, ~$768.64+ vs OR high $768.36).

**Post-trade analysis:** Entry was timely (signal age 0, ~25 s after seen) and the option drew down to $0.45 (−25%) over the first 5 minutes as SPY slipped from 769.10 back to ~768.65, only $0.30 above the OR high, before reversing and running to 769.72 for both targets inside 15 minutes. Structural never fired but came within about $0.30. Contrast with 10/19, 10/21 and 10/26: here the confirmation came 90 minutes into the session after repeated tests of the boundary, and the +$0.60 premium meant a TP was only $0.15/$0.24 away. Worth watching in reviews: the later, grinding confirmation (one cent miss at 10:50) worked; Δ0.35 flag.

## 2026-10-28 — NO TRADE  [PAPER]
OR $769.25-$771.70 (w 0.32%) | Macro NO TRADE: FOMC decision day (Oct 27-28, 2:00 PM ET; confirmed by search) | VIX 17.47 (-0.2%) | Status: NO TRADE — FOMC decision day per config.md § Macro Event Rules. Opening range recorded only; no signal scan, no position.

## 2026-10-29 — BULLISH BREAKOUT  [PAPER]

**Opening range:** $767.27 - $769.41 (width 0.28%)
**Signal:** 10:30 close $770.04 and 10:35 close $770.01 (both above OR high); seen at 10:40:28 ET, age 0 bars. WATCHING bullish at the 10:30 bar (10:00 close $769.76 was a bullish 1 of 2 that reverted at 10:05; 10:15 closed $769.40, one cent short of the OR high). Price sat inside the range 10:05-10:25
**Context:** prior day $769.24-$773.36 | 63-day high $781.62 | VWAP $768.51 | VIX 17.07 (+0.95%) | Macro DATA DAY: Sept PCE/personal income, Q3 GDP advance and jobless claims at 8:30 ET (mega-cap earnings after yesterday's close could not be verified: calendar search returned nothing, so macro check partly unavailable)
**Filters:** all passed — OR width 0.28% <= 0.30% (data day); SPY 769.87 > VWAP 768.51; VIX 17.07 (<30, +0.95%); prior-day high 773.36 and 63-day high 781.62 more than $1.00 above price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $771.55 via measured move (OR height 2.14 above 769.41)
- Strike: $771 call 0DTE (SPY261029C00771000), Δ0.40 — only strikes inside the bound [769.87, 771.55] were 770C (Δ0.50, ask $1.63) and 771C (Δ0.4016, ask $1.16); 771C is closest to 0.40 at or above it
- Contracts: 1 (floor($200 / ($1.16 x 100)))
- Fill [PAPER]: $1.16 x 1 = $116.00 at ~10:40:40 ET (ask; ~12 s after the signal was seen)

**Lot A (1 ct, TP +25% = $1.45):** DISASTER SL @ $0.68 10:47:58 (stop $0.70 minus $0.02) — −$48.00 gross (−41%) | peak bid after fill $1.08 (−7%, never above entry), low bid $0.68

**Day:** held 10:40-10:48 (7 min) | P&L −$48.02 (gross −$48.00, fees $0.01 x 2), −41.4% of cost
**Stop history:** disaster $0.70 never ratcheted; breakeven trigger $1.33 never reached (peak bid after fill $1.08).
**Execution deviations:** none (no orders placed; PAPER). The stop was seen at a ~20-30 s poll (bid $0.73 at 10:47:48, then $0.68 at 10:47:58); fill simulated at stop minus $0.02 per config. The ask paid ($1.16) was a quote that dropped to $1.05/$1.06 within 12 s as SPY slipped from 769.87 to 769.64.

**Post-trade analysis:** Entry was timely and filters were clean, but the premium started leaking immediately: SPY fell from $769.87 at entry to $768.30 within 7 minutes, never closing a bar back inside the range (structural not tested: the 10:40 bar closed $769.66, the 10:45 bar was still open), so the -40% premium stop was the binding exit. Fifth thin-signal loss in six trades: the confirmation bars closed only $0.60-0.63 above the OR high, and the entry sat at the local high after a 1-of-2 reversion earlier (10:00). The $1.16 premium on a Δ0.40 strike with theta ~$1.2/hr meant a $1.5 SPY pullback was enough for -40%. Worth watching in reviews: confirmation margin over the OR high (here $0.60) as a filter, and the -40% stop vs normal 0DTE noise on ~$1 premiums, same as 10/26. Data day with a gap-down open (-0.45%): a bullish breakout into prior-day lows-to-highs overhead was fighting the gap.

## 2026-10-30 — SKIPPED — VIX (up 14.5% on the day)  [PAPER]

**Opening range:** $759.44 - $761.46 (width 0.27%)
**Signal:** PUT — 10:10 close $758.60 and 10:15 close $758.25 both below OR low; seen at 10:20:30 ET, age 0 bars (fresh, SPY $758.40, inside the 0.10% chase limit). WATCHING bearish at the 10:10 bar (10:05 close $759.59 was inside by $0.15). SPY gapped down -1.0% at the open (prior close $769.36) and sat inside the range 09:30-10:05
**Context:** prior day $766.62-$770.76 | 63-day high $781.62 | VWAP $760.23 | VIX 19.69 (+14.48% vs prev close 17.20; 18.92, +10.00% at 09:51; 18.58, +8.2% at 10:00) | Macro NORMAL-ish (month-end; calendar search showed PCE/ECI were Thu 10/29; AAPL/AMZN earnings date unconfirmed, so macro check partly unavailable)
**Filters:** macro pass (unverified); OR width 0.27% <= 0.40% pass; SPY 758.40 < VWAP 760.23 pass; **VIX FAIL** — +14.48% > +10% limit. Levels and budget not evaluated (first failure ends the day).
**Outcome:** no trade, no position. Hypothetical outcome not computed. Second VIX-spike gap-down skip (10/22, 10/30) with a PUT signal; track in the monthly filter scorecard.

## 2026-11-02 — BULLISH BREAKOUT  [PAPER]

**Opening range:** $758.72 - $760.57 (width 0.24%)
**Signal:** 10:05 close $760.78 and 10:10 close $761.53 (both above OR high); seen at 10:15:28 ET, age 0 bars. WATCHING bullish at the 10:05 bar (10:00 close $760.35 inside). Price sat inside the range 09:30-10:00. (OR high printed $760.56 on the first read and settled at $760.57 once the 09:50 bar was final; no effect on any decision.)
**Context:** prior day $750.36-$761.46 | 63-day high $781.62 | VWAP $759.96 | VIX 18.27 (−6.0%) | Macro NORMAL — ISM Manufacturing 10:00 ET and AMD/PFE/SMCI earnings are not on the config.md data-day list; midterm elections are Tue 11/3. Gap up +0.5% at the open (prior close $755.72)
**Filters:** all passed — OR width 0.24% <= 0.40%; SPY 761.67 > VWAP 759.96; VIX 18.27 (<30, −6.0%); prior-day high 761.46 already cleared, 63-day high 781.62 more than $1.00 above price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $762.42 via measured move (OR height 1.85 above 760.57); IV method (~$770.9) was farther away
- Strike: $762 call 0DTE (SPY261102C00762000), Δ0.47 — only integer strike inside the bound [761.67, 762.42]
- Contracts: 1 (floor($200 / ($1.48 x 100)))
- Fill [PAPER]: $1.48 x 1 = $148.00 at ~10:15:40 ET (ask; ~12 s after the signal was seen)

**Lot A (1 ct, TP +25% = $1.85):** TP @ $1.87 10:20:58 — +$39.00 gross (+26%) | low bid while held $1.39 (−6%), peak bid $1.87

**Day:** held 10:15-10:21 (5 min) | P&L +$38.98 (gross +$39.00, fees $0.01 x 2), +26.3% of cost
**Stop history:** disaster $0.89 never touched (low bid $1.39 at 10:16:15); breakeven trigger $1.70 hit at 10:17:25 (bid $1.72), stop raised to $1.50, never touched (low bid afterward $1.70).
**Execution deviations:** none (no orders placed; PAPER). TP fill assumes a LIMIT at the polled bid filled in full, per config.md § Mode. Polls ran about every 25-35 s; structural check passed (10:15 bar closed $761.68 vs OR high $760.57).

**Post-trade analysis:** Entry was timely (signal age 0, ~12 s after seen) and the option dipped to $1.39 (−6%) in the first minute as SPY slipped from 761.72 to 761.49, then ran with SPY to 762.47 and hit the +25% target in under 6 minutes. Confirmation margin over the OR high was $0.21 and $0.96, the thin first bar again, but the second bar's strength and a gap-up open into a cleared prior-day high gave follow-through, in contrast to the 10/26 and 10/29 stop-outs. Trade used the full measured-move target almost exactly (SPY high so far ~762.47 vs 762.42). Worth watching in reviews: the 1-contract $1.48 premium needed only ~$0.60 SPY for +25%; the ~$0.39 gain was a quick scalp, not a runner.

## 2026-11-03 — BULLISH BREAKOUT  [PAPER]

**Opening range:** $752.39 - $754.68 (width 0.30%)
**Signal:** 10:40 close $754.85 and 10:45 close $755.39 (both above OR high); seen at 10:50:25 ET, age 0 bars. WATCHING bullish at the 10:40 bar. Price sat inside the range 09:30-10:35 (10:15 close $754.48, 10:25 close $754.57 missed the OR high by $0.20 and $0.11)
**Context:** prior day $751.60-$763.86 | 63-day high $781.62 | VWAP $753.88 | VIX 20.42 (−3.45%) | Macro NORMAL — JOLTS 10:00 ET and midterm elections are not on the config.md data-day list; no Fed speaker or mega-cap earnings found for the day (calendar search was partial, so macro check partly unavailable). Gap up +0.1% at the open (prior close $752.10)
**Filters:** all passed — OR width 0.30% <= 0.40%; SPY 755.48 > VWAP 753.88; VIX 20.42 (<30, −3.45%); prior-day high 763.86 and 63-day high 781.62 more than $1.00 above price

**Entry:**
- Underlying: SPY (fits $200 budget)
- Projected target: $756.97 via measured move (OR height 2.29 above 754.68); IV method (~$765) was farther away
- Strike: $756 call 0DTE (SPY261103C00756000), Δ0.45 — only integer strike inside the bound [755.48, 756.97]
- Contracts: 1 (floor($200 / ($1.45 x 100)))
- Fill [PAPER]: $1.45 x 1 = $145.00 at ~10:50:45 ET (ask; ~20 s after the signal was seen)

**Lot A (1 ct, TP +25% = $1.81):** BREAKEVEN stop @ $1.45 11:18:31 (stop $1.47 minus $0.02) — $0.00 gross (0%) | low bid while held $1.33 (−8%), peak bid $1.80 (+24%, one cent short of the TP trigger $1.8125)

**Day:** held 10:51-11:18 (27 min) | P&L −$0.02 (gross $0.00, fees $0.01 x 2), −0.0% of cost
**Stop history:** disaster $0.87 never touched (low bid $1.33 at 10:51:09); breakeven trigger $1.67 hit at 11:00:02 (bid $1.73), stop raised to $1.47; bid reached $1.80 at 11:00:34, then ranged $1.54-$1.74 and fell to $1.39 at 11:18:31, stop filled.
**Execution deviations:** none (no orders placed; PAPER). The stop was seen at a ~35 s poll (bid $1.54 at 11:17:55, then $1.39 at 11:18:31); fill simulated at stop minus $0.02 per config. Structural check passed at every bar close (10:50 $755.31 through 11:10 $756.16, vs OR high $754.68; SPY $755.67 at the exit, still above).

**Post-trade analysis:** Entry was timely (signal age 0, ~20 s after seen). The option dipped to $1.33 (−8%) in the first 20 s, then ran with SPY to ~756.4 and reached a $1.80 bid, one cent below the +25% trigger, twice more touching $1.74. SPY then faded about $0.75 and the option lost 20% of premium in two minutes (1.54 to 1.39) as SPY slipped from 756.4 to 755.67, so the breakeven stop at $1.47 was the binding exit; SPY never came near the OR high ($1.0 above at exit), so the structural exit did not fire. Confirmation margin over the OR high was $0.17 and $0.71, a thin first bar again, but the trade had no follow-through beyond the measured-move target ($756.97 vs SPY high ~756.50): the target sat only $0.5 above the entry, a capped move. Worth watching in reviews: the +25% TP missed by one cent while the breakeven ratchet turned a near-winner into a scratch (same shape as the 10/23 breakeven stop).

## 2026-11-04 — NO BREAKOUT all day  [PAPER]

**Opening range:** $762.65 - $768.11 (width 0.71%, wide: > 0.40% limit). SPY gapped up +1.15% at the open (prior close $754.35) and ran to the range top by 09:45.
**Signal:** none. WATCHING bullish (1 of 2) at the 10:10 bar (close $768.24, $0.13 above OR high); 10:15 closed back inside at $767.84. Price then drifted to $764.16-$766 for the rest of the window and never closed below the OR low ($762.65). Final scan at 11:35 ET covered every completed bar through 11:30.
**Context:** prior day $752.39-$757.07 | 63-day high $781.62 | VWAP ~$766.29 (through 11:30) | VIX 18.14 (-12.9%) | Macro NORMAL (ADP 8:15 and ISM Services 10:00 are not on the config.md data-day list; Fed speakers Logan/Williams/Musalem/Bowman, not the Chair; no mega-cap earnings found; calendar search partial)
**Filters:** not evaluated (no signal). A signal would have been SKIPPED on OR width 0.71% > 0.40%.
**Outcome:** no trade, no position, no P&L. Equity unchanged at $957.64.
**Data note:** the API later revised the 10:00 bar high to $775.64 (apparent stray print); no effect on the OR (09:30-09:55) or any close.
