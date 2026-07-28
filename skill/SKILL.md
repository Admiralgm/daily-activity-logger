---
name: daily-activity-logger
category: productivity
description: Log daily activities, job scans, scores, and progress across Hermes profiles. Maintains a chronological daily journal in the wiki activity directory.
version: 1.2.0
author: hermes-agent
license: MIT
platforms: [macos, linux]
---

# Daily Activity Logger

Class-level skill for logging daily activities, job search progress, scan events, and key decisions. Designed for multi-profile setups (AGENT/AGENT/AGENT/AGENT) where each agent produces output that needs tracking.

## When to Use

- User asks "log today's activity", "keep a daily log", "what did we do yesterday"
- After completing a scan, scoring session, or batch of JD extractions
- At end of a session that produced tangible results (tracker updates, scores, CVs)
- Before handing off context between profiles
- When user says "record this" or "note that"

## Where to Store

All daily activity logs go in:
```
config/wiki/activity/
```

Each entry is a dated file: `YYYY-MM-DD.md` (e.g., `2026-06-05.md`).

This matches the cron script at `config/scripts/log-daily-activity.sh` which creates the template structure. The agent fills in content via session_search.

**Do NOT use** `~/Downloads/DATA_REPOSITORY/DAILY_ACTIVITY_LOG.md` — that path is wrong.

## Entry Format

Each daily file uses this structure:

```markdown
---
date: YYYY-MM-DD
type: activity
status: complete
---

# Daily Activity Log — YYYY-MM-DD

## Summary
[One-paragraph summary of the day]

## Key Activities
- [Activity 1]
- [Activity 2]

## Decisions Made
- [Decision 1]

## New Knowledge / Skills
- [New learning]

## Outcomes
- [Tangible result]

## Next Steps / Follow-ups
- [Pending item]
```

### Minimal Entry (quick log)
```
YYYY-MM-DD | agent | Short summary of what was accomplished
```

## Quick Start (ALWAYS DO THIS)

When user says "execute daily activity logger" or "log today's activity":

1. **Check ALL profiles** — `ls -lt` the sessions dirs for agent, agent, agent, agent (and default). A profile with session files modified today had activity.
   ⚠️ WARNING: Do NOT use `session_search(profile='hermesN', ...)` to check per-profile activity - the profile parameter in session_search does NOT filter by profile and will return sessions from all profiles. Always check the filesystem session directories directly as shown above.

2. **Check existing log** — `cat config/wiki/activity/YYYY-MM-DD.md` to see if a file already exists and what it covers.
3. **Write ONE file per date** — `config/wiki/activity/YYYY-MM-DD.md` with a `## Cross-Profile Activity` section listing EVERY profile (even "No user sessions").
4. **Never log only the current profile** — if you only checked your own session, you WILL miss activity from other profiles. The user will ask "did it cover all profiles?"

## Usage Patterns

### Pattern 0: Backfill Missing Dates (Post-Hoc Reconstruction)

When asked to "execute daily activity logger" and existing logs show gaps:

1. **Discover gaps** — `ls -la config/wiki/activity/YYYY-MM-DD.md` or iterate dates since last log.
2. **Search session DB per date** — One `session_search(query="YYYY-MM-DD")` per missing date. Required field: `sort=newest`.
3. **Handle compression gracefully** — Discovery results are heavily compacted. Use the `bookend_start` and `messages` arrays as the primary source. Read persisted tool output files from `/var/folders/fr/v7gh2ttj2q97xyb40q1g6qx40000gp/T/hermes-results/` if you need detail not visible in the snippet.
4. **Cross-reference with filesystem** — Check known output directories (tracker dirs, `~/Desktop/`, skill workdirs) for files with matching timestamps to infer what was done on that day even when session text is sparse.
5. **Cross-reference with memory** — `memory.md` and `user.md` often contain date-stamped facts ("SINHRO MERGE 2026-06-20: +0h1 +1h2 +0h3") that anchor what happened.
6. **Write one file per missing date** — Same full entry format as Pattern 1, even if some sections are brief. "Session data heavily compacted; recovered from session_search + filesystem state" is a valid entry.
7. **Check `-l` flag resolution** — Hermes `memory.md` may list `SINHRO MERGE <date>` entries that confirm activity even when session_search returns 0 sessions. Do NOT create zero-day logs if no evidence exists, but DO create them if memory, filesystem, or session artifacts confirm work happened.

### Pattern 4: Full-History Backfill (All Profiles, All Dates)

When asked to "log everything which was not logged on all hermes profiles":

1. **Inventory existing logs** — List all *.md files in `config/wiki/activity/`, extract date strings.
2. **Inventory session dates per profile** — For each profile (agent, agent, agent, agent), parse session filenames in `config/profiles/<profile>/sessions/` to extract unique dates. Use `ls -1` + regex on filenames, not individual session_search calls.
3. **Cross-reference** — For each date with sessions, check if a log file exists AND which profiles are mentioned in it. Use `grep -c 'hermesN'` per file.
4. **Classify gaps** into three categories:
   - **Missing log file** (sessions exist but no YYYY-MM-DD.md at all) -> create new file with write_file
   - **Missing profile coverage** (log exists but doesn't mention a profile that had sessions that day) -> append section to existing file
   - **Fully covered** -> skip
5. **For missing log files**: Use write_file with full frontmatter + per-profile sections.
6. **For missing profile coverage**: Use patch to append a "## hermesN Session" section before the closing "---". NEVER use write_file to append to an existing file -- write_file overwrites the entire file, destroying existing content. Use patch with the closing "---" as the anchor.
7. **Verify after every batch** — Check that no files were corrupted (first line should be "---" or "# "). If corruption occurred, restore from the last known-good state.
8. **Batch efficiently** — Use execute_code with terminal() calls to search multiple dates/profiles in one script. Avoid one session_search call per date -- that is 30+ tool calls for a full backfill. Parse session filenames directly from the filesystem instead.

### Pattern 1: Full Entry at End of Session

```bash
cat > config/wiki/activity/YYYY-MM-DD.md << 'ENTRY'
---
date: YYYY-MM-DD
type: activity
status: complete
---

# Daily Activity Log — YYYY-MM-DD

## Summary
[Summary]

## Key Activities
- [Activities]
ENTRY
```

### Pattern 2: Quick One-Liner

```bash
echo "YYYY-MM-DD | agent | Short summary" >> config/wiki/activity/YYYY-MM-DD.md
```

### Pattern 3: Read Today's or Yesterday's Log

```bash
cat config/wiki/activity/YYYY-MM-DD.md
```

Search for a specific date:
```bash
grep -l "YYYY-MM-DD" config/wiki/activity/*.md
```

## Cross-Profile Coordination

When multiple profiles share the same activity directory:

1. Each profile writes to its own dated file or appends to the same file
2. Profiles read the latest entries before starting to see what was already done
3. Use the `profile` field to distinguish which agent did what

Example for a handoff log between profiles:
```
2026-06-04 | agent | Camoufox fix + 4-portal scrape, 12 new JDs, tracker updated
2026-06-04 | agent | Impactpool scan: 8 JDs scored, updated tracker
```

## Pitfalls

- **Cross-profile coverage is MANDATORY and FIRST step** — When asked to "execute daily activity logger", the VERY FIRST thing you do is check ALL profile session directories. Do NOT write a log for just your own profile and stop. The user will correct you ("I want logs across all the profiles") if you skip this. Check `ls -lt config/profiles/hermesN/sessions/` for N=0,1,2,3 AND default. **IMPORTANT**: `session_search(profile="hermesN")` does NOT filter by profile — it returns the same session regardless of the `profile` parameter. To detect per-profile activity, check filesystem session directories directly:
  ```bash
  # List session files per profile
  ls -la config/sessions/ | tail -20
  ls -la config/profiles/agent/sessions/ | tail -20
  ls -la config/profiles/agent/sessions/ | tail -20
  ```
  Session filenames contain dates (e.g., `session_20260623_165329_*.json`). Extract the date prefix to determine which days had activity per profile. If a profile had sessions that day, include it in the log. If a profile had no sessions, note it explicitly ("No sessions on June 20"). If a profile had sessions but they weren't logged, backfill them. The user will ask "did it cover all profiles?" if you miss one — prevent that question by checking proactively.
- **`session_search` profile parameter does NOT filter by profile** — When called with `profile="hermesN"`, `session_search` returns the SAME sessions regardless of the parameter value. The `profile` parameter only changes which database file is read for resolving `@session:<profile>/<id>` links, NOT which sessions are searched. Do NOT rely on `session_search(profile="agent")` to find agent-specific activity — use filesystem session directories instead (see pitfall above). See `references/filesystem-session-detection.md` for the canonical detection script.
- **Filesystem detection misses TUI sessions — ALWAYS run session_search in parallel** — `ls -1 sessions/ | grep "YYYYMMDD"` finds `request_dump_*` files, but TUI sessions that don't generate API request dumps are invisible to this method. On 2026-07-25, agent had 5 active TUI sessions (14-JD scoring, cross-profile CV generation, OCR migration, 3 job scans) but filesystem detection returned zero for all profiles — all 5 sessions were only found via `session_search(query="2026-07-25")`. **Correct approach**: run BOTH `ls -1` and `session_search` in parallel, then cross-reference. If either method finds activity, include it. Filter `session_search` results by `session_id.startswith('YYYYMMDD')` to avoid false-positive date matches from deadline fields.
- **session_search date queries produce false positives** — `session_search(query="2026-06-28")` matches
  ANY occurrence of that date string in session content (deadline fields, tracker data, etc.), not just
  sessions that occurred on that date. Always filter results by `session_id` prefix
  (`r['session_id'].startswith('YYYYMMDD')`) to isolate genuine date matches.
- **Empty session directory `..` line matches date grep** — When checking `ls -la | grep "Jun 27"`, the `..` parent directory entry often matches the date grep, producing a false positive (looks like sessions existed when they didn't). Use `ls -1 | grep "YYYYMMDD"` on session filenames directly, NOT `ls -la | grep "Mon DD"`.
- **Don't overwrite** — Always append or create new dated files. Never overwrite existing entries.
- **Same-session placeholder exception:** If you create an activity file early in the session as a "starter" (e.g. a placeholder with fuzzy session-start context), and later in the same session the user asks to "execute the activity logger", do NOT append — replace the placeholder entirely with the real end-of-session activity content. The placeholder has no value once the session completes; the final log is the source of truth.
- **Backfill by date, not today only:** When asked to "execute activity logger" or "update wiki with yesterday", search `session_search` for ALL missing dates since the last activity log, not just today. If there are gaps (e.g. missing Saturday and Sunday), create both files, not just today's.
- **Session rectification:** If a session produced tangible results (tracker updates, scores, CVs, skill patches, wiki updates, cron changes) but wasn't logged, the next session must backfill it. Use `session_search(query="<date>")` to reconstruct events and write the activity log post-hoc.
- **Wiki update + logger in one session:** When a single session contains both a wiki update task AND a "execute activity logger" command, the activity log for that date MUST include the wiki changes (pages created, skills patched, log.md/index.md updates) under Key Activities and Outcomes. Do NOT create a separate wiki-only log — there is one activity log per date, and it reflects everything done that day.
- **Don't log trivial tool calls** — Only log session-level outcomes (scan results, scores, file updates, decisions).
- **Don't log in the middle of a task** — Log at session end or at natural breakpoints.
- **Be consistent with date format** — Use `YYYY-MM-DD` only.
- **Profile label is mandatory** — Essential for multi-profile setups.
- **Don't create files in the wrong location** — Always use `config/wiki/activity/`, never `~/Downloads/DATA_REPOSITORY/`.
- **Combined "update wiki AND execute logger" → one activity log per date**: When the user stacks both requests into a single session (e.g., "please update LLM-WIKI and execute daily activity logger"), the activity logger runs once at the end of the session and writes a single `YYYY-MM-DD.md` for that date. If the session happens late in the day (past midnight UTC), use the user's local date (CEST) — don't create two files for the same local day. If the file already exists for that date, append to it or report "already exists" rather than overwrite.
- **`write_file ~` vs `$HOME` path mismatch:** If you create activity logs via `write_file` with a `~` path, it writes to the Hermes profile's home (`config/profiles/agent/home/`). Subsequent `ls` or `cat` via `terminal` using `~` will look at the REAL home (`~/`) and not find the file. **Always use absolute paths** (`config/wiki/activity/...`) when verifying existence of files written via `write_file`.
- **Cross-profile wiki editing:** The activity log directory `config/wiki/activity/` is shared across profiles. When running under agent/2/3, you can write to it directly via `terminal()` without cross-profile guard issues (it's not a skill/plugin/cron file). However, `write_file` with `~` paths will resolve to the profile's home, not the real home. Use absolute paths or `terminal()` with heredocs/cat for reliable writes.
- **`read_file`/`patch` pagination warning:** When you `read_file` a file with `offset`/ `limit` and then immediately `patch` the same file, the patch tool warns: "was last read with offset/limit pagination (partial view)." The patch still succeeds, but the warning is noisy during wiki maintenance at scale. Do a full `read_file` (no offset) before the patch, or simply ignore the warning.
- **NEVER use `replace_all=true` when appending to log.md via `patch()`** — The `old_string` anchor (e.g. `- agent: No sessions`) can appear multiple times in log.md: once in the new entry being added and once in a prior entry. Using `replace_all=true` replaces ALL matches, creating duplicate copies of the new entry. Hit live on 2026-07-23: the wiki update entry was duplicated because `- agent: No sessions\n- agent: No sessions` appeared twice. **Always use `replace_all=false` (default) and include enough surrounding context (the preceding `## [date] header` line) to make the `old_string` unique.** If patch reports "Found 2 matches", expand the `old_string` — do NOT switch to `replace_all=true`.

## Integration Points

- **Wiki activity directory**: The canonical location is `config/wiki/activity/` — same directory the cron script uses.
- **Cron auto-log**: The cron job at `config/scripts/log-daily-activity.sh` creates a template daily; the agent fills it via session_search.
- **Obsidian vault**: The wiki activity directory can be synced to Obsidian for visualization.

## References

- `references/backfill-reconstruction-recipe.md` — step-by-step protocol for reconstructing missing daily logs when session data is compacted or inaccessible.
- `tracker-file-format` skill — companion for the tracker files referenced in the activity log.
- `cv-repository` skill — companion for CV/score entries in the activity log.
- `llm-wiki` skill — the wiki system that hosts the activity directory.
