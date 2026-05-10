# tmux Direct Handoff Extension

This is an optional, lazy-loaded AIMemory extension. Read it only when
all conditions are true:

1. The current agent is running inside tmux (`TMUX` is non-empty or
   `tmux display-message -p '#{pane_id}'` works).
2. The user explicitly asked to enable tmux handoff (`tmux handoff on`),
   asked to deliver a handoff to a tmux pane, an AICP handoff target
   starts with `tmux-pane:`, the user asked for a tmux high-five smoke
   test, the user message is the literal phrase
   `HIGHFIVE_CONFIRMED`, or the user asked in English
   or their own language to name or rename the current pane.
3. The user has not disabled tmux handoff for this session with
   `tmux handoff off`.

If any condition is false, do not load this file. Use normal AICP
handoff files only.

## Manual activation

Treat this command, after trimming surrounding whitespace and ignoring
case, as an explicit request to load the tmux handoff extension:

```text
tmux handoff on
```

When the user says `tmux handoff on`:

1. Clear any session-local "tmux handoff disabled" flag.
2. Check whether the current agent is inside tmux (`TMUX` is non-empty or
   `tmux display-message -p '#{pane_id}'` works).
3. If not inside tmux, say that tmux handoff is unavailable in this
   session and do not read, fetch, or create `AIMemory/tmux-handoff.md`.
4. If inside tmux, read `AIMemory/tmux-handoff.md`. If it is missing and
   web fetch is available, fetch this file from the upstream
   `agent-work-mem` repository into `AIMemory/tmux-handoff.md`.
5. After loading, report that tmux handoff is ready and include the
   current pane id plus the stable pane name if one is configured.

## Manual deactivation

Treat this command, after trimming surrounding whitespace and ignoring
case, as an explicit request to disable tmux handoff for the current
agent session:

```text
tmux handoff off
```

When the user says `tmux handoff off`:

1. Set a session-local "tmux handoff disabled" flag in active reasoning.
2. Conceptually remove `AIMemory/tmux-handoff.md` from the active
   context: do not follow this file's pane lookup, paste, high-five, pane
   naming, or return-handoff rules after this point.
3. Do not run tmux checks, do not read or fetch `AIMemory/tmux-handoff.md`,
   and do not delete any cached `AIMemory/tmux-handoff.md` file.
4. Invalidate any in-memory route cache. On `tmux handoff on` again, the
   cache starts empty.
5. Process future handoffs through normal AICP only, even if the target
   mentions `tmux-pane:<name-or-id>`, until the user says
   `tmux handoff on` again.
6. Say that tmux handoff is off for this session.

## Invariants

- AICP remains the source of truth. Always create the handoff file and
  append the normal `FILES_CREATED` and `HANDOFF` events before any tmux
  delivery.
- tmux delivery is a local convenience: it pastes a prompt into another
  pane. It does not replace `AIMemory/work.log` or `handoff_*.md`.
- Do not inject work into a pane unless the user named that pane, or the
  handoff `To` field is explicitly `tmux-pane:<name-or-id>`.
- If the pane match is missing or ambiguous, stop after the normal AICP
  handoff and ask the user to pick a pane.
- Never paste secrets or hidden chain-of-thought into another pane. Send
  only the handoff path, action summary, and public instructions.
- Do not run arbitrary shell commands inside the target pane. Paste an
  agent-facing instruction and press Enter only when the target pane was
  explicitly selected for this handoff.
- Do not scan AIMemory files to determine "who I am", "which pane I am",
  or whether this pane is the source or receiver. Those facts come only
  from the pasted tmux prompt, current runtime context, and direct tmux
  current-pane inspection. AIMemory files are shared state and are not
  identity authority.
- Once `@awm_pane_name` is set, treat it as the stable routing name. Do
  not change or unset it unless the user explicitly asks to rename that
  pane.

## Pane names

Prefer stable AIMemory pane names. The user can set one with:

```bash
pane_id="${TMUX_PANE:-$(tmux display-message -p '#{pane_id}')}"
tmux select-pane -t "$pane_id" -T codex-review
```

### Rename the current pane

When the user asks to name or rename the current tmux pane, do it
directly and keep it lightweight. Do not create AICP files or write
`work.log` entries for pane naming unless the user explicitly asks to
record the UI change.

Trigger examples:

- `rename this pane to codex-review`
- `set the current pane name to codex-review`
- `name this tmux pane codex-review`
- `change the current pane name to "codex-review"`

Flow:

1. Confirm the current process is inside tmux.
2. Extract the requested pane name from English or any user-language
   phrasing. If the request contains a quoted name, use the quoted text
   exactly. Otherwise, use the final explicit name phrase. If the name is
   ambiguous, ask the user to repeat it in quotes.
3. Resolve the pane id for this agent process before renaming. Prefer
   `$TMUX_PANE`; if it is empty, ask tmux for the current pane. Do not use
   a bare `tmux display-message` result as authority when `$TMUX_PANE`
   is set, because the active tmux client can be focused on a different
   pane than the agent process. Every rename command below must target
   the resolved `pane_id` explicitly.
4. Rename that resolved pane and store the stable AIMemory pane name:

   ```bash
   name='<pane-name>'
   pane_id="${TMUX_PANE:-$(tmux display-message -p '#{pane_id}')}"
   tmux display-message -t "$pane_id" -p '#{pane_id}	#{session_name}:#{window_index}.#{pane_index}	#{pane_title}'
   tmux set-option -p -t "$pane_id" @awm_pane_name "$name"
   tmux select-pane -t "$pane_id" -T "$name"
   tmux set-option -p -t "$pane_id" allow-rename off 2>/dev/null || true
   ```

   `allow-rename off` is defensive for tmux/window rename escape
   sequences. The stable pane identity is still `@awm_pane_name`.

5. Make the rename visible in the top border. This step is **required**,
   not optional cosmetics. Agent CLIs such as Claude Code, Codex CLI, and
   OpenCode continuously rewrite `#{pane_title}` via OSC terminal title
   escape sequences (spinner glyphs, current task labels, `Ready (...)`
   states). Without this step the border will display the agent's
   transient status text instead of the stable routing name, and the
   rename will appear to revert within seconds.

   Set the window's `pane-border-format` to prefer `@awm_pane_name` over
   `#{pane_title}` so the stable name wins:

   ```bash
   window_id="$(tmux display-message -t "$pane_id" -p '#{window_id}')"
   tmux set-window-option -t "$window_id" pane-border-status top
   tmux set-window-option -t "$window_id" pane-border-format '#[bold] #{?@awm_pane_name,#{@awm_pane_name},#{pane_title}}#{?@awm_agent_kind, (#{@awm_agent_kind}),} #[default]'
   ```

   This is a window-level option, so a single call covers every pane in
   the current window. Re-run it whenever a renamed pane lands in a
   window whose border format does not yet consult `@awm_pane_name`, for
   example after a fresh tmux server start, after a layout reset, or
   after the window was created by a startup script that hard-coded
   `pane-border-format '#{pane_title}'`.

6. Confirm with the resolved pane id and stable name:

   ```bash
   tmux display-message -t "$pane_id" -p 'Pane #{pane_id} named #{?@awm_pane_name,#{@awm_pane_name},#{pane_title}}'
   ```

Stable pane names use the pane-local tmux user option `@awm_pane_name`.
This is deliberate: programs running inside a pane can change
`#{pane_title}` with terminal title escape sequences, but they should not
overwrite `@awm_pane_name`. Use `#{pane_title}` only as a fallback for
unnamed panes.

Also store the agent or harness type in the pane-local tmux user option
`@awm_agent_kind` when it is known. Keep this separate from
`@awm_pane_name`: the pane name is the human routing name (`gemini`,
`claude`, `opencode`), while the agent kind is the implementation detail
used for behavior switches (`gemini-cli`, `claude-code`, `opencode`,
`codex`). Do not overload pane names with harness details unless the user
explicitly asks.

Discover panes with:

```bash
tmux list-panes -a -F '#{pane_id}	#{session_name}:#{window_index}.#{pane_index}	#{@awm_pane_name}	#{@awm_agent_kind}	#{pane_title}	#{window_name}	#{pane_current_command}'
```

### Troubleshooting: renamed label reverts to spinner or status text

Symptom: after a successful rename, the pane border briefly shows the
new routing name and then visibly reverts within seconds to the agent's
spinner glyph, current-task phrase, or `Ready (...)` text.

Cause: the window's `pane-border-format` is still the default
`#{pane_title}` (or a prior layout's setting that does not consult
`@awm_pane_name`), so as soon as the agent CLI emits its next OSC title
escape, the border re-renders with that transient title.

`@awm_pane_name` itself is unaffected — it is a tmux user option, not a
terminal title — but the border format must be told to read from it.

Fix: re-run step 4 of the rename flow against the affected window.
Verify with:

```bash
tmux show-window-option -t "$window_id" pane-border-format
tmux list-panes -t "$window_id" -F '#{pane_id}	awm=#{@awm_pane_name}	kind=#{@awm_agent_kind}	title=#{pane_title}'
```

The window option should reference `@awm_pane_name` ahead of
`#{pane_title}`. The per-pane listing should show the stable routing
name in `awm=`, with the mutable `title=` field free to drift as the
agent CLI updates its status.

### Record agent kind

When the user identifies what agent or harness a pane is running, record
it separately from the stable pane name.

Trigger examples:

- `record this pane as gemini-cli`
- `this pane is using claude-code`
- `set agent kind to opencode`
- `제미니 클라이언트로 기록해줘`

Flow:

1. Confirm the current process is inside tmux, or use the explicit pane id
   if the user named another pane.
2. Normalize obvious agent kinds:
   - `gemini`, `gemini cli`, `gemini-cli` -> `gemini-cli`
   - `claude`, `claude code`, `claude-code` -> `claude-code`
   - `opencode`, `open code`, `deepseek` -> `opencode`
   - `codex`, `gpt`, `openai` -> `codex`
3. Store the pane-local metadata:

   ```bash
   pane_id="${TMUX_PANE:-$(tmux display-message -p '#{pane_id}')}"
   tmux set-option -p -t "$pane_id" @awm_agent_kind '<agent-kind>'
   ```

4. Keep `@awm_pane_name` unchanged unless the user also asked to rename
   the pane.

Match order:

1. Exact pane id, such as `%12`
2. Exact pane location in the current session, such as
   `session:window.pane`.
3. For a short pane name, search the current window first:
   - exact stable AIMemory pane name (`@awm_pane_name`)
   - exact pane title (`#{pane_title}`), only as a fallback for unnamed
     panes
4. Exact `window-name:pane-name`, where `pane-name` matches
   `@awm_pane_name` first, then `#{pane_title}` only for unnamed panes.
   This may match panes in any session only if it resolves to exactly one
   pane.
5. Exact `session-name:window-name:pane-name`, using the same pane-name
   rule as above. Use this when the same window and pane names appear in
   multiple sessions.
6. Exact stable AIMemory pane name across the full tmux server, only if
   the current-window search found nothing and it resolves to exactly one
   pane.
7. Exact pane title (`#{pane_title}`), only as a fallback and only if it
   resolves to exactly one pane after current-window and stable-name
   searches found nothing.
8. Exact window name, only if it resolves to one pane.

Do not fuzzy-match pane names for direct delivery.
For short names like `claude` or `gemini`, same-window matches win over
cross-window matches. Cross-window search is only a fallback when the
current window has no exact match.

### Pane name aliases

Before matching, normalize obvious agent-name aliases from the user's
language into canonical pane names. This is not fuzzy matching: only use
known aliases with a single clear canonical target, then apply the exact
match order above.

If the user's exact text matches a real pane's `@awm_pane_name` or
`#{pane_title}` in the current window, prefer that pane over alias
normalization. Aliases only apply when no exact match exists in the local
window.

Default aliases:

| User text | Canonical pane name |
|-----------|---------------------|
| `클로드`, `끌로드`, `claude code`, `anthropic` | `claude` |
| `제미니`, `gemini cli`, `google` | `gemini` |
| `코덱스`, `코드엑스`, `gpt`, `openai` | `codex` |
| `오픈코드`, `오픈 코드`, `open code` | `opencode` |
| `커서`, `cursor agent` | `cursor` |
| `에이더`, `aider` | `aider` |

Preserve explicit pane ids, `session:window.pane`,
`window-name:pane-name`, and `session-name:window-name:pane-name`
addresses exactly, except that only the final `pane-name` segment may use
a known alias. If an alias could reasonably map to more than one local
pane, ask for a quoted exact pane name instead of guessing.

### Route cache

Cache scope: the agent's current chat session only — never persisted to
disk, never shared across agents.

Do not rediscover every pane on every tmux delivery. Once a target pane
has been successfully used, keep a session-local route cache from the
user-facing target name and its canonical alias to the resolved pane id.

A route is verified when either:

- a high-five target returns `HIGHFIVE_CONFIRMED`; or
- a tmux handoff prompt was pasted and submitted to the target pane
  without tmux command failure.

Before using a cached route, re-verify that the cached pane id still
exists and that its live `@awm_pane_name` or fallback `#{pane_title}`
still exactly matches the requested target name or canonical alias:

```bash
tmux display-message -t '<cached-pane-id>' -p '#{pane_id}	#{@awm_pane_name}	#{pane_title}	#{session_name}:#{window_index}.#{pane_index}'
```

Use the cached pane id directly if the command succeeds and one of these
is true:

- the user requested that exact pane id;
- `@awm_pane_name` exactly matches the requested name or its canonical
  alias;
- the requested name is an exact `session:window.pane`; or
- the requested name is an exact `window-name:pane-name` or
  `session-name:window-name:pane-name` for this cached pane; or
- no stable name is set and `#{pane_title}` exactly matches the requested
  name or its canonical alias.

Only fall back to full pane remapping with `tmux list-panes -a ...` when
the cached pane id is missing, the live name no longer matches, or the
paste / send command fails. During remapping, check the current window
before the rest of the tmux server for short pane names. After a
successful fallback delivery, refresh the session-local route cache with
the new pane id.

## Thumb-up ASCII

Show this when a tmux handoff is accepted and again when the work is
complete:

```text
       __
      |  |
   ___|  |
  |      |
  |======|
  |======|
  |______|
```

## High-five ASCII

Show this for high-five smoke tests:

```text
   ___   _   __
  / _ \ | | / /
 | | | || |/ /
 | | | ||   <
 | |_| || |\ \
  \___/ |_| \_\
Sent by: <pane-name-or-id>
```

## High-five smoke test

This is the smallest tmux transport test. Keep it intentionally light:
do not create AICP handoff files, do not read or write `AIMemory/work.log`,
and do not create persistent files for the test. It only proves that pane
lookup, prompt injection, and return delivery work.

The confirmation phrase is the exact English token
`HIGHFIVE_CONFIRMED`. Do not translate it, localize it, abbreviate it as
"high-five response", or paraphrase it in the user's language. Natural
language around the token may be localized, but the return prompt must
start with `HIGHFIVE_CONFIRMED` on its own line.

Trigger examples:

- `send a high-five to tmux pane codex-review`
- `tmux-pane:codex-review high-five`

Sender flow:

1. Confirm the current process is inside tmux.
2. Resolve the current source pane:

   ```bash
   tmux display-message -p '#{pane_id}	#{?@awm_pane_name,#{@awm_pane_name},#{pane_title}}	#{session_name}:#{window_index}.#{pane_index}'
   ```

3. Resolve the requested target pane by the route-cache rules above. Use
   a previously verified cached pane id first. Fall back to the full
   pane-name rules only if the cached route is missing, stale, mismatched,
   or delivery fails. For natural-language requests in English or any
   user language, extract the stable pane name or pane id exactly as the
   user wrote it. If the pane name is ambiguous, ask the user to repeat it
   in quotes.
4. Load the high-five prompt into a temporary tmux buffer without writing
   a file:

   ```bash
   tmux set-buffer -b awm-highfive-<source-pane-id> "$(cat <<'EOF'
AIMemory tmux high-five test received.

Print this high-five ASCII art in your reply:

   ___   _   __
  / _ \ | | / /
 | | | || |/ /
 | | | ||   <
 | |_| || |\ \
  \___/ |_| \_\
Sent by: <source-pane-name-or-id>

Then return the test to the source pane by pasting exactly this prompt.
The first line must be the literal English token `HIGHFIVE_CONFIRMED`;
do not translate, rename, or paraphrase it:

HIGHFIVE_CONFIRMED
High-five sender: <target-pane-name-or-id>

Source pane: <source-pane-id>
Source pane name: <source-pane-name>

After pasting the return prompt, also show a tmux status-line notice on
the source pane so the human can see the confirmation even if the source
agent TUI has not redrawn its input buffer yet:

tmux display-message -t <source-pane-id> 'HIGHFIVE_CONFIRMED from <target-pane-name-or-id>'
EOF
)"
   ```

5. Paste it into the target pane and press Enter:

   ```bash
   tmux paste-buffer -t '<target-pane>' -b awm-highfive-<source-pane-id>
   tmux send-keys -t '<target-pane>' Enter
   tmux delete-buffer -b awm-highfive-<source-pane-id> 2>/dev/null || true
   ```
6. Stop after the paste and Enter. Do not poll either pane with
   `tmux capture-pane`, do not inspect the source pane for the returned
   prompt, and do not send a follow-up "please return
   HIGHFIVE_CONFIRMED" prompt. Report only that the high-five prompt was
   sent to the target pane. The user or a later inbound prompt will tell
   the source pane whether the return arrived.

Target pane behavior:

1. When the prompt says `AIMemory tmux high-five test received`, print the
   high-five ASCII art and a `Sent by: <source-pane-name-or-id>` line
   directly below it.
2. Resolve the target pane's own stable pane name or pane id. Use that as
   `High-five sender` in the return prompt.
3. Resolve the `Source pane` value exactly.
4. Paste the confirmation phrase exactly as `HIGHFIVE_CONFIRMED`, plus
   `High-five sender: <target-pane-name-or-id>` to the source pane and
   press Enter. Use a target-pane-specific temporary buffer such as
   `awm-highfive-<target-pane-id>` for this return delivery. Do not use
   localized variants such as "하이파이브 응답".
5. Also run a best-effort tmux status-line notice against the source pane:

   ```bash
   tmux display-message -t '<source-pane>' 'HIGHFIVE_CONFIRMED from <target-pane-name-or-id>' 2>/dev/null || true
   ```

   This is only a visual hint for the human. It does not replace the
   returned prompt and it must not be retried in a loop.
6. Return the confirmation at most once for each received high-five
   prompt. If the source pane later asks for the same confirmation again
   because it failed to observe the first return, do not paste
   `HIGHFIVE_CONFIRMED` a second time unless the human user explicitly
   asks for a new high-five test. Instead, state that the confirmation was
   already sent once and include the source pane id and target pane id.

Source pane behavior:

When the source pane receives `HIGHFIVE_CONFIRMED`, print the high-five
ASCII art and a `Sent by: <High-five sender>` line directly below it. Do
not create AICP files or write log entries for this smoke test.

Do not proactively check for the confirmation with `tmux capture-pane`.
If the source pane naturally receives a visible or submitted
`HIGHFIVE_CONFIRMED` prompt, handle it then. If the user asks whether a
return arrived, inspect once and report exactly what is visible. Never ask
the target pane to return the same high-five again unless the human user
explicitly requests a new high-five test.

Some terminal UIs do not redraw pasted inbound text while the source
agent is busy or while the text is sitting in the input buffer. In that
case the returned prompt may become visible only after the user presses
Enter, the current agent turn finishes, or the UI redraws. Treat this as a
terminal/TUI limitation, not as a reason to resend the confirmation. The
receiver-side `tmux display-message` notice is the preferred immediate
visibility mechanism because it is tmux UI state, not source-agent input.

## Receiver roles

tmux handoff delivery must include the requested receiver roles. Do not
paste one generic "read this handoff" prompt when the user asked the
target pane to implement, review, inspect, test, verify, or fix
something.

Infer receiver roles from the user's wording in English or their own
language. Roles can be combined. Preserve the user's requested order when
it is clear. If the wording is ambiguous, ask one short clarification
before delivery.

Use these roles:

- `IMPLEMENT`: The user asks the target pane to implement or build the
  current design or requested change, then report back.
- `REVIEW`: The user asks the target pane to review consistency,
  alignment, or correctness against the current design, requirements, or
  implementation. This is a conformance review, not an improvement pass.
- `INSPECT`: The user asks the target pane to examine the design or
  implementation and include improvement opportunities, quality concerns,
  or suggested changes. Do not edit unless this is combined with
  `IMPLEMENT` or `FIX`.
- `TEST`: The user asks the target pane to run, add, or assess tests.
- `VERIFY`: The user asks the target pane to validate behavior,
  acceptance criteria, integration, or end-to-end correctness.
- `FIX`: The user asks the target pane to modify files to correct
  confirmed issues, then rerun focused checks.
- `GENERAL_STATUS`: The user asks only for a status/report handoff or the
  request does not fit the roles above.

Examples:

- "handoff the current design to gemini pane, have it implement it, then
  send a report handoff back" -> `IMPLEMENT`
- "handoff the current design to gemini pane for consistency review, then
  send a report handoff back" -> `REVIEW`
- "ask gemini pane to inspect the current design for improvements, then
  send a report handoff back" -> `INSPECT`
- "ask gemini pane to inspect the current implementation, test, validate,
  fix confirmed issues, then hand off the result" ->
  `INSPECT+TEST+VERIFY+FIX`

Each role changes the prompt pasted into the target pane:

- `IMPLEMENT` prompt:
  "Action roles: IMPLEMENT. Implement the design or requested change in
  the handoff. Edit files as needed, run focused validation, then create a
  STATUS_REPORT handoff back to the source pane with changed files, tests,
  and any blockers."
- `REVIEW` prompt:
  "Action roles: REVIEW. Check the handoff for consistency, alignment, and
  correctness against the stated design, requirements, or implementation.
  Do not implement. Create a REVIEW_RESPONSE handoff back to the source
  pane with mismatches, risks, open questions, and residual risk."
- `INSPECT` prompt:
  "Action roles: INSPECT. Examine the handoff for improvement
  opportunities, quality issues, simplifications, and design or
  implementation concerns. Do not edit unless paired with IMPLEMENT or
  FIX. Create a REVIEW_RESPONSE handoff back to the source pane with
  improvement suggestions and risks."
- `TEST` prompt:
  "Action roles: TEST. Run, add, or assess the relevant tests requested by
  the handoff. Report commands, results, coverage, and gaps."
- `VERIFY` prompt:
  "Action roles: VERIFY. Validate the requested behavior, acceptance
  criteria, integration path, or end-to-end result. Report evidence,
  failures, and confidence."
- `FIX` prompt:
  "Action roles: FIX. Fix confirmed issues when safe, rerun focused
  checks, then create a STATUS_REPORT handoff back to the source pane with
  fixes, changed files, commands run, and remaining risk."
- `GENERAL_STATUS` prompt:
  "Action roles: GENERAL_STATUS. Follow the action items in the handoff
  and create the appropriate STATUS_REPORT, REVIEW_RESPONSE, or
  BLOCKER_RAISED handoff back to the source pane."

If multiple roles are present, compose the role instructions in order and
make sure the pasted prompt names all roles.

## Autonomy default

tmux handoff is an execution channel, not a clarification channel. The
target pane should proceed from the injected handoff and prompt without
asking the user for routine confirmation.

The receiver MUST make reasonable assumptions and execute automatically
when the handoff path, action roles, and target pane are explicit. Ask a
question only when one of these is true:

- the exact handoff file path is missing or unreadable;
- the requested action is destructive, externally visible, or requires
  credentials / purchases / sending external messages (email, Slack, IM,
  webhook calls);
- two or more incompatible interpretations would cause materially
  different file changes;
- required context is absent and cannot be recovered from AIMemory or the
  repository; or
- the target pane cannot safely complete the requested role.

If a question is not required, do not ask permission to start, do not ask
which file to open, and do not ask whether to run the assigned role.
Proceed, record assumptions in `work.log` or the response handoff, and
report the result.

Autonomy does not permit role escalation. If the injected action roles do
not include `IMPLEMENT` or `FIX`, the receiver must not edit files and
must not create or deliver a new `IMPLEMENT` / `FIX` handoff on its own.
It may recommend follow-up work in its response handoff, but that
recommendation is not approval to execute it.

## Gemini CLI compatibility mode

Some tmux peers can accept pasted prompts but fail after write-oriented
tool chains. Gemini CLI has shown this failure mode as a Google API
`INVALID_ARGUMENT` error saying the number of function response parts
must equal the number of function call parts. Treat that as a harness
tool-call state error, not as a tmux Enter, routing, or handoff content
error.

Use this compatibility mode only when the target is Gemini CLI, determined
by one of these signals:

- the target pane has `@awm_agent_kind=gemini-cli`;
- the pasted/user-visible target explicitly says `gemini-cli`;
- recent `AIMemory/work.log` entries for that pane show the
  `INVALID_ARGUMENT` function-response mismatch; or
- the user asks to avoid the repeated Gemini CLI error for that pane.

Do not apply inline compatibility mode merely because a pane is named
`gemini` if `@awm_agent_kind` says it is not Gemini CLI. If no
`@awm_agent_kind` is recorded and the pane name is exactly `gemini`, it is
acceptable to infer `gemini-cli` only as a conservative fallback after
checking the current pane context or recent work.log history.

Compatibility mode is only for non-mutating roles: `REVIEW`, `INSPECT`,
`VERIFY`, and status-only `GENERAL_STATUS`. Do not send `IMPLEMENT`,
`FIX`, or other file-mutating work to a tool-call fragile receiver unless
the user explicitly accepts the risk.

In compatibility mode:

1. The source pane still creates the normal AICP handoff file and appends
   `FILES_CREATED` and `HANDOFF`.
2. The source pane prepares an inline receiver packet in the pasted prompt.
   Include the exact task, role, expected output format, and any content
   that the fragile receiver would otherwise need to write into files.
   For review-style work, include key file excerpts or summaries,
   validation results, and the specific questions to answer. The receiver
   may still read files or run read-only inspection commands if needed,
   but the prompt should make routine file discovery unnecessary.
3. The pasted prompt must say `Gemini CLI compatibility mode: do not run
   write commands, do not edit files, do not append work.log, do not
   create handoff/report files, and do not tmux-deliver a returned
   prompt. Read-only file inspection is allowed if needed. Provide the
   exact content you want written as inline text in this pane.`
4. The receiver produces a plain text inline response. For reviews, use
   `REVIEW_RESPONSE_INLINE` with judgment, findings, evidence reviewed,
   evidence limitations, residual risk, and next action. For status-only
   outputs, use the requested inline status format.
5. If the receiver wants a file, log entry, or handoff/report content to
   be written, it must include the exact proposed content in fenced blocks
   or clearly labeled sections. The source pane is responsible for
   applying that content to disk after checking it.
6. The source pane transcribes the inline response into the normal
   `AIMemory/handoff_<topic>-report.<receiver>.md`, appends the normal
   `FILES_CREATED`, `HANDOFF`, `HANDOFF_CLOSED`, and source judgment
   `NOTE`, and records that the receiver response was source-relayed due
   to Gemini CLI compatibility mode.
7. Do not ask the fragile receiver to verify delivery with tmux capture,
   append `HANDOFF_RECEIVED`, create the report file, close the handoff,
   or tmux-deliver a returned prompt. Those actions require the exact
   write-oriented tool-call path that is failing.

## Sending flow

1. Infer receiver roles from the user's request.
2. Create `AIMemory/handoff_<topic>.<sender-model>.md` with the normal
   AICP header, the inferred receiver roles, and action items.
3. Append the normal `FILES_CREATED` and `HANDOFF` events to
   `AIMemory/work.log`.
4. Resolve the current source pane:

   ```bash
   tmux display-message -p '#{pane_id}	#{?@awm_pane_name,#{@awm_pane_name},#{pane_title}}	#{session_name}:#{window_index}.#{pane_index}'
   ```

5. Resolve the target pane by the route-cache rules above. Use a
   previously verified cached pane id first. Fall back to the full
   pane-name rules only if the cached route is missing, stale, mismatched,
   or delivery fails.
6. If the resolved target is in **Gemini CLI compatibility mode**, follow
   that section exactly instead of the normal receiver prompt below.
   Produce a self-contained inline review prompt, allow read-only
   inspection if needed, and do not ask Gemini CLI to write files, append
   logs, create handoffs, or run tmux delivery.
7. Create a local prompt file under ignored local state. Inject the exact
   handoff file path. Do not ask the receiver to search `INDEX.md`,
   `work.log`, or the filesystem to discover which handoff to open:

   ```bash
   mkdir -p .agent-work-mem
   cat > .agent-work-mem/tmux-handoff-message.txt <<'EOF'
       __
      |  |
   ___|  |
  |      |
  |======|
  |======|
  |______|

Invocation identity:
- You are the receiver of this pasted tmux prompt.
- Your current pane is the pane where this prompt was pasted.
- Do not infer your own identity from AIMemory files, INDEX.md,
  work.log, or old handoff files. Those files are shared by all agents.
- Use AIMemory files only for project context and the exact handoff
  content.

Route facts:
- Source pane: <source-pane-id-or-title>
- Source pane name: <source-pane-name-or-empty>
- Target pane: <target-pane-id-or-title>
- Target pane name: <target-pane-name-or-empty>
- Sender model: <sender-model>

AIMemory tmux handoff received.

Read AIMemory/INDEX.md, AIMemory/PROJECT_OVERVIEW.md, and the tail of
AIMemory/work.log for current project context. Then open this exact
handoff file:

  AIMemory/handoff_<topic>.<sender-model>.md

Action roles: <IMPLEMENT|REVIEW|INSPECT|TEST|VERIFY|FIX|GENERAL_STATUS>
Receiver instruction:
  <role-specific instruction from "Receiver roles">

Autonomy: do not ask routine clarification questions. Make reasonable
assumptions and execute the assigned role now unless blocked by a missing
handoff file, destructive / externally visible action, credential need, or
material ambiguity.

Accept it by appending HANDOFF_RECEIVED, then execute the receiver
instruction and the handoff action items. When complete, create the
role-appropriate STATUS_REPORT, REVIEW_RESPONSE, or BLOCKER_RAISED handoff
back to the sender, append HANDOFF_CLOSED for the original handoff, show
the thumb-up ASCII again, and tmux-deliver the report back to the source
pane if it can be resolved safely.

EOF
   ```

8. Paste it into the target pane and press Enter. This injection plus
   Enter is mandatory: after creating the AICP handoff, the sender must
   actively deliver the prompt to the target pane so the receiver starts
   the assigned role without requiring the user to repeat the command. If
   paste or send fails against a cached pane id, remap the target using
   the full pane-name rules, refresh the route cache, and retry once. Use
   a source-pane-specific buffer name based on the pane id resolved in
   step 4:

   ```bash
   tmux load-buffer -b awm-handoff-<source-pane-id> .agent-work-mem/tmux-handoff-message.txt
   tmux paste-buffer -t '<target-pane>' -b awm-handoff-<source-pane-id>
   tmux send-keys -t '<target-pane>' Enter
   tmux delete-buffer -b awm-handoff-<source-pane-id> 2>/dev/null || true
   ```

9. Verify that the receiver actually started. Check for a new
   `HANDOFF_RECEIVED` entry for the exact handoff file after a short
   delay. If it is missing, send one submit-only Enter to the same target
   pane and check once more. If it is still missing, append a
   `NEEDS_MANUAL_SUBMIT` note with the target pane and handoff path; do
   not keep pressing Enter in a loop.
10. Append a `NOTE` to `work.log` saying the AICP handoff was also
   delivered to `tmux-pane:<target-pane>`.
11. When the peer returns a handoff to this pane, run **Source follow-up
   flow** to issue and record the final judgment.

## Receiving flow

When a pane receives a tmux handoff prompt:

1. First establish invocation identity from the pasted prompt and current
   runtime context. Treat `Invocation identity` and `Route facts` in the
   pasted prompt as authoritative for this handoff. Do not use
   `AIMemory/INDEX.md`, `AIMemory/work.log`, old handoff files, or any
   shared file to decide whether this pane is the source or receiver.
2. Show the thumb-up ASCII in the agent reply.
3. Read `AIMemory/INDEX.md`, `AIMemory/PROJECT_OVERVIEW.md`, and the
   tail of `AIMemory/work.log`.
4. Open the exact handoff file path named in the prompt. Do not search
   for a handoff file unless the pasted prompt is missing the path or the
   file is absent.
5. Append `HANDOFF_RECEIVED`.
6. Execute the requested receiver roles and handoff action items
   autonomously. Do not ask the user to confirm routine steps. Use the
   Autonomy default above: make reasonable assumptions, document them,
   and proceed unless the task is blocked, destructive, externally
   visible, credential-gated, or materially ambiguous. Stay within the
   injected action roles; `REVIEW`, `INSPECT`, `TEST`, and `VERIFY` do
   not authorize implementation unless paired with `IMPLEMENT` or `FIX`.
7. Create the normal response handoff:
   - For `IMPLEMENT`, use `STATUS_REPORT` with changed files, validation,
     and remaining risk.
   - For `REVIEW`, use `REVIEW_RESPONSE` with consistency findings,
     mismatches, risks, open questions, and residual risk.
   - For `INSPECT`, use `REVIEW_RESPONSE` with improvement opportunities,
     quality concerns, suggested changes, and residual risk. If combined
     with `FIX`, use `STATUS_REPORT` after the fixes.
   - For `TEST`, use `STATUS_REPORT` with tests run or added, commands,
     results, coverage, and gaps.
   - For `VERIFY`, use `STATUS_REPORT` with validation evidence,
     acceptance criteria checked, failures, and confidence.
   - For `FIX`, use `STATUS_REPORT` with fixes made, changed files,
     checks rerun, and remaining risk.
   - For `GENERAL_STATUS`, use the response type requested by the handoff.
   - Use `BLOCKER_RAISED` if the request cannot be completed.
8. Append `FILES_CREATED`, `HANDOFF`, and `HANDOFF_CLOSED`.
9. Show the thumb-up ASCII again.
10. If still inside tmux and the source pane resolves exactly, paste a
   short report prompt back to the source pane and press Enter. This
   return injection plus Enter is mandatory: the source pane must start
   Source follow-up flow without requiring the user to manually submit the
   returned prompt.

   ```text
       __
      |  |
   ___|  |
  |      |
  |======|
  |======|
  |______|

Invocation identity:
- You are the original source pane receiving a returned handoff.
- Do not infer whether you are source or receiver from AIMemory files,
  INDEX.md, work.log, or old handoff files. Those files are shared by all
  agents.
- This pasted prompt is authoritative: run Source follow-up flow now.

Route facts:
- Source pane: <source-pane-id-or-title>
- Source pane name: <source-pane-name-or-empty>
- Returning pane: <receiver-pane-id-or-title>
- Returning pane name: <receiver-pane-name-or-empty>

AIMemory tmux handoff completed.

Read the returned handoff now:
  AIMemory/handoff_<topic>-report.<receiver-model>.md

Then review it and tell the user your judgment: accept, request changes,
ask for more validation, or mark blocked. Include the reason, evidence,
and next action. The original handoff was closed in AIMemory/work.log.
   ```

   Delivery command pattern:

   ```bash
   tmux load-buffer -b awm-handoff-return-<receiver-pane-id> .agent-work-mem/tmux-return-message.txt
   tmux paste-buffer -t '<source-pane>' -b awm-handoff-return-<receiver-pane-id>
   tmux send-keys -t '<source-pane>' Enter
   tmux delete-buffer -b awm-handoff-return-<receiver-pane-id> 2>/dev/null || true
   ```

   The full returned prompt may be pasted at most once for a given
   returned handoff path. Never re-paste the same returned prompt as a
   verification or retry mechanism. After a short delay, verify that the
   source pane started Source follow-up only by checking for the expected
   judgment `NOTE` or visible progress in `AIMemory/work.log`. Do not run
   tmux capture or other interactive diagnostics against the source pane
   as part of normal return delivery. If no progress is visible, send one
   submit-only Enter to the same source pane and check `work.log` once
   more. If it still does not start, append `NEEDS_MANUAL_SUBMIT` with
   the source pane and returned handoff path; do not keep pressing Enter
   and do not paste the full returned prompt again unless the user
   explicitly requests a new full redelivery.

If the source pane cannot be resolved, stop after the normal AICP report.
The sender will still see it through `INDEX.md` and `work.log`.

## Source follow-up flow

When the source pane receives a returned tmux handoff prompt, it must not
stop at acknowledging receipt. The source pane owns the final judgment
for the user's original handoff command.

1. First establish invocation identity from the pasted prompt and current
   runtime context. Treat the returned prompt as authoritative that this
   pane is the original source for this handoff. Do not use
   `AIMemory/INDEX.md`, `AIMemory/work.log`, old handoff files, or any
   shared file to decide whether this pane is the source or receiver.
2. Read `AIMemory/INDEX.md`, `AIMemory/PROJECT_OVERVIEW.md`, and the tail
   of `AIMemory/work.log`.
3. Open the exact returned handoff file path named in the prompt.
4. If `AIMemory/work.log` already contains a source judgment `NOTE` for
   this exact returned handoff path, treat the returned prompt as an
   idempotent duplicate. Do not re-review the same report in depth. Tell
   the user the prior judgment briefly, append at most one short duplicate
   receipt `NOTE` if useful, and stop.
5. Review the peer's result against the original request and action
   roles.
6. Tell the user the judgment in the source pane:
   - `ACCEPTED` — the peer result satisfies the request.
   - `CHANGES_REQUESTED` — the peer result is useful but needs specific
     fixes or follow-up work.
   - `NEEDS_VALIDATION` — the peer result may be right but lacks enough
     evidence, tests, or reproduction.
   - `BLOCKED` — the peer could not complete the task or the source pane
     cannot safely assess it.
7. Include the reason, the evidence reviewed, any files or commands that
   matter, and the next action.
8. Append a `NOTE` to `AIMemory/work.log` with the judgment and returned
   handoff path. If the original user request was review-only or otherwise
   did not authorize implementation, do not edit files, and do not create
   or deliver a new `IMPLEMENT` / `FIX` handoff yet. Report the proposed
   next action and wait for explicit user approval. `CHANGES_REQUESTED`
   and `NEEDS_VALIDATION` are judgments, not implementation
   authorization.
9. If the user explicitly approves follow-up implementation or fixing,
   create a new handoff rather than silently continuing the old thread.
