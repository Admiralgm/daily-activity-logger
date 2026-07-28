# Filesystem-Based Per-Profile Session Detection

## Problem
`session_search(profile="hermesN")` does NOT filter by profile — it returns the same sessions regardless. To detect which profiles were active on a given day, you must inspect the filesystem directly.

## Method

### Per-profile session directory
Each profile stores sessions at:
```
config/profiles/<profile>/sessions/
```
Session filenames contain dates in format `YYYYMMDD` (e.g., `request_dump_20260626_191925_287c14_20260626_211519_532983.json`).

### Detection script (optimal)
```bash
# List session filenames per profile, filtered by date
for profile in default agent agent agent agent; do
  dir="config/profiles/${profile}/sessions"
  if [ -d "$dir" ]; then
    count=$(ls -1 "$dir" 2>/dev/null | grep "YYYYMMDD" | wc -l)
    echo "${profile}: ${count} sessions"
  fi
done
```

### ⚠️ CRITICAL: Filesystem detection has gaps — ALWAYS cross-check with session_search

Filesystem detection finds `request_dump_*` files in the `sessions/` directory, but **TUI sessions that don't generate API request dumps are invisible to this method**. On 2026-07-25, agent had 5 active TUI sessions with major work (14-JD scoring, cross-profile CV generation, OCR migration, UN scan, NGO scan, remote vacancies scan) but `ls -1 | grep "20260725"` returned **zero** results for all profiles. The sessions were only discoverable via `session_search(query="2026-07-25 OR 2026-07-26")`.

**Correct approach: run BOTH methods in parallel, then cross-reference:**
1. Filesystem `ls -1 | grep "YYYYMMDD"` for quick per-profile check
2. `session_search(query="YYYY-MM-DD", sort="newest")` for content-based discovery
3. Cross-reference: if either method finds activity, include it in the log
4. Filter `session_search` results by `session_id` prefix (`r['session_id'].startswith('YYYYMMDD')`) to isolate genuine date matches (avoids false positives from deadline fields)

### Where profiles live
| Profile | Path |
|---------|------|
| default | `config/sessions.db` (single DB, no per-profile subdir) |
| agent | `config/sessions/` |
| agent | `config/profiles/agent/sessions/` |
| agent | `config/profiles/agent/sessions/` |
| agent | `config/profiles/agent/sessions/` |

### Known gotcha — parent directory false positive
Using `ls -la | grep "Jun 27"` matches the `..` parent directory entry (which shows the dir's own mod date), creating FALSE positives. Always use `ls -1 | grep "YYYYMMDD"` on filenames only.

### Centos/quick check
```bash
# One-liner: did any profile have sessions today?
for p in agent agent agent agent; do
  n=$(ls -1 config/profiles/$p/sessions/ 2>/dev/null | grep "20260627" | wc -l)
  [ "$n" -gt 0 ] && echo "$p: YES ($n)" || echo "$p: no"
done
```
