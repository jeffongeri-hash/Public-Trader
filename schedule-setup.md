# Schedule Setup — Opening Range Breakout

## Current live setup (revised 9/22/26): single daily check, cloud routine

**This strategy actually runs as a Claude Code cloud Routine named
"Public ODTE Trader," on a native Schedule trigger.** That trigger type has
a **1-hour minimum interval**, so the 5-minute intraday polling this file
originally described (below, kept for reference/local-machine setups) was
never really compatible with it — the routine can only fire once per
market morning, not every 5 minutes.

**Set the routine's Schedule trigger to 10:10 AM ET (9:10 AM CT), weekdays.**
This is the earliest time the 2-bar confirmation rule can even be
evaluated (it needs the 10:00–10:05 and 10:05–10:10 ET bars, both
complete). A breakout that confirms later than 10:10 AM ET will not be
caught — this is a known, accepted limitation of the single-run design,
traded off against not being able to poll every 5 minutes anyway.

Jeff needs to set/verify this trigger time himself in the routine's own
settings (claude.ai/code/routines) — a skill run can't change its own
routine's trigger from inside a run.

To match, the strategy itself changed too (see `SKILL.md` and
`config.md` § Exit Rule): single lot, one real STOP order at -30%, no
take-profit order, no bot-managed end-of-day close — Jeff exits manually.
Logging also moved from a Google Doc to a chat message Jeff copies
himself (`trade-log.md` § Log Storage) since Google Drive isn't reliably
available in the routine's session.

---

## Historical: 5-minute polling (retired, local-machine only)

The sections below describe a **5-minute polling setup for a tiered
TP/SL, bot-managed-EOD-close version of this strategy that has been
retired** (see the revision notes in `config.md`, `signal-logic.md`, and
`public-submission.md`, all dated 9/22/26). They're kept only because they
still work as a reference for running this kind of check-loop from a local
Mac/Windows machine, where an every-5-minutes cron IS possible (unlike the
cloud routine's 1-hour-minimum Schedule trigger) — if Jeff ever wants to
resurrect the tiered/polling design locally instead of the single-daily
cloud-routine design above, this is how. It does not reflect how the
strategy runs today.

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
review run**. (Retired — see note above.)

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

## Claude Code Routines (cloud) — this is the live setup

Routines (claude.ai/code/routines) run this skill on Anthropic's cloud
instead of a local Mac/Windows machine. This is how "Public ODTE Trader"
actually runs today. Two cloud-specific constraints shaped the current
design (see the revision notes throughout `SKILL.md` and its references,
all dated 9/22/26):

- **A native Schedule trigger has a 1-hour minimum interval** — it cannot
  run 5-minute intraday polling. Rather than route around that with an
  external cron hitting an API trigger, the strategy itself was
  simplified to a **single daily check** at 10:10 AM ET (9:10 AM CT) —
  the earliest the 2-bar confirmation can fire — using the native Schedule
  trigger directly, with a single stop-loss order and manual exits instead
  of a bot-managed intraday exit loop.
- **No persistent local filesystem between runs, and Google Drive isn't
  reliably enabled in the routine's session** — so the trade log is no
  longer auto-written anywhere. Each run prints its log entry in chat
  (see `references/trade-log.md` § Log Storage) and Jeff copies it into
  his own record at the end of the day.

Set up as **two separate routines**:
1. **Public ODTE Trader** (a.k.a. "ORB Check") — trigger: Schedule, 10:10
   AM ET (9:10 AM CT), weekdays. Connect a repo containing this
   `public-trader/` folder; instructions tell Claude to read `SKILL.md`
   and everything under `references/` from that repo and follow it for
   the run.
2. **Weekly Review** — trigger: Schedule, first trading day of the week,
   8:00 AM CT. Same repo connection; instructions point at
   `references/trade-log.md` § Weekly Review instead, and should include
   the week's chat-logged entries (or ask Jeff for them) since there's no
   Doc to read automatically anymore.

Connectors needed: **Public** (trading) at minimum. **Twelve Data** is
preferred for 5-min bars but optional — `Public:get_price_history` is a
working fallback if Twelve Data isn't enabled in the session (see
`references/public-submission.md`). Google Drive is no longer required by
either routine. No Notifications tab setup is needed for exit alerts —
Public itself sends order-fill notifications for every order (entry, stop)
automatically; see `references/public-submission.md` § Exit Alerts.
