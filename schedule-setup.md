# Schedule Setup — ORB v2 (one long-lived session per day)

v1 tried to run 70 separate 5-minute checks a day. In practice the cloud
routine fired roughly hourly, so signals were seen late or not at all
(10/6/26: confirmed 10:15 ET, never traded). v2 runs **one session per
trading day** that stays alive and does its own 5-minute bar checks and
30-second position polls (SKILL.md § Session Flow).

## Routines (Claude Code cloud — claude.ai/code/routines)

| Routine | Trigger | Prompt |
|---|---|---|
| **ORB Session** | Schedule: weekdays **8:55 AM CT (9:55 AM ET)**, once | "Read SKILL.md and every reference file in the connected repo and run today's ORB session per SKILL.md. Live account 5OI27877 — orders execute immediately; no confirmation needed. Report the Phase 3 summary." |
| **ORB Weekly Review** | Schedule: Mondays 8:00 AM CT (fires Tuesday too; skips if this ISO week's review already exists in the log) | "Run the ORB weekly review per trade-log.md § Weekly Review." |
| **ORB Monthly Review** | Schedule: 1st–3rd of each month 8:00 AM CT (runs on the first trading day; skips if this month's review already exists) | "Run the ORB monthly review per trade-log.md § Monthly Review." |

All three need the **Public** and **Google Drive** connectors. The session
also uses WebSearch for the macro calendar (it degrades gracefully if
unavailable).

**Retire the old schedule:** the existing "Public ODTE Trader" routine
(hourly, combined with a monthly review) should be replaced by the three
routines above — not run alongside them. If both run, the session lock
(config.md § Session Lock) and the fills-based one-trade gate prevent
double entries, but an hourly routine still wastes runs and can't manage
exits in between.

### Market holidays (no session)
2026: Jan 1, Jan 19, Feb 16, Apr 3, May 25, Jun 19, Jul 3, Sep 7, Nov 26,
Dec 25. Half days (no trade): Nov 27, Dec 24. Phase 0 checks this list.

### Session length
A normal day: 9:55 ET start → entry decision by 11:30 → if a trade is
open, watching until it closes (often within an hour; at most 15:45 ET).
No-trade days end by 11:30 ET at the latest, usually much earlier
(SKIPPED/MISSED/NO TRADE end the session immediately after logging).

If the platform ends the session early (time limit, crash), the resting
disaster stop protects the open lot, Public's own fill notifications
report any stop fill, and a re-run of the ORB Session routine resumes
management (Phase 0 step 3).

## Local alternative (Mac launchd)
One job instead of 70:
```bash
cat > ~/orb-session-run.sh << 'EOF'
#!/bin/bash
[ "$(date +%u)" -ge 6 ] && exit 0
claude -p "run today's ORB session per SKILL.md" >> ~/orb-session.log 2>&1
EOF
chmod +x ~/orb-session-run.sh
```
`~/Library/LaunchAgents/com.jeff.orb-session.plist` → `StartCalendarInterval`
Hour 8, Minute 55 (CT), `ProgramArguments` `/bin/bash
~/orb-session-run.sh`. Unload any old `com.jeff.orb-breakout` (70-entry)
plist first:
```bash
launchctl unload ~/Library/LaunchAgents/com.jeff.orb-breakout.plist 2>/dev/null
rm -f ~/Library/LaunchAgents/com.jeff.orb-breakout.plist
launchctl load ~/Library/LaunchAgents/com.jeff.orb-session.plist
```
