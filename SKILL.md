---
name: ocas-usercontext
description: >
  Generates a compressed daily context section for the owner's USER.md.
  Infers mood, location, week theme, and day-level bullets (yesterday/today/tomorrow)
  from calendar, session history, and interaction patterns.
  Runs as cron, patches USER.md, delivers via Telegram.
license: MIT
source: https://github.com/indigokarasu/ocas-usercontext
metadata:
  author: Indigo Karasu (indigokarasu)
  version: 1.3.0
  hermes:
    tags: [context, daily, cron]
    category: infrastructure
includes:
  - references/**
---

# User Daily Context

## What it Produces

A `## Daily Context` section written into `~/.hermes/profiles/indigo/memories/USER.md` with this structure:

```markdown
# Daily Context YYYY-MM-DD

## Snapshot
**Mood:** [inferred from interaction patterns]
**Location:** [current or inferred]
**Week:** [1-line week theme]

## Yesterday (YYYY-MM-DD)
[2-3 bullets of what happened]

## Today (YYYY-MM-DD)
[2-3 bullets of known/planned activity]

## Tomorrow (YYYY-MM-DD)
[2-3 bullets of upcoming]
```

### Compression rules

- Total output: **under 300 words** (this is a snapshot, not a briefing)
- Bullets: 2-3 per day, **max 12 words each**
- Mood: single word or short phrase, inferred not stated
- Location: city name + travel indicator if applicable
- Week: one line, max 10 words
- **No filler, no weather, no system details, no advice**
- If calendar is empty for a day, write "No scheduled events" — don't fabricate

## Mood Inference

Mood is inferred from session patterns via `session_search`. See `references/mood-inference.md` for the full signal table and confidence rules.

## Data Sources

See `references/data-sources.md` for the full tool-to-signal mapping. Primary sources: `mcp_google_workspace_get_events`, `session_search`, `chronicle_ask_about`.

## Generation Workflow

- [ ] Step 1: Gather signals in parallel
  - Calendar events for yesterday, today, tomorrow (3 separate `mcp_google_workspace_get_events` calls)
  - Recent session activity (`session_search`, limit=5)
  - Chronicle context (`chronicle_ask_about`)
- [ ] Step 2: Extract and compress
  - For each day (yesterday, today, tomorrow): list calendar events (title + time), note session activity patterns, compress to 2-3 bullets (max 12 words each)
- [ ] Step 3: Infer mood
  - Scan last 3-5 sessions for tone signals
  - Apply mood inference table
  - If no sessions found, mood = "unknown"
- [ ] Step 4: Determine location
  - Check chronicle for known travel
  - Check calendar for out-of-city events
  - Output the city name (e.g. "San Francisco", "Honolulu", "unknown")
- [ ] Step 5: Write week theme
  - Look at full week's calendar, identify dominant theme (travel, deadline, meeting-heavy, quiet, project push)
  - Compress to one line, max 10 words
- [ ] Step 6: Patch USER.md
  - Replace the `## Daily Context` section of `~/.hermes/profiles/indigo/memories/USER.md` with the freshly generated snapshot
  - Use `patch` to replace only that section — do not touch any other part of USER.md
  - Section marker: `## Daily Context` (everything from that heading until the next `##` or end of file)
- [ ] Step 7: Deliver
  - Send the snapshot content to the owner via Telegram (origin channel)

## Schedule

| Job | Time | Action |
|-----|------|--------|
| `ocas-usercontext` | `0 7 * * *` (07:00 PT daily) | Generate + patch USER.md + deliver to Telegram |

## Constraints

See `references/constraints.md` for full rules and design rationale. Hard limits: 300 words, no fabrication, no weather, no advice, no em dashes, no meta-narration, never touch other USER.md sections.

## Target file

`~/.hermes/profiles/indigo/memories/USER.md` — the `## Daily Context` section.

This section is loaded every session as part of the user profile. It is the freshest signal about what's happening in the owner's life.

## Gotchas

- **USER.md not at `~/.hermes/profiles/indigo/USER.md`** — the actual file is at `memories/USER.md` inside the profile. Always use the full path.
- **Missing `## Daily Context` section in USER.md** — on first run, you must create the section. Use `patch` to add it after the last existing `##` heading. If USER.md itself is missing, create it with the daily context section as first content.
- **Patch targets wrong section** — the `patch` tool uses fuzzy matching. Ensure `## Daily Context` is unique in USER.md. If patch fails, read the raw file first and use exact `old_string`.
- **Calendar returns empty** — this is normal (quiet days). Write "No scheduled events" for that day. Don't fabricate.
- **Mood is ambiguous** — pick the more neutral option. Never output "mixed" or list multiple moods. One word only.
- **Cron deliver** — the cron job delivers to Telegram (origin channel). If deliver fails, the section is still written to USER.md. The context is not lost.
- **Stale context** — if the cron fails for multiple days, the context section becomes outdated. Check cron health if context seems wrong.

## Error Handling

| Failure | Response |
|---------|----------|
| Calendar API call fails | Skip calendar data for that day, write "No available calendar data" |
| `session_search` returns nothing | Mood = "unknown", no session pattern bullets |
| `chronicle_ask_about` fails | Location = "unknown" |
| `patch` on USER.md fails (section not found) | Add the `## Daily Context` section at the end of file instead |
| `patch` fails (fuzzy match) | Read raw USER.md, use exact string match for old_string |
| Telegram deliver fails | Context is still written to USER.md. Log failure. Retry next run. |
| USER.md not found | Create it with the daily context section as first content |

## Initialization

On first run:
1. Verify USER.md exists at `~/.hermes/profiles/indigo/memories/USER.md`
2. Add a `## Daily Context` section with a placeholder snapshot
3. Register the cron job `ocas-usercontext` (schedule `0 7 * * *`, deliver `origin`)
4. Log to journal
