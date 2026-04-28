# AIMemory Protocol — Multi-LLM Collaboration Rules

> Applies to every AI agent invoked on this project, regardless of model
> or vendor. Read this before acting on any user request.

---

## The five rules

### 1. AIMemory is the home of all AI-authored markdown

Every markdown file you author lives under `AIMemory/`.

Exceptions — files that specific harnesses load from fixed paths
(e.g. Claude Code reads `CLAUDE.md` from project root; Cursor reads
`.cursorrules`). If your harness has such a file, it stays where the
harness expects it.

Everything else — notes, plans, reviews, scratch, design docs, analyses
— goes in `AIMemory/`.

### 2. work.log is the shared memory

`AIMemory/work.log` is an append-only event log. Every AI MUST append the
following events:

- **PROMPT** — the user's message, verbatim, in a `> ` blockquote
- **WORK_START** — when you begin acting; include a one-line task summary
- **FILES_CREATED / FILES_MODIFIED / FILES_MOVED / FILES_DELETED** — full
  absolute paths whenever you touch the filesystem
- **WORK_END** — when finished; include status (complete / blocked / partial)
- **NOTE** — assumptions, uncertainties, open questions for next agent
- **HANDOFF / HANDOFF_RECEIVED / HANDOFF_CLOSED** — see AICP below
- **PROJECT_BOOTSTRAPPED / RE_ENGAGED** — session start markers; include
  the agent's capabilities (see §9 below)

Event format:

```
### YYYY-MM-DD HH:MM | <your-model-id> | <EVENT>
<body, free-form>
```

### 3. Read work.log before working — every new turn

At session start, AND at the start of any new user request after a long idle:

1. Read the tail of `AIMemory/work.log` (last ~50 lines is enough)
2. Look for any `WORK_START` without a matching `WORK_END`
3. If found, ASK the user: "Previous session has an unfinished task: <one-line
   summary>. Resume, or start fresh?" before doing anything else

This is a HARD RULE. Skipping it risks clobbering another AI's in-flight work.

### 4. Model name in every filename you create

Files you author must carry your model identifier:

```
AIMemory/{slug}.{your-model-id}.md
```

Examples:
- `refactor-plan.claude-opus-4-5.md`
- `auth-review.gpt-5-codex.md`
- `data-schema.gemini-2-5-pro.md`

If multiple agents edit the same document, the **originator's** model-id
stays in the filename; subsequent editors note their contribution in
`work.log`, not in the filename.

### 5. One agent owns work.log writes during multi-agent work

When multiple agents run in parallel (e.g. one orchestrator firing several
sub-tasks), only the **orchestrator** writes to `work.log`. Sub-agents
return their results to the orchestrator; they do not touch the log.

If two orchestrators run in parallel (rare), each records in its own dated
block and includes a `HANDOFF` event when work transitions between them.

### 6. Race-free append discipline (CRITICAL for multi-LLM)

`work.log` is a shared file. When two LLMs (different sessions, possibly
different machines) write at the same time, naive writes lose data.
Apply the following tiered discipline.

#### 6.1 Baseline — required for every agent, every append

**Rule A — single atomic append per event.** Use ONE shell call (heredoc
to `>>`) per event. POSIX `O_APPEND` guarantees that appends ≤ 4096 bytes
are atomic on local filesystems, so concurrent agents on the same machine
won't interleave bytes:

```bash
cat >> AIMemory/work.log <<'EOF'

### YYYY-MM-DD HH:MM | <model-id> | <EVENT>
<body>
EOF
```

DO NOT split a single event across multiple appends. DO NOT use a
high-level "edit file" tool that performs read-modify-write — that
pattern races even when both agents are well-behaved.

**Rule B — keep events under 4 KB.** If your message body is longer than
~3000 chars, split: write a short event in `work.log` and put the bulk
in a separate `AIMemory/<slug>.<model-id>.md` file. The work.log entry
just links to it.

**Rule C — never edit-in-place.** Never open `work.log`, modify, save.
Append-only. If you must correct an earlier entry, append a NEW event
of type `CORRECTION` referencing the original timestamp.

**Rule D — read tail right before write.** When you're about to append,
read the last ~20 lines first. If you see another agent's `WORK_START`
within the last 5 minutes without a matching `WORK_END`, that agent is
likely active. Append a `NOTE` BEFORE your own event flagging concurrent
work (so the human reader can spot interleavings):

```
### HH:MM | <my-model> | NOTE
Concurrent work detected — <other-model>'s WORK_START at HH:MM is still
open. My events below may interleave with theirs.
```

#### 6.2 Strong lock — recommended when truly concurrent

If your shell has `flock` (Linux, macOS, git-bash on Windows), wrap
appends in an exclusive lock. This serializes appends across processes
on the same machine, eliminating same-machine race entirely:

```bash
flock AIMemory/work.log -c "cat >> AIMemory/work.log <<'EOF'

### YYYY-MM-DD HH:MM | <model-id> | <EVENT>
<body>
EOF"
```

If `flock` is not available (Windows cmd/PowerShell without git-bash),
fall back to the baseline rules — same-machine atomicity from `O_APPEND`
covers the common case.

#### 6.3 Cloud-synced AIMemory — switch to per-session files

If `AIMemory/` lives on Dropbox / iCloud / Google Drive / OneDrive (i.e.
synced across multiple machines), `work.log` will produce conflicted
copies (`work.log (conflict 2026-04-26).log` etc.) when two machines
write before sync converges. POSIX atomicity does not help here — the
race is at the sync layer.

**Mitigation — per-session log files.** Each session writes to its OWN
file under `AIMemory/sessions/`:

```
AIMemory/
├── work.log                  ← legacy / index (optional, mostly empty)
└── sessions/
    ├── 2026-04-26T14-30__claude-opus-4-5__claude-code.log
    ├── 2026-04-26T14-32__gpt-5-codex__chatgpt-codex-cli.log
    └── 2026-04-26T15-10__gemini-2-5-pro__antigravity.log
```

Naming: `<UTC ISO8601 sortable>__<model-id>__<harness>.log`

Each session file is exclusively owned by ONE session — zero contention
even across machines, because no two sessions ever write the same path.

When READING the project history, agents merge all `AIMemory/sessions/*.log`
files sorted by timestamp:

```bash
# tail of merged view
cat AIMemory/sessions/*.log | grep -E '^### ' | sort | tail -50
# or with bodies:
ls -1 AIMemory/sessions/*.log | xargs -I{} sh -c 'cat {}; echo'
```

The legacy `AIMemory/work.log` may still exist as a manually curated
digest, but it is no longer the primary write target.

How to know if you're in this mode: look for `AIMemory/sessions/` —
if the directory exists, use it. If not, you're in shared-`work.log`
mode (default for local-only projects).

#### 6.4 Decision matrix — quick reference

| Setup                                             | Mode                  |
|---------------------------------------------------|-----------------------|
| Local project, single agent at a time             | shared work.log + 6.1 |
| Local project, occasionally 2+ agents same machine| shared work.log + 6.2 |
| Project on Dropbox/iCloud/Drive, multi-machine    | per-session files 6.3 |
| Git-versioned AIMemory, manual merge OK           | shared work.log + 6.1 |

If unsure → start with shared work.log + 6.1. Migrate to 6.3 the first
time you see a conflict file.

### 7. Tiered storage — keep context small

After a few days of multi-agent use, `work.log` + handoff files + scratch
notes accumulate. Reading the whole pile on every session burns LLM
context for no benefit. Tiered storage separates recent-and-relevant
(always read) from older-and-rarely-needed (lazy-load).

#### 7.1 The three tiers

- **Hot** — `AIMemory/work.log`. The N most recent events. Every new
  session reads this in full.
- **Warm** — `AIMemory/archive/work-<YYYY-MM-DD>.log`. Older events
  grouped by UTC date. Read only when the user's request reaches into
  history older than the hot tier covers.
- **Cold** — `AIMemory/cold/digest-<period>.md`. Multi-week or
  multi-month summaries. Fetch only on explicit need.

#### 7.2 INDEX.md — read this FIRST every session

`AIMemory/INDEX.md` is the directory's table of contents AND search index.
It's small and cheap to load. It tells you:
- which warm archives exist and what each contains (so you skip irrelevant ones)
- a flat **topic index** mapping keywords to the files that cover them
  (so you can `grep` for a topic instead of reading anything)

Required sections of INDEX.md:

```markdown
# AIMemory Index

## Configuration
- HOT_RETENTION_EVENTS: 50

## Hot — read every session
- work.log — last N events, append-only

## Warm — read only when needed
| File | Date range | Events | Topics | Summary |
|------|------------|--------|--------|---------|
| archive/work-2026-04-26.log | 2026-04-26 | 87 | auth, jwt, refresh-tokens, toctou-race | JWT auth implementation; TOCTOU race in refresh-rotation found by gpt-5-codex review, fixed via SELECT FOR UPDATE. |
| archive/work-2026-04-25.log | 2026-04-25 | 23 | bootstrap, obsidian, project-setup | Initial project bootstrap, Obsidian vault opened. |

## Cold — fetch only on explicit need
| File | Period covered | Topics | Summary |
|------|----------------|--------|---------|
| cold/digest-2026-03.md | 2026-03 | foo, bar, baz | One-paragraph summary for the whole month. |

## Topic index — grep me
auth          → archive/work-2026-04-26.log
jwt           → archive/work-2026-04-26.log
refresh-tokens → archive/work-2026-04-26.log
toctou-race   → archive/work-2026-04-26.log
bootstrap     → archive/work-2026-04-25.log
obsidian      → archive/work-2026-04-25.log

## Active handoffs (open AICP threads)
- handoff_perf.gemini-2-5-pro.md — to claude-opus, NORMAL

## Other notable files
- PROJECT_OVERVIEW.md — onboarding primer (read after INDEX.md)
- session-checkpoint-2026-04-26.claude-opus-4-7.md — major checkpoint

---
Last update: 2026-04-26 22:30 by claude-opus-4-7
```

How agents use the topic index:

```bash
# user asks: "did we ever discuss auth?"
grep -i "auth" AIMemory/INDEX.md
# → archive/work-2026-04-26.log appears in the topic index
# load only that file; skip the rest.
```

**Maintaining topics**: when an agent rotates events to an archive, it
extracts 3–7 topic keywords (kebab-case, lowercase) summarizing what
the archived events covered, and updates both:
1. The Warm table's `Topics` column for that archive
2. The flat Topic index (one line per topic → file mapping)

If a new archive covers a topic that already exists, append the new
file to the existing topic line:

```
auth → archive/work-2026-04-26.log, archive/work-2026-05-12.log
```

Don't be precious about topic granularity — over-tagging is fine,
under-tagging hurts findability.

#### 7.3 Configuration (user-tunable)

The user controls the rotation threshold via `HOT_RETENTION_EVENTS`
in INDEX.md. Recommended defaults:

| Project type                       | HOT_RETENTION_EVENTS |
|------------------------------------|----------------------|
| Active multi-agent (≥3 agents/day) | 30                   |
| Standard (default)                 | 50                   |
| Long-running solo                  | 100                  |

If unset, agents default to 50.

#### 7.4 When to rotate

At WORK_START, count current events. If over threshold × 1.5, rotate.
The 1.5× hysteresis avoids thrashing on every single event.

```bash
EVENT_COUNT=$(grep -c '^### ' AIMemory/work.log)
THRESHOLD=$(grep '^- HOT_RETENTION_EVENTS:' AIMemory/INDEX.md 2>/dev/null \
  | awk -F: '{print $2}' | tr -d ' ')
[ -z "$THRESHOLD" ] && THRESHOLD=50

if [ "$EVENT_COUNT" -gt $((THRESHOLD * 3 / 2)) ]; then
  # rotate (see §7.5)
fi
```

#### 7.5 Rotation algorithm

This is the one place where the protocol modifies (not appends to)
`work.log`. Treat it as a single atomic maintenance operation.

1. **Acquire lock** (use flock if available; else verify no orphan
   `WORK_START` from another agent):
   ```bash
   exec 9>AIMemory/.rotation.lock
   flock -n 9 || { echo "rotation in progress, skipping"; exit 0; }
   ```
2. Read all events from `work.log`.
3. Keep the most recent THRESHOLD events. The rest get archived.
4. For each archived event, append to
   `AIMemory/archive/work-<UTC-date-from-event-timestamp>.log`. Group
   by date.
5. **Atomically replace** `work.log` with the kept events:
   ```bash
   # write kept-events to a temp file, then rename:
   mv AIMemory/work.log.new AIMemory/work.log
   ```
6. **Update INDEX.md**: add or refresh the row in the warm table for
   each archive file you wrote to (event count, date range, one-line
   summary).
7. Append a NOTE event in the (now small) `work.log`:
   ```
   ### YYYY-MM-DD HH:MM | <model-id> | NOTE
   Rotated <N> events to AIMemory/archive/. Hot kept at <THRESHOLD>.
   ```
8. Release lock.

If you cannot rotate safely (concurrent agent active, no flock), append
a NOTE flagging "rotation needed" and let a future single-agent
session do it. Don't half-rotate.

#### 7.6 Cold digesting + PROJECT_OVERVIEW maintenance

Cold digests are heavy — they read and summarize many archive files.
Don't do this automatically. Trigger only on:
- Explicit user request ("digest last month into cold")
- Archive directory > 1 year old or > 100 files

Cold digest naming: `AIMemory/cold/digest-<YYYY-MM>.md` (monthly) or
`AIMemory/cold/digest-<YYYY-Wnn>.md` (weekly).

Each cold digest must contain:

```markdown
# Cold digest — <period>

**Period covered**: YYYY-MM-DD → YYYY-MM-DD
**Source archives**: archive/work-YYYY-MM-DD.log, ...
**Distilled by**: <model-id> on YYYY-MM-DD

## What happened (chronological)
<3–10 bullets, each one decision or major work item with date>

## Decisions locked in
<list of decisions that future sessions should NOT revisit>

## Open questions / unresolved
<anything still hanging from this period>

## Topics
<comma-separated keywords for INDEX.md topic index>
```

**After writing a cold digest, ALWAYS update:**
1. INDEX.md — add the digest to the Cold table; update Topic index
2. **PROJECT_OVERVIEW.md** — re-merge new "Decisions locked in" + major
   work into the running overview (see §7.8)

You may compress or delete the underlying archives that the digest
covers — but ONLY with explicit user approval. Default: keep them
("cold" doesn't mean deleted).

#### 7.7 Reading discipline (every new session)

Read in this order, stopping when you have enough context:

1. **INDEX.md** (always — small, free; tells you what exists + topic search)
2. **PROJECT_OVERVIEW.md** if it exists (always — small project primer)
3. **work.log tail** ~50 lines (always)
4. Active handoffs listed in INDEX (if relevant to current request)
5. Specific warm archive — only after grepping INDEX's Topic index;
   load the matched file(s) only
6. Cold digest — only on explicit user request

Lazy-load. Don't pre-fetch warm/cold "just in case". Use the Topic
index in INDEX.md to find what's relevant before opening anything.

#### 7.8 PROJECT_OVERVIEW.md — onboarding primer for new sessions/LLMs

`AIMemory/PROJECT_OVERVIEW.md` is a 1-screen briefing for any LLM
joining the project (whether a brand-new session or a different vendor's
agent stepping in). It answers: "what is this project, what's been
decided, what's still in motion?"

It is **derived** from `work.log` + cold digests, not authored from
scratch. It is rebuilt/extended whenever a cold digest is created.

Required structure:

```markdown
# Project Overview

> Onboarding for new LLMs joining this project. Read this AFTER
> AIMemory/INDEX.md (which tells you what files exist) and BEFORE
> AIMemory/work.log tail (which tells you what's happening right now).

## What is this project?
<2–4 sentences. What it does, who uses it, why it exists.>

## Tech stack
- <bullets of major techs>

## Key decisions locked in
- <decision 1> (YYYY-MM-DD, <model-id>) — <one-line rationale>
- <decision 2> ...

## Major work completed
- <YYYY-MM-DD>: <what shipped or finished>
- ...

## Active concerns
- <what's currently being worked on or blocked>

## Where to look
- Recent activity → AIMemory/work.log
- Topic-based history → AIMemory/INDEX.md (Topic index section)
- Long-term history → AIMemory/cold/digest-*.md

---
Last rebuild: YYYY-MM-DD by <model-id>
Source: cold/digest-* + open work.log + active handoffs
```

**When to rebuild**:
- After writing any cold digest (mandatory — the digest's "Decisions
  locked in" + "What happened" must propagate up)
- On user request ("regenerate the overview")
- When a major architectural decision is made (optional — adding it to
  PROJECT_OVERVIEW now saves a future session from having to dig)

**Don't over-edit**: this is a derived view. Single-line changes that
parrot a recent work.log event don't belong here. PROJECT_OVERVIEW is
for things a new LLM MUST know to be productive — small enough to
read in 60 seconds.

---

## 8. Why these rules exist

- **Continuity across sessions**: AI sessions are stateless. `work.log` is
  the persistence layer that lets a new session pick up where the last
  one stopped — sometimes minutes, sometimes weeks later.
- **Cross-model collaboration**: each model has different strengths,
  tool access, and context windows. `work.log` is the shared blackboard;
  model-named filenames + capability declarations let everyone see who
  authored what and what they could do at the time.
- **Conflict avoidance**: parallel writes to the same log file produce
  silent corruption. Designating one owner per logical agent team
  prevents interleaved events.

---

## 9. Per-LLM type — capability declaration

`<model-id>` in event headers is the type tag. But model-ids are opaque to
future readers (human or other AI). To make the log self-describing, every
session-start event MUST also declare capabilities using a **vendor-neutral
vocabulary** so any LLM can read and act on them.

### Tier classification (one of three)

| Tier | Means |
|------|-------|
| A    | Filesystem read+write AND shell exec on user's project |
| B    | Sandbox interpreter only — cannot touch user's files |
| C    | Chat only — no tools |

### Generic capability tags (use these, NOT vendor tool names)

- `filesystem-read`, `filesystem-write`, `shell-exec`
- `code-sandbox`
- `web-fetch`, `web-search`
- `image-input`, `image-output`, `multimodal-video`, `audio`

Do NOT write `Bash`, `Read`, `Edit`, `FileCreate`, `Browser`, `python` —
those are vendor-specific and confusing cross-team. Map your actual tools
to the generic tags above.

### Required body for `PROJECT_BOOTSTRAPPED` and `RE_ENGAGED`

```
### YYYY-MM-DD HH:MM | <model-id> | <PROJECT_BOOTSTRAPPED|RE_ENGAGED>
Tier: <A | B | C>
Vendor: <Anthropic | OpenAI | Google | xAI | Mistral | DeepSeek | Meta | other>
Capabilities: <comma-separated generic tags from the list above>
Strengths: <1 line>
Context: <token budget or "unknown">
Harness: <Claude Code | Cursor | Aider | ChatGPT-tools | Continue | Web chat | mobile | API direct | unknown>
Notes: <anything relevant — e.g. "preview model", "no internet", "Korean UI">
```

### Optional capability tag in WORK_START

If a task depends on a specific capability:

```
### YYYY-MM-DD HH:MM | <model-id> | WORK_START
Task: <one-liner>
Capability used: <e.g. web-search, code-sandbox, image-input>
```

This lets the next session know "the previous agent had `web-search`; I
don't — I should hand off if I need to verify a URL."

### Known LLM types — quick reference

A non-exhaustive registry. Add to it as new models appear (via NOTE event).

| model-id pattern              | Vendor    | Typical strengths                              |
|-------------------------------|-----------|------------------------------------------------|
| `claude-opus-*`               | Anthropic | long-context reasoning, code synthesis, refactors |
| `claude-sonnet-*`             | Anthropic | balanced speed/quality, daily driver           |
| `claude-haiku-*`              | Anthropic | fast searches, simple edits                    |
| `gpt-5-*`, `gpt-4o-*`         | OpenAI    | general reasoning, code interpreter            |
| `gpt-5-codex`, `o3-mini`      | OpenAI    | tight code completion, agentic coding          |
| `gemini-2-5-pro`              | Google    | long-context, multimodal, Google search        |
| `gemini-2-5-flash`            | Google    | fast multimodal                                |
| `grok-*`                      | xAI       | real-time web, irreverent                      |
| `qwen-*`, `deepseek-*`        | various   | open-weight code models                        |
| `antigravity`, `codex-gpt-*`  | (custom)  | use whatever vendor reports                    |

When a session uses a model not on this list, just declare its capabilities
in the bootstrap event — readers will infer.

---

## When in doubt

Append a `NOTE` event to `work.log` describing your uncertainty and what
you assumed. The next agent (possibly the user) will see it and can
correct course.

---

# Part II — Inter-AI Communication Protocol (AICP)

`work.log` records events. But when AI-A wants AI-B to **act** on something
— review a doc, implement from a spec, take over a task — we need a
**structured message**. AICP defines one artifact per handoff: a handoff
file in `AIMemory/`. `work.log` gets a pointer via a `HANDOFF` event.

## Handoff file naming

```
AIMemory/handoff_<topic-slug>.<authoring-model-id>.md
```

Topic-slugs: short (under ~40 chars), kebab-case.

## Message types (pick one per file)

| Type             | Purpose                                                | Expects reply? |
|------------------|--------------------------------------------------------|:--------------:|
| DECISION_RELAY   | Passing user-confirmed decisions for another AI to act | No (action)    |
| REVIEW_REQUEST   | Asking another AI to review a spec/plan/code/data      | Yes            |
| REVIEW_RESPONSE  | Returning a review (cite the REQUEST it answers)       | Optional       |
| QUESTION         | Open question that needs another AI's answer           | Yes            |
| ANSWER           | Responding to a QUESTION (cite the request)            | No             |
| BLOCKER_RAISED   | Flagging something that blocks further work            | Yes            |
| STATUS_REPORT    | Informing of work completed / current state            | No             |
| PROPOSAL         | Suggesting a new design / approach for discussion      | Optional       |

## Required header

```markdown
# <Short title>

**From**: <authoring-model-id>
**From-vendor**: <Anthropic | OpenAI | Google | …>
**To**: <target-model-id | "all" | "any-capable">
**Date**: YYYY-MM-DD HH:MM
**Type**: <one of the message types above>
**Priority**: <BLOCKING | HIGH | NORMAL | LOW>
**Reply by**: <YYYY-MM-DD | "no reply needed" | "when convenient">
**Re**: <link to triggering file or work.log timestamp, or "new topic">
**Required capability**: <e.g. "WebSearch", "Multimodal", "none">
```

`Required capability` lets the target check feasibility before accepting.
If the target lacks it, they should `BLOCKER_RAISED` rather than attempt.

Priority rules:
- **BLOCKING** — receiver cannot proceed without responding; sender should
  also ask the user to flag it
- **HIGH** — receiver should respond in their next turn if possible
- **NORMAL** — respond when convenient (default)
- **LOW** — informational; no reply expected

## Required body sections

```markdown
## Summary
<2–4 sentences, plain language>

## Context
<minimal background — link to prior files rather than restate>

## Content
<the actual payload>

## Action items
- [ ] AI-B: <imperative, specific action>

## Waiting on
<what sender is blocked by, if anything>
```

For REVIEW_REQUEST, add:

```markdown
## Review checklist (for AI-B)
- [ ] Correctness of X
- [ ] Completeness of Y
- [ ] Feasibility of Z
```

## work.log integration

When you create a handoff file, append TWO events:

```
### HH:MM | <model> | FILES_CREATED
- AIMemory/handoff_<slug>.<model>.md

### HH:MM | <model> | HANDOFF
→ <target-model>: <one-line purpose>. See handoff_<slug>.<model>.md.
Priority: <...>. Reply by: <...>.
```

Receiving AI:

```
### HH:MM | <my-model> | HANDOFF_RECEIVED
← <sender-model>: handoff_<slug>.<sender-model>.md
Acknowledged. <Acting on action items | Replying in handoff_<reply-slug>.<my-model>.md>.
```

When done:

```
### HH:MM | <my-model> | HANDOFF_CLOSED
← <sender-model>: handoff_<slug>.<sender-model>.md
Completed: <short status>. See <deliverable files, if any>.
```

Unclosed handoffs are like unfinished work — next session should check.

## Amending a handoff after sending

Don't edit after acknowledgment. Create `handoff_<slug>_v2.<model>.md`
with `Re:` pointing to v1. Typo fixes before any HANDOFF_RECEIVED are
fine in place.

---

## Quick checklist — every new turn

Before responding to a user prompt:
- [ ] Read AIMemory/work.log tail
- [ ] Check for orphan WORK_START → ask about resumption
- [ ] If proceeding, append PROMPT entry
- [ ] At session start, also append RE_ENGAGED with capabilities
- [ ] Before finishing, append WORK_END

Before creating a new markdown file:
- [ ] Is this a harness-required file? → use its required location
- [ ] Otherwise → AIMemory/{slug}.{my-model-id}.md
- [ ] Append FILES_CREATED to work.log

Before sending work to another AI:
- [ ] Create AIMemory/handoff_<slug>.<my-model>.md with full AICP header
- [ ] Append FILES_CREATED + HANDOFF events

---

## Non-goals

- Logging every Read/Grep/Edit tool call.
- Logging subagent-internal thinking.
- Heavy ceremony for trivial single-turn requests. Use judgment.

---

## Amendment

If you (any AI) want to amend this protocol, append a NOTE to work.log
proposing the change. Don't unilaterally edit PROTOCOL.md.
