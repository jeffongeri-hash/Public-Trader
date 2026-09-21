# Schedule Setup — Opening Range Breakout (5-minute polling, 2-bar confirmation)

This strategy requires 2 consecutive completed 5-minute bars to close beyond
the opening range boundary before a trade fires. The schedule runs **every
5 minutes**, not every 15 — checking less often risks missing or delaying a
confirmed signal by a full bar or more.

## Run Times (all weekdays, US market holidays skipped)

| Time (CT) | Time (ET) | Purpose |
|-----------|-----------|---------|
| 9:00 AM | 10:00 AM | Capture opening range (30-min H/L) + prior-day context, run first breakout check |
| 9:05 AM | 10:05 AM | Breakout check (2-bar confirmation) |
| 9:10 AM | 10:10 AM | Breakout check |
| ... every 5 min ... | ... | Breakout check |
| 2:45 PM | 3:45 PM | Final breakout check **+ forced end-of-day close** |

That's **70 runs per day** between 9:00 AM and 2:45 PM CT, plus **1 weekly
review run**.

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

## Mac — launchd (recommended, intraday polling)

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
```

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
  <array>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>5</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>10</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>15</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>20</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>25</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>35</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>40</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>45</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>50</integer></dict>
    <dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>55</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>5</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>10</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>15</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>20</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>25</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>35</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>40</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>45</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>50</integer></dict>
    <dict><key>Hour</key><integer>10</integer><key>Minute</key><integer>55</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>5</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>10</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>15</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>20</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>25</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>35</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>40</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>45</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>50</integer></dict>
    <dict><key>Hour</key><integer>11</integer><key>Minute</key><integer>55</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>5</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>10</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>15</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>20</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>25</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>35</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>40</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>45</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>50</integer></dict>
    <dict><key>Hour</key><integer>12</integer><key>Minute</key><integer>55</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>5</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>10</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>15</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>20</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>25</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>35</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>40</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>45</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>50</integer></dict>
    <dict><key>Hour</key><integer>13</integer><key>Minute</key><integer>55</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>0</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>5</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>10</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>15</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>20</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>25</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>30</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>35</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>40</integer></dict>
    <dict><key>Hour</key><integer>14</integer><key>Minute</key><integer>45</integer></dict>
  </array>
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
launchctl list | grep orb-breakout
```

**Note on load:** 70 calendar-interval entries firing `claude -p` every 5
minutes for ~5.75 hours is a lot of process spin-up overhead. If that proves
too heavy in practice (API costs, launchd reliability, overlapping runs if
one invocation takes >5 min to finish), consider switching to a single
long-running process with an internal sleep loop instead of 70 separate
launchd triggers — flag it if you hit that wall and I'll rework this.

### Retiring an old public-trader schedule
If `com.jeff.public-trader-morning` / `-close` (twice-daily equity era) or
any prior version of `com.jeff.orb-breakout` is still loaded, unload/replace
it — this schedule replaces it entirely, it doesn't run alongside it:
```bash
launchctl unload ~/Library/LaunchAgents/com.jeff.public-trader-morning.plist 2>/dev/null
launchctl unload ~/Library/LaunchAgents/com.jeff.public-trader-close.plist 2>/dev/null
launchctl unload ~/Library/LaunchAgents/com.jeff.orb-breakout.plist 2>/dev/null
rm -f ~/Library/LaunchAgents/com.jeff.public-trader-morning.plist
rm -f ~/Library/LaunchAgents/com.jeff.public-trader-close.plist
```

## Windows — Task Scheduler (alternative)

```powershell
schtasks /create /tn "ORB Breakout Check" /tr "claude -p 'run the ORB breakout check'" /sc minute /mo 5 /st 09:00 /et 14:45
```
Task Scheduler's `/sc minute /mo 5` runs every 5 minutes within the
start/end window on the days configured in the GUI (weekday recurrence is
easier to set via the Task Scheduler GUI than the command line for this
trigger type).

## Alternative: Claude Code Routines (cloud, no laptop dependency)

Routines (claude.ai/code/routines, research preview) run this skill on
Anthropic's cloud instead of a local Mac/Windows machine. Two constraints
change the setup:

- **A native Schedule trigger has a 1-hour minimum interval** — it cannot
  run the 5-minute intraday breakout check directly. Use the routine's
  **API trigger** instead: it exposes a per-run HTTP endpoint with a
  bearer token, and an external 5-minute cron (still your local launchd,
  or any cron-as-a-service) calls that endpoint instead of running
  `claude -p` directly. The weekly review's cadence (once a week) is well
  above the 1-hour floor, so it can use the native Schedule trigger as-is.
- **No persistent local filesystem between runs** — this is exactly why
  the trade log lives in a Google Doc (see `references/trade-log.md`)
  rather than `~/orb-trade-log.md`: a cloud routine has nowhere durable to
  keep a local file across separate invocations, but Drive persists fine.

Set up as **two separate routines**:
1. **ORB Check** — trigger: API. Connect a repo containing this
   `public-trader/` folder; instructions tell Claude to read `SKILL.md`
   and everything under `references/` from that repo and follow it for
   the run. An external 5-minute cron calls the routine's endpoint during
   market hours.
2. **Weekly Review** — trigger: Schedule, first trading day of the week,
   8:00 AM CT. Same repo connection; instructions point at
   `references/trade-log.md` § Weekly Review instead.

Connectors needed on both: **Public** (trading) and **Twelve Data** (5-min
bars) at minimum; the weekly review additionally needs **Google Drive**
(reading the trade log) — the ORB Check routine needs it too, since it
writes the daily log entry as part of Step 8. No Notifications tab setup
is needed for exit alerts — Public itself sends order-fill notifications
for every leg (entry, TP, SL) automatically; see
`references/public-submission.md` § Exit Alerts.
