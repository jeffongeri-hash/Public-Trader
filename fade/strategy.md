# SPY 1-Week Fade — PAPER ONLY ($5,000 assumed account)

Second, independent strategy, added 10/12/26. **Paper only — never place,
cancel or replace a real order** (no `place_order`, `cancel_order`,
`cancel_and_replace_order`). Read-only Public calls only (quotes, bars,
option chain, greeks). Separate ledger (`fade/ledger.md`), separate equity
($5,000). Never blend with the ORB paper account (`paper-ledger.md`). The ORB
session (SKILL.md) ignores this folder.

## Idea
Fade a normal-sized daily move in SPY: **buy a put after an up day, a call
after a down day**, using a ~1-week-out, ~0.70-delta option (mostly intrinsic,
low theta per dollar), expecting a partial reversion within a few days. It is
the opposite direction of the ORB breakout trade.

## Parameters (single source of truth — all are starting guesses to calibrate)
| Name | Value | Notes |
|---|---|---|
| `PAPER_START` | $5,000 | assumed |
| `RUN_TIME` | 15:30 ET, weekdays | day's move is nearly final; session starts ~15:25 ET, waits to 15:30:20 |
| `MIN_MOVE` | $1.50 | `abs(SPY last − previous close)`; below this the day has no direction → no entry |
| `MOVE_BAND` | $7.00 | `abs(move) ≤ $7.00`; bigger moves are news/trend days, not fades. $7 ≈ 1σ daily move at VIX 15 (SPY × VIX/100 / √252); log both numbers each day |
| `TARGET_DTE` | 7 calendar days (accept 5–9) | nearest listed expiration to 7 days |
| `TARGET_DELTA` | 0.70 | pick strike with abs(delta) closest to 0.70; reject if outside 0.65–0.75 |
| `MAX_OPTION_IV` | 13% (annualized) | greeks `impliedVolatility` of the chosen option. Week-out SPY IV was ~9–10% on 10/9. Reject if IV is outside a 5–60% sanity band (bad data) and flag it |
| `VIX_MAX` | 22, and VIX day change ≤ ±8% | stand down in stress |
| `MAX_SPREAD` | ask−bid ≤ 3% of mid | paper realism |
| `MAX_TRADE_COST` | 25% of current paper equity | ≈ $1,250 → 1 contract of a ~$8 option |
| `MAX_OPEN` | 2 positions | and max 1 new entry per day |
| Macro skip | FOMC decision day, CPI/PPI/NFP/PCE/Retail Sales day, half-days, holidays | same calendar as config.md § Macro Event Rules; if the lookup fails, trade and flag `macro check unavailable` |

## Daily session (one short run, ~15:25–15:45 ET)
1. Holiday/weekend → stop. Read `fade/ledger.md` for open positions and equity.
2. At 15:30:20 ET: SPY quote (`last`, `previousClose`), VIX quote
   (`last`, `previousClose`).
3. **Exits first** — for each open position, option quote (use **bid**), then
   apply in order, first match wins:
   - `TP`: bid ≥ entry × 1.30
   - `SL`: bid ≤ entry × 0.75
   - `REVERT`: SPY has retraced ≥ 50% of the entry-day move (from the entry-day close toward the prior close) → thesis played out
   - `TIME`: 3rd trading day after entry has arrived, or ≤ 2 days to expiry
   Exit price = bid − $0.02. (Daily granularity only: stops are checked once a day at 15:30 ET, so a bad intraday move can overshoot — record the overshoot.)
4. **Entry** (only if all pass; first failure = `NO TRADE — <reason>` with the values):
   macro ok → `MIN_MOVE ≤ abs(move) ≤ MOVE_BAND` → VIX filter → open < `MAX_OPEN` →
   no entry already today. Direction: move > 0 → PUT; move < 0 → CALL.
   Then `get_option_expirations("SPY")` → choose expiry → candidate strikes
   around the money (`SPY` $1 steps; OSI `SPY<YYMMDD><C|P><strike×1000, 8 digits>`)
   → `get_option_greeks` → closest to 0.70 delta → `get_quotes` →
   `MAX_SPREAD`, `MAX_OPTION_IV`, `cost = ask × 100 ≤ MAX_TRADE_COST` →
   paper fill at the **ask**, 1 contract (`floor(MAX_TRADE_COST / cost)`, min 1 only if 1 fits).
5. Fees $0.01/contract/side. Record everything in `fade/ledger.md`, commit and
   push to `main` (if the push fails, create a dated Google Doc and say so).
6. Summary tagged `[PAPER · FADE]`: date, SPY move vs band and 1σ, VIX, action
   (ENTRY / EXIT / HOLD / NO TRADE + reason), open positions with mark-to-bid P&L, equity.

## Ledger entry format (fade/ledger.md)
```
## YYYY-MM-DD — ENTRY | EXIT | HOLD | NO TRADE
SPY $x (move ±$m vs prev close $p; band $7.00; 1σ-day $e) | VIX v (±y%)
[ENTRY] BUY 1 <osi> Δd IV i% DTE n @ ask $a (spread s%) cost $c
[EXIT]  SELL 1 <osi> @ $b — TP|SL|REVERT|TIME — P&L $x (±y%), held n days
Open: <positions, mark P&L>   Day P&L $x   Equity $y
```

## Reviews
Weekly (Mondays) and monthly (first trading day) like trade-log.md §§ Weekly /
Monthly, computed from `fade/ledger.md` only: trades, win rate, avg win/loss,
expectancy, exit-type counts (TP/SL/REVERT/TIME), average hold, max drawdown,
% of NO TRADE days by reason, and **filter scorecard** (hypothetical result of
trades skipped by `MIN_MOVE`, `MOVE_BAND`, IV, VIX, spread). Recommend keep /
adjust one named parameter / not enough data (< ~8 trades). Any move to real
money is Jeff's decision only, after ≥ 20 paper trades with positive expectancy.

## Known limits
- Once-a-day exit checks (no intraday stops) and quotes at 15:30 ET only.
- No IV-rank/history available from the connector; `MAX_OPTION_IV` is an absolute cap and needs calibrating from logged IVs.
- Week-out SPY option spreads can be 1–4%; fills at ask/bid are conservative, not guaranteed.
- A fade loses on trend days; `MOVE_BAND` and the 25%/2-position cap are the only protections beyond the −25% stop.
