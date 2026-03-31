> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Quickstart

> Install Claude Code, authenticate, and complete your first coding task in under 5 minutes.

## Prerequisites

* Node.js 18 or higher
* npm

To check your Node.js version:

```bash  theme={null}
node --version
```

If you need to install or upgrade Node.js, download it from [nodejs.org](https://nodejs.org).

***

## Step 1: Install Claude Code

Install the package globally with npm:

```bash  theme={null}
npm install -g @anthropic-ai/claude-code
```

Verify the installation:

```bash  theme={null}
claude --version
```

***

## Step 2: Authenticate

Run `claude` from any directory to start the first-time setup:

```bash  theme={null}
claude
```

On first run, Claude Code opens your browser and walks you through signing in with your Anthropic account. Once complete, your credentials are stored securely and reused for future sessions.

**Alternatively**, set an API key directly if you prefer not to use OAuth:

```bash  theme={null}
export ANTHROPIC_API_KEY=sk-ant-...
```

<Note>
  API keys take priority when the `ANTHROPIC_API_KEY` environment variable is set. For interactive use, OAuth is recommended because it handles token refresh automatically.
</Note>

***

## Step 3: Navigate to your project

Claude Code works within your current directory. Navigate to any project you want to work on:

```bash  theme={null}
cd my-project
```

***

## Step 4: Start an interactive session

Run `claude` to open an interactive session:

```bash  theme={null}
claude
```

You'll see a prompt where you can type tasks in plain language. Claude reads your project files, runs commands, and makes changes based on what you describe.

**Example first tasks to try:**

```
> explain the structure of this codebase
> add input validation to the signup form
> write tests for the UserService class
> find all places where we catch and swallow errors
```

Claude shows you what it plans to do before making changes that require permission. You can approve, deny, or ask for a different approach at each step.

***

## Step 5: Initialize a CLAUDE.md file

Run `/init` inside your session to generate a `CLAUDE.md` file for your project:

```
/init
```

Claude analyzes your repository — reading manifest files, existing documentation, and code structure — and produces a `CLAUDE.md` with the commands and context it needs to work effectively in your codebase. You should review and edit the generated file before committing it.

<Tip>
  `CLAUDE.md` is loaded at the start of every session. It's the best place to capture things like non-standard build commands, testing quirks, and coding conventions specific to your team.
</Tip>

***

## Running a non-interactive command

Use the `-p` flag to run a single task and print the result, without entering an interactive session. This is useful for scripting or one-off queries:

```bash  theme={null}
claude -p "explain this codebase"
```

```bash  theme={null}
claude -p "list all TODO comments and the files they appear in"
```

```bash  theme={null}
claude -p "check for unused exports in src/"
```

***

## Key slash commands

Once you're inside an interactive session, these slash commands are the most useful to know:

| Command        | Description                                                 |
| -------------- | ----------------------------------------------------------- |
| `/help`        | Show available commands and keyboard shortcuts              |
| `/init`        | Generate or update a `CLAUDE.md` for the current project    |
| `/memory`      | View and edit memory files (`CLAUDE.md`, `CLAUDE.local.md`) |
| `/permissions` | View or change the current permission mode                  |
| `/mcp`         | Manage connected MCP servers                                |
| `/clear`       | Clear the current conversation context                      |
| `/exit`        | End the session                                             |

***

## Next steps

<CardGroup cols={2}>
  <Card title="Core concepts" icon="lightbulb" href="/concepts/how-it-works">
    Understand how Claude Code thinks, plans, and acts on tasks
  </Card>

  <Card title="Permission modes" icon="shield" href="/concepts/permissions">
    Learn how to configure how much autonomy Claude has
  </Card>

  <Card title="CLAUDE.md reference" icon="file-text" href="/configuration/claudemd">
    Write effective memory files for your projects
  </Card>

  <Card title="MCP servers" icon="plug" href="/guides/mcp-servers">
    Extend Claude with databases, APIs, and internal tools
  </Card>
</CardGroup>


Built with [Mintlify](https://mintlify.com).