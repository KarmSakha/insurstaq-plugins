# Guard Hooks: tested agents, events and schemas

InsurStaq Guard Hooks call one command, `insurstaq-cli hook <agent> <event>`, from the tool bundled
in InsurStaq.app. Agents change their hook schemas often, so each one is pinned here to the
version and the documentation it was checked against. **Doctor** (InsurStaq → Settings → Coding
Agents) runs synthetic events per agent through the real decision function and checks each
output's exact shape; a mismatch shows ✗. If an agent's input is no longer recognised (no `cwd` /
`workspace_roots`), hooks answer "unable to verify" per mode (Warn notes it; Enforce blocks a Stop
once and pauses commit/push), never a silent pass.

Checked: 11 October 2026 (schemas); live end-to-end with Claude Code: 12 October 2026.

| Agent | Version checked | Source | Events InsurStaq uses | Status |
|---|---|---|---|---|
| Claude Code | v2.1.295 (newest version named on the hooks page) | https://code.claude.com/docs/en/hooks | `SessionStart`, `PostToolUse` (`Edit\|Write\|MultiEdit`, `async`), `Stop`, `PreToolUse` | Verified against the official page |
| Cursor | 2.3.x (from a forum post; the release number could not be confirmed) | https://cursor.com/docs/hooks | `sessionStart`, `afterFileEdit`, `stop`, `beforeReadFile`, `beforeShellExecution` | **Partially verified** against the published documentation; fields come from the page's published excerpts |
| Codex CLI | `rust-v0.162.1` (source `openai/codex` at cc7ba336, 10 Oct 2026) | `codex-rs/hooks/schema/generated/*.schema.json`, `codex-rs/config/src/hook_config.rs`, `docs/config.md` in https://github.com/openai/codex | `SessionStart`, `PostToolUse` (`apply_patch`), `Stop`, `PreToolUse` (`Bash\|apply_patch`) | Verified from the Codex source |
| GitHub Copilot (VS Code) | — | — | none: MCP tools plus `copilot/.github/copilot-instructions.md` | **Advisory only** |

## Live end-to-end runs

Each hook package is also run with the real agent, headless, on a staged copy of a seeded change
case: an invoice route whose ownership check is removed (an insecure direct object reference,
`SEC-102`). The run uses project-level hook settings only (the user's own agent settings are never
written) and a fresh InsurStaq data directory, then checks what the hooks decided, what the agent
saw, the files and the Git history.

| Scenario | What must hold | Claude Code 2.1.296 (12 October 2026) | Cursor | Codex CLI |
|---|---|---|---|---|
| Doctor | every event's output has the agent's exact shape | ✓ | not run yet | not run yet |
| Observe | Stop never blocks; the change is made and recorded | ✓ | not run yet | not run yet |
| Warn | Stop never blocks; a warning is shown | ✓ | not run yet | not run yet |
| Enforce, fix | the regression is reported (right after the edit, or at Stop) with its catalog id, file and line and no code; the agent fixes it; the final Stop allows the turn; an independent Guard run on the result is not blocked | ✓ | not run yet | not run yet |
| Enforce, loop safety | the first Stop blocks and names `SEC-102`; a repeated Stop with the same regression is released with a notice; at most 3 blocks a turn | ✓ | not run yet | not run yet |
| Private path | reading a file under a private path is denied with the policy's reason; its content never reaches the agent | ✓ | not run yet | not run yet |
| Commit, Enforce | `git commit` is denied while the change is blocked; HEAD unchanged | ✓ | not run yet | not run yet |
| Push, Enforce | `git push` is denied while the change is blocked or unable to verify; the remote is unchanged | ✓ | not run yet | not run yet |

Cursor and Codex are checked against their published hooks documentation and source (table
above) and with Doctor's synthetic events; their live runs are still to come. Copilot has no
hooks to run.

## Claude Code

- Configuration: plugin `claude/hooks/hooks.json` (`${CLAUDE_PLUGIN_ROOT}`), or `settings.json`.
  Handler: `{"type": "command", "command": …, "timeout": <seconds>, "async": true?}`.
- Input read: `session_id`, `cwd`, `stop_hook_active` (Stop), `tool_name`, `tool_input.file_path` /
  `path` / `notebook_path` / `command`.
- Output:
  - Stop block: `{"decision": "block", "reason": "…"}`;
  - PreToolUse deny: `{"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny", "permissionDecisionReason": "…"}}`;
  - context: `hookSpecificOutput.additionalContext` (SessionStart, PostToolUse; capped at 10,000 characters by Claude Code; InsurStaq sends at most 1,500);
  - user message: `systemMessage`;
  - allow: `{}`.
- Claude Code's own Stop block cap is 8 consecutive blocks; InsurStaq stops at 3 per turn.
- Exit code: always 0 with JSON.

## Cursor

- Configuration: `.cursor/hooks.json` (project, trusted workspaces) or `~/.cursor/hooks.json`:
  `{"version": 1, "hooks": {"<event>": [{"command": "…", "timeout": …}]}}`. The `stop` hook takes
  `loop_limit` (InsurStaq sets 3).
- Input read: `conversation_id`, `workspace_roots`, `cwd`, `file_path`, `command`, `loop_count`.
- Output:
  - permission hooks: `{"permission": "allow" | "deny", "user_message", "agent_message"}`;
  - stop: `{"followup_message": "…"}` continues the turn. InsurStaq sends a block, and the first
    notice of a turn about a `BLOCKED` or `UNABLE_TO_VERIFY` change (Warn, "unable to verify",
    loop safety), this way; a later notice in the same turn, and a note on a passing verdict
    (limited coverage, warnings), goes as `user_message` / `agent_message` and never starts another
    turn (not yet confirmed in Cursor);
  - sessionStart / afterFileEdit: `{"additional_context": "…"}` (Cursor may not pass post-edit context to the model);
  - allow: `{"permission": "allow"}` or `{}`.
- Known gap: forum reports say only `deny` is reliably enforced. InsurStaq relies on `deny` alone.

## Codex CLI

- Configuration: `[[hooks.<Event>]]` tables in `~/.codex/config.toml` (or `hooks.json`); handler
  `type = "command"`, `command`, `timeout` (seconds). **Codex runs a new hook only after the user
  trusts it**: until then nothing is checked.
- File edits arrive as `tool_name: "apply_patch"`; InsurStaq reads the files from the patch
  envelope (`*** Update File: …`).
- Stop input: if `stop_hook_active` is absent, InsurStaq tracks the turn by `session_id` (a Stop
  within 30 minutes after a blocking Stop continues the turn): in the project's turn state when
  InsurStaq's engine answers, and in a per-session marker on the Mac when it does not (or the
  input is not recognised), so loop safety ends the turn after the same cap on both paths.
  Whether Codex shows `systemMessage` is not verified yet.
- Output:
  - Stop/PostToolUse block: `{"decision": "block", "reason": …}`;
  - PreToolUse deny: `hookSpecificOutput.permissionDecision: "deny"` with a reason;
  - allow: `{}` (Codex rejects `permissionDecision: "allow"` without `updatedInput`).

## GitHub Copilot

No hook support is assumed. The `copilot/` package configures the MCP server and an instructions
file. It is documented as advisory: an agent can skip the verification step.
