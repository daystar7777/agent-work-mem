# tmux Direct Handoff Extension

This is an optional, lazy-loaded AIMemory extension. Read it only when
all conditions are true:

1. The current agent is running inside tmux (`TMUX` is non-empty or
   `tmux display-message -p '#{pane_id}'` works).
2. The user explicitly asked to enable tmux handoff (`tmux handoff on`),
   asked to deliver a handoff to a tmux pane, an AICP handoff target
   starts with `tmux-pane:`, the user asked for a tmux high-five smoke
   test, the user message is the configured high-five confirmation
   phrase (`HIGHFIVE_CONFIRMED` by default), or the user asked in English
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
4. Process future handoffs through normal AICP only, even if the target
   mentions `tmux-pane:<name-or-id>`, until the user says
   `tmux handoff on` again.
5. Say that tmux handoff is off for this session.

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
- Once `@awm_pane_name` is set, treat it as the stable routing name. Do
  not change or unset it unless the user explicitly asks to rename that
  pane.

## Pane names

Prefer stable AIMemory pane names. The user can set one with:

```bash
tmux select-pane -T codex-review
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
3. Rename the current pane and store the stable AIMemory pane name:

   ```bash
   name='<pane-name>'
   pane_id="$(tmux display-message -p '#{pane_id}')"
   tmux set-option -p -t "$pane_id" @awm_pane_name "$name"
   tmux select-pane -T "$name"
   tmux set-option -p -t "$pane_id" allow-rename off 2>/dev/null || true
   ```

   `allow-rename off` is defensive for tmux/window rename escape
   sequences. The stable pane identity is still `@awm_pane_name`.

4. Ensure pane titles are visible in the top border of the current tmux
   window and prefer the stable AIMemory pane name over mutable
   `#{pane_title}`:

   ```bash
   window_id="$(tmux display-message -p '#{window_id}')"
   tmux set-window-option -t "$window_id" pane-border-status top
   tmux set-window-option -t "$window_id" pane-border-format '#[bold] #{?@awm_pane_name,#{@awm_pane_name},#{pane_title}} #[default]'
   ```

5. Confirm with the current pane id and stable name:

   ```bash
   tmux display-message -p 'Pane #{pane_id} named #{?@awm_pane_name,#{@awm_pane_name},#{pane_title}}'
   ```

Stable pane names use the pane-local tmux user option `@awm_pane_name`.
This is deliberate: programs running inside a pane can change
`#{pane_title}` with terminal title escape sequences, but they should not
overwrite `@awm_pane_name`. Use `#{pane_title}` only as a fallback for
unnamed panes.

Discover panes with:

```bash
tmux list-panes -a -F '#{pane_id}	#{session_name}:#{window_index}.#{pane_index}	#{@awm_pane_name}	#{pane_title}	#{window_name}	#{pane_current_command}'
```

Match order:

1. Exact pane id, such as `%12`
2. Exact stable AIMemory pane name (`@awm_pane_name`)
3. Exact `session:window.pane`
4. Exact pane title (`#{pane_title}`), only as a fallback
5. Exact window name, only if it resolves to one pane

Do not fuzzy-match pane names for direct delivery.

## Thumb-up ASCII

Show this when a tmux handoff is accepted and again when the work is
complete:

```text
      _
     / )
    / /
 __/ /__
/__  ___)
   | |
   | |
   |_|
```

## High-five ASCII

Show this for high-five smoke tests:

```text
   _     _
  / )   ( \
 / /     \ \
/ /       \ \
\ \       / /
 \ \     / /
  \_)   (_/
Sent by: <pane-name-or-id>
```

## High-five smoke test

This is the smallest tmux transport test. Keep it intentionally light:
do not create AICP handoff files, do not read or write `AIMemory/work.log`,
and do not create persistent files for the test. It only proves that pane
lookup, prompt injection, and return delivery work.

Trigger examples:

- `send a high-five to tmux pane codex-review`
- `tmux-pane:codex-review high-five`

Sender flow:

1. Confirm the current process is inside tmux.
2. Resolve the current source pane:

   ```bash
   tmux display-message -p '#{pane_id}	#{?@awm_pane_name,#{@awm_pane_name},#{pane_title}}	#{session_name}:#{window_index}.#{pane_index}'
   ```

3. Resolve the requested target pane by the pane-name rules above. For
   natural-language requests in English or any user language, extract the
   stable pane name or pane id exactly as the user wrote it. If the pane
   name is ambiguous, ask the user to repeat it in quotes.
4. Load the high-five prompt into a temporary tmux buffer without writing
   a file:

   ```bash
   tmux set-buffer -b awm-highfive "$(cat <<'EOF'
AIMemory tmux high-five test received.

Print this high-five ASCII art in your reply:

   _     _
  / )   ( \
 / /     \ \
/ /       \ \
\ \       / /
 \ \     / /
  \_)   (_/
Sent by: <source-pane-name-or-id>

Then return the test to the source pane by pasting exactly this prompt:

<confirmation phrase, default HIGHFIVE_CONFIRMED>
High-five sender: <target-pane-name-or-id>

Source pane: <source-pane-id>
Source pane name: <source-pane-name>
EOF
)"
   ```

5. Paste it into the target pane and press Enter:

   ```bash
   tmux paste-buffer -t '<target-pane>' -b awm-highfive
   tmux send-keys -t '<target-pane>' Enter
   tmux delete-buffer -b awm-highfive 2>/dev/null || true
   ```

Target pane behavior:

1. When the prompt says `AIMemory tmux high-five test received`, print the
   high-five ASCII art and a `Sent by: <source-pane-name-or-id>` line
   directly below it.
2. Resolve the target pane's own stable pane name or pane id. Use that as
   `High-five sender` in the return prompt.
3. Resolve the `Source pane` value exactly.
4. Paste the configured confirmation phrase plus `High-five sender:
   <target-pane-name-or-id>` to the source pane and press Enter.

Source pane behavior:

When the source pane receives the configured confirmation phrase, print
the high-five ASCII art and a `Sent by: <High-five sender>` line directly
below it. Do not create AICP files or write log entries for this smoke
test.

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

5. Resolve the target pane by the rules above.
6. Create a local prompt file under ignored local state:

   ```bash
   mkdir -p .agent-work-mem
   cat > .agent-work-mem/tmux-handoff-message.txt <<'EOF'
      _
     / )
    / /
 __/ /__
/__  ___)
   | |
   | |
   |_|

AIMemory tmux handoff received.

Read AIMemory/INDEX.md, AIMemory/PROJECT_OVERVIEW.md, and the tail of
AIMemory/work.log. Then open:

  AIMemory/handoff_<topic>.<sender-model>.md

Action roles: <IMPLEMENT|REVIEW|INSPECT|TEST|VERIFY|FIX|GENERAL_STATUS>
Receiver instruction:
  <role-specific instruction from "Receiver roles">

Accept it by appending HANDOFF_RECEIVED, then execute the receiver
instruction and the handoff action items. When complete, create the
role-appropriate STATUS_REPORT, REVIEW_RESPONSE, or BLOCKER_RAISED handoff
back to the sender, append HANDOFF_CLOSED for the original handoff, show
the thumb-up ASCII again, and tmux-deliver the report back to the source
pane if it can be resolved safely.

Source pane: <source-pane-id-or-title>
Sender model: <sender-model>
EOF
   ```

7. Paste it into the target pane:

   ```bash
   tmux load-buffer -b awm-handoff .agent-work-mem/tmux-handoff-message.txt
   tmux paste-buffer -t '<target-pane>' -b awm-handoff
   tmux send-keys -t '<target-pane>' Enter
   tmux delete-buffer -b awm-handoff 2>/dev/null || true
   ```

8. Append a `NOTE` to `work.log` saying the AICP handoff was also
   delivered to `tmux-pane:<target-pane>`.

## Receiving flow

When a pane receives a tmux handoff prompt:

1. Show the thumb-up ASCII in the agent reply.
2. Read the normal AIMemory entry points.
3. Open the handoff file named in the prompt.
4. Append `HANDOFF_RECEIVED`.
5. Execute the requested receiver roles and handoff action items.
6. Create the normal response handoff:
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
7. Append `FILES_CREATED`, `HANDOFF`, and `HANDOFF_CLOSED`.
8. Show the thumb-up ASCII again.
9. If still inside tmux and the source pane resolves exactly, paste a
   short report prompt back to the source pane:

   ```text
      _
     / )
    / /
 __/ /__
/__  ___)
   | |
   | |
   |_|

AIMemory tmux handoff completed.

Read the returned handoff:
  AIMemory/handoff_<topic>-report.<receiver-model>.md

The original handoff was closed in AIMemory/work.log.
   ```

If the source pane cannot be resolved, stop after the normal AICP report.
The sender will still see it through `INDEX.md` and `work.log`.
