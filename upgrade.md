# Upgrade: agent-work-mem v1 → v2 (tiered storage + index + onboarding primer)

> If your project already has `AIMemory/PROTOCOL.md` and `AIMemory/work.log`
> from a previous bootstrap of agent-work-mem, paste this prompt into your
> agent (Claude Code, ChatGPT Codex CLI, OpenCode, Antigravity, Cursor, etc.)
> to migrate to the v2 layout: `INDEX.md`, `PROJECT_OVERVIEW.md`,
> `archive/`, `cold/`, and the new tiering rules.
>
> Easiest one-line invocation:
>
> > "Fetch <https://raw.githubusercontent.com/daystar7777/agent-work-mem/main/upgrade.md>
> >  and execute it on this project."
>
> The agent will WebFetch this file and run the tasks below.

---

You are upgrading an existing AIMemory installation from the original
flat-file layout (just `PROTOCOL.md` + `work.log`) to the v2 tiered
layout. The new layout adds:

- `AIMemory/INDEX.md` — file inventory + topic search
- `AIMemory/PROJECT_OVERVIEW.md` — onboarding primer for new sessions/LLMs
- `AIMemory/archive/` — warm tier (rotated old events)
- `AIMemory/cold/` — cold tier (period digests, on-demand)
- New rules in `PROTOCOL.md` §7 (tiered storage)

The migration is non-destructive — your existing `work.log` keeps every
byte; older events just get moved into dated archive files if your log
has grown past the rotation threshold.

## Your tasks (in order)

### Task 0 — Verify state

Check that `AIMemory/PROTOCOL.md` and `AIMemory/work.log` already exist.
If they don't, this isn't an upgrade — run the bootstrap prompt instead
(<https://raw.githubusercontent.com/daystar7777/agent-work-mem/main/prompt.md>).

```bash
test -f AIMemory/PROTOCOL.md && test -f AIMemory/work.log && echo OK || echo "no v1 install — run bootstrap instead"
```

### Task 1 — Identify yourself

State at the top of your reply:

- **Model-id** (lowercase kebab-case)
- **Vendor**
- **Harness** (Claude Code / ChatGPT Codex CLI / OpenCode / Antigravity /
  Cursor / Aider / Cline / Continue / Windsurf / gemini-cli / other)
- **Capabilities** (vendor-neutral tags: `filesystem-read`,
  `filesystem-write`, `shell-exec`, `web-fetch`, `web-search`, `code-sandbox`,
  `image-input`, `subagent-spawn`)

### Task 2 — Fetch the canonical PROTOCOL.md

```bash
mkdir -p AIMemory/archive AIMemory/cold
```

Then fetch the latest PROTOCOL.md from the upstream repo, replacing the
existing file (your data in `work.log` is untouched):

```bash
curl -fsSL https://raw.githubusercontent.com/daystar7777/agent-work-mem/main/PROTOCOL.md \
  -o AIMemory/PROTOCOL.md
```

If your harness doesn't allow `curl`, use your `web-fetch` tool to load
the URL above and write the content to `AIMemory/PROTOCOL.md`.

### Task 3 — Create INDEX.md (with current state reflected)

Write `AIMemory/INDEX.md` using the template below. Populate the
"Other notable files" section by scanning the existing `AIMemory/`
directory for any files beyond `PROTOCOL.md` / `work.log` /
`INDEX.md` / `PROJECT_OVERVIEW.md`. List handoff files under
"Active handoffs" if their content suggests they are still open
(no `HANDOFF_CLOSED` corresponding event in work.log).

```markdown
# AIMemory Index

## Configuration
- HOT_RETENTION_EVENTS: 50

## Hot — read every session
- work.log — last N events, append-only

## Warm — read only when needed
| File | Date range | Events | Topics | Summary |
|------|------------|--------|--------|---------|
(none yet — populated when work.log first rotates)

## Cold — fetch only on explicit need
| File | Period covered | Topics | Summary |
|------|----------------|--------|---------|
(none yet)

## Topic index — grep me
(none yet — keywords appear here as archives are created)

## Active handoffs (open AICP threads)
<list any handoff_*.md files in AIMemory/ that haven't been HANDOFF_CLOSED
in work.log; if none, write "(none)">

## Other notable files
- PROTOCOL.md — the rules
- PROJECT_OVERVIEW.md — onboarding primer
<add any other markdown files you find in AIMemory/, with a 1-line summary>

---
Last update: <YYYY-MM-DD HH:MM> by <your-model-id>
```

### Task 4 — Generate PROJECT_OVERVIEW.md from existing work.log

This is the most useful part of the upgrade for existing projects: you
read the entire `work.log` and synthesize a project primer from it.

1. Read all of `AIMemory/work.log`.
2. Extract:
   - The project's purpose (often visible from the first PROMPT events)
   - Tech stack (from FILES_CREATED paths, package install events, etc.)
   - Key decisions (look for NOTE events, decisions in PROMPT/WORK_END)
   - Major work completed (WORK_END events with significant FILES_CREATED)
   - Active concerns (open WORK_START without WORK_END, open handoffs)
3. Write `AIMemory/PROJECT_OVERVIEW.md` using this structure:

```markdown
# Project Overview

> Onboarding for new LLMs joining this project. Read this AFTER
> AIMemory/INDEX.md (which tells you what files exist) and BEFORE
> AIMemory/work.log tail (which tells you what's happening right now).

## What is this project?
<2–4 sentences synthesized from the log. If unclear, ASK the user
during this upgrade and incorporate their answer.>

## Tech stack
- <bullets — infer from work.log: file extensions, install events,
  framework mentions in PROMPT/WORK_START events>

## Key decisions locked in
- <decision> (YYYY-MM-DD, <model-id>) — <one-line rationale>
- <decision 2> ...
<at least 3–8 bullets if the log has any meaningful history. If only a
few events, just list what you can.>

## Major work completed
- <YYYY-MM-DD>: <what shipped or finished>
- ...

## Active concerns
- <what's currently being worked on or blocked>

## Where to look
- Recent activity → AIMemory/work.log
- Topic-based history → AIMemory/INDEX.md (Topic index section)
- Long-term history → AIMemory/cold/digest-*.md (none yet)

---
Last rebuild: <YYYY-MM-DD> by <your-model-id>
Source: synthesized from full work.log on upgrade (was v1 → v2)
```

If you cannot synthesize a section from the log (genuinely no
information), write `<TODO: ask user>` instead of inventing content.

### Task 5 — Rotate if work.log is over threshold

```bash
EVENT_COUNT=$(grep -c '^### ' AIMemory/work.log)
THRESHOLD=$(grep '^- HOT_RETENTION_EVENTS:' AIMemory/INDEX.md 2>/dev/null \
  | awk -F: '{print $2}' | tr -d ' ')
[ -z "$THRESHOLD" ] && THRESHOLD=50

if [ "$EVENT_COUNT" -gt $((THRESHOLD * 3 / 2)) ]; then
  echo "Rotation needed: $EVENT_COUNT events; will reduce to $THRESHOLD."
fi
```

If rotation is needed, perform it per PROTOCOL.md §7.5:

1. Acquire flock if available:
   ```bash
   exec 9>AIMemory/.rotation.lock
   flock -n 9 || { echo "lock busy; skip rotation"; }
   ```
2. Read all events. Keep the most recent `THRESHOLD` events; archive the
   rest.
3. For each archived event, append to
   `AIMemory/archive/work-<UTC-date-from-event>.log`.
4. Atomically replace work.log with the kept events
   (`mv AIMemory/work.log.new AIMemory/work.log`).
5. Update INDEX.md: add a row in the Warm table for each archive file
   created. Extract 3–7 topic keywords per archive (kebab-case,
   lowercase, scanning the events for meaningful nouns/topics).
6. Append the topic mappings to the "Topic index — grep me" section
   of INDEX.md (one line per topic → file).
7. Append a NOTE event in the (now small) work.log:
   ```
   ### YYYY-MM-DD HH:MM | <model-id> | NOTE
   Rotated <N> events to AIMemory/archive/. Hot kept at <THRESHOLD>.
   v1 → v2 upgrade.
   ```

If `EVENT_COUNT <= THRESHOLD * 1.5`, skip rotation — log is small enough.

### Task 6 — Append the upgrade event

```bash
cat >> AIMemory/work.log <<'EOF'

### YYYY-MM-DD HH:MM | <your-model-id> | RE_ENGAGED
Vendor: <...>
Harness: <...>
Capabilities: <...>
Strengths: <...>
Context: <...>
Notes: AIMemory upgraded v1 → v2 (tiered storage + INDEX.md + PROJECT_OVERVIEW.md).
EOF
```

### Task 7 — Confirm

Reply with:
- Your model-id, vendor, harness, capabilities
- New files created (`INDEX.md`, `PROJECT_OVERVIEW.md`, `archive/`, `cold/`)
- Whether rotation ran (and if so, how many events archived to which dates)
- Topic keywords extracted (if you rotated)
- Any sections in PROJECT_OVERVIEW.md that you marked `<TODO: ask user>`
  so the user can fill them in
- One sentence: "I will follow the v2 PROTOCOL.md from this turn forward."

### Optional Task 8 — Suggest topic refinement

After your synthesis, the user may want to refine PROJECT_OVERVIEW.md or
add more topic keywords to INDEX.md. Offer:

> "I've synthesized PROJECT_OVERVIEW.md from your existing work.log. The
> 'What is this project?' / 'Tech stack' / 'Key decisions' sections are
> based on what I could infer. Want me to ask you specific questions to
> sharpen any of those, or run with what I have?"

---

Now execute the eight tasks. Begin.
