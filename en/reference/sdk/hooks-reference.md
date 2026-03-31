> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Hooks reference

> Full reference for every hook event, its input payload, output schema, and the effect each exit code has on Claude's behavior.

Hooks are shell commands, HTTP endpoints, LLM prompts, or in-process callbacks that fire at defined points in Claude's agentic loop. They let you inject logic before or after tool calls, intercept permission requests, react to session lifecycle events, and more.

## Configuration

Hooks are configured in any Claude Code settings file. The top-level `hooks` key maps event names to an array of matcher objects.

```json  theme={null}
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to run bash command' >&2"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "/usr/local/bin/lint-changed-file '$TOOL_INPUT'"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude finished'"
          }
        ]
      }
    ]
  }
}
```

### Settings file locations

| Scope   | Path                          | Priority |
| ------- | ----------------------------- | -------- |
| User    | `~/.claude/settings.json`     | Low      |
| Project | `.claude/settings.json`       | Medium   |
| Local   | `.claude/settings.local.json` | High     |

Higher-priority settings files take precedence. All hooks across all files run; they are not overridden.

### Matcher configuration

<ResponseField name="matcher" type="string">
  A string pattern that filters when these hooks run. The field matched depends on the event:

  * `PreToolUse` / `PostToolUse` / `PostToolUseFailure` / `PermissionRequest` / `PermissionDenied` — matched against `tool_name`
  * `Notification` — matched against `notification_type`
  * `SessionStart` — matched against `source` (`startup`, `resume`, `clear`, `compact`)
  * `Setup` — matched against `trigger` (`init`, `maintenance`)
  * `SubagentStart` / `SubagentStop` — matched against `agent_type`
  * `PreCompact` / `PostCompact` — matched against `trigger` (`manual`, `auto`)
  * `StopFailure` — matched against `error`
  * `ConfigChange` — matched against `source`
  * `InstructionsLoaded` — matched against `load_reason`
  * `Elicitation` / `ElicitationResult` — matched against `mcp_server_name`
  * `FileChanged` — matched against filenames (e.g., `".envrc|.env"`)

  Omit `matcher` to run the hook for all instances of the event.
</ResponseField>

<ResponseField name="hooks" type="HookCommand[]" required>
  One or more hook definitions to execute when the matcher fires.
</ResponseField>

***

## Hook types

### `command` — shell command

```json  theme={null}
{
  "type": "command",
  "command": "jq '.tool_name' && my-validator",
  "timeout": 30,
  "shell": "bash",
  "async": false,
  "once": false,
  "if": "Bash(git *)",
  "statusMessage": "Validating command..."
}
```

The hook input JSON is piped to the command's stdin. The `CLAUDE_ENV_FILE` environment variable is set for `CwdChanged` and `FileChanged` hooks — write `export KEY=value` lines there to inject environment variables into subsequent Bash tool calls.

<ResponseField name="command" type="string" required>
  Shell command to execute.
</ResponseField>

<ResponseField name="timeout" type="number">
  Timeout in seconds. Defaults to the global hook timeout (60 s).
</ResponseField>

<ResponseField name="shell" type="'bash' | 'powershell'">
  Shell interpreter. `bash` uses your `$SHELL` (bash/zsh/sh). Defaults to `bash`.
</ResponseField>

<ResponseField name="async" type="boolean">
  When `true`, the hook runs in the background without blocking Claude. Output is ignored.
</ResponseField>

<ResponseField name="asyncRewake" type="boolean">
  When `true`, the hook runs in the background but wakes the model if it exits with code 2. Implies `async: true`.
</ResponseField>

<ResponseField name="once" type="boolean">
  When `true`, the hook runs once and is removed from the configuration after execution.
</ResponseField>

<ResponseField name="if" type="string">
  Permission rule syntax (e.g., `"Bash(git *)"`) evaluated against the hook input. The hook is skipped if the condition does not match. Avoids spawning processes for non-matching tool calls.
</ResponseField>

<ResponseField name="statusMessage" type="string">
  Custom message shown in the spinner while the hook runs.
</ResponseField>

### `prompt` — LLM evaluation

```json  theme={null}
{
  "type": "prompt",
  "prompt": "Check whether this bash command is safe: $ARGUMENTS",
  "model": "claude-haiku-4-5",
  "timeout": 30
}
```

The `$ARGUMENTS` placeholder is replaced with the hook input JSON. The model's response is treated as the hook output.

<ResponseField name="prompt" type="string" required>
  Prompt sent to the model. Use `$ARGUMENTS` to embed the hook input.
</ResponseField>

<ResponseField name="model" type="string">
  Model to use. Defaults to the small fast model.
</ResponseField>

### `agent` — agentic verifier

```json  theme={null}
{
  "type": "agent",
  "prompt": "Verify that unit tests ran and passed.",
  "model": "claude-haiku-4-5",
  "timeout": 120
}
```

Runs a short agentic loop that can call tools to verify the action. Use for `PostToolUse` hooks where you want the verifier to read files or run commands.

<ResponseField name="prompt" type="string" required>
  Verification prompt. Use `$ARGUMENTS` to embed the hook input.
</ResponseField>

### `http` — HTTP endpoint

```json  theme={null}
{
  "type": "http",
  "url": "https://my-server.example.com/hook",
  "headers": {
    "Authorization": "Bearer $MY_TOKEN"
  },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 10
}
```

POSTs the hook input JSON to the given URL. Header values can reference environment variables using `$VAR_NAME` syntax, but only variables listed in `allowedEnvVars` are interpolated.

<ResponseField name="url" type="string" required>
  URL to POST the hook input JSON to.
</ResponseField>

<ResponseField name="headers" type="Record<string, string>">
  Additional request headers.
</ResponseField>

<ResponseField name="allowedEnvVars" type="string[]">
  Environment variable names that may be interpolated in header values.
</ResponseField>

***

## Base hook input

Every hook receives a JSON object on stdin with these fields present for all event types.

<ResponseField name="hook_event_name" type="string" required>
  The event that fired (e.g., `"PreToolUse"`).
</ResponseField>

<ResponseField name="session_id" type="string" required>
  Current session identifier.
</ResponseField>

<ResponseField name="transcript_path" type="string" required>
  Absolute path to the JSONL transcript file for this session.
</ResponseField>

<ResponseField name="cwd" type="string" required>
  Current working directory at the time the hook fired.
</ResponseField>

<ResponseField name="permission_mode" type="string">
  Active permission mode (`"default"`, `"acceptEdits"`, `"bypassPermissions"`, `"plan"`, `"dontAsk"`).
</ResponseField>

<ResponseField name="agent_id" type="string">
  Subagent identifier. Present only when the hook fires from within a subagent. Use this field to distinguish subagent calls from main-thread calls.
</ResponseField>

<ResponseField name="agent_type" type="string">
  Agent type name (e.g., `"general-purpose"`, `"code-reviewer"`). Present when the hook fires from a subagent, or on the main thread of a session started with `--agent`.
</ResponseField>

***

## Sync hook output (JSON on stdout)

For blocking hooks, write a JSON object to stdout before exiting. The schema is the same for all events, with event-specific fields nested under `hookSpecificOutput`.

```json  theme={null}
{
  "continue": true,
  "suppressOutput": false,
  "decision": "approve",
  "reason": "Command looks safe",
  "systemMessage": "The hook approved this action.",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "additionalContext": "Verified by security scanner."
  }
}
```

<ResponseField name="continue" type="boolean">
  When `false`, Claude stops the current turn immediately.
</ResponseField>

<ResponseField name="suppressOutput" type="boolean">
  When `true`, the hook's stdout is not shown in transcript mode.
</ResponseField>

<ResponseField name="decision" type="'approve' | 'block'">
  Explicit approve/block decision. Takes effect only when the CLI reads it.
</ResponseField>

<ResponseField name="systemMessage" type="string">
  Message injected into Claude's context as a system turn.
</ResponseField>

<ResponseField name="reason" type="string">
  Human-readable reason for the decision. Shown to the user when a hook blocks an action.
</ResponseField>

<ResponseField name="hookSpecificOutput" type="object">
  Event-specific output. See each event section below for the allowed fields.
</ResponseField>

***

## Hook events

### `PreToolUse`

Fires immediately before a tool executes. You can inspect the tool input, approve or block the call, or modify the input before it reaches the tool.

**When it fires:** Before every tool invocation.

**Input fields:**

<ResponseField name="tool_name" type="string">
  Name of the tool about to run (e.g., `"Bash"`, `"Write"`, `"mcp__myserver__my_tool"`).
</ResponseField>

<ResponseField name="tool_input" type="unknown">
  The raw tool input object as Claude submitted it.
</ResponseField>

<ResponseField name="tool_use_id" type="string">
  Unique ID for this tool invocation.
</ResponseField>

**Exit codes:**

| Exit code | Effect                                                      |
| --------- | ----------------------------------------------------------- |
| `0`       | Stdout/stderr not shown. Hook output JSON applied if valid. |
| `2`       | Stderr shown to Claude; tool call is **blocked**.           |
| Other     | Stderr shown to user only; tool call continues.             |

**`hookSpecificOutput` fields:**

<ResponseField name="hookEventName" type="literal: 'PreToolUse'" required>
  Must be `"PreToolUse"`.
</ResponseField>

<ResponseField name="permissionDecision" type="'allow' | 'deny' | 'ask'">
  Override the permission decision for this tool call. `"allow"` approves the call; `"deny"` blocks it; `"ask"` forces the permission dialog.
</ResponseField>

<ResponseField name="permissionDecisionReason" type="string">
  Reason string shown to the user when the decision is `"deny"` or `"ask"`.
</ResponseField>

<ResponseField name="updatedInput" type="Record<string, unknown>">
  Replacement tool input. When provided, the tool receives this object instead of Claude's original input.
</ResponseField>

<ResponseField name="additionalContext" type="string">
  Text injected into Claude's context for this turn.
</ResponseField>

***

### `PostToolUse`

Fires after a tool completes successfully. You can observe the tool output or inject context for Claude to act on.

**When it fires:** After every successful tool execution.

**Input fields:**

<ResponseField name="tool_name" type="string">Tool that ran.</ResponseField>
<ResponseField name="tool_input" type="unknown">Input that was passed to the tool.</ResponseField>
<ResponseField name="tool_response" type="unknown">The tool's output.</ResponseField>
<ResponseField name="tool_use_id" type="string">Unique ID for this invocation.</ResponseField>

**Exit codes:**

| Exit code | Effect                                                  |
| --------- | ------------------------------------------------------- |
| `0`       | Stdout shown in transcript mode (Ctrl+O).               |
| `2`       | Stderr shown to Claude immediately as a system message. |
| Other     | Stderr shown to user only.                              |

**`hookSpecificOutput` fields:**

<ResponseField name="additionalContext" type="string">
  Context injected into Claude's conversation after the tool result.
</ResponseField>

<ResponseField name="updatedMCPToolOutput" type="unknown">
  Replacement for the MCP tool's output. Only effective for MCP tool calls.
</ResponseField>

***

### `PostToolUseFailure`

Fires when a tool call ends in an error or is interrupted.

**When it fires:** When a tool throws or is aborted.

**Input fields:**

<ResponseField name="tool_name" type="string">Tool that failed.</ResponseField>
<ResponseField name="tool_input" type="unknown">Input that was passed to the tool.</ResponseField>
<ResponseField name="tool_use_id" type="string">Unique ID for this invocation.</ResponseField>
<ResponseField name="error" type="string">Error message from the tool.</ResponseField>
<ResponseField name="is_interrupt" type="boolean">Whether the failure was caused by an interrupt signal.</ResponseField>

**Exit codes:** Same as `PostToolUse`. Hook output and exit codes are logged but do not affect the failed tool result.

***

### `PermissionRequest`

Fires when a permission dialog would be shown to the user. Hooks can programmatically approve or deny without showing any UI.

**When it fires:** When Claude requests permission for a tool call and the default behavior is to prompt.

**Input fields:**

<ResponseField name="tool_name" type="string">Tool requesting permission.</ResponseField>
<ResponseField name="tool_input" type="unknown">Input the tool would receive if approved.</ResponseField>
<ResponseField name="permission_suggestions" type="PermissionUpdate[]">Suggested permission rules (allow/deny) that the UI would offer.</ResponseField>

**Exit codes:**

| Exit code | Effect                                                                                                   |
| --------- | -------------------------------------------------------------------------------------------------------- |
| `0`       | Hook decision applied if `hookSpecificOutput.decision` is set; otherwise falls through to normal dialog. |
| Other     | Stderr shown to user; falls through to normal dialog.                                                    |

**`hookSpecificOutput` fields:**

<ResponseField name="hookEventName" type="literal: 'PermissionRequest'" required>
  Must be `"PermissionRequest"`.
</ResponseField>

<ResponseField name="decision" type="object">
  The approval or denial decision.

  <Expandable title="Allow decision">
    ```json  theme={null}
    {
      "behavior": "allow",
      "updatedInput": { ... },
      "updatedPermissions": [...]
    }
    ```
  </Expandable>

  <Expandable title="Deny decision">
    ```json  theme={null}
    {
      "behavior": "deny",
      "message": "Blocked by security policy.",
      "interrupt": false
    }
    ```

    When `interrupt` is `true`, the current turn is aborted after denial.
  </Expandable>
</ResponseField>

***

### `PermissionDenied`

Fires when a tool call is denied (by rules, mode, or classifier). You can instruct Claude to retry the action.

**When it fires:** After every permission denial.

**Input fields:**

<ResponseField name="tool_name" type="string">Denied tool.</ResponseField>
<ResponseField name="tool_input" type="unknown">Input that was denied.</ResponseField>
<ResponseField name="tool_use_id" type="string">Unique ID for this invocation.</ResponseField>
<ResponseField name="reason" type="string">Human-readable denial reason.</ResponseField>

**Exit codes:**

| Exit code | Effect                           |
| --------- | -------------------------------- |
| `0`       | Stdout shown in transcript mode. |
| Other     | Stderr shown to user only.       |

**`hookSpecificOutput` fields:**

<ResponseField name="retry" type="boolean">
  When `true`, Claude is told it may retry the denied action.
</ResponseField>

***

### `Stop`

Fires just before Claude concludes its response for the current turn.

**When it fires:** When the model is about to stop and return control to the user.

**Input fields:**

<ResponseField name="stop_hook_active" type="boolean">
  Whether a Stop hook is currently running (prevents infinite loops if your Stop hook itself would trigger a Stop).
</ResponseField>

<ResponseField name="last_assistant_message" type="string">
  Text content of the last assistant message before stopping. Saves you from parsing the transcript file.
</ResponseField>

**Exit codes:**

| Exit code | Effect                                                                      |
| --------- | --------------------------------------------------------------------------- |
| `0`       | Stdout/stderr not shown.                                                    |
| `2`       | Stderr injected as a system message; Claude **continues** the conversation. |
| Other     | Stderr shown to user only; Claude stops.                                    |

<Tip>
  Use exit code 2 from a Stop hook to check Claude's output and keep the conversation going if a condition is unmet — for example, if tests are still failing.
</Tip>

***

### `StopFailure`

Fires instead of `Stop` when the turn ends due to an API error.

**When it fires:** When a rate limit, authentication failure, or other API error ends the turn.

**Input fields:**

<ResponseField name="error" type="'authentication_failed' | 'billing_error' | 'rate_limit' | 'invalid_request' | 'server_error' | 'unknown' | 'max_output_tokens'">
  The error category.
</ResponseField>

<ResponseField name="error_details" type="string">
  Detailed error message.
</ResponseField>

<ResponseField name="last_assistant_message" type="string">
  Last assistant message text, if any was produced before the error.
</ResponseField>

**Behavior:** Fire-and-forget. Hook output and exit codes are ignored.

***

### `SubagentStart`

Fires when Claude spawns a subagent via the `Agent` tool.

**When it fires:** When an Agent tool call begins.

**Input fields:**

<ResponseField name="agent_id" type="string">Unique ID for this subagent instance.</ResponseField>
<ResponseField name="agent_type" type="string">Agent type name (e.g., `"general-purpose"`).</ResponseField>

**Exit codes:**

| Exit code | Effect                                   |
| --------- | ---------------------------------------- |
| `0`       | Stdout shown to the subagent as context. |
| Other     | Stderr shown to user only.               |

**`hookSpecificOutput` fields:**

<ResponseField name="additionalContext" type="string">
  Context injected into the subagent's conversation at the start.
</ResponseField>

***

### `SubagentStop`

Fires just before a subagent concludes its response. Mirrors `Stop` but for subagents.

**When it fires:** When a subagent is about to return its result to the parent.

**Input fields:**

<ResponseField name="agent_id" type="string">Subagent instance ID.</ResponseField>
<ResponseField name="agent_type" type="string">Agent type name.</ResponseField>
<ResponseField name="agent_transcript_path" type="string">Path to the subagent's JSONL transcript.</ResponseField>
<ResponseField name="stop_hook_active" type="boolean">Whether a SubagentStop hook is already running.</ResponseField>
<ResponseField name="last_assistant_message" type="string">Last message from the subagent.</ResponseField>

**Exit codes:**

| Exit code | Effect                                                    |
| --------- | --------------------------------------------------------- |
| `0`       | Stdout/stderr not shown.                                  |
| `2`       | Stderr shown to subagent; subagent **continues** running. |
| Other     | Stderr shown to user only; subagent stops.                |

***

### `SessionStart`

Fires when a session begins. Use this to inject initial context or set up the environment.

**When it fires:** On session startup, resume, clear (`/clear`), or after compaction.

**Input fields:**

<ResponseField name="source" type="'startup' | 'resume' | 'clear' | 'compact'">
  What triggered the session start.
</ResponseField>

<ResponseField name="model" type="string">Active model for the session.</ResponseField>

**Exit codes:**

| Exit code | Effect                                     |
| --------- | ------------------------------------------ |
| `0`       | Stdout shown to Claude as initial context. |
| Other     | Stderr shown to user only.                 |

**`hookSpecificOutput` fields:**

<ResponseField name="additionalContext" type="string">
  Context injected into Claude's system prompt for this session.
</ResponseField>

<ResponseField name="initialUserMessage" type="string">
  Auto-submitted as the first user message of the session.
</ResponseField>

<ResponseField name="watchPaths" type="string[]">
  Absolute file paths to register with the `FileChanged` watcher.
</ResponseField>

***

### `SessionEnd`

Fires when a session is about to end.

**When it fires:** On clear, logout, prompt-input exit, or other termination reasons.

**Input fields:**

<ResponseField name="reason" type="'clear' | 'resume' | 'logout' | 'prompt_input_exit' | 'other' | 'bypass_permissions_disabled'">
  The reason the session is ending.
</ResponseField>

**Exit codes:** Exit code 0 completes successfully. Other exit codes show stderr to the user.

***

### `Setup`

Fires during repository initialization and maintenance checks.

**When it fires:** On `init` (first time Claude Code runs in a directory) or `maintenance` (periodic checks).

**Input fields:**

<ResponseField name="trigger" type="'init' | 'maintenance'">
  What triggered the setup hook.
</ResponseField>

**Exit codes:**

| Exit code | Effect                                                  |
| --------- | ------------------------------------------------------- |
| `0`       | Stdout shown to Claude.                                 |
| Other     | Stderr shown to user only. Blocking errors are ignored. |

**`hookSpecificOutput` fields:**

<ResponseField name="additionalContext" type="string">
  Context provided to Claude for the setup phase.
</ResponseField>

***

### `PreCompact`

Fires before context compaction begins.

**When it fires:** Before the compaction summary is generated, whether triggered manually (`/compact`) or automatically.

**Input fields:**

<ResponseField name="trigger" type="'manual' | 'auto'">Whether compaction was requested by the user or triggered automatically.</ResponseField>
<ResponseField name="custom_instructions" type="string | null">Any custom compaction instructions already configured.</ResponseField>

**Exit codes:**

| Exit code | Effect                                             |
| --------- | -------------------------------------------------- |
| `0`       | Stdout appended as custom compaction instructions. |
| `2`       | Compaction is **blocked**.                         |
| Other     | Stderr shown to user; compaction continues.        |

***

### `PostCompact`

Fires after compaction completes.

**When it fires:** After the compaction summary has been generated and applied.

**Input fields:**

<ResponseField name="trigger" type="'manual' | 'auto'">How compaction was triggered.</ResponseField>
<ResponseField name="compact_summary" type="string">The summary produced by compaction.</ResponseField>

**Exit codes:** Exit code 0 shows stdout to the user. Other exit codes show stderr to the user.

***

### `UserPromptSubmit`

Fires when the user submits a prompt, before Claude processes it.

**When it fires:** Each time you press Enter with a message in the terminal.

**Input fields:**

<ResponseField name="prompt" type="string">The raw prompt text the user submitted.</ResponseField>

**Exit codes:**

| Exit code | Effect                                                                |
| --------- | --------------------------------------------------------------------- |
| `0`       | Stdout shown to Claude as additional context.                         |
| `2`       | Processing **blocked**; original prompt erased; stderr shown to user. |
| Other     | Stderr shown to user only.                                            |

**`hookSpecificOutput` fields:**

<ResponseField name="additionalContext" type="string">
  Context appended to Claude's view of the user message.
</ResponseField>

***

### `Notification`

Fires when Claude Code sends a notification (e.g., permission prompts, idle alerts).

**When it fires:** When a notification event is raised internally.

**Input fields:**

<ResponseField name="message" type="string">Notification message text.</ResponseField>
<ResponseField name="title" type="string">Notification title.</ResponseField>

<ResponseField name="notification_type" type="'permission_prompt' | 'idle_prompt' | 'auth_success' | 'elicitation_dialog' | 'elicitation_complete' | 'elicitation_response'">
  The type of notification.
</ResponseField>

**Exit codes:** Exit code 0 produces no output. Other exit codes show stderr to the user.

***

### `Elicitation`

Fires when an MCP server requests user input. Hooks can auto-respond without showing the dialog.

**When it fires:** When an MCP server sends an elicitation request (a structured input form or URL).

**Input fields:**

<ResponseField name="mcp_server_name" type="string">Name of the MCP server requesting input.</ResponseField>
<ResponseField name="message" type="string">The prompt message from the server.</ResponseField>
<ResponseField name="mode" type="'form' | 'url'">Input mode.</ResponseField>
<ResponseField name="elicitation_id" type="string">Request identifier.</ResponseField>
<ResponseField name="requested_schema" type="Record<string, unknown>">JSON schema describing the expected input structure.</ResponseField>

**Exit codes:**

| Exit code | Effect                                                                        |
| --------- | ----------------------------------------------------------------------------- |
| `0`       | Use hook response if `hookSpecificOutput` is provided; otherwise show dialog. |
| `2`       | Deny the elicitation.                                                         |
| Other     | Stderr shown to user only.                                                    |

**`hookSpecificOutput` fields:**

<ResponseField name="action" type="'accept' | 'decline' | 'cancel'">
  The programmatic response to the elicitation.
</ResponseField>

<ResponseField name="content" type="Record<string, unknown>">
  Form data to submit when `action` is `"accept"`.
</ResponseField>

***

### `ElicitationResult`

Fires after a user responds to an MCP elicitation. Hooks can observe or override the response.

**When it fires:** After the user (or an Elicitation hook) responds to the elicitation.

**Input fields:**

<ResponseField name="mcp_server_name" type="string">Name of the MCP server.</ResponseField>
<ResponseField name="elicitation_id" type="string">Request identifier.</ResponseField>
<ResponseField name="action" type="'accept' | 'decline' | 'cancel'">How the user responded.</ResponseField>
<ResponseField name="content" type="Record<string, unknown>">Submitted form data, if accepted.</ResponseField>

**`hookSpecificOutput` fields:**

<ResponseField name="action" type="'accept' | 'decline' | 'cancel'">
  Override the user's action before it is sent to the server.
</ResponseField>

<ResponseField name="content" type="Record<string, unknown>">
  Override the submitted content.
</ResponseField>

***

### `ConfigChange`

Fires when a settings file changes during a session.

**When it fires:** When `user_settings`, `project_settings`, `local_settings`, `policy_settings`, or `skills` files are modified on disk.

**Input fields:**

<ResponseField name="source" type="'user_settings' | 'project_settings' | 'local_settings' | 'policy_settings' | 'skills'">
  Which settings source changed.
</ResponseField>

<ResponseField name="file_path" type="string">Absolute path to the changed file.</ResponseField>

**Exit codes:**

| Exit code | Effect                                                  |
| --------- | ------------------------------------------------------- |
| `0`       | Allow the change to be applied.                         |
| `2`       | **Block** the change from being applied to the session. |
| Other     | Stderr shown to user only.                              |

***

### `InstructionsLoaded`

Fires when a CLAUDE.md or instruction rule file is loaded. This event is observability-only.

**When it fires:** When any instruction file is loaded into context.

**Input fields:**

<ResponseField name="file_path" type="string">Path to the loaded file.</ResponseField>
<ResponseField name="memory_type" type="'User' | 'Project' | 'Local' | 'Managed'">The memory scope of the file.</ResponseField>
<ResponseField name="load_reason" type="'session_start' | 'nested_traversal' | 'path_glob_match' | 'include' | 'compact'">Why the file was loaded.</ResponseField>
<ResponseField name="globs" type="string[]">The `paths:` frontmatter patterns that matched (if `load_reason` is `path_glob_match`).</ResponseField>
<ResponseField name="trigger_file_path" type="string">The file Claude accessed that caused this load (for `path_glob_match`).</ResponseField>
<ResponseField name="parent_file_path" type="string">The file that `@include`d this one (for `include`).</ResponseField>

**Behavior:** Blocking not supported. Exit code 0 completes normally. Other exit codes show stderr to the user.

***

### `WorktreeCreate`

Fires when Claude Code needs to create an isolated worktree. The hook is responsible for actually creating the worktree and reporting its path.

**When it fires:** When worktree isolation is requested for a task.

**Input fields:**

<ResponseField name="name" type="string">Suggested slug for the worktree directory.</ResponseField>

**Behavior:** Write the absolute path of the created worktree directory to stdout and exit with code 0. Any other exit code signals a failure.

***

### `WorktreeRemove`

Fires when Claude Code needs to remove a previously created worktree.

**When it fires:** When a worktree task completes and the worktree should be cleaned up.

**Input fields:**

<ResponseField name="worktree_path" type="string">Absolute path of the worktree to remove.</ResponseField>

**Behavior:** Exit code 0 means success. Other exit codes show stderr to the user.

***

### `CwdChanged`

Fires after the working directory changes.

**When it fires:** When Claude changes directories during a session.

**Input fields:**

<ResponseField name="old_cwd" type="string">Previous working directory.</ResponseField>
<ResponseField name="new_cwd" type="string">New working directory.</ResponseField>

**`hookSpecificOutput` fields:**

<ResponseField name="watchPaths" type="string[]">
  Absolute paths to add to the `FileChanged` watcher. Return this to start watching files in the new directory.
</ResponseField>

The `CLAUDE_ENV_FILE` environment variable is set — write `export KEY=value` lines to apply environment variables to subsequent Bash tool commands.

***

### `FileChanged`

Fires when a watched file is modified, added, or removed.

**When it fires:** When a file registered via `SessionStart.watchPaths` or `CwdChanged.watchPaths` changes on disk.

**Input fields:**

<ResponseField name="file_path" type="string">Absolute path of the changed file.</ResponseField>
<ResponseField name="event" type="'change' | 'add' | 'unlink'">The type of filesystem event.</ResponseField>

**`hookSpecificOutput` fields:**

<ResponseField name="watchPaths" type="string[]">
  Update the watch list with these absolute paths.
</ResponseField>

The `CLAUDE_ENV_FILE` environment variable is set for this hook as well.

***

## Async hooks

For hooks that need to run in the background without delaying Claude, output an async acknowledgment on stdout instead of the normal sync output:

```json  theme={null}
{
  "async": true,
  "asyncTimeout": 30
}
```

<ResponseField name="async" type="literal: true" required>
  Signals that this hook is running asynchronously.
</ResponseField>

<ResponseField name="asyncTimeout" type="number">
  How long (in seconds) the async hook is allowed to run before it is cancelled.
</ResponseField>

<Warning>
  Async hooks cannot block tool execution or inject context. Use them for side effects like notifications, logging, or metrics that must not slow down the agentic loop.
</Warning>


Built with [Mintlify](https://mintlify.com).