> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Slash commands

> Commands you type inside a running Claude Code session. Type / at the input prompt to activate them.

Slash commands are typed at the Claude Code input prompt during an active session. Every command starts with `/`.

```
/command [arguments]
```

<Tip>
  Type `/help` at any time to see all commands available in the current session, including any added by plugins and skills.
</Tip>

## Quick reference

| Command                        | Description                                                          |
| ------------------------------ | -------------------------------------------------------------------- |
| [`/init`](#init)               | Generate `CLAUDE.md` files and optional skills/hooks for the project |
| [`/memory`](#memory)           | Edit Claude memory files (global, project, local)                    |
| [`/config`](#config)           | Open the settings panel                                              |
| [`/hooks`](#hooks)             | View hook configurations for tool events                             |
| [`/mcp`](#mcp)                 | Manage MCP servers — enable, disable, reconnect                      |
| [`/permissions`](#permissions) | Manage allow and deny rules for tools                                |
| [`/plan`](#plan)               | Enable plan mode or open/describe the current session plan           |
| [`/model`](#model)             | Set the AI model for the current session                             |
| [`/commit`](#commit)           | Create a git commit with an AI-generated message                     |
| [`/review`](#review)           | Review a pull request                                                |
| [`/skills`](#skills)           | List available skills                                                |
| [`/compact`](#compact)         | Summarize conversation history to reduce context usage               |
| [`/clear`](#clear)             | Clear conversation history and free up context                       |
| [`/help`](#help)               | Show help and available commands                                     |
| [`/login`](#login)             | Sign in or switch Anthropic accounts                                 |
| [`/logout`](#logout)           | Sign out from your Anthropic account                                 |

***

## Project and memory commands

<AccordionGroup>
  <Accordion title="/init">
    **Syntax:** `/init`

    Analyzes your codebase and sets up `CLAUDE.md` file(s), and optionally skills and hooks. Claude surveys key project files — manifests, CI config, build scripts, README — then interviews you to fill in any gaps before writing the output files.

    What it sets up, depending on your choices:

    * **Project `CLAUDE.md`** — team-shared instructions checked into source control. Covers build/test/lint commands, coding conventions, architecture notes, and non-obvious gotchas.
    * **Personal `CLAUDE.local.md`** — your private preferences for this project (gitignored). Covers your role, sandbox URLs, communication preferences.
    * **Skills** (`.claude/skills/<name>/SKILL.md`) — on-demand workflows you or Claude invoke with `/<skill-name>`.
    * **Hooks** (`.claude/settings.json`) — deterministic shell commands that run automatically on tool events (e.g. format on every edit).

    **Example:**

    ```
    /init
    ```

    <Tip>
      Run `/init` again at any time. If `CLAUDE.md` already exists, Claude will propose specific changes rather than overwriting the file.
    </Tip>
  </Accordion>

  <Accordion title="/memory">
    **Syntax:** `/memory`

    Opens an interactive editor for Claude's memory files. Memory files are loaded into every session and persist across conversations.

    The three memory scopes are:

    | Scope   | File                              | Who it applies to                       |
    | ------- | --------------------------------- | --------------------------------------- |
    | Global  | `~/.claude/CLAUDE.md`             | You, across all projects                |
    | Project | `CLAUDE.md` at project root       | Everyone on the team                    |
    | Local   | `CLAUDE.local.md` at project root | You, for this project only (gitignored) |

    **Example:**

    ```
    /memory
    ```
  </Accordion>
</AccordionGroup>

***

## Configuration commands

<AccordionGroup>
  <Accordion title="/config">
    **Syntax:** `/config`

    **Alias:** `/settings`

    Opens the configuration panel where you can view and edit Claude Code settings, including model preferences, theme, verbose mode, and more.

    **Example:**

    ```
    /config
    ```
  </Accordion>

  <Accordion title="/hooks">
    **Syntax:** `/hooks`

    Displays the hook configurations currently active for this session. Hooks are shell commands that run automatically when tool events occur (e.g. after every file edit, before a Bash command).

    **Example:**

    ```
    /hooks
    ```

    <Note>
      To create or edit hooks, use `/init` or edit `.claude/settings.json` directly. See the [hooks guide](/guides/hooks) for the full schema.
    </Note>
  </Accordion>

  <Accordion title="/mcp">
    **Syntax:** `/mcp [enable|disable [server-name]]`

    Manages MCP (Model Context Protocol) servers for the current session. Without arguments, opens the MCP management panel. With arguments, enables or disables specific servers.

    | Argument                  | Effect                            |
    | ------------------------- | --------------------------------- |
    | *(none)*                  | Open the MCP management panel     |
    | `enable`                  | Enable all disabled MCP servers   |
    | `enable <server-name>`    | Enable a specific server by name  |
    | `disable`                 | Disable all active MCP servers    |
    | `disable <server-name>`   | Disable a specific server by name |
    | `reconnect <server-name>` | Reconnect to a specific server    |

    **Examples:**

    ```
    /mcp
    /mcp enable
    /mcp enable my-database-server
    /mcp disable analytics-server
    /mcp reconnect filesystem
    ```

    <Note>
      To add or remove MCP servers permanently, use the `claude mcp` CLI subcommand or edit your MCP config file. Changes made with `/mcp enable`/`disable` apply for the current session only.
    </Note>
  </Accordion>

  <Accordion title="/permissions">
    **Syntax:** `/permissions`

    **Alias:** `/allowed-tools`

    Opens the permissions panel where you can view and manage allow and deny rules for tools. Rules control which tools Claude can use without prompting (allow rules) and which are blocked entirely (deny rules).

    **Example:**

    ```
    /permissions
    ```

    Rules use glob-style patterns:

    ```
    Bash(git:*)         # allow all git commands
    Bash(npm:*)         # allow all npm commands
    Edit(src/**/*.ts)   # allow edits to TypeScript files in src/
    ```

    See [permissions](/concepts/permissions) for the full rule syntax.
  </Accordion>

  <Accordion title="/model">
    **Syntax:** `/model [model]`

    Sets the AI model used for the rest of the session. Without an argument, opens an interactive model picker. With a model name or alias, switches immediately.

    | Argument            | Effect                                |
    | ------------------- | ------------------------------------- |
    | *(none)*            | Open the interactive model picker     |
    | `sonnet`            | Switch to the latest Claude Sonnet    |
    | `opus`              | Switch to the latest Claude Opus      |
    | `haiku`             | Switch to the latest Claude Haiku     |
    | `claude-sonnet-4-6` | Switch to a specific model by full ID |

    **Examples:**

    ```
    /model
    /model sonnet
    /model claude-opus-4-5
    ```
  </Accordion>
</AccordionGroup>

***

## Session management commands

<AccordionGroup>
  <Accordion title="/plan">
    **Syntax:** `/plan [open|<description>]`

    Enables plan mode or manages the current session plan. In plan mode, Claude produces a written plan before taking any action and waits for your approval.

    | Argument        | Effect                                       |
    | --------------- | -------------------------------------------- |
    | *(none)*        | Toggle plan mode on/off                      |
    | `open`          | Open and display the current plan            |
    | `<description>` | Create a new plan with the given description |

    **Examples:**

    ```
    /plan
    /plan open
    /plan refactor the auth module to use JWT
    ```

    <Tip>
      Plan mode is equivalent to `--permission-mode plan` at launch. Use it when you want to review what Claude intends to do before any files are touched.
    </Tip>
  </Accordion>

  <Accordion title="/compact">
    **Syntax:** `/compact [instructions]`

    Summarizes the conversation history and replaces it with a condensed version in context. Use this when the context window is filling up and you want to continue working without starting a new session.

    An optional argument lets you give Claude specific instructions for how to summarize.

    **Examples:**

    ```
    /compact
    /compact focus only on the database schema changes
    /compact summarize the last three completed tasks
    ```
  </Accordion>

  <Accordion title="/clear">
    **Syntax:** `/clear`

    **Aliases:** `/reset`, `/new`

    Clears the entire conversation history and frees up context, starting a fresh session in the same working directory. Unlike `/compact`, this removes all history rather than summarizing it.

    **Example:**

    ```
    /clear
    ```
  </Accordion>

  <Accordion title="/skills">
    **Syntax:** `/skills`

    Lists all skills available in the current session. Skills are on-demand capabilities defined in `.claude/skills/` that you or Claude can invoke with `/<skill-name>`.

    **Example:**

    ```
    /skills
    ```
  </Accordion>
</AccordionGroup>

***

## Git commands

<AccordionGroup>
  <Accordion title="/commit">
    **Syntax:** `/commit`

    Creates a git commit using AI-generated commit message. Claude reads the current git status and diff, analyzes staged and unstaged changes, and drafts a concise commit message that focuses on the "why" rather than the "what". It then stages the relevant files and creates the commit.

    Claude follows the existing commit message style in the repository and applies these safety rules:

    * Never amends existing commits (always creates a new commit)
    * Never skips hooks (`--no-verify`)
    * Never commits files that likely contain secrets (`.env`, credentials files)
    * Does not create empty commits when there are no changes

    **Example:**

    ```
    /commit
    ```

    <Note>
      `/commit` only has access to `git add`, `git status`, and `git commit`. It cannot push, rebase, or run other git operations.
    </Note>
  </Accordion>

  <Accordion title="/review">
    **Syntax:** `/review [PR-number]`

    Runs an AI code review on a pull request using the GitHub CLI (`gh`). Without a PR number, Claude runs `gh pr list` to show open PRs. With a PR number, it fetches the PR details and diff, then provides a structured review covering:

    * Overview of what the PR does
    * Code quality and style analysis
    * Specific improvement suggestions
    * Potential issues or risks
    * Performance, test coverage, and security considerations

    **Examples:**

    ```
    /review
    /review 142
    ```

    <Note>
      `/review` requires the [GitHub CLI](https://cli.github.com/) (`gh`) to be installed and authenticated.
    </Note>
  </Accordion>
</AccordionGroup>

***

## Account and help commands

<AccordionGroup>
  <Accordion title="/help">
    **Syntax:** `/help`

    Shows help and lists all slash commands available in the current session, including built-in commands, skill commands, and any commands added by installed plugins.

    **Example:**

    ```
    /help
    ```
  </Accordion>

  <Accordion title="/login">
    **Syntax:** `/login`

    Signs in to your Anthropic account or switches between accounts. Opens a browser-based OAuth flow if not already authenticated, or presents the account switcher if you are.

    **Example:**

    ```
    /login
    ```
  </Accordion>

  <Accordion title="/logout">
    **Syntax:** `/logout`

    Signs out from your Anthropic account. After logging out, Claude Code will prompt you to authenticate again on the next session.

    **Example:**

    ```
    /logout
    ```
  </Accordion>
</AccordionGroup>

***

## Custom skill commands

When you or a plugin author creates a skill in `.claude/skills/<skill-name>/SKILL.md`, it becomes available as `/<skill-name>` in any session where that skill is loaded.

```
/verify
/deploy staging
/fix-issue 123
```

Run `/skills` to see all loaded skills and their descriptions. See the [skills guide](/guides/skills) for how to create your own.


Built with [Mintlify](https://mintlify.com).