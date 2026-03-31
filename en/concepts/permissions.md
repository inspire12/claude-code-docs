> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Permissions

> How Claude Code controls which operations Claude can perform automatically versus which require your explicit approval.

Claude Code runs tools on your local machine — executing shell commands, editing files, fetching URLs. The permission system gives you precise control over which operations Claude performs automatically and which require your explicit approval.

## What permissions control

Permissions apply to three categories of operations:

<CardGroup cols={3}>
  <Card title="File operations" icon="file">
    Reading, editing, and writing files on your local filesystem via the `Read`, `Edit`, and `Write` tools.
  </Card>

  <Card title="Bash commands" icon="terminal">
    Any shell command executed through the `Bash` tool, including installs, builds, git operations, and arbitrary scripts.
  </Card>

  <Card title="MCP tool calls" icon="plug">
    Tools exposed by connected MCP servers, which may include database queries, API calls, or browser automation.
  </Card>
</CardGroup>

## Permission modes

The permission mode determines the default behaviour when no specific allow/deny rule matches a tool call. Set the mode once and it applies for the entire session.

<AccordionGroup>
  <Accordion title="default — ask for potentially dangerous operations">
    The standard mode. Claude Code evaluates each tool call and prompts you for confirmation on operations that could have side effects: running shell commands, editing files, making network requests. Read-only operations (file reads, searches) are auto-approved.

    This is the recommended mode for everyday use.
  </Accordion>

  <Accordion title="acceptEdits — auto-approve file edits">
    File edit and write operations (`Edit`, `Write`) are automatically approved without prompting. Bash commands still require confirmation.

    Useful when you trust Claude to make file changes freely but still want to review shell commands.
  </Accordion>

  <Accordion title="plan — read-only planning mode">
    Claude can read files, search the codebase, and discuss changes, but cannot execute any write or bash operations. All mutating tool calls are blocked.

    Use this mode when you want Claude to analyse a problem and produce a plan before you authorise any changes. The model can exit plan mode and request permission to proceed via the `ExitPlanMode` tool.
  </Accordion>

  <Accordion title="bypassPermissions — skip all permission checks">
    All permission checks are disabled. Every tool call is executed immediately without any confirmation prompts.

    <Warning>
      This mode is intended for automated, fully-scripted workflows where you have audited what Claude will do in advance. Never use `bypassPermissions` in an interactive session where Claude might take unexpected actions.
    </Warning>
  </Accordion>

  <Accordion title="dontAsk — suppress prompts">
    Similar to `bypassPermissions` but uses a slightly different internal path. Tool calls that would normally prompt are auto-approved. Intended for scripted/non-interactive scenarios.
  </Accordion>

  <Accordion title="auto — transcript-classifier mode (feature-gated)">
    An experimental mode that uses a secondary AI classifier to evaluate each proposed tool call against the conversation transcript. The classifier decides whether the operation is within the scope of what was requested, and either auto-approves or escalates to a human prompt.

    This mode is only available when the `TRANSCRIPT_CLASSIFIER` feature flag is enabled.
  </Accordion>
</AccordionGroup>

## Setting the permission mode

<Tabs>
  <Tab title="CLI flag">
    Pass `--permission-mode` when starting Claude Code:

    ```bash  theme={null}
    claude --permission-mode acceptEdits
    claude --permission-mode bypassPermissions
    claude --permission-mode plan
    ```
  </Tab>

  <Tab title="/permissions command">
    Change the mode mid-session without restarting:

    ```
    /permissions
    ```

    An interactive menu lets you select the new mode. The change takes effect for the remainder of the session.
  </Tab>

  <Tab title="settings.json">
    Set a persistent default in your user or project settings:

    ```json  theme={null}
    {
      "defaultMode": "acceptEdits"
    }
    ```

    Valid values: `"default"`, `"acceptEdits"`, `"bypassPermissions"`, `"plan"`, `"dontAsk"`.
  </Tab>
</Tabs>

## Permission rules (allow/deny lists)

Beyond the global mode, you can create fine-grained rules that always allow or always deny specific tool calls — regardless of the active mode.

Rules have three components:

| Field         | Description                                                                            |
| ------------- | -------------------------------------------------------------------------------------- |
| `toolName`    | The name of the tool the rule applies to (e.g., `"Bash"`, `"Edit"`, `"mcp__myserver"`) |
| `ruleContent` | An optional pattern that must match the tool's input (e.g., a command prefix for Bash) |
| `behavior`    | `"allow"`, `"deny"`, or `"ask"`                                                        |

Rules are evaluated **before** the permission mode. If a matching rule exists, its behavior is applied immediately.

### Rule sources and persistence

Rules can originate from different sources and are stored accordingly:

| Source            | Where stored                  | Scope                             |
| ----------------- | ----------------------------- | --------------------------------- |
| `userSettings`    | `~/.claude/settings.json`     | All projects for the current user |
| `projectSettings` | `.claude/settings.json`       | All users of this project         |
| `localSettings`   | `.claude/settings.local.json` | Current user, this project only   |
| `session`         | In-memory                     | Current session only              |
| `cliArg`          | CLI flags                     | Current invocation only           |

### Example: always allow specific git commands

```json  theme={null}
{
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Read(*)"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)"
    ]
  }
}
```

## How bash permissions work

Bash command permission checking deserves special attention because shell commands can be complex and ambiguous.

### Pattern matching

A `Bash` permission rule with a `ruleContent` string is matched against the command using wildcard pattern matching:

* `git status` — exact match only
* `git *` — matches any `git` subcommand
* `npm run *` — matches any `npm run` script
* `*` — matches any bash command (use with extreme caution)

### Compound commands

When a bash command contains multiple sub-commands joined by `&&`, `||`, `;`, or pipes (`|`), each sub-command is checked independently. The overall permission is the most restrictive result: if any sub-command is denied, the entire compound command is blocked.

### Operator restrictions

Certain shell constructs are flagged for extra scrutiny regardless of rules:

* Output redirections (`>`, `>>`) to paths outside the project directory
* Commands that change the current directory (`cd`) to outside the working tree
* `sed -i` edit-in-place commands (handled specially to track file modifications)

### Safety checks

Regardless of the active permission mode, certain operations are always blocked or escalated:

* Commands targeting `.claude/` or `.git/` configuration directories
* Modifications to shell config files (`.bashrc`, `.zshrc`, etc.)
* Attempts to bypass path restrictions using cross-platform path tricks

<Note>
  In `auto` mode (feature-gated), safety checks marked `classifierApprovable: true` are sent to the transcript classifier rather than forcing a prompt. The classifier has visibility into the full conversation context and can decide whether the operation is appropriate.
</Note>

## MCP tool permissions

MCP tools follow the same rule system as built-in tools. You can allow or deny an entire MCP server or individual tools within a server:

```json  theme={null}
{
  "permissions": {
    "deny": [
      "mcp__myserver"
    ],
    "allow": [
      "mcp__myserver__read_database"
    ]
  }
}
```

Using `mcp__servername` as the rule (without a specific tool name) blanket-denies all tools from that server — they are filtered out of the tool list before the model even sees them.

## Safety recommendations

<Warning>
  `bypassPermissions` and `dontAsk` modes remove all safeguards. Only use them in isolated environments (containers, CI sandboxes) where Claude's actions are constrained by the environment itself, not by permission prompts.
</Warning>

* **Start with `default` mode** for any interactive session.
* **Use `plan` mode** when exploring an unfamiliar codebase or designing a large change — review the plan before enabling writes.
* **Use `acceptEdits`** for coding sessions where you trust Claude to edit files freely but want to review shell commands.
* **Prefer granular allow rules** over broad mode escalation. Allowing `Bash(git *)` is safer than switching to `bypassPermissions`.
* **Scope deny rules tightly.** A blanket `Bash(*)` deny prevents Claude from running any shell command, including safe read-only operations.
* **Review project settings files** (`.claude/settings.json`) when cloning unfamiliar repositories — they may pre-configure permission rules.


Built with [Mintlify](https://mintlify.com).