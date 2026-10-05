# ORB Trade Log

Entries for 9/29–10/2 were reconstructed from Public transaction history
(account 5OI27877), not written live. Opening range, confirmation bars,
delta and projected target were not recorded and are marked n/a. Times ET.
Win/loss labels below are vs. entry premium.

## 2026-09-28 — NO TRADE
Opening Range: n/a | Final status: no trades in account history

## 2026-09-29 — BEARISH BREAKDOWN (reconstructed)

**Opening Range:** n/a
**Confirmation bars:** n/a
**Entry decision:**
- Underlying: SPY 0DTE $764 put (SPY260929P00764000)
- Contracts: 2
- Entry fill: $1.36 x 2 = $272.00 (11:38 ET)

**Exit — Lot A (1 ct):** exit $1.32 at 11:47 (−3%); P&L −$4.00 — neither TP (+25%, 1.70) nor SL (−30%, 0.95): early/manual close
**Exit — Lot B (1 ct):** exit $1.57 at 11:52 (+15%); P&L +$21.00 — below +40% target (1.90): early/manual close

**Blended outcome:** held 9–14 min. Net P&L +$17.22 after fees (+6% on cost).
**Deviation:** both lots closed by hand/other logic, not by the tiered rule — does not test the strategy.

## 2026-09-30 — BULLISH BREAKOUT (reconstructed)

**Opening Range:** n/a
**Confirmation bars:** n/a
**Entry decision:**
- Underlying: SPY 0DTE $769 call (SPY260930C00769000)
- Contracts: 3
- Entry fill: $1.00 x 3 = $300.00 (10:15 ET)

**Exit — Lot A (2 ct):** SL-A hit, exit $0.70 at 10:28 (−30%); P&L −$60.00
**Exit — Lot B (1 ct):** SL-B hit, exit $0.69 at 10:28 (−31%); P&L −$31.00

**Blended outcome:** held 13 min. Net P&L −$90.67 after fees (−30%).
**Post-trade analysis:** full stop-out within 13 minutes of entry — the only trade this week that followed the rules. Prompted the delayed-stop change (10/5, now 5 minutes after entry).

## 2026-10-01 — BEARISH BREAKDOWN (reconstructed)

**Opening Range:** n/a
**Confirmation bars:** n/a
**Entry decision:**
- Underlying: SPY 0DTE $760 put (SPY261001P00760000)
- Contracts: 2
- Entry fill: $1.25 x 2 = $250.00 (10:17 ET)

**Exit — Lot A (1 ct):** exit $1.50 at 10:20 (+20%); P&L +$25.00 — short of +25% (1.56): early/manual close
**Exit — Lot B (1 ct):** exit $1.40 at 10:27 (+12%); P&L +$15.00 — short of +40% (1.75): early/manual close

**Blended outcome:** held 3–10 min. Net P&L +$40.22 after fees (+16%).
**Deviation:** both lots closed before their targets, not by the tiered rule.

## 2026-10-02 — NO TRADE
Opening Range: $769.07-$772.09 | Final status: NO BREAKOUT (last bars closed $771.47, $771.93 inside range at 9:18 AM CT check)

## Entry-quality notes (started 10/5)
From this week's reconstructed trades, the three entries so far were at 11:38, 10:15 and 10:17 ET; only the 10:15 call lost (−30% in 13 min). Candidate entry filters to start collecting data on (not live rules yet): volume on the confirmation bars vs. the opening-range average, SPY side of VWAP, close distance beyond the boundary, and range width. Live entries from 10/5 onward log these fields, so the first real check is after ~5 logged trades.
