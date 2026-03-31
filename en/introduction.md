> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Introduction

> Claude Code is an AI coding agent that runs in your terminal — reading, editing, and executing code across your entire codebase.

Claude Code is a terminal-based AI agent built on Claude. It has direct access to your filesystem, shell, and tools, so you can describe a task in plain language and Claude handles the implementation end-to-end — no copy-pasting, no context switching.

## What Claude Code can do

<CardGroup cols={2}>
  <Card title="Read and edit files" icon="file-code" href="/reference/tools/file-operations">
    Claude reads source files, writes new content, and makes targeted edits. It shows diffs before applying changes so you stay in control.
  </Card>

  <Card title="Run shell commands" icon="square-terminal" href="/reference/tools/bash">
    Execute tests, build scripts, git operations, and any shell command — with configurable permission controls to keep you safe.
  </Card>

  <Card title="Search your codebase" icon="magnifying-glass" href="/reference/tools/search">
    Find files by glob pattern, search content with regular expressions, and navigate large codebases without reading every file manually.
  </Card>

  <Card title="Fetch from the web" icon="globe" href="/reference/tools/web">
    Pull documentation, read API specs, or search the web — all without leaving your terminal session.
  </Card>

  <Card title="Spawn sub-agents" icon="sitemap" href="/guides/multi-agent">
    Break complex tasks into parallel workstreams. Claude can spin up and coordinate multiple agents to work on different parts of a problem simultaneously.
  </Card>

  <Card title="Connect MCP servers" icon="plug" href="/guides/mcp-servers">
    Extend Claude's capabilities with Model Context Protocol servers for databases, APIs, internal tools, and more.
  </Card>
</CardGroup>

## Permission system

Every tool use in Claude Code goes through a permission check. You control how much autonomy Claude has.

| Mode                | Behavior                                                                                                                         |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `default`           | Claude asks before running shell commands and making edits. You approve or deny each action.                                     |
| `acceptEdits`       | File edits are applied automatically. Shell commands still require approval.                                                     |
| `plan`              | Claude produces a plan and asks for your sign-off before taking any action. Good for reviewing large changes before they happen. |
| `bypassPermissions` | All actions run without prompts. Intended for automated pipelines in sandboxed environments — not for interactive use.           |

<Warning>
  `bypassPermissions` mode disables all confirmation prompts. Only use it in isolated environments (Docker containers, CI sandboxes) where Claude cannot affect systems outside the task.
</Warning>

Set your permission mode with the `--permission-mode` flag when starting Claude, or change it with `/permissions` which also lets you manage allow/deny rules for specific tools:

```bash  theme={null}
claude --permission-mode acceptEdits
```

## CLAUDE.md memory system

Claude Code reads `CLAUDE.md` files from your repository at the start of every session. These files let you encode project-specific knowledge — build commands, coding conventions, architecture notes, required environment variables — so Claude doesn't need to re-discover them from scratch every time.

There are three scopes:

* **Project** (`CLAUDE.md` at the repo root) — shared by everyone on the team, checked into source control.
* **Personal** (`CLAUDE.local.md` at the repo root) — your private preferences for this project, gitignored.
* **Subdirectory** (`CLAUDE.md` inside a subdirectory) — loaded automatically when Claude works in that directory. Useful for monorepos with distinct modules.

Run `/init` inside any Claude Code session to generate a `CLAUDE.md` for your project automatically. Claude analyzes your codebase and produces a file with the commands and context that matter most.

```bash  theme={null}
# Inside a Claude Code session
/init
```

<Tip>
  Keep `CLAUDE.md` concise. Every line should pass this test: "Would removing this cause Claude to make mistakes?" If not, cut it.
</Tip>

## Authentication

Claude Code authenticates in two ways:

1. **OAuth (recommended)** — Sign in with your Anthropic account at `claude.ai`. Run `claude` for the first time and follow the browser prompt.
2. **API key** — Set the `ANTHROPIC_API_KEY` environment variable. Useful for CI pipelines and non-interactive sessions.

See [Authentication](/guides/authentication) for the full priority order and cloud provider options (AWS Bedrock, GCP Vertex AI).

## Ready to get started?

<CardGroup cols={2}>
  <Card title="Quickstart" icon="rocket" href="/quickstart">
    Get Claude Code running in under 5 minutes
  </Card>

  <Card title="Installation" icon="download" href="/installation">
    Detailed installation instructions for all platforms
  </Card>
</CardGroup>


Built with [Mintlify](https://mintlify.com).