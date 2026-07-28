# Post-Hoc Activity Logger Reconstruction

One-page recipe for rebuilding missing daily activity logs when session data is compacted or lost from context.

## 1. Detect gaps quickly

```bash
for d in $(seq -f "%02g" 16 20); do
  f="$HOME/.hermes/wiki/activity/2026-06-${d}.md"
  echo "2026-06-${d}: $(test -f "$f" && echo EXISTS || echo MISSING)"
done
```

## 2. Search the session DB

One call per missing date. Required: `sort=newest`.

```bash
# Example: what happened on 2026-06-16?
session_search(query="2026-06-16", limit=5, sort=newest)
```

- `results[].snippet` and `results[].bookend_start` are your primary sources.
- `messages[]` array inside each result may contain the actual content.
- If `count > 0` but the response says `sessions_searched: 0`, look for a persisted file path in the output — large results are written to `/var/folders/…/T/hermes-results/`. Read that file with `read_file`.

## 3. Recover from a persisted file

When `session_search` returns a path like `/var/folders/…/T/hermes-results/functions.session_search:N.txt`, read it directly:

```bash
read_file(path="<persisted_path>", limit=500)
```

The file is raw JSON (one `session_search` result object). Parse the `results[]` array manually.

## 4. Cross-reference with filesystem

Even when sessions are fully compacted, artifacts survive on disk:

| Search target | What it reveals |
|---|---|
| `ls -lt ~/Downloads/DATA_REPOSITORY/ | head` | Tracker updates, scan outputs |
| `ls -lt ~/Desktop/ | head -20` | Exported files (presentations, reports) |
| `ls -lt config/wiki/ | head` | Wiki page creation/modification dates |
| `ls -lt skills/ | head` | Skill patches, new skills |
| `grep "2026-06-DD" config/MEMORY.md` | Date-stamped memory notes |

## 5. Cross-reference with memory

`memory.md` and `user.md` often contain lines like:

- `SINHRO MERGE 2026-06-20: +0h1 +1h2 +0h3`
- `Camoufox v2.4.3 updated 2026-05-05`

These anchor dates when session text is sparse. Use them to populate the **Summary** and **Key Activities** lines of the backfilled log.

## 6. Write the reconstructed log

Use the standard frontmatter, even if some sections are inferred:

```yaml
---
date: 2026-06-16
type: activity
status: complete
profile: agent
---
```

If the only recoverable fact is "impactpool-scan skill execution", that is sufficient for a minimal entry. If more detail is available (scored N entries, fixed bug X, output files Y and Z), include it.

## 7. Verify

```bash
ls -la config/wiki/activity/2026-06-*.md
```

Confirm no gaps remain.

---

**Note:** These steps are additive to the main SKILL.md workflow — a fallback when real-time logging was missed, not a replacement for per-session logging.
