> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Commands overview

> Claude Code accepts two categories of commands: CLI flags you pass at launch, and slash commands you type during a session.

Claude Code has two categories of commands:

* **CLI flags** — options you pass when starting Claude from your terminal (e.g. `claude --model sonnet`). They configure the session before it starts.
* **Slash commands** — text commands you type inside an active session (e.g. `/help`). They control Claude's behavior while the session is running.

## Getting help

```bash  theme={null}
# Show all CLI flags
claude --help

# Inside a session, list slash commands
/help
```

`/help` lists every slash command available in the current session, including commands added by plugins and skills.

## Two types of commands

| Type           | When to use                                                                   | Example                                                |
| -------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------ |
| CLI flags      | Configure a session at launch — set the model, output format, permission mode | `claude --permission-mode acceptEdits "fix the tests"` |
| Slash commands | Interact with a running session — manage memory, switch models, commit code   | `/commit`                                              |

CLI flags are consumed once and cannot be changed mid-session (with the exception of `/model` and `/permissions`, which change the running configuration in place).

## CLI flags

Pass flags directly after `claude`:

```bash  theme={null}
claude [flags] [prompt]
```

```bash  theme={null}
# Non-interactive: print response and exit
claude -p "summarize this file" < README.md

# Set model for the session
claude --model opus

# Accept all file edits automatically
claude --permission-mode acceptEdits
```

See [CLI flags](/reference/commands/cli-flags) for the full list.

## Slash commands

Type a slash command at the input prompt inside any session:

```
/command [arguments]
```

```
/init
/compact summarize only the last three tasks
/model claude-opus-4-5
```

See [Slash commands](/reference/commands/slash-commands) for the full list.

## Keyboard shortcuts

These shortcuts work inside any interactive Claude Code session:

| Key           | Action                                                           |
| ------------- | ---------------------------------------------------------------- |
| `Ctrl+C`      | Interrupt the current response (Claude stops mid-turn)           |
| `Ctrl+D`      | Exit Claude Code                                                 |
| `Ctrl+L`      | Clear the terminal display (does not clear conversation history) |
| `Up` / `Down` | Navigate input history                                           |
| `Tab`         | Autocomplete slash command names                                 |
| `Escape`      | Cancel an in-progress permission prompt                          |

<Note>
  `Ctrl+C` interrupts the current response but keeps the conversation alive. Use `Ctrl+D` or `/exit` to end the session entirely.
</Note>

## Subcommands

In addition to the main `claude` command, a few subcommands are available in your terminal:

| Subcommand         | Description                                    |
| ------------------ | ---------------------------------------------- |
| `claude mcp`       | Configure and manage MCP servers               |
| `claude mcp serve` | Start Claude Code as an MCP server             |
| `claude doctor`    | Diagnose installation and configuration issues |
| `claude update`    | Update Claude Code to the latest version       |

```bash  theme={null}
claude mcp --help
claude doctor
```


Built with [Mintlify](https://mintlify.com).