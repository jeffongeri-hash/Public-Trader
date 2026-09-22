# Config Reference — Opening Range Breakout (ORB), SPY/SPX 0DTE

## Connector Account
Public.com account **5OI27877 (CASH, options Level 2, BUY_AND_SELL)**.
Hardcode this account_id in every Public tool call in this skill.
Account 5LG81210 (MARGIN) runs a SEPARATE playbook — never touch it from
this skill.

## Strategy Summary
Pure Opening Range Breakout on SPY, trading same-day-expiration (0DTE)
options on whichever of SPY/SPX fits the budget. This is NOT the old
16-ticker/5-strategy swing matrix — that content has been fully removed
from this skill (it belonged to a different conversation/skill).

## Underlying for the breakout signal
**SPY only.** The opening range and breakout trigger are always computed
from the SPY chart, regardless of which underlying's options end up being
bought.

## Opening Range Window
- **9:30–10:00 AM ET (8:30–9:00 AM CT)** — first 30 minutes of the regular session
- OR_high = highest high in that window
- OR_low = lowest low in that window

## Previous Day High/Low
Fetched and displayed as **context only** — NOT part of the trigger condition.
Shows Jeff where the opening range sits relative to yesterday's range.

## Breakout Trigger — requires 2-bar confirmation on the 5-minute chart
```
Using completed 5-minute SPY candles (not the live/forming price):

if last_closed_5min_bar.close > OR_high AND prior_5min_bar.close > OR_high
    → BULLISH BREAKOUT CONFIRMED → buy a CALL

if last_closed_5min_bar.close < OR_low AND prior_5min_bar.close < OR_low
    → BEARISH BREAKDOWN CONFIRMED → buy a PUT

if ONLY the last_closed_5min_bar (the newest one) is beyond a boundary,
and prior_5min_bar was not
    → WATCHING, no trade yet — log it, re-check next run

if prior_5min_bar had breached but last_closed_5min_bar is back inside
the range → that breach is stale/reverted → NO BREAKOUT, not WATCHING

else → no trade, no action
```
A single tick or single 5-min close beyond the range is NOT enough — both
of the two most recently completed 5-minute bars must close beyond the
same boundary. This is a confirmation filter to avoid single-bar fakeouts
at the range edge. `WATCHING` only applies when the newest bar is the one
that just breached — a reverted older breach doesn't linger as `WATCHING`.

Prior-day H/L is still not part of this condition — opening range alone
(with 2-bar confirmation) triggers.

## One Trade Per Day
Once a breakout fires and an order is placed, no further entries the same
day even if price re-crosses back the other way. Check `get_orders` /
`get_portfolio` for an existing same-day options fill from this strategy
before allowing a new entry.

## Underlying Selection for the Actual Option (SPY vs SPX)
1. Get 0DTE ATM premium for both SPY and SPX in the breakout direction.
2. contract_cost = premium × 100 for either (both are 100-multiplier).
3. If both fit under $400 → default to **SPY** (tighter spreads, more liquid).
4. If only one fits under $400 → use that one.
5. If neither fits under $400 → skip the trade, log why (this will be rare —
   SPX 0DTE ATM premiums are typically far above $400, so SPY will almost
   always be the one used in practice).

## Price Target Projection (informs strike selection only — added 9/18/26)
Before picking a strike, project a realistic SPY target for the rest of the
day, using the more conservative (closer to current price) of two methods:

**Method 1 — Measured move:** project the opening range's own height from
the boundary that broke:
```
range_height = OR_high - OR_low
Bearish breakdown: measured_target = OR_low - range_height
Bullish breakout:  measured_target = OR_high + range_height
```

**Method 2 — IV-implied expected move:** using the IV already returned by
`get_option_greeks` for a near-ATM strike, compute a statistical expected
move for the remaining trading day:
```
hours_remaining = trading hours left until 4:00 PM ET
T_years = hours_remaining / (252 * 6.5)   # 252 trading days/yr, 6.5 hrs/day
expected_move = current_price × IV × sqrt(T_years)
Bearish breakdown: iv_target = current_price - expected_move
Bullish breakout:  iv_target = current_price + expected_move
```

**Final projected_target** = whichever of `measured_target` / `iv_target`
is closer to the current price (the more conservative, i.e. smaller,
projected move). This target does NOT change the TP/SL percentages below
— it exists solely to keep strike selection realistic (see next section).

## Strike / Delta Target
Same convention as before — ATM or slightly ITM, delta magnitude ≥ 0.40 —
but candidate strikes are now **bounded by `projected_target`**: only
consider strikes between the current price and `projected_target`
(inclusive), never deeper ITM than the projection justifies. For example,
if SPY is at $762 and `projected_target` is $759, don't consider a $750
strike even if it has an appealing delta — it assumes a move the day's own
data doesn't support. Use `get_option_greeks` on the bounded candidates to
confirm delta before committing. If nothing in the bounded range reaches
delta ≥ 0.40, use the closest available within that range and flag it —
don't reach past `projected_target` to find a higher delta.

Log the `projected_target` and which method produced it in the trade log
entry (see `references/trade-log.md`) — the weekly review should track
whether these projections are actually tracking realized moves.

## Position Sizing
```
contracts = floor($400 / (premium × 100)), minimum 1
```

## Exit Rule — single stop-loss only, manual exit (revised 9/22/26)
The tiered two-lot TP/SL bracket was retired after the first live trade
showed two problems: (1) this CASH account won't let multiple unlinked
resting SELL orders on the same contracts total more than the quantity
held, so TP and SL legs couldn't both rest at once; and (2) a SELL LIMIT
priced below the live market fills immediately instead of resting as
protection — a real stop needs `order_type="STOP"`. Jeff also asked to
manage exits himself rather than have the bot force-close at end of day.

```
N = contracts bought (single lot, no tiering)

STOP: SELL, order_type="STOP", stop_price = round(entry_premium × 0.70, 2)
      (−30%), quantity = N, time_in_force="DAY"
```

No take-profit order is placed. No forced end-of-day close — Jeff decides
when to exit and does it himself; the STOP order is the only automated
protection resting on the position.

## End-of-Day Close
No bot-managed forced close (revised 9/22/26). Jeff exits manually
whenever he chooses — the STOP order is the only automated exit. Reminder
in every run's summary if a position is still open: SPY 0DTE is
physical-settlement, so an unclosed position expiring ITM/OTM matters —
that risk is now Jeff's to manage, not the bot's.

## Schedule (revised 9/22/26 — start time moved, polling itself kept)
- **9:10 AM CT (10:10 AM ET):** capture opening range (30-min high/low),
  fetch previous day H/L for context, run the first breakout check
  immediately using whatever 5-min bars have already closed by then (at
  minimum the 10:00-10:05 and 10:05-10:10 ET bars, the earliest pair that
  can satisfy 2-bar confirmation).
- **Every 5 minutes from 9:10 AM CT through 2:45 PM CT:** breakout check,
  aligned to each new 5-min bar close (skipped once a trade has already
  been taken today).
- **2:45 PM CT:** final run of the day — no forced close (Jeff exits
  manually), but this is when the day's chat-log entry gets printed (see
  `references/trade-log.md`).

**Platform note:** a native cloud-routine Schedule trigger has a 1-hour
minimum interval and cannot fire every 5 minutes on its own — see
`references/schedule-setup.md` for the API-trigger + external 5-minute
caller this actually needs.

## Budget Warning
0DTE SPY ATM premiums typically run $150–$400 depending on realized
volatility that morning — this is expected and the reason the budget is
$400, not $200 like the equity-era config.
