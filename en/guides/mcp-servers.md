> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# MCP servers

> Extend Claude Code with Model Context Protocol servers to connect databases, APIs, and custom tools.

Model Context Protocol (MCP) is an open standard that lets Claude Code connect to external data sources and services. When you add an MCP server, Claude gains access to new tools — for example, querying a database, reading Jira tickets, or interacting with a Slack workspace.

<CardGroup cols={2}>
  <Card title="Extend Claude's capabilities" icon="plug">
    Connect Claude to any service that exposes an MCP server: databases, APIs, file systems, and more.
  </Card>

  <Card title="Project or user scope" icon="folder">
    Save server configs per-project in `.mcp.json` or globally in your user settings.
  </Card>

  <Card title="Enable and disable at runtime" icon="toggle-right">
    Turn individual servers on and off with `/mcp enable` and `/mcp disable` without editing config files.
  </Card>

  <Card title="Permission gating" icon="shield">
    Claude Code prompts before calling any MCP tool, giving you control over what actions are taken.
  </Card>
</CardGroup>

## Adding a server

The primary way to add an MCP server is via the `claude mcp add` CLI command, or by editing a config file directly.

### Via the CLI

Run `/mcp` to open the MCP management panel where you can enable, disable, and reconnect servers.

To add a server from the command line using the `claude` CLI:

```bash  theme={null}
claude mcp add <name> -- <command> [args...]
```

For example, to add the official filesystem MCP server:

```bash  theme={null}
claude mcp add filesystem -- npx -y @modelcontextprotocol/server-filesystem /tmp
```

Specify the scope with `--scope`:

```bash  theme={null}
# Save to .mcp.json in the current directory (shared with your team)
claude mcp add --scope project filesystem -- npx -y @modelcontextprotocol/server-filesystem /tmp

# Save to your user config (available in all projects)
claude mcp add --scope user my-db -- npx -y @my-org/mcp-server-postgres
```

### Via the `--mcp-config` flag

Pass a JSON config file when starting Claude Code:

```bash  theme={null}
claude --mcp-config ./my-mcp-config.json
```

This is useful for CI environments or when you want a self-contained configuration that is not persisted to any settings file.

## Config file format

MCP server configurations use JSON with a top-level `mcpServers` key. Each server entry has a name and a configuration object.

<Tabs>
  <Tab title="Stdio (local process)">
    Most MCP servers run as a local subprocess communicating over stdin/stdout:

    ```json  theme={null}
    {
      "mcpServers": {
        "filesystem": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"],
          "env": {
            "NODE_ENV": "production"
          }
        }
      }
    }
    ```

    Fields:

    * `command` — the executable to run (required)
    * `args` — array of command-line arguments
    * `env` — environment variables to pass to the process
  </Tab>

  <Tab title="HTTP (remote server)">
    For servers hosted as HTTP endpoints:

    ```json  theme={null}
    {
      "mcpServers": {
        "my-api": {
          "type": "http",
          "url": "https://mcp.example.com/v1",
          "headers": {
            "Authorization": "Bearer $MY_API_TOKEN"
          }
        }
      }
    }
    ```

    Fields:

    * `type` — set to `"http"`
    * `url` — the server URL (required)
    * `headers` — HTTP headers; values support `$VAR` environment variable expansion
  </Tab>

  <Tab title="SSE (server-sent events)">
    For servers using the SSE transport:

    ```json  theme={null}
    {
      "mcpServers": {
        "events-server": {
          "type": "sse",
          "url": "https://mcp.example.com/sse"
        }
      }
    }
    ```
  </Tab>
</Tabs>

Server names may only contain letters, numbers, hyphens, and underscores.

### Environment variable expansion

Values in `command`, `args`, `url`, and `headers` support `$VAR` and `${VAR}` syntax. Variables are expanded at startup from the shell environment. If a referenced variable is missing, Claude Code logs a warning but still attempts to connect.

## Configuration scopes

MCP configs are stored in different places depending on scope:

| Scope     | Location                                                                 | Use case                                             |
| --------- | ------------------------------------------------------------------------ | ---------------------------------------------------- |
| `project` | `.mcp.json` in the current directory (and parent directories up to root) | Team-shared server configs                           |
| `user`    | `~/.claude.json` (global config)                                         | Personal servers available everywhere                |
| `local`   | `.claude/settings.local.json` in the current project                     | Per-project personal overrides, not committed to VCS |

When the same server name appears in multiple scopes, `local` takes precedence over `project`, which takes precedence over `user`.

## Managing servers

### Enable and disable

```
/mcp enable <server-name>
/mcp disable <server-name>
/mcp enable all
/mcp disable all
```

Disabled servers remain in the config but are not connected at startup. This is useful for servers you want to keep configured but not always running.

### Reconnect a server

If a server fails to connect or you need to force a reconnection:

```
/mcp reconnect <server-name>
```

### View server status

Run `/mcp` to see a list of all configured servers and their current connection status:

* **connected** — server is running and ready to use
* **pending** — server is starting up
* **failed** — server could not connect (check the error message)
* **needs-auth** — server requires OAuth authorization
* **disabled** — server is configured but turned off

## Approving MCP tool calls

Claude Code displays a permission prompt before calling any MCP tool. The prompt shows the tool name and its input arguments, so you can review what Claude is about to do before it executes.

You can:

* **Allow once** — approve this specific call
* **Allow always** — approve all calls to this tool in this session
* **Deny** — block the call; Claude receives an error and can try a different approach

<Note>
  In auto mode (`--allowedTools`), MCP tools can be pre-approved by including their full name (formatted as `mcp__<server-name>__<tool-name>`) in the allowed tools list.
</Note>

## Example: filesystem server

The official filesystem MCP server gives Claude the ability to read and write files in directories you specify.

<Steps>
  <Step title="Add the server">
    ```bash  theme={null}
    claude mcp add --scope project filesystem -- npx -y @modelcontextprotocol/server-filesystem /home/user/projects
    ```
  </Step>

  <Step title="Verify the connection">
    Run `/mcp` and confirm `filesystem` shows as **connected**.
  </Step>

  <Step title="Use it">
    Claude can now read and write files in `/home/user/projects` using the `mcp__filesystem__read_file` and `mcp__filesystem__write_file` tools.
  </Step>
</Steps>

## Example: database server

```json  theme={null}
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "$DATABASE_URL"
      }
    }
  }
}
```

Set `DATABASE_URL` in your environment before starting Claude Code, and the MCP server will receive it automatically.

## Official MCP registry

Anthropic and the community maintain an MCP server registry at [modelcontextprotocol.io](https://modelcontextprotocol.io). Browse available servers for databases, productivity tools, cloud providers, and more.

## Troubleshooting

<AccordionGroup>
  <Accordion title="Server shows as 'failed'">
    * Check that the command exists and is executable: `which npx`
    * Run the command manually in your terminal to see if it starts without errors
    * Check that required environment variables (like API keys) are set
    * Run `claude --debug` to see detailed connection logs
  </Accordion>

  <Accordion title="MCP tools not appearing">
    A server that is connected but unauthenticated will not expose any tools. Look for a **needs-auth** status in `/mcp` and follow the OAuth flow to authorize.
  </Accordion>

  <Accordion title="Environment variable not expanded">
    Claude Code expands variables from the process environment at startup. If a variable is not set in your shell profile, it will not be available. Verify with `echo $YOUR_VAR` in the same terminal before starting `claude`.
  </Accordion>

  <Accordion title="Windows: npx fails to start">
    On Windows, `npx` requires a `cmd /c` wrapper:

    ```json  theme={null}
    {
      "mcpServers": {
        "my-server": {
          "command": "cmd",
          "args": ["/c", "npx", "-y", "@my-org/mcp-server"]
        }
      }
    }
    ```
  </Accordion>

  <Accordion title="Server works locally but not in CI">
    Use the `--mcp-config` flag to pass a config file explicitly, and ensure all referenced environment variables are set in the CI environment. For stdio servers, make sure the command (e.g., `npx`, `node`) is available in the CI PATH.
  </Accordion>
</AccordionGroup>


Built with [Mintlify](https://mintlify.com).