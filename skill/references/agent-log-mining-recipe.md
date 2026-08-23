# Agent-Log Mining for Per-Date Activity Reconstruction

Proven 2026-08-19 backfill (11-day gap, Aug 9–18). When session files are
missing or empty but agent.log exists, the log is the most reliable per-date
activity source — it records every conversation turn with the user's message.

## Core commands

Per profile, per date:

```bash
# 1. Which dates had ANY activity (fast scan across the gap)
grep -oE "^2026-08-[0-9]{2}" config/profiles/hermesN/logs/agent.log | sort -u

# 2. What the user actually asked (session presence + message text)
grep -E "^2026-08-DD" config/profiles/hermesN/logs/agent.log \
  | grep -viE "telegram|gateway.run|keepalive|Reconnect|reconnect" \
  | grep -E "turn_context|tool_executor|conversation_loop" | head -20
```

- `turn_context` lines contain `msg='<user message>'` — the actual user
  request for that session (e.g. `msg='execute NEKRETNINE skill, scan all
  three portals deeply...'`).
- `tool_executor` lines show which skills were loaded that turn.
- Filtering out telegram/gateway noise is ESSENTIAL: AGENT's agent.log is
  dominated by Telegram reconnect attempts (every 5 min, token rejected) that
  would otherwise drown the signal. `gateway.run`, `keepalive`, and
  `Reconnect` lines are infrastructure noise, not user activity.

## Cross-reference: skill Scan History tables

The nekretnine-vlasnik-scan SKILL.md contains a Scan History table with one
row per scan:

```
| 2026-08-15 AGENT | 1 | 8 | 5 | 14 | SEQUENTIAL DEEP SCAN (15 pages): ... |
```

Columns: date+profile | nek.rs | 4zida | halooglasi | total | details. This is
an authoritative per-day record of real-estate work — exact ad counts per
portal, which profile ran the scan, emails sent, 0 UNSENT status. Extract with:

```bash
grep -oE "\| 2026-08-1[0-9][a-z]? HERMES[0-9] \| [0-9]+ \| [0-9]+ \| [0-9]+ \| [0-9]+ \| [^|]{0,220}" \
  skills/automation/nekretnine-vlasnik-scan/SKILL.md
```

## Reconstruction order (what worked)

1. `ls -1 sessions/` per profile → dates with session files
2. `ps aux | grep "hermes -p"` + `ps -o lstart` → live TUI processes (alive
   today even with no session files)
3. agent.log date scan (step 1 above) → full per-profile activity map
4. `session_search` per missing date — filter results by `session_id`
   prefix `YYYYMMDD`; deadline fields in tracker content cause false-positive
   date matches
5. Skill scan-history tables for exact per-day scan records
6. Write one file per date; note "recovered from agent.log + session_search"
   when session data is sparse

## Output shape

One file per date at `config/wiki/activity/YYYY-MM-DD.md`
with full frontmatter + `## Cross-Profile Activity` section listing EVERY
profile (even "No sessions"). Backfill files are full entries, not one-liners.
