> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# CLI flags

> All options you can pass when launching Claude Code from the terminal. Run claude --help to see the full list.

Pass CLI flags when you launch Claude Code:

```bash  theme={null}
claude [flags] [prompt]
```

Run `claude --help` to see all available flags in your installed version.

<Tip>
  Flags that configure session behavior (like `--model` and `--permission-mode`) can also be changed mid-session with the corresponding slash commands: `/model` and `/permissions`.
</Tip>

***

## Core flags

<AccordionGroup>
  <Accordion title="-p, --print">
    Run Claude non-interactively. Claude processes the prompt (from the argument or stdin), prints the response, and exits. No REPL is started.

    ```bash  theme={null}
    claude -p "explain the main function in src/index.ts"
    echo "what does this do?" | claude -p
    ```

    <Warning>
      The workspace trust dialog is skipped in `--print` mode. Only use this flag in directories you trust.
    </Warning>

    Works with: `--output-format`, `--model`, `--system-prompt`, `--permission-mode`, `--max-turns`, `--allowed-tools`.
  </Accordion>

  <Accordion title="--output-format <format>">
    Set the output format. Only works with `--print`.

    | Value         | Description                                         |
    | ------------- | --------------------------------------------------- |
    | `text`        | Plain text output (default)                         |
    | `json`        | Single JSON object with the complete result         |
    | `stream-json` | Newline-delimited JSON stream with real-time events |

    ```bash  theme={null}
    claude -p "list the exported functions" --output-format json
    claude -p "refactor this file" --output-format stream-json
    ```

    Use `stream-json` when you want to process Claude's output incrementally as it arrives (useful for long-running tasks or piping into other tools).
  </Accordion>

  <Accordion title="--input-format <format>">
    Set the input format for stdin. Only works with `--print`.

    | Value         | Description                         |
    | ------------- | ----------------------------------- |
    | `text`        | Plain text input (default)          |
    | `stream-json` | Newline-delimited JSON stream input |

    `stream-json` input requires `--output-format stream-json`.

    ```bash  theme={null}
    cat messages.jsonl | claude -p --input-format stream-json --output-format stream-json
    ```
  </Accordion>

  <Accordion title="--verbose">
    Enable verbose output. Overrides the `verbose` setting in your config file.

    ```bash  theme={null}
    claude --verbose
    claude -p "debug this" --verbose
    ```
  </Accordion>

  <Accordion title="-v, --version">
    Print the version number and exit.

    ```bash  theme={null}
    claude --version
    claude -v
    ```
  </Accordion>

  <Accordion title="-h, --help">
    Display help for the command and exit.

    ```bash  theme={null}
    claude --help
    claude mcp --help
    ```
  </Accordion>
</AccordionGroup>

***

## Session continuation flags

<AccordionGroup>
  <Accordion title="-c, --continue">
    Resume the most recent conversation in the current directory without prompting for a session to resume.

    ```bash  theme={null}
    claude --continue
    claude -c "now add tests for that"
    ```
  </Accordion>

  <Accordion title="-r, --resume [session-id]">
    Resume a conversation by session ID. Without a value, opens an interactive picker where you can search through past sessions. Accepts an optional search term to filter the list.

    ```bash  theme={null}
    # Open interactive picker
    claude --resume

    # Resume by session ID
    claude --resume 550e8400-e29b-41d4-a716-446655440000

    # Open picker filtered by search term
    claude --resume "auth refactor"
    ```
  </Accordion>

  <Accordion title="--fork-session">
    When used with `--continue` or `--resume`, creates a new session branched from the resumed conversation rather than continuing it in place.

    ```bash  theme={null}
    claude --resume <session-id> --fork-session
    ```
  </Accordion>

  <Accordion title="-n, --name <name>">
    Set a display name for the session. The name appears in `/resume` and in the terminal title.

    ```bash  theme={null}
    claude --name "auth-refactor"
    ```
  </Accordion>

  <Accordion title="--session-id <uuid>">
    Use a specific UUID as the session ID instead of a generated one. Must be a valid UUID. Cannot be used with `--continue` or `--resume` unless `--fork-session` is also specified.

    ```bash  theme={null}
    claude --session-id 550e8400-e29b-41d4-a716-446655440000
    ```
  </Accordion>

  <Accordion title="--no-session-persistence">
    Disable session persistence. The session will not be saved to disk and cannot be resumed. Only works with `--print`.

    ```bash  theme={null}
    claude -p "one-off task" --no-session-persistence
    ```
  </Accordion>
</AccordionGroup>

***

## Model and capability flags

<AccordionGroup>
  <Accordion title="--model <model>">
    Set the model for the session. Accepts an alias (e.g. `sonnet`, `opus`, `haiku`) or a full model ID (e.g. `claude-sonnet-4-6`).

    ```bash  theme={null}
    claude --model sonnet
    claude --model opus
    claude --model claude-sonnet-4-6
    ```

    You can also change the model mid-session with `/model`.
  </Accordion>

  <Accordion title="--effort <level>">
    Set the effort level for the session. Controls how much computation Claude applies to each response.

    | Value    | Description               |
    | -------- | ------------------------- |
    | `low`    | Faster, lighter responses |
    | `medium` | Balanced (default)        |
    | `high`   | More thorough reasoning   |
    | `max`    | Maximum effort            |

    ```bash  theme={null}
    claude --effort high "review this architecture"
    ```
  </Accordion>

  <Accordion title="--fallback-model <model>">
    Enable automatic fallback to a different model when the primary model is overloaded. Only works with `--print`.

    ```bash  theme={null}
    claude -p "analyze this" --model opus --fallback-model sonnet
    ```
  </Accordion>
</AccordionGroup>

***

## Permission and safety flags

<AccordionGroup>
  <Accordion title="--permission-mode <mode>">
    Set the permission mode for the session.

    | Mode                | Behavior                                                                             |
    | ------------------- | ------------------------------------------------------------------------------------ |
    | `default`           | Claude prompts before running commands and making edits                              |
    | `acceptEdits`       | File edits are applied automatically; shell commands still require approval          |
    | `plan`              | Claude proposes a plan and waits for your approval before acting                     |
    | `bypassPermissions` | All actions run without prompts — intended for sandboxed automated environments only |

    ```bash  theme={null}
    claude --permission-mode acceptEdits
    claude --permission-mode plan "refactor the payment module"
    claude --permission-mode bypassPermissions   # only in isolated sandboxes
    ```

    <Warning>
      `bypassPermissions` disables all confirmation prompts. Only use it inside Docker containers, CI sandboxes, or other isolated environments with no internet access.
    </Warning>
  </Accordion>

  <Accordion title="--dangerously-skip-permissions">
    Bypass all permission checks. Claude takes all actions (file edits, shell commands) without asking. Equivalent to `--permission-mode bypassPermissions`.

    ```bash  theme={null}
    claude --dangerously-skip-permissions -p "run the full test suite and fix failures"
    ```

    <Warning>
      Only use this in sandboxed environments with no internet access. Claude Code enforces this: the flag is rejected when running with root/sudo privileges, or outside a Docker or bubblewrap container.
    </Warning>
  </Accordion>

  <Accordion title="--allow-dangerously-skip-permissions">
    Make bypassing all permission checks available as an option during the session, without enabling it by default. Useful for automated pipelines that may need to escalate mid-session.

    ```bash  theme={null}
    claude --allow-dangerously-skip-permissions -p "..."
    ```
  </Accordion>

  <Accordion title="--allowed-tools <tools...>">
    **Aliases:** `--allowedTools`

    Comma- or space-separated list of tools Claude is allowed to use. Tools not in this list are blocked.

    ```bash  theme={null}
    claude --allowed-tools "Bash(git:*) Edit Read"
    claude --allowed-tools Bash,Edit,Read
    ```

    Tool patterns support glob-style matching: `Bash(git:*)` permits any git command; `Edit(src/**)` permits edits under `src/`.
  </Accordion>

  <Accordion title="--disallowed-tools <tools...>">
    **Aliases:** `--disallowedTools`

    Comma- or space-separated list of tools Claude is not allowed to use.

    ```bash  theme={null}
    claude --disallowed-tools "Bash(rm:*)"
    ```
  </Accordion>

  <Accordion title="--tools <tools...>">
    Specify the exact set of built-in tools available for the session. Use `""` to disable all tools, `default` to enable all tools, or name specific tools.

    ```bash  theme={null}
    # Disable all tools
    claude --tools ""

    # Enable only Bash and Read
    claude --tools "Bash Read"

    # Enable the default set
    claude --tools default
    ```
  </Accordion>
</AccordionGroup>

***

## Context and prompt flags

<AccordionGroup>
  <Accordion title="--add-dir <directories...>">
    Add one or more directories to the tool access context. Claude will be able to read and edit files in these directories in addition to the current working directory.

    ```bash  theme={null}
    claude --add-dir /shared/libs --add-dir /shared/config
    ```

    Useful for monorepos or projects where related code lives outside the current directory.
  </Accordion>

  <Accordion title="--system-prompt <prompt>">
    Override the default system prompt with a custom prompt. Cannot be used with `--system-prompt-file`.

    ```bash  theme={null}
    claude --system-prompt "You are a security auditor. Focus only on vulnerabilities."
    ```
  </Accordion>

  <Accordion title="--append-system-prompt <text>">
    Append text to the default system prompt. Unlike `--system-prompt`, this preserves Claude's built-in instructions and adds to them.

    ```bash  theme={null}
    claude --append-system-prompt "Always suggest test cases for every function you write."
    ```
  </Accordion>

  <Accordion title="--mcp-config <configs...>">
    Load MCP servers from one or more JSON config files or inline JSON strings. Multiple values are space-separated.

    ```bash  theme={null}
    # Load from a file
    claude --mcp-config ./mcp-servers.json

    # Load from multiple files
    claude --mcp-config ./local-tools.json ./db-tools.json

    # Pass inline JSON
    claude --mcp-config '{"mcpServers":{"filesystem":{"command":"npx","args":["@modelcontextprotocol/server-filesystem","/tmp"]}}}'
    ```

    See the [MCP servers guide](/guides/mcp-servers) for the config file format.
  </Accordion>

  <Accordion title="--strict-mcp-config">
    Only use MCP servers from `--mcp-config`, ignoring all other MCP configurations (user config, project config, etc.).

    ```bash  theme={null}
    claude --mcp-config ./ci-tools.json --strict-mcp-config
    ```
  </Accordion>

  <Accordion title="--settings <file-or-json>">
    Load additional settings from a JSON file path or an inline JSON string.

    ```bash  theme={null}
    # From a file
    claude --settings ./team-settings.json

    # Inline JSON
    claude --settings '{"model":"claude-sonnet-4-6","verbose":true}'
    ```
  </Accordion>

  <Accordion title="--setting-sources <sources>">
    Comma-separated list of settings sources to load. Controls which settings files are read at startup.

    | Value     | Description                                                 |
    | --------- | ----------------------------------------------------------- |
    | `user`    | Load `~/.claude/settings.json`                              |
    | `project` | Load `.claude/settings.json` in the current directory       |
    | `local`   | Load `.claude/settings.local.json` in the current directory |

    ```bash  theme={null}
    # Load only user-level settings (ignore project settings)
    claude --setting-sources user

    # Load user and project settings
    claude --setting-sources user,project
    ```
  </Accordion>

  <Accordion title="--agents <json>">
    Define custom agents inline as a JSON object. Each key is the agent name; the value is an object with `description` and `prompt`.

    ```bash  theme={null}
    claude --agents '{"reviewer":{"description":"Reviews code for security issues","prompt":"You are a security-focused code reviewer."}}'
    ```
  </Accordion>
</AccordionGroup>

***

## Output control flags

<AccordionGroup>
  <Accordion title="--include-hook-events">
    Include all hook lifecycle events in the output stream. Only works with `--output-format stream-json`.

    ```bash  theme={null}
    claude -p "run task" --output-format stream-json --include-hook-events
    ```
  </Accordion>

  <Accordion title="--max-turns <n>">
    Limit the number of agentic turns in non-interactive mode. Claude stops after this many turns even if the task is incomplete. Only works with `--print`.

    ```bash  theme={null}
    claude -p "refactor this module" --max-turns 10
    ```
  </Accordion>

  <Accordion title="--max-budget-usd <amount>">
    Set a maximum dollar amount to spend on API calls. Claude stops when the budget is reached. Only works with `--print`.

    ```bash  theme={null}
    claude -p "large analysis task" --max-budget-usd 2.50
    ```
  </Accordion>

  <Accordion title="--json-schema <schema>">
    Provide a JSON Schema for structured output validation. Claude's response will be validated against this schema.

    ```bash  theme={null}
    claude -p "extract the function names" \
      --output-format json \
      --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}'
    ```
  </Accordion>
</AccordionGroup>

***

## Worktree flags

<AccordionGroup>
  <Accordion title="-w, --worktree [name]">
    Create a new git worktree for this session. Optionally specify a name for the worktree branch. Accepts a PR number or GitHub PR URL to create a worktree from that PR.

    ```bash  theme={null}
    claude --worktree
    claude --worktree feature-auth
    claude --worktree "#142"
    ```
  </Accordion>

  <Accordion title="--tmux">
    Create a tmux session alongside the worktree. Requires `--worktree`. Uses iTerm2 native panes when available; pass `--tmux=classic` to force standard tmux behavior.

    ```bash  theme={null}
    claude --worktree feature-auth --tmux
    ```
  </Accordion>
</AccordionGroup>

***

## Debug flags

<AccordionGroup>
  <Accordion title="-d, --debug [filter]">
    Enable debug mode. Optionally pass a filter to restrict which debug categories are shown.

    ```bash  theme={null}
    # Show all debug output
    claude --debug

    # Show only api and hooks categories
    claude --debug "api,hooks"

    # Exclude specific categories
    claude --debug "!file,!1p"
    ```
  </Accordion>

  <Accordion title="--debug-file <path>">
    Write debug logs to a specific file path instead of displaying them inline. Implicitly enables debug mode.

    ```bash  theme={null}
    claude --debug-file /tmp/claude-debug.log
    ```
  </Accordion>

  <Accordion title="--bare">
    Minimal mode. Skips hooks, LSP, plugin sync, attribution, auto-memory, background prefetches, keychain reads, and `CLAUDE.md` auto-discovery. Sets `CLAUDE_CODE_SIMPLE=1`.

    Authentication is limited to `ANTHROPIC_API_KEY` or `apiKeyHelper` via `--settings` (OAuth and keychain are not used).

    Use `--bare` in scripted pipelines where startup latency matters and the features it disables are not needed. You can still provide context explicitly:

    ```bash  theme={null}
    claude --bare \
      --system-prompt "$(cat context.md)" \
      --add-dir /project/libs \
      --mcp-config ./tools.json \
      -p "perform the analysis"
    ```
  </Accordion>
</AccordionGroup>

***

## Flag combinations

Common flag patterns for scripting and automation:

```bash  theme={null}
# Non-interactive with JSON output
claude -p "list all exported types" --output-format json

# Bypass permissions in CI (sandboxed environment only)
claude -p "run full test suite and fix failures" --dangerously-skip-permissions

# Resume last session and continue non-interactively
claude --continue -p "now write the tests for that"

# Use a custom MCP config with strict isolation
claude --mcp-config ./ci-mcp.json --strict-mcp-config -p "analyze the schema"

# Append to system prompt without replacing it
claude --append-system-prompt "Always output TypeScript, not JavaScript."
```


Built with [Mintlify](https://mintlify.com).