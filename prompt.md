# Bootstrap: Multi-LLM Collaboration Protocol (with Obsidian)

You are about to set up a lightweight protocol that lets multiple AI
assistants (Claude, GPT, Gemini, Codex, etc.) collaborate on this project
without losing state across sessions or stepping on each other's work.
Optionally also wire it up to Obsidian for a clean reading/querying UI.

## Prerequisite — you must be an agentic LLM

This protocol assumes you have **filesystem read+write AND shell execution**
on the user's project. Known compatible agent platforms:

| Agent platform        | Underlying model(s)                  |
|-----------------------|--------------------------------------|
| Claude Code           | Claude Opus / Sonnet / Haiku         |
| ChatGPT Codex (CLI)   | GPT-5 / GPT-5-Codex                  |
| OpenCode              | any (via provider config)            |
| Antigravity           | Gemini family                        |
| Cursor (agent mode)   | Claude / GPT / Gemini                |
| Aider                 | any (via provider config)            |
| Cline / Continue      | any (via provider config)            |
| Windsurf              | proprietary + others                 |
| Codex CLI / gemini-cli| GPT-5-Codex / Gemini-2.5-Pro         |

If you're a chat-only LLM (web ChatGPT free, Claude.ai chat without tools,
Gemini chat, etc.) without filesystem/shell access — see the
"Chat-only LLM fallback" section of [README.md](README.md). The remainder
of this prompt assumes agentic capability.

Quick self-check: try `ls -la` (or your platform's equivalent) and confirm
you see the user's project files. If yes, proceed.

## Your tasks (in order)

### Task 1 — Identify yourself

State at the top of your reply:

- **Model-id** in lowercase kebab-case. Examples:
  `claude-opus-4-5`, `claude-sonnet-4-6`, `gpt-5`, `gpt-5-codex`,
  `gemini-2-5-pro`, `gemini-2-5-flash`, `o3-mini`,
  `qwen-2-5-coder`, `deepseek-coder-v3`, `grok-4`.
  If your exact version is uncertain, use `<vendor>-unknown-<YYYY-MM>`
  and ASK the user for the precise name. Do not guess.
- **Vendor**: Anthropic / OpenAI / Google / xAI / Mistral / DeepSeek /
  Meta / Alibaba / other.
- **Harness** (the agent platform running you): Claude Code / ChatGPT
  Codex CLI / OpenCode / Antigravity / Cursor / Aider / Cline /
  Continue / Windsurf / gemini-cli / other (name it).
- **Capabilities** — pick from this **vendor-neutral** set,
  comma-separated. Map your actual tools to these generic tags
  (do NOT report vendor names like "Bash", "Read", "Edit", "Browser"):
  - `filesystem-read`, `filesystem-write`, `shell-exec` (Tier A baseline)
  - `web-fetch`, `web-search`
  - `code-sandbox` (separate from shell — e.g. Python sandbox)
  - `image-input`, `image-output`, `multimodal-video`, `audio`
  - `subagent-spawn` (e.g. Claude Code's Agent tool, OpenCode tasks)
- **Strengths** — one line, e.g. "long-context reasoning + code
  synthesis", "fast multimodal", "tight code completion".
- **Context window** in tokens if you know it.

### Task 2 — Establish the user's OS

Run a probe appropriate to your shell:
- macOS / Linux / git-bash: `uname -a`
- Windows cmd.exe: `ver`
- Windows PowerShell: `(Get-CimInstance Win32_OperatingSystem).Caption`

You need this for Task 7 (Obsidian).

### Task 3 — Create the AIMemory directory

```
mkdir -p AIMemory
```

at the project root.

### Task 4 — Write `AIMemory/PROTOCOL.md`

Content in **§A** below — write verbatim.

### Task 5 — Write `AIMemory/work.log`

Content in **§B** below — write verbatim.

### Task 6 — Append the first event

Append a `PROJECT_BOOTSTRAPPED` entry to `AIMemory/work.log` using the
template in **§C**. Include capabilities from Task 1 using the
vendor-neutral vocabulary.

### Task 7 — Obsidian integration (optional, recommended)

Follow **§E** below. Detect → ask user → install (with consent) → instruct
"Open folder as vault: AIMemory/".

### Task 8 — Behavior commitment

From this turn forward, follow the protocol on EVERY user turn. **§D**
is the non-negotiable summary.

### Task 9 — Confirm

Reply with:
- Model-id, vendor, harness, capabilities (generic tags)
- Files you created (absolute paths)
- Obsidian outcome (already installed / installed now / declined)
- One sentence: "I will follow AIMemory/PROTOCOL.md from this turn forward."

### If files already exist (re-bootstrap)

DO NOT overwrite. Read them, append a `RE_ENGAGED` event to `work.log`
with your current capabilities, and confirm.

---

## §A — Content for `AIMemory/PROTOCOL.md`

Write this file verbatim. This IS the protocol.

```markdown
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
  the agent's capabilities (see §A.8 below)

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

---

## 7. Why these rules exist

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

## 8. Per-LLM type — capability declaration

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
```

---

## §B — Content for `AIMemory/work.log`

```
# AIMemory work.log
#
# Append-only event log. Newest events at the bottom.
#
# Event grammar:
#
#   ### YYYY-MM-DD HH:MM | <model-id> | <EVENT>
#   <body, free-form>
#
# Events: PROMPT, WORK_START, WORK_END, FILES_CREATED, FILES_MODIFIED,
#         FILES_MOVED, FILES_DELETED, HANDOFF, HANDOFF_RECEIVED,
#         HANDOFF_CLOSED, NOTE, PROJECT_BOOTSTRAPPED, RE_ENGAGED
#
# Session-start events (PROJECT_BOOTSTRAPPED, RE_ENGAGED) MUST declare
# Vendor / Tools / Strengths / Context / Harness — see PROTOCOL.md §8.
# Race-free append discipline — see PROTOCOL.md §6.
#
# Read tail before every new user turn. See AIMemory/PROTOCOL.md.
# ============================================================
```

---

## §C — Your first event to append

Append this to `work.log` (replace placeholders with REAL values from
your own self-introspection):

```
### YYYY-MM-DD HH:MM | <your-model-id> | PROJECT_BOOTSTRAPPED
Vendor: <Anthropic|OpenAI|Google|xAI|Mistral|DeepSeek|Meta|other>
Harness: <Claude Code|ChatGPT Codex CLI|OpenCode|Antigravity|Cursor|Aider|Cline|Continue|Windsurf|gemini-cli|other>
Capabilities: <comma-separated generic tags — filesystem-read, filesystem-write, shell-exec, web-fetch, web-search, code-sandbox, image-input, subagent-spawn, ...>
Strengths: <one line>
Context: <token budget or "unknown">
Notes: AIMemory protocol installed. Multi-LLM collaboration enabled.
```

Concrete example (a Claude Code session bootstrapping the protocol):

```
### 2026-04-26 14:30 | claude-opus-4-5 | PROJECT_BOOTSTRAPPED
Vendor: Anthropic
Harness: Claude Code
Capabilities: filesystem-read, filesystem-write, shell-exec, web-fetch, web-search, subagent-spawn
Strengths: long-context reasoning + code synthesis + multi-step orchestration
Context: 200000
Notes: AIMemory protocol installed. Multi-LLM collaboration enabled.
```

Another example (an OpenAI Codex CLI session):

```
### 2026-04-26 15:10 | gpt-5-codex | PROJECT_BOOTSTRAPPED
Vendor: OpenAI
Harness: ChatGPT Codex CLI
Capabilities: filesystem-read, filesystem-write, shell-exec, code-sandbox
Strengths: tight code completion, agentic coding loop
Context: 200000
Notes: AIMemory protocol installed.
```

Use the actual current date+time. Be honest about capabilities — claiming
a capability you don't have will mislead future sessions and cause
handoffs to fail silently.

### Atomic-append form (use this exact pattern)

To stay race-safe (PROTOCOL.md §6), append via a single heredoc per event:

```bash
cat >> AIMemory/work.log <<'EOF'

### 2026-04-26 14:30 | claude-opus-4-5 | PROJECT_BOOTSTRAPPED
Vendor: Anthropic
Harness: Claude Code
Capabilities: filesystem-read, filesystem-write, shell-exec, web-fetch, web-search, subagent-spawn
Strengths: long-context reasoning + code synthesis + multi-step orchestration
Context: 200000
Notes: AIMemory protocol installed. Multi-LLM collaboration enabled.
EOF
```

If `flock` is available and you expect concurrent agents:

```bash
flock AIMemory/work.log -c "cat >> AIMemory/work.log <<'EOF'

### 2026-04-26 14:30 | claude-opus-4-5 | PROJECT_BOOTSTRAPPED
[...]
EOF"
```

If the project uses cloud-synced AIMemory (per §6.3), append to your own
session file under `AIMemory/sessions/` instead:

```bash
mkdir -p AIMemory/sessions
SESSION_LOG="AIMemory/sessions/$(date -u +%Y-%m-%dT%H-%M)__claude-opus-4-5__claude-code.log"
cat >> "$SESSION_LOG" <<'EOF'

### 2026-04-26 14:30 | claude-opus-4-5 | PROJECT_BOOTSTRAPPED
[...]
EOF
```

---

## §D — Behavior commitment (memorize this)

From the next user message onward, on EVERY user turn:

1. **Read work.log tail first** (last ~50 lines).
2. **Check for orphan WORK_START** — ask the user about resumption.
3. **Append a PROMPT event** with the user's verbatim message.
4. **Append WORK_START** when you begin acting (and for sessions that
   span a long gap, also `RE_ENGAGED` with capabilities at session start).
5. **Use absolute paths** in any FILES_* events.
6. **All new markdown files** go under `AIMemory/` with
   `{slug}.{your-model-id}.md` naming.
7. **Append WORK_END** when done, with status.
8. **For cross-AI handoffs**, write `handoff_*.{your-model-id}.md` per
   AICP, and log a `HANDOFF` event.

If the user explicitly opts out of logging for a trivial turn ("don't log
this, just answer X"), respect it — but the default is always log.

---

## §E — Obsidian integration (perform during step 7)

Obsidian (https://obsidian.md) is a free local markdown editor that treats
any folder as a "vault". Opening `AIMemory/` as a vault gives the user:
- Graph view of cross-references between log entries and handoff files
- Dataview queries over `work.log` (e.g. "all open WORK_STARTs by model")
- Search/backlinks/tags
- Templater-based handoff templates

This step is **optional**. Always ask the user before installing software.

### E.1 — Detect Obsidian by platform

```bash
# macOS
test -d "/Applications/Obsidian.app" && echo "INSTALLED" || echo "MISSING"

# Linux (any of the following)
which obsidian || flatpak list 2>/dev/null | grep -i obsidian || \
  snap list 2>/dev/null | grep -i obsidian || echo "MISSING"

# Windows (PowerShell or git-bash)
where.exe obsidian 2>/dev/null || \
  test -d "$LOCALAPPDATA/Obsidian" && echo "INSTALLED" || echo "MISSING"
```

### E.2 — If MISSING, ask the user

> "Obsidian is not installed. It would let you read/query the AIMemory
> folder visually. Install it now?
> - macOS: `brew install --cask obsidian` (if Homebrew is available, else
>   download from https://obsidian.md)
> - Linux: `flatpak install flathub md.obsidian.Obsidian` or AppImage
> - Windows: `winget install Obsidian.Obsidian`
>
> Reply Y / install / skip."

If the user says yes:
- macOS: prefer `brew install --cask obsidian`; fallback: print the URL
- Linux: prefer Flatpak (broadest), fallback: snap, then AppImage
- Windows: prefer `winget`, fallback: print the URL

If the user declines, just print E.3 instructions for later.

### E.3 — Open AIMemory as a vault

After install (or if already installed), instruct the user:

> "Open Obsidian → 'Open folder as vault' → select the path
> `<absolute path to project>/AIMemory`. On first open, Obsidian creates
> `.obsidian/` inside the vault for its config — that's normal.
>
> Recommended community plugins (install from Settings → Community plugins):
> - **Dataview** — query `work.log` events as a table
> - **Templater** — pre-fill new handoff files
> - **Calendar** — daily activity view
>
> Sample Dataview query (paste in any note inside the vault):
>
> ```dataview
> TABLE WITHOUT ID file.link AS \"Handoff\", From, To, Type, Priority
> FROM \"\"
> WHERE startswith(file.name, \"handoff_\")
> SORT file.mtime DESC
> ```"

### E.4 — Log the result

Append one of:

```
### HH:MM | <model> | NOTE
Obsidian: already installed; instructed user to open AIMemory/ as vault.
```

```
### HH:MM | <model> | NOTE
Obsidian: installed via <brew|winget|flatpak|...>; instructed user to open
AIMemory/ as vault.
```

```
### HH:MM | <model> | NOTE
Obsidian: install declined by user. Protocol still functional via plain
markdown.
```

---

Now execute the nine tasks. Begin.
