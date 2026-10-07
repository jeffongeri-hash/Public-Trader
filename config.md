# Config Reference — Opening Range Breakout (ORB), SPY/SPX 0DTE

> **v2 — revised 10/7/26.** Review of live results Sep 18 – Oct 5 (−$500 on
> $1,000) found the trigger itself was sound but execution was not: hourly
> runs entered late or missed signals entirely (10/6), the one-trade gate
> allowed re-entries after a stop-out (9/21, 9/25 — the 9/25 re-entry alone
> lost $329), and winners averaged +$31 vs losers −$164. v2 keeps the trigger
> and fixes execution, exits and filtering. Every parameter below is the
> single source of truth — the other files reference these names.

## Connector Account
Public.com account **5OI27877 (CASH, options Level 2, BUY_AND_SELL)**.
Hardcode this account_id in every Public tool call in this skill.
Account 5LG81210 (MARGIN) runs a SEPARATE playbook — never touch it from
this skill.

## Mode
`MODE = LIVE` — orders execute for real. Set to `PAPER` to run the full
session (signals, filters, simulated fills at the quoted ask/bid, simulated
exits) and log it with a `[PAPER]` tag, without calling `place_order`,
`cancel_order` or `cancel_and_replace_order`.

## Data Sources (all via the Public connector)
| Need | Call |
|---|---|
| SPY 5-min bars (today) | `get_price_history(symbol="SPY", period="DAY", aggregation="FIVE_MINUTES", trading_session_toggle="REGULAR_HOURS")` |
| SPY daily bars (prior day H/L, 63-day high — YEAR is too large for a tool result) | `get_price_history(symbol="SPY", period="QUARTER", aggregation="ONE_DAY")` |
| SPY / option live quote | `get_quotes(..., instrument_type="EQUITY" / "OPTION")` |
| VIX | `get_quotes(symbols=["VIX"], instrument_type="INDEX")` — `last` and `previousClose` |

Twelve Data is no longer used (it isn't connected to the routine).

**Completed-bar rule:** a 5-min bar stamped `HH:MM` is complete only once
`now ≥ HH:MM + 5 min`. The API returns the forming bar too (often with an
odd timestamp like `10:23`) — always drop it. Never evaluate a forming bar.

## Opening Range Window
- **9:30–10:00 ET** — the six bars stamped 09:30 … 09:55
- `OR_high` = max high, `OR_low` = min low, `OR_mid` = (high+low)/2
- `OR_width_pct` = (OR_high − OR_low) / OR_mid

## Breakout Trigger — unchanged from v1 (2-bar confirmation)
Two consecutive **completed** 5-min bars both close above `OR_high` → CALL;
both below `OR_low` → PUT. In the Sep 25 – Oct 6 replay this rule, entered
on time, reached +$0.75 SPY before −$0.60 on 6 of 8 days; 1-bar and
15-minute-range variants did worse (3–5 of 8). Don't loosen it.

**Scan every completed bar since 10:00, not just the latest two.** The
first bar pair that confirms is THE signal for the day (`signal_bar` = the
second bar of the pair, `signal_close` = its close). Nothing later can
create a second signal.

## Entry Window & No-Chase Rule
- `ENTRY_START = 10:05 ET` (earliest possible confirmation is the 10:05 bar)
- `ENTRY_CUTOFF = 11:30 ET` — no new entries after this. Breakouts that
  hold past midday faded on 6 of 8 sample days.
- `MAX_SIGNAL_AGE = 1 bar` — enter only if `signal_bar` is the newest or
  second-newest completed bar.
- `MAX_CHASE_PCT = 0.10%` — enter only if live SPY is within 0.10% of
  `signal_close` in the trade's direction and still beyond the OR boundary.
- If the signal is older, or price has run past the chase limit, or price
  is back inside the range → status `MISSED`, log it, **no trade today**.

## Entry Filters (all must pass; any fail → `SKIPPED — <filter>`, no trade today)
| Filter | Rule |
|---|---|
| Macro calendar | See § Macro Event Rules |
| OR width | `OR_width_pct ≤ 0.40%` (≤ 0.30% on a data day). Wide ranges = chop |
| VWAP | CALL needs live SPY > session VWAP; PUT needs SPY < VWAP. VWAP = Σ(typical price × volume) / Σ volume over today's completed bars, typical = (H+L+C)/3 |
| VIX regime | No trade if VIX `last > 30`, or VIX up > 10% vs `previousClose`. No CALL if VIX is up > 5% on the day |
| Overhead / underfoot level | CALL: skip if the prior-day high or the 63-day high sits above entry by less than `$1.00` (target is capped). PUT: same with the prior-day low below entry. A level price has already cleared doesn't count |
| Budget | At least one contract must cost ≤ `BUDGET` |

## Macro Event Rules
Checked once per session before 10:00 ET (see SKILL.md Phase 0).
| Event (today, ET) | Rule |
|---|---|
| FOMC decision day | **No trade** (2:00 PM decision dominates the tape). 2026 meetings — verify against federalreserve.gov: Jan 27–28, Mar 17–18, Apr 28–29, Jun 16–17, Jul 28–29, Sep 15–16, **Oct 27–28**, Dec 8–9 (decision on the 2nd day) |
| CPI, PPI, Nonfarm Payrolls, PCE, Retail Sales (8:30 AM release) | "Data day" — trade allowed only if `OR_width_pct ≤ 0.30%` |
| Fed Chair speaking 10:00–11:30 ET | No trade |
| NVDA / AAPL / MSFT / AMZN / GOOGL / META / TSLA reported after yesterday's close | Treat as a data day |
| Monthly opex (3rd Friday), quarter-end, half-day sessions | Treat as a data day; half-days: no trade |
If the calendar lookup fails, proceed as a normal day but flag
`macro check unavailable` in the summary and log.

## Underlying Selection (SPY vs SPX) — unchanged
Compare 0DTE premium for both; if both fit `BUDGET` use SPY; if only one
fits use that; if neither, skip and log both costs. SPX will almost never
fit at this budget.

## Price Target Projection & Strike — unchanged from v1
`projected_target` = the more conservative of measured move
(OR height projected from the broken boundary) and IV expected move
(`price × IV × sqrt(hours_left / (252 × 6.5))`). Candidate strikes are
bounded between current price and `projected_target`; pick |delta| ≥ 0.40
closest to 0.40, else the closest available inside the bound (flag it).

## Position Sizing
```
BUDGET    = min($200, optionsBuyingPower)        # was $400; account has ~$250
contracts = floor(BUDGET / (premium × 100)), minimum 1 only if 1 contract ≤ BUDGET
```
Never size up after a loss.

## Exit Rules (v2) — one resting stop per lot + a live watcher
```
tier1_qty = ceil(N/2)   (Lot A)        tier2_qty = N − tier1_qty   (Lot B, runner)

Resting at all times (the safety net if the watcher dies):
  DISASTER_SL: SELL STOP_LIMIT per lot, stop = entry × 0.60 (−40%),
               limit = stop − 0.05

Watched live (Phase 2 in SKILL.md, polled every POLL_SECONDS):
  TP_A        bid ≥ entry × 1.25 → close Lot A
  TP_B        bid ≥ entry × 1.40 → close Lot B
  BREAKEVEN   bid ≥ entry × 1.15 → raise every open lot's stop to entry + 0.02
              (limit = stop − 0.05). After Lot A's TP fills, Lot B's stop
              goes to breakeven immediately if not already there.
  STRUCTURAL  a completed 5-min SPY bar closes back inside the opening range
              (CALL: close < OR_high; PUT: close > OR_low) → close all lots
  TIME_STOP   60 min after entry, if bid < entry × 1.10 → close all lots
  EOD         15:45 ET → close anything still open
N = 1: Lot A only (TP at +25%).
```
`POLL_SECONDS = 30` while a position is open.

**How a watched exit executes — `cancel_and_replace_order`, never
cancel-then-sell.** The lot's resting stop is atomically replaced by the
exit order (`LIMIT` at the current bid for TP; `MARKET` for STRUCTURAL,
TIME_STOP and EOD). There is never a moment with two exit orders (no
oversell, stays within the account's no-OCO close-quantity cap) and never
a moment with none. Raising the stop (BREAKEVEN) uses the same call with
`order_type="STOP_LIMIT"` and the new prices. Details and fallbacks:
`public-submission.md` § Watched Exits.

**Never submit a stop-loss as `order_type="LIMIT"`** (9/23/26 incident — a
sell limit below the market fills instantly). Stops are always
`STOP_LIMIT`.

## One Trade Per Day — hard gate (fixes the 9/21 and 9/25 re-entries)
The gate is **today's fills**, not open positions: if
`get_history(start=<today 00:00 ET>)` shows any `BUY` of a SPY/SPX option
on this account today, the day's trade is used up — even if it's already
closed. Also blocked if any SPY/SPX option position or open order exists.

## Session Lock
Only one session may manage the account at a time. Google Doc
`ORB Session Lock` holds one line: `<ISO timestamp> <session note>`.
- Phase 0 reads it. If the timestamp is < 10 minutes old → another session
  is live → exit immediately with `LOCKED — another session is running`.
- Otherwise write a fresh timestamp, and refresh it at least every 5 minutes
  for as long as this session runs. Clear it (write `released`) at the end.
- If Drive is unavailable, proceed but flag it: the only guard left is the
  one-trade gate.
