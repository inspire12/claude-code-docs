> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Tools

> Reference for every built-in tool Claude Code can use, including file operations, shell execution, web access, and sub-agent spawning.

Claude Code gives Claude a set of built-in tools it can call to interact with your machine. Each tool call is subject to the active [permission mode and rules](/concepts/permissions). Below is a reference for every tool available by default.

## File tools

<CardGroup cols={2}>
  <Card title="Read" icon="file-lines">
    **Read a file from the local filesystem.**

    Reads up to 2 000 lines by default. Supports an `offset` and `limit` for targeted reads of large files. Returns content in `cat -n` format with line numbers.

    Also supports reading images (PNG, JPG, etc.) as visual input, PDF files (up to 20 pages at a time), and Jupyter notebooks (`.ipynb`).

    Read-only. Always auto-approved in `default` mode.
  </Card>

  <Card title="Edit" icon="file-pen">
    **Perform exact string replacements in a file.**

    Requires a prior `Read` call on the file in the same conversation. Replaces `old_string` with `new_string` — the match must be unique in the file. Use `replace_all: true` to rename across the entire file.

    Fails if `old_string` appears more than once (unless `replace_all` is set). This precision prevents unintended edits.
  </Card>

  <Card title="Write" icon="file-arrow-up">
    **Create a new file or completely overwrite an existing one.**

    For existing files, a prior `Read` call is required in the same conversation. Prefer `Edit` for modifying existing files — `Write` sends the entire file content and is better suited for new files or full rewrites.
  </Card>

  <Card title="Glob" icon="folder-magnifying-glass">
    **Find files by name pattern.**

    Fast pattern matching that works on any codebase size. Returns matching file paths sorted by modification time (most recently modified first).

    Supports patterns like `**/*.ts`, `src/**/*.test.js`, `**/CLAUDE.md`. For open-ended multi-step searches, use the `Task` tool instead.

    Read-only. Always auto-approved.
  </Card>
</CardGroup>

## Shell tool

<Card title="Bash" icon="terminal">
  **Execute a shell command in a persistent shell session.**

  Runs the command in a shell that persists across tool calls within a conversation — environment variables and working-directory changes carry over between calls. Supports a `timeout` parameter (default and max values are configurable).

  Key behaviours:

  * **Compound commands** (`&&`, `||`, `;`, `|`) are parsed and each sub-command is permission-checked independently.
  * **Background execution** — pass `run_in_background: true` to run a long-running command without blocking. You are notified when it completes.
  * **Output limits** — stdout/stderr is truncated if it exceeds the per-tool result size budget; a preview and file path are returned instead.
  * **Search commands** (`find`, `grep`, `rg`) — for content search, prefer the dedicated `Grep` tool, which has optimised permissions and access.

  Subject to permission prompts in `default` mode. Auto-approved in `acceptEdits` mode only for commands covered by an allow rule.
</Card>

## Search tools

<CardGroup cols={2}>
  <Card title="Grep" icon="magnifying-glass">
    **Search file contents using regular expressions.**

    Built on ripgrep. Supports full regex syntax (`log.*Error`, `function\s+\w+`), file-type filtering (`*.ts`, `**/*.py`), and three output modes:

    * `files_with_matches` (default) — returns only file paths
    * `content` — returns matching lines with context
    * `count` — returns match counts per file

    Multiline patterns are supported with `multiline: true`.

    Read-only. Always auto-approved.
  </Card>

  <Card title="LS" icon="list">
    **List directory contents.**

    Returns files and subdirectories in a structured format. Useful for exploring project structure before reading or editing files.

    Read-only. Always auto-approved.

    <Note>In bare-minimum mode (`CLAUDE_CODE_SIMPLE=1`) only `Bash`, `Read`, and `Edit` are available — use `ls` via `Bash` instead.</Note>
  </Card>
</CardGroup>

## Web tools

<CardGroup cols={2}>
  <Card title="WebFetch" icon="globe">
    **Fetch a URL and extract information from it.**

    Takes a URL and a prompt describing what to extract. Converts HTML to Markdown, then passes the content through a secondary model to produce a focused answer.

    Features:

    * HTTP URLs are automatically upgraded to HTTPS.
    * Includes a 15-minute self-cleaning cache — repeated fetches of the same URL are fast.
    * When a URL redirects to a different host, the tool returns the redirect URL for a follow-up request.
    * For GitHub URLs, prefer the `gh` CLI via `Bash` (e.g., `gh pr view`, `gh api`).

    Read-only. Prompts for approval in `default` mode.
  </Card>

  <Card title="WebSearch" icon="magnifying-glass-arrow-right">
    **Search the web and return results.**

    Returns search results with titles, snippets, and URLs formatted as Markdown links. Useful for accessing information beyond the model's training cutoff.

    After answering, Claude automatically appends a `Sources:` section listing all referenced URLs.

    Domain filtering is supported to include or exclude specific sites. Currently only available in the US.

    Prompts for approval in `default` mode.
  </Card>
</CardGroup>

## Agent and task tools

<CardGroup cols={2}>
  <Card title="Task (Agent)" icon="robot">
    **Spawn a sub-agent to complete a task.**

    Starts a nested agentic loop in a separate context. The sub-agent has its own conversation history, its own tool set (optionally restricted), and runs to completion before returning a result to the parent agent.

    Sub-agents can run:

    * **Locally** — in-process, sharing the parent's filesystem and shell
    * **Remotely** — on separate compute when remote agent eligibility criteria are met

    Use `Task` for open-ended multi-step searches, parallel workstreams, or delegating distinct sub-problems to isolated agents.

    The parent agent receives the sub-agent's final output as a tool result.
  </Card>

  <Card title="TodoWrite" icon="list-check">
    **Create and manage a structured task list.**

    Writes a list of todo items with statuses (`pending`, `in_progress`, `completed`) to a persistent panel in the terminal UI. Helps Claude track progress on complex multi-step tasks and shows you what it is working on.

    Use proactively for tasks with 3 or more distinct steps. Not necessary for simple single-step requests.

    Results are rendered in the todo panel, not the conversation transcript.
  </Card>
</CardGroup>

## MCP tools

MCP (Model Context Protocol) servers can expose additional tools to Claude Code. Connected tools appear in the tool list alongside built-in tools and follow the same permission system.

MCP tools are named with a `mcp__` prefix:

```
mcp__<server-name>__<tool-name>
```

For example, a tool named `query` on a server named `mydb` appears as `mcp__mydb__query`.

Common MCP tool categories include:

* Database query and management tools
* Browser and web automation tools
* Cloud provider APIs (AWS, GCP, Azure)
* Issue tracker integrations (GitHub, Linear, Jira)
* Internal company tools and APIs

<Info>
  MCP servers are configured in `~/.claude/mcp_servers.json` or `.claude/mcp_servers.json`. Tools from a connected server are automatically included in Claude's context once the server is running.
</Info>

To deny all tools from a specific MCP server, add a deny rule for the server prefix:

```json  theme={null}
{
  "permissions": {
    "deny": ["mcp__untrusted-server"]
  }
}
```

## Notebook tool

<Card title="NotebookEdit" icon="book">
  **Edit cells in a Jupyter notebook.**

  Allows Claude to insert, replace, or delete cells in a `.ipynb` file with line-level precision. Read notebooks using the standard `Read` tool, which returns all cells and their outputs.
</Card>

## Tool availability

Not all tools are available in every context. The active tool set is determined at startup and can be affected by:

* `CLAUDE_CODE_SIMPLE=1` — restricts to `Bash`, `Read`, and `Edit` only
* Permission deny rules — tools blanket-denied by a rule are removed from the tool list before the model sees them
* `isEnabled()` checks — each tool can self-disable based on environment conditions (e.g., `WebSearch` is gated by region)
* MCP server connection state — MCP tools are only available when the server is running and connected

You can inspect the active tool set with the `/tools` command in the REPL.


Built with [Mintlify](https://mintlify.com).