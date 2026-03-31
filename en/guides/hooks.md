> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Hooks

> Run shell commands, HTTP requests, or prompts automatically when Claude uses tools or reaches session milestones.

Hooks let you attach automation to Claude Code's tool lifecycle. When Claude reads a file, runs a bash command, or finishes a response, your configured hooks execute automatically. Use hooks to enforce code style, run tests, log tool usage, or gate what Claude is allowed to do.

## How hooks work

A hook is a command (shell script, HTTP endpoint, or LLM prompt) bound to a specific **event**. When that event fires, Claude Code runs every matching hook and uses the exit code and output to decide what to do next.

The input to each hook is a JSON object on stdin describing what happened — for example, the tool name and its arguments for `PreToolUse`, or the tool name and response for `PostToolUse`.

### Exit code semantics

Exit code behavior varies by event. The full table is documented in each event's description, but the general pattern is:

| Exit code | Meaning                                                                              |
| --------- | ------------------------------------------------------------------------------------ |
| `0`       | Success. Stdout may be shown to Claude (event-specific).                             |
| `2`       | Block or inject. Show stderr to Claude and (for `PreToolUse`) prevent the tool call. |
| Other     | Show stderr to the user only; execution continues.                                   |

## Hook events

<AccordionGroup>
  <Accordion title="PreToolUse — before tool execution">
    Fires before every tool call. The hook input contains the tool name and arguments as JSON.

    * Exit `0`: tool proceeds normally (stdout not shown)
    * Exit `2`: block the tool call and show stderr to Claude so it can respond
    * Other: show stderr to the user but allow the tool call to continue

    Use matchers to restrict this hook to specific tools (e.g., `Bash`, `Write`).
  </Accordion>

  <Accordion title="PostToolUse — after tool execution">
    Fires after every successful tool call. The hook input contains `inputs` (the tool arguments) and `response` (the tool result).

    * Exit `0`: stdout is shown in transcript mode (Ctrl+O)
    * Exit `2`: show stderr to Claude immediately (Claude can respond)
    * Other: show stderr to the user only

    Use this to run formatters, linters, or test runners after file edits.
  </Accordion>

  <Accordion title="PostToolUseFailure — after a tool error">
    Fires when a tool call results in an error. Input contains `tool_name`, `tool_input`, `error`, `error_type`, `is_interrupt`, and `is_timeout`.

    Same exit code semantics as `PostToolUse`.
  </Accordion>

  <Accordion title="Stop — before Claude concludes a response">
    Fires just before Claude's turn ends. No matcher support.

    * Exit `0`: no output shown
    * Exit `2`: show stderr to Claude and continue the conversation (Claude gets another turn)
    * Other: show stderr to the user only

    Use this to check that all required tasks are complete before Claude finishes.
  </Accordion>

  <Accordion title="SubagentStop — before a subagent concludes">
    Like `Stop`, but fires when a subagent (launched via the Agent tool) finishes. Input includes `agent_id`, `agent_type`, and `agent_transcript_path`. Same exit code semantics as `Stop`.
  </Accordion>

  <Accordion title="SubagentStart — when a subagent starts">
    Fires when a new subagent is launched. Input includes `agent_id` and `agent_type`.

    * Exit `0`: stdout is shown to the subagent's initial prompt
    * Other: show stderr to user only
  </Accordion>

  <Accordion title="SessionStart — when a session begins">
    Fires at the start of every session (startup, resume, `/clear`, or `/compact`). Input contains the start `source`.

    * Exit `0`: stdout is shown to Claude
    * Other: show stderr to user only (blocking errors are ignored)

    Match on `source` values: `startup`, `resume`, `clear`, `compact`.
  </Accordion>

  <Accordion title="UserPromptSubmit — when you submit a prompt">
    Fires when you press Enter to submit a prompt. Input contains your original prompt text.

    * Exit `0`: stdout is shown to Claude (can prepend context)
    * Exit `2`: block the prompt and show stderr to the user only
    * Other: show stderr to user only
  </Accordion>

  <Accordion title="PreCompact — before conversation compaction">
    Fires before Claude Code compacts the conversation (auto or manual). Input contains compaction details.

    * Exit `0`: stdout is appended as custom compact instructions
    * Exit `2`: block the compaction
    * Other: show stderr to user but proceed

    Match on `trigger`: `manual` or `auto`.
  </Accordion>

  <Accordion title="PostCompact — after conversation compaction">
    Fires after compaction completes. Input contains compaction details and the summary.

    * Exit `0`: stdout shown to user
    * Other: show stderr to user only
  </Accordion>

  <Accordion title="Setup — repository setup and maintenance">
    Fires with `trigger: init` (project onboarding) or `trigger: maintenance` (periodic). Use this for one-time setup scripts or periodic maintenance tasks.

    * Exit `0`: stdout shown to Claude
    * Other: show stderr to user only
  </Accordion>

  <Accordion title="PermissionRequest — when a permission dialog appears">
    Fires when Claude Code would show a permission prompt. Output JSON with `hookSpecificOutput.decision` to approve or deny programmatically.

    * Exit `0`: use the hook's decision if provided
    * Other: show stderr to user only
  </Accordion>

  <Accordion title="PermissionDenied — after auto mode denies a tool call">
    Fires when the auto mode classifier denies a tool call. Return `{"hookSpecificOutput":{"hookEventName":"PermissionDenied","retry":true}}` to tell Claude it may retry.
  </Accordion>

  <Accordion title="Notification — when notifications are sent">
    Fires for permission prompts, idle prompts, auth success, and elicitation events. Match on `notification_type`.

    * Exit `0`: no output shown
    * Other: show stderr to user only
  </Accordion>

  <Accordion title="CwdChanged — after working directory changes">
    Fires after the working directory changes. Input includes `old_cwd` and `new_cwd`. The `CLAUDE_ENV_FILE` environment variable is set — write bash export lines to that file to apply new env vars to subsequent Bash tool calls.
  </Accordion>

  <Accordion title="FileChanged — when a watched file changes">
    Fires when a file matching the hook's `matcher` pattern changes on disk. The matcher specifies filename patterns to watch (e.g., `.envrc|.env`). Like `CwdChanged`, supports `CLAUDE_ENV_FILE` for injecting environment.
  </Accordion>

  <Accordion title="SessionEnd — when a session ends">
    Fires when the session is ending (clear, logout, or exit). Match on `reason`: `clear`, `logout`, `prompt_input_exit`, or `other`.
  </Accordion>

  <Accordion title="ConfigChange — when config files change">
    Fires when settings files change during a session. Match on `source`: `user_settings`, `project_settings`, `local_settings`, `policy_settings`, or `skills`.

    * Exit `0`: allow the change
    * Exit `2`: block the change from being applied
  </Accordion>

  <Accordion title="InstructionsLoaded — when a CLAUDE.md file is loaded">
    Fires when any instruction file (CLAUDE.md or rule) is loaded. Observability-only — does not support blocking.
  </Accordion>

  <Accordion title="WorktreeCreate / WorktreeRemove — worktree lifecycle">
    `WorktreeCreate` fires when an isolated worktree needs to be created. Stdout should contain the absolute path of the created worktree. `WorktreeRemove` fires when a worktree should be cleaned up.
  </Accordion>

  <Accordion title="Task events — task lifecycle">
    `TaskCreated` and `TaskCompleted` fire when tasks are created or marked complete. Input includes `task_id`, `task_subject`, `task_description`, `teammate_name`, and `team_name`. Exit `2` prevents the state change.
  </Accordion>

  <Accordion title="TeammateIdle — when a teammate is about to go idle">
    Fires before a teammate goes idle. Exit `2` to send stderr to the teammate and prevent it from going idle.
  </Accordion>

  <Accordion title="Elicitation / ElicitationResult — MCP elicitation">
    `Elicitation` fires when an MCP server requests user input. Return JSON in `hookSpecificOutput` to provide the response programmatically. `ElicitationResult` fires after the user responds, allowing you to modify or block the response.
  </Accordion>
</AccordionGroup>

## Configuring hooks

Run `/hooks` inside Claude Code to open the hooks configuration menu. The menu shows all configured hooks grouped by event and lets you add, edit, or remove them interactively.

Hooks are stored in the `hooks` field of settings files:

* `~/.claude/settings.json` — user-level hooks (apply everywhere)
* `.claude/settings.json` — project-level hooks (apply for this project)
* `.claude/settings.local.json` — local hooks (not checked into VCS)

### Configuration format

```json  theme={null}
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $CLAUDE_FILE_PATH"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Session complete' >> ~/.claude-log.txt"
          }
        ]
      }
    ]
  }
}
```

Each event maps to an array of **matcher objects**. Each matcher object has:

* `matcher` (optional) — a string pattern to match against the event's matchable field (for example, `tool_name` for `PreToolUse`/`PostToolUse`, `trigger` for `Setup`, `source` for `SessionStart`)
* `hooks` — an array of hook commands to run when the matcher matches

### Hook command types

<Tabs>
  <Tab title="Shell command">
    ```json  theme={null}
    {
      "type": "command",
      "command": "npm test",
      "timeout": 60,
      "shell": "bash"
    }
    ```

    Fields:

    * `command` — the shell command to run (required)
    * `timeout` — timeout in seconds (default: no limit)
    * `shell` — `"bash"` (default) or `"powershell"`
    * `statusMessage` — custom spinner text shown while the hook runs
    * `async` — run in background without blocking (`true`/`false`)
    * `once` — run once and remove the hook automatically
    * `if` — permission rule syntax to conditionally skip the hook (e.g., `"Bash(git *)"`)
  </Tab>

  <Tab title="HTTP request">
    ```json  theme={null}
    {
      "type": "http",
      "url": "https://hooks.example.com/claude-event",
      "headers": {
        "Authorization": "Bearer $MY_TOKEN"
      },
      "allowedEnvVars": ["MY_TOKEN"],
      "timeout": 10
    }
    ```

    Claude Code POSTs the hook input JSON to the URL. Headers support `$VAR` expansion for variables listed in `allowedEnvVars`.
  </Tab>

  <Tab title="LLM prompt">
    ```json  theme={null}
    {
      "type": "prompt",
      "prompt": "Review this tool call for security issues: $ARGUMENTS. If you find a problem, output an explanation and exit with code 2.",
      "model": "claude-haiku-4-5",
      "timeout": 30
    }
    ```

    The hook prompt is evaluated by an LLM. `$ARGUMENTS` is replaced with the hook input JSON. The LLM's response becomes the hook output.
  </Tab>

  <Tab title="Agent hook">
    ```json  theme={null}
    {
      "type": "agent",
      "prompt": "Verify that the unit tests in $ARGUMENTS passed and all assertions are meaningful.",
      "timeout": 60
    }
    ```

    Like a prompt hook, but runs as a full agent with tool access. Useful for verification tasks that require reading files or running commands.
  </Tab>
</Tabs>

## Matcher patterns

For events that support matching (like `PreToolUse`, `PostToolUse`, `SessionStart`), the `matcher` field filters which inputs trigger the hook.

* An empty or absent `matcher` matches all inputs for that event.
* For tool events, the `matcher` is matched against the `tool_name` (e.g., `"Bash"`, `"Write"`, `"Read"`).
* For `SessionStart`, it matches `source` (e.g., `"startup"`, `"compact"`).
* For `Setup`, it matches `trigger` (e.g., `"init"`, `"maintenance"`).
* For `FileChanged`, the `matcher` specifies filename patterns to watch (e.g., `".envrc|.env"`).

## Example hooks

### Auto-format files after editing

Run Prettier after every file write:

```json  theme={null}
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write \"$CLAUDE_FILE_PATH\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### Run tests after bash commands

Run the test suite after any bash command that touches source files:

```json  theme={null}
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if git diff --name-only HEAD | grep -q '\\.ts$'; then npm test; fi",
            "timeout": 120,
            "async": true
          }
        ]
      }
    ]
  }
}
```

### Log all tool usage

Append every tool call to a log file:

```json  theme={null}
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"$(date -u +%Y-%m-%dT%H:%M:%SZ) $CLAUDE_TOOL_NAME\" >> ~/.claude-tool-log.txt",
            "async": true
          }
        ]
      }
    ]
  }
}
```

### Block dangerous commands

Use `PreToolUse` to prevent `rm -rf` from being called:

```json  theme={null}
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$CLAUDE_TOOL_INPUT\" | grep -q 'rm -rf'; then echo 'Blocked: rm -rf is not allowed' >&2; exit 2; fi"
          }
        ]
      }
    ]
  }
}
```

### Inject environment on directory change

Use `CwdChanged` with `CLAUDE_ENV_FILE` to load `.envrc` when you change directories:

```json  theme={null}
{
  "hooks": {
    "CwdChanged": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "if [ -f .envrc ]; then grep '^export ' .envrc >> \"$CLAUDE_ENV_FILE\"; fi"
          }
        ]
      }
    ]
  }
}
```

## Hook timeout configuration

Set a per-hook timeout in seconds using the `timeout` field:

```json  theme={null}
{
  "type": "command",
  "command": "npm run integration-tests",
  "timeout": 300
}
```

Hooks without a `timeout` run until they exit naturally. For long-running hooks that should not block Claude, use `"async": true`.

## Hooks vs. skills

| Feature       | Hooks                               | Skills                                             |
| ------------- | ----------------------------------- | -------------------------------------------------- |
| When they run | Automatically on tool events        | When Claude or you explicitly invoke `/skill-name` |
| Purpose       | Side effects, gating, observability | On-demand workflows and capabilities               |
| Configuration | Settings JSON                       | Markdown files in `.claude/skills/`                |
| Input         | JSON from the tool event            | The arguments you pass to the skill                |

Use hooks for things that should happen automatically every time (formatting, logging, enforcement). Use skills for repeatable workflows that you want to trigger on demand.


Built with [Mintlify](https://mintlify.com).