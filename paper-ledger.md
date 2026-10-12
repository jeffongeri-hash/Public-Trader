# ORB Paper Ledger  [PAPER]

Paper trading started **2026-10-12** on an assumed **$1,000** account
(`PAPER_START`), after live trading was cancelled (live account 5OI27877 ended
at $149.43). Fill rules: config.md § Mode. One entry per trading day, formats
in trade-log.md. Reviews (weekly/monthly) read this file.

Running equity = $1,000.00 + sum of `Day P&L` below.

| Date | Status | Day P&L | Equity |
|---|---|---|---|
| 2026-10-12 | TRADED — PUT, BREAKEVEN stop both lots | −$0.04 | $999.96 |

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
