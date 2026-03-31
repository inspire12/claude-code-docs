> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Permissions API

> Reference for permission modes, allow/deny rule syntax, and configuration methods for controlling which tools Claude Code can use without prompting.

The permissions system controls whether Claude Code runs a tool immediately, prompts you for confirmation, or blocks the call entirely. You configure it through permission modes and rule lists that can be set globally, per project, or per session.

## Permission modes

A permission mode sets the baseline behavior for all tool calls. Individual allow/deny rules can override this baseline for specific tools or commands.

| Mode                | Description                                                                                                                |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `default`           | Prompt for approval on potentially dangerous operations. Safe read-only tools run without prompting.                       |
| `acceptEdits`       | Auto-approve all file edit operations (`Write`, `Edit`, `MultiEdit`). Bash commands still prompt.                          |
| `bypassPermissions` | Skip all permission checks. All tools run without prompting. Requires `allowDangerouslySkipPermissions: true` in settings. |
| `plan`              | Read-only planning mode. No tool execution; Claude can only read files and explain what it would do.                       |
| `dontAsk`           | Do not prompt the user. Deny any tool call that is not pre-approved by an explicit allow rule.                             |

<Warning>
  `bypassPermissions` mode disables all safeguards. Only use it in sandboxed environments or CI pipelines where you fully control the input. The mode requires `allowDangerouslySkipPermissions: true` in your settings file.
</Warning>

### Setting the permission mode

<Tabs>
  <Tab title="CLI flag">
    Pass `--permission-mode` to set the mode for a single invocation:

    ```bash  theme={null}
    claude --permission-mode acceptEdits
    claude --permission-mode bypassPermissions --dangerously-skip-permissions
    claude --permission-mode plan
    ```
  </Tab>

  <Tab title="Settings file">
    Set `defaultPermissionMode` in any Claude Code settings file to make it the default for all sessions in that scope:

    ```json  theme={null}
    {
      "defaultPermissionMode": "acceptEdits"
    }
    ```

    Settings files load in this priority order (highest wins):

    * `.claude/settings.local.json` — local overrides, not committed
    * `.claude/settings.json` — project-level defaults
    * `~/.claude/settings.json` — user-level defaults
  </Tab>

  <Tab title="/permissions command">
    Run `/permissions` inside a session to open the interactive permissions panel where you can view and edit allow/deny rules. Session-level changes are not persisted to disk.

    ```
    /permissions
    ```
  </Tab>

  <Tab title="SDK control request">
    Change the permission mode programmatically via the control protocol:

    ```json  theme={null}
    {
      "type": "control_request",
      "request_id": "pm-1",
      "request": {
        "subtype": "set_permission_mode",
        "mode": "acceptEdits"
      }
    }
    ```
  </Tab>
</Tabs>

***

## Permission rules

Rules let you pre-approve or block specific tools and commands without changing the global permission mode. Rules are additive across settings scopes — allow rules from all files merge together.

### Configuration format

```json  theme={null}
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm run *)",
      "Read",
      "Write(src/**)",
      "mcp__myserver"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(curl * | bash)",
      "Write(/etc/**)"
    ]
  }
}
```

<ResponseField name="allow" type="string[]">
  Rules that auto-approve matching tool calls without prompting.
</ResponseField>

<ResponseField name="deny" type="string[]">
  Rules that unconditionally block matching tool calls.
</ResponseField>

<Note>
  Deny rules always take precedence over allow rules. If both an allow rule and a deny rule match a tool call, the call is blocked.
</Note>

### Rule syntax

A rule is a string that matches a tool name, optionally with a parenthesized content pattern.

**Tool name only** — matches every call to that tool:

```
Read
Write
Edit
Bash
```

**Tool name with content pattern** — matches calls where the tool's primary input matches the glob:

```
Bash(git *)
Bash(npm run *)
Write(src/*)
Edit(*.ts)
```

For `Bash`, the pattern is matched against the full command string. For file tools (`Write`, `Edit`, `Read`, `Glob`), the pattern is matched against the file path argument.

**MCP server** — matches all tools from a specific MCP server:

```
mcp__myserver
```

**MCP server wildcard** — same as above, explicit wildcard form:

```
mcp__myserver__*
```

**Specific MCP tool** — matches one tool from a server:

```
mcp__myserver__query_database
```

**Agent type** — blocks or allows a specific subagent:

```
Agent(Explore)
Agent(CodeReviewer)
```

### Pattern matching for `Bash`

Bash rule patterns use shell-style glob matching on the full command string.

```json  theme={null}
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)",
      "Bash(npm run test*)",
      "Bash(make *)"
    ],
    "deny": [
      "Bash(rm *)",
      "Bash(sudo *)",
      "Bash(* | bash)",
      "Bash(* | sh)"
    ]
  }
}
```

<Warning>
  Bash patterns match prefix-first. A rule like `Bash(git *)` matches `git status`, `git log --oneline`, and `git push --force`. Be specific when writing deny rules.
</Warning>

### Pattern matching for file tools

File path patterns are matched against the absolute path of the file being read, written, or edited.

```json  theme={null}
{
  "permissions": {
    "allow": [
      "Write(src/**)",
      "Write(tests/**)",
      "Edit(*.md)",
      "Read"
    ],
    "deny": [
      "Write(/etc/**)",
      "Write(~/.ssh/**)",
      "Edit(.env*)"
    ]
  }
}
```

***

## Permission rule sources and priority

Rules are collected from multiple sources and evaluated in a defined order. When the same tool call matches rules from different sources, deny takes precedence over allow, and the most restrictive result wins.

| Source            | Where configured                             | Editable       |
| ----------------- | -------------------------------------------- | -------------- |
| `policySettings`  | Managed policy layer                         | No             |
| `flagSettings`    | CLI flags and SDK control requests           | Per-session    |
| `userSettings`    | `~/.claude/settings.json`                    | Yes            |
| `projectSettings` | `.claude/settings.json`                      | Yes            |
| `localSettings`   | `.claude/settings.local.json`                | Yes            |
| `cliArg`          | `--allowedTools` / `--disallowedTools` flags | Per-invocation |
| `session`         | `/permissions` command, SDK updates          | Per-session    |

### `--allowedTools` and `--disallowedTools` CLI flags

Pass comma-separated rule strings directly on the command line:

```bash  theme={null}
claude --allowedTools "Bash(git *),Read,Write" --print "Run the tests"
claude --disallowedTools "Bash,Write" --print "Summarize this project"
```

These map to the `cliArg` source and apply for the duration of the invocation.

***

## Permission decisions

The permission engine evaluates each tool call through a pipeline and returns one of three outcomes.

| Decision | Meaning                                                |
| -------- | ------------------------------------------------------ |
| `allow`  | Tool runs immediately.                                 |
| `ask`    | User is prompted for confirmation.                     |
| `deny`   | Tool call is blocked; Claude receives an error result. |

### Decision pipeline

Claude Code evaluates tool calls in this order:

1. **Deny rules** — if any deny rule matches, the call is blocked immediately.
2. **Ask rules** — if any ask rule matches, the permission dialog is shown.
3. **Tool's own permission check** — the tool's `checkPermissions` method runs (e.g., Bash checks individual subcommands).
4. **Safety checks** — paths inside `.git/`, `.claude/`, `.vscode/`, and shell config files always prompt, even in `bypassPermissions` mode.
5. **Mode check** — `bypassPermissions` and plan mode apply here.
6. **Allow rules** — if an allow rule matches, the call is approved.
7. **Default behavior** — if no rule matched, prompt the user.

<Note>
  Safety checks (step 4) are bypass-immune. Even with `bypassPermissions` mode, Claude Code will prompt before modifying files in `.git/` or shell configuration files like `~/.bashrc`.
</Note>

***

## Working directories

By default, Claude Code restricts file operations to the current working directory and its subdirectories. You can grant access to additional directories.

### Via CLI flag

```bash  theme={null}
claude --add-dir /path/to/extra/dir
```

### Via settings

```json  theme={null}
{
  "permissions": {
    "additionalDirectories": [
      "/shared/data",
      "/home/user/configs"
    ]
  }
}
```

### Via SDK control request

```json  theme={null}
{
  "type": "control_request",
  "request_id": "dirs-1",
  "request": {
    "subtype": "apply_flag_settings",
    "settings": {
      "permissions": {
        "additionalDirectories": ["/shared/data"]
      }
    }
  }
}
```

***

## Permission updates via the SDK

SDK hosts (IDEs, desktop apps) can respond to `can_use_tool` control requests and include permission updates to persist rule changes alongside their decisions.

### `PermissionUpdate` object

```json  theme={null}
{
  "type": "addRules",
  "rules": [
    { "toolName": "Bash", "ruleContent": "git *" }
  ],
  "behavior": "allow",
  "destination": "userSettings"
}
```

<ResponseField name="type" type="'addRules' | 'replaceRules' | 'removeRules' | 'setMode' | 'addDirectories' | 'removeDirectories'" required>
  The update operation.
</ResponseField>

<ResponseField name="rules" type="PermissionRuleValue[]">
  Rules to add, replace, or remove. Each rule has `toolName` and an optional `ruleContent` (the parenthesized pattern).
</ResponseField>

<ResponseField name="behavior" type="'allow' | 'deny' | 'ask'" required>
  Which rule list to modify.
</ResponseField>

<ResponseField name="destination" type="'userSettings' | 'projectSettings' | 'localSettings' | 'session' | 'cliArg'" required>
  Where to persist the update. `session` applies only to the current session; `userSettings`, `projectSettings`, and `localSettings` write to the corresponding settings files on disk.
</ResponseField>

### Permission decision response

When responding to a `can_use_tool` request, include `updatedPermissions` to persist rule changes:

```json  theme={null}
{
  "type": "control_response",
  "response": {
    "subtype": "success",
    "request_id": "<request_id>",
    "response": {
      "behavior": "allow",
      "updatedPermissions": [
        {
          "type": "addRules",
          "rules": [{ "toolName": "Bash", "ruleContent": "git *" }],
          "behavior": "allow",
          "destination": "userSettings"
        }
      ],
      "decisionClassification": "user_permanent"
    }
  }
}
```

<ResponseField name="behavior" type="'allow' | 'deny'" required>
  The permission decision.
</ResponseField>

<ResponseField name="updatedInput" type="Record<string, unknown>">
  Modified tool input to use instead of the original. Only valid when `behavior` is `"allow"`.
</ResponseField>

<ResponseField name="updatedPermissions" type="PermissionUpdate[]">
  Permission rule updates to apply and persist alongside this decision.
</ResponseField>

<ResponseField name="decisionClassification" type="'user_temporary' | 'user_permanent' | 'user_reject'">
  How the user responded, for telemetry. `user_temporary` for allow-once; `user_permanent` for always-allow; `user_reject` for deny.
</ResponseField>

***

## Hooks and permissions

`PreToolUse` and `PermissionRequest` hooks can inject permission decisions programmatically. See [Hooks reference](/reference/sdk/hooks-reference) for details on `permissionDecision` output and `hookSpecificOutput.decision`.

The hook-based permission flow is especially useful for headless agents that cannot show interactive prompts. When `shouldAvoidPermissionPrompts` is true (background agent mode), Claude Code runs `PermissionRequest` hooks before falling back to auto-deny.

***

## Safety recommendations

<AccordionGroup>
  <Accordion title="CI/CD pipelines">
    Use `bypassPermissions` only in isolated, short-lived environments where you control all inputs. Set explicit deny rules for destructive operations as a defense-in-depth measure:

    ```json  theme={null}
    {
      "defaultPermissionMode": "bypassPermissions",
      "allowDangerouslySkipPermissions": true,
      "permissions": {
        "deny": [
          "Bash(rm -rf *)",
          "Bash(sudo *)",
          "Bash(curl * | bash)"
        ]
      }
    }
    ```
  </Accordion>

  <Accordion title="IDE and interactive use">
    Use `default` mode with allow rules for common safe operations. This minimizes interruptions while keeping you in control of destructive actions:

    ```json  theme={null}
    {
      "defaultPermissionMode": "default",
      "permissions": {
        "allow": [
          "Read",
          "Glob",
          "Grep",
          "Bash(git status)",
          "Bash(git log *)",
          "Bash(git diff *)",
          "Bash(npm run *)",
          "Bash(make *)"
        ]
      }
    }
    ```
  </Accordion>

  <Accordion title="Code review and read-only analysis">
    Use `plan` mode when you want Claude to reason about code without making any changes. Claude can read files and explain what it would do, but cannot write, edit, or run commands:

    ```bash  theme={null}
    claude --permission-mode plan "Explain the architecture of this codebase"
    ```
  </Accordion>

  <Accordion title="Automated agents with human approval">
    Use `dontAsk` mode combined with an SDK `PermissionRequest` hook to replace the interactive dialog with your own approval UI. The hook receives every tool call that would have prompted and can allow, deny, or forward it to a human reviewer:

    ```json  theme={null}
    {
      "defaultPermissionMode": "dontAsk",
      "permissions": {
        "allow": [
          "Read",
          "Bash(git *)"
        ]
      }
    }
    ```

    Any tool call not matching an allow rule is sent to your `PermissionRequest` hook handler instead of being auto-denied.
  </Accordion>
</AccordionGroup>


Built with [Mintlify](https://mintlify.com).