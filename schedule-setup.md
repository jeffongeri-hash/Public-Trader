# Schedule Setup — Opening Range Breakout (5-minute polling, 2-bar confirmation)

## Current live setup (revised 9/22/26): 5-minute polling starting at 10:10 AM ET

**This strategy runs as a Claude Code cloud Routine named "Public ODTE
Trader."** Jeff confirmed on 9/22/26 that he wants intraday polling kept
(not a single daily check) — only the start time moved, from 10:00 AM ET
to **10:10 AM ET (9:10 AM CT)**, the earliest time the 2-bar confirmation
rule can be evaluated (it needs the 10:00–10:05 and 10:05–10:10 ET bars,
both complete).

**Platform constraint: a native Schedule trigger has a 1-hour minimum
interval** — it cannot fire every 5 minutes by itself. To get real 5-minute
polling on a cloud routine, use the routine's **API trigger** instead: it
exposes a per-run HTTP endpoint with a bearer token, and an external
5-minute caller (a cron-as-a-service, or a local machine's launchd/Task
Scheduler hitting that endpoint — see below for both local setups) invokes
it instead of the routine firing itself on a schedule. **Jeff needs to set
this up (switch the routine to an API trigger, and point some external
5-minute caller at it) — a skill run can't create standing external
infrastructure or change its own routine's trigger type from inside a
run.** Tell me if you want help picking/wiring the external caller.

The Weekly Review is unaffected — once a week is well above the 1-hour
floor, so it can keep using a native Schedule trigger directly.

To match the account's real order-placement limits found in the first live
trade, the strategy's exit design also changed (see `SKILL.md` and
`config.md` § Exit Rule): single lot, one real STOP order at -30%, no
take-profit order, no bot-managed end-of-day close — Jeff exits manually.
Logging also moved from a Google Doc to a chat message Jeff copies
himself (`trade-log.md` § Log Storage) since Google Drive isn't reliably
available in the routine's session.

---

This strategy requires 2 consecutive completed 5-minute bars to close beyond
the opening range boundary before a trade fires. The schedule runs **every
5 minutes**, not every 15 — checking less often risks missing or delaying a
confirmed signal by a full bar or more.

## Run Times (all weekdays, US market holidays skipped)

| Time (CT) | Time (ET) | Purpose |
|-----------|-----------|---------|
| 9:10 AM | 10:10 AM | Capture opening range (30-min H/L) + prior-day context, run first breakout check |
| 9:15 AM | 10:15 AM | Breakout check (2-bar confirmation) |
| 9:20 AM | 10:20 AM | Breakout check |
| ... every 5 min ... | ... | Breakout check |
| 2:45 PM | 3:45 PM | Final breakout check — no forced close (Jeff exits manually); this run prints the day's log entry if no trade fired earlier |

That's **68 runs per day** between 9:10 AM and 2:45 PM CT, plus **1 weekly
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

**Note on load:** 68 calendar-interval entries firing `claude -p` every 5
minutes for ~5.6 hours is a lot of process spin-up overhead. If that proves
too heavy in practice (API costs, launchd reliability, overlapping runs if
one invocation takes >5 min to finish), consider switching to a single
long-running process with an internal sleep loop instead of 68 separate
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
schtasks /create /tn "ORB Breakout Check" /tr "claude -p 'run the ORB breakout check'" /sc minute /mo 5 /st 09:10 /et 14:45
```
Task Scheduler's `/sc minute /mo 5` runs every 5 minutes within the
start/end window on the days configured in the GUI (weekday recurrence is
easier to set via the Task Scheduler GUI than the command line for this
trigger type).

## Claude Code Routines (cloud) — this is the live setup

Routines (claude.ai/code/routines) run this skill on Anthropic's cloud.
This is how "Public ODTE Trader" actually runs today, and Jeff confirmed
on 9/22/26 that the routine's own Schedule-trigger UI refuses a 5-minute
interval no matter how it's set — consistent with the 1-hour-minimum
platform limit noted above. A native Schedule trigger literally cannot
poll every 5 minutes; there is no setting that gets around this.

**The fix: switch the routine to an API trigger, and have something
external call it every 5 minutes.**

1. In the routine's settings (claude.ai/code/routines → "Public ODTE
   Trader"), change the trigger type from Schedule to **API trigger**.
   This generates a per-run HTTPS endpoint plus a bearer token, and
   usually shows a sample `curl` call — paste that sample here so the
   exact request shape (headers/method/body) can be wired up correctly.
2. **Recommended external caller: a GitHub Actions scheduled workflow in
   this repo** (`jeffongeri-hash/Public-Trader`), since it's already
   connected and needs no new service signup:
   ```yaml
   # .github/workflows/orb-poll.yml
   name: ORB breakout poll
   on:
     schedule:
       - cron: '10-45/5 14 * * 1-5'   # 9:10-2:45 CT = 14:10-19:45 UTC (CDT); adjust for standard time
   jobs:
     poll:
       runs-on: ubuntu-latest
       steps:
         - run: |
             curl -s -X POST "$ROUTINE_ENDPOINT" \
               -H "Authorization: Bearer $ROUTINE_TOKEN"
           env:
             ROUTINE_ENDPOINT: ${{ secrets.ORB_ROUTINE_ENDPOINT }}
             ROUTINE_TOKEN: ${{ secrets.ORB_ROUTINE_TOKEN }}
   ```
   Store the endpoint URL and bearer token as **repo secrets** (Settings →
   Secrets and variables → Actions) — never commit the token in plaintext,
   since it can place real trades. GitHub Actions cron is known to fire a
   few minutes late under load; that's tolerable here since every run
   re-derives the breakout from the two most recently *completed* bars
   rather than assuming which exact 5-minute slot it's in — a late run
   just detects a confirmed breakout a few minutes after it happened, it
   doesn't miscompute it. A local machine's launchd/Task Scheduler (below)
   is the more precisely-timed alternative if Jeff has one he can leave on
   during market hours instead.
3. The CDT/CST cron offset above will drift twice a year at DST
   transitions — flag it and it can be split into two cron lines (one for
   each UTC offset) if that matters.

The Weekly Review is unaffected by any of this — once a week is well
above the 1-hour floor, so it can keep its own native Schedule trigger.

Once Jeff has the API trigger's endpoint + token, paste them (or just the
sample curl command) here and the workflow file above can be finalized and
committed for real, plus the repo secrets can be named precisely.

Connectors needed: **Public** (trading) at minimum. **Twelve Data** is
preferred for 5-min bars but optional — `Public:get_price_history` is a
working fallback if Twelve Data isn't enabled in the session (see
`references/public-submission.md`). Google Drive is no longer required.
No Notifications tab setup is needed for exit alerts — Public itself sends
order-fill notifications for every order (entry, stop) automatically; see
`references/public-submission.md` § Exit Alerts.
