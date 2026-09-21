# Schedule Setup — Opening Range Breakout (single daily check, 2-bar confirmation)

> **Rebuild note (9/20/26):** This originally polled every 5 minutes,
> 9:00 AM–2:45 PM CT, so the 2-bar confirmation could catch a breakout as
> soon as it happened. Jeff opted into a simpler, cheaper design instead:
> **one breakout check at 10:10 AM CT**, evaluating whatever the two most
> recently completed 5-minute SPY bars are at that exact moment, plus a
> forced close moved up to **12:00 PM CT** (also at Jeff's request — any
> lot still open by then gets sold regardless, rather than waiting until
> end of day). This trades signal timeliness for simplicity — see
> `config.md` § Schedule and `SKILL.md` § Schedule for the tradeoff this
> implies (a breakout that confirms before or after 10:10 AM CT is simply
> missed for the day).

## Run Times (all weekdays, US market holidays skipped)

| Time (CT) | Time (ET) | Purpose |
|-----------|-----------|---------|
| 10:10 AM | 11:10 AM | Capture opening range (30-min H/L) + prior-day context + the single 2-bar breakout check for the day |
| 12:00 PM | 1:00 PM | Forced close (no new entries — just closes anything still open) |

That's **2 runs per day**, plus **1 weekly review run**.

## Weekly Review Schedule

**First *trading* day of each week, 8:00 AM CT** — normally Monday,
reviewing the prior Mon-Fri from the "ORB Trade Log" Google Doc. Uses the
same holiday-skip logic as the intraday script, plus an "already ran this
week" marker so it doesn't fire twice if Monday is a holiday and Tuesday
ends up being the actual first trading day of the week.

### Step 1: Wrapper script

```bash
cat > ~/orb-weekly-review-run.sh << 'EOF'
#!/bin/bash
TODAY=$(date +%Y-%m-%d)
HOLIDAYS=("2026-07-03" "2026-09-07" "2026-11-26" "2026-12-25")
for h in "${HOLIDAYS[@]}"; do
  if [ "$TODAY" = "$h" ]; then
    echo "Market holiday $TODAY — skipping."
    exit 0
  fi
done

# Guard against running twice in one week: this fires on Monday AND
# Tuesday (see plist below) so it still catches the real first trading
# day when Monday is a holiday — but only the earliest of those that's
# actually a trading day should run.
THIS_WEEK=$(date +%G-W%V)   # ISO year-week, e.g. 2026-W38
MARKER=~/.orb-weekly-review-last-run
if [ -f "$MARKER" ] && [ "$(cat "$MARKER")" = "$THIS_WEEK" ]; then
  echo "Weekly review already ran for $THIS_WEEK — skipping."
  exit 0
fi

claude -p "run the ORB weekly review" >> ~/orb-weekly-review.log 2>&1
echo "$THIS_WEEK" > "$MARKER"
EOF
chmod +x ~/orb-weekly-review-run.sh
```
(No separate weekend check needed here — Monday/Tuesday are never
weekend days, so the holiday check alone is sufficient.)

### Step 2: Plist — fires Monday and Tuesday, wrapper decides which one actually runs

`~/Library/LaunchAgents/com.jeff.orb-weekly-review.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.jeff.orb-weekly-review</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/YOUR_USERNAME/orb-weekly-review-run.sh</string>
  </array>
  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Weekday</key><integer>1</integer><key>Hour</key><integer>8</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Weekday</key><integer>2</integer><key>Hour</key><integer>8</integer><key>Minute</key><integer>0</integer></dict>
  </array>
  <key>StandardOutPath</key>
  <string>/Users/YOUR_USERNAME/orb-weekly-review.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/YOUR_USERNAME/orb-weekly-review-err.log</string>
  <key>RunAtLoad</key>
  <false/>
</dict>
</plist>
```
```bash
launchctl load ~/Library/LaunchAgents/com.jeff.orb-weekly-review.plist
```

**Why Monday and Tuesday instead of just Monday:** if Monday is a market
holiday, the wrapper skips it and Tuesday (the real first trading day of
the week) is the one that actually runs and writes the marker. Without
this, a Monday-only trigger would just never fire that week on a holiday
Monday — launchd doesn't retry. (`Weekday: 1` = Monday, `2` = Tuesday in
launchd's convention.)

## Mac — launchd (recommended, daily breakout check + forced close)

```bash
cat > ~/orb-breakout-run.sh << 'EOF'
#!/bin/bash
DAY=$(date +%u)
if [ "$DAY" -ge 6 ]; then
  echo "Weekend — skipping."
  exit 0
fi

TODAY=$(date +%Y-%m-%d)
HOLIDAYS=("2026-07-03" "2026-09-07" "2026-11-26" "2026-12-25")
for h in "${HOLIDAYS[@]}"; do
  if [ "$TODAY" = "$h" ]; then
    echo "Market holiday $TODAY — skipping."
    exit 0
  fi
done

claude -p "run the ORB breakout check" >> ~/orb-breakout.log 2>&1
EOF
chmod +x ~/orb-breakout-run.sh

cp ~/orb-breakout-run.sh ~/orb-forced-close-run.sh
```
Both scripts run the exact same prompt — `"run the ORB breakout check"` —
at different times of day. The skill itself branches internally on the
explicitly-computed current America/Chicago time (see `SKILL.md` §
Schedule): at 10:10 AM CT it runs Steps 1-6 and 9; at 12:00 PM CT it skips
straight to Steps 3, 7, 8 and 9 (the forced-close check). There's no
separate trigger phrase to keep in sync with SKILL.md's ALWAYS-trigger
list — this mirrors how the original design handled the single trigger
phrase firing repeatedly with time-based branching inside the skill.

`~/Library/LaunchAgents/com.jeff.orb-breakout.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.jeff.orb-breakout</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/YOUR_USERNAME/orb-breakout-run.sh</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>10</integer></dict>
  <key>StandardOutPath</key>
  <string>/Users/YOUR_USERNAME/orb-breakout.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/YOUR_USERNAME/orb-breakout-err.log</string>
  <key>RunAtLoad</key>
  <false/>
</dict>
</plist>
```

`~/Library/LaunchAgents/com.jeff.orb-forced-close.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.jeff.orb-forced-close</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/YOUR_USERNAME/orb-forced-close-run.sh</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardOutPath</key>
  <string>/Users/YOUR_USERNAME/orb-breakout.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/YOUR_USERNAME/orb-breakout-err.log</string>
  <key>RunAtLoad</key>
  <false/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.jeff.orb-breakout.plist
launchctl load ~/Library/LaunchAgents/com.jeff.orb-forced-close.plist
launchctl list | grep orb-
```

**Tradeoff of the single 10:10 AM CT check (vs. the original 5-minute
polling):** the 2-bar confirmation is only evaluated once, using whatever
the two most recently completed 5-minute bars happen to be at that moment
(as of 10:10 AM CT / 11:10 AM ET, that's over an hour after the opening
range itself closes). A breakout that confirms earlier or later than that
one moment is simply missed for the day — there is no later re-check to
catch it, unlike the original design. A `WATCHING` result at the 10:10 AM
CT check also means no trade fires that day (there's no follow-up run
within the day to confirm the second bar) — it's logged for the weekly
review's fakeout-rate tracking, but functionally equivalent to `NO
BREAKOUT` for that day's trading outcome.

### Retiring an old public-trader schedule
If `com.jeff.public-trader-morning` / `-close` (twice-daily equity era),
any prior 5-minute-interval version of `com.jeff.orb-breakout`, or a
`com.jeff.orb-eod-close` from an earlier 2:45 PM CT setup is still loaded,
unload/replace it — this schedule replaces any prior version entirely, it
doesn't run alongside one:
```bash
launchctl unload ~/Library/LaunchAgents/com.jeff.public-trader-morning.plist 2>/dev/null
launchctl unload ~/Library/LaunchAgents/com.jeff.public-trader-close.plist 2>/dev/null
launchctl unload ~/Library/LaunchAgents/com.jeff.orb-breakout.plist 2>/dev/null
launchctl unload ~/Library/LaunchAgents/com.jeff.orb-eod-close.plist 2>/dev/null
rm -f ~/Library/LaunchAgents/com.jeff.public-trader-morning.plist
rm -f ~/Library/LaunchAgents/com.jeff.public-trader-close.plist
rm -f ~/Library/LaunchAgents/com.jeff.orb-eod-close.plist
```

## Windows — Task Scheduler (alternative)

```powershell
schtasks /create /tn "ORB Breakout Check" /tr "claude -p 'run the ORB breakout check'" /sc weekly /d MON,TUE,WED,THU,FRI /st 10:10
schtasks /create /tn "ORB Forced Close" /tr "claude -p 'run the ORB breakout check'" /sc weekly /d MON,TUE,WED,THU,FRI /st 12:00
```
Both tasks run the identical prompt; the skill branches on the current
America/Chicago time, same as the Mac setup above.

## Alternative: Claude Code Routines (cloud, no laptop dependency)

Routines (claude.ai/code/routines, research preview) run this skill on
Anthropic's cloud instead of a local Mac/Windows machine.

- **A native Schedule trigger has a 1-hour minimum interval.** The
  original every-5-minute design couldn't use it and needed an external
  cron hitting an API trigger instead. The current twice-daily design
  (10:10 AM CT breakout check, 12:00 PM CT forced close — nearly 2 hours
  apart) is well above that 1-hour floor, so **both runs can now use the
  native Schedule trigger directly** — no external cron or API-trigger
  workaround needed. The weekly review already used the native trigger.
- **No persistent local filesystem between runs** — this is exactly why
  the trade log lives in a Google Doc (see `references/trade-log.md`)
  rather than `~/orb-trade-log.md`: a cloud routine has nowhere durable to
  keep a local file across separate invocations, but Drive persists fine.

Set up as **three separate routines**, all on the native Schedule trigger:
1. **ORB Breakout Check** — trigger: Schedule, 10:10 AM CT weekdays.
   Connect a repo containing this `public-trader/` folder; instructions
   tell Claude to read `SKILL.md` and everything under `references/` from
   that repo and follow it for the run (the skill itself runs Steps 1-6
   and 9 when the explicitly-computed current CT time is 10:10 AM).
2. **ORB Forced Close** — trigger: Schedule, 12:00 PM CT weekdays. Same
   repo connection and identical instructions to routine 1 above — the
   skill branches internally on the current CT time, running just Steps
   3, 7, 8 and 9 (force-close and log the day's outcome) when it's 12:00
   PM CT instead of 10:10 AM CT.
3. **Weekly Review** — trigger: Schedule, first trading day of the week,
   8:00 AM CT. Same repo connection; instructions point at
   `references/trade-log.md` § Weekly Review instead.

Connectors needed on all three: **Public** only, for both trading and
5-min bars (`get_price_history` covers the opening-range and
breakout-check data — no separate market-data connector required); the
Breakout Check and Forced Close routines additionally need **Google
Drive** (writing/reading the trade log as part of Step 8), and so does the
Weekly Review (reading it). No Notifications tab setup is needed for exit
alerts — Public itself sends order-fill notifications for every leg
(entry, TP, SL) automatically; see `references/public-submission.md`
§ Exit Alerts.

**Note:** this skill previously used the Twelve Data connector for 5-min
bars. That dependency was dropped in favor of Public's own
`get_price_history` (proven out via a historical practice run) — Twelve
Data can be ignored/left disabled for this skill unless re-added later.
