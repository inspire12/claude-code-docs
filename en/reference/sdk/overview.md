> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# SDK overview

> Embed Claude Code in your own tools using the stdin/stdout control protocol. Reference for the SDK session API, message types, and output formats.

The Claude Code SDK is a control protocol for embedding Claude Code in other applications — IDEs, automation scripts, CI/CD pipelines, or any host that can spawn a subprocess and communicate over stdin/stdout.

Rather than exposing a library API directly, the SDK communicates with a running `claude` process over a structured JSON message stream. The host process sends user messages and control requests; the CLI process streams back assistant messages, tool progress events, and result payloads.

<Note>
  The TypeScript types described on this page are exported from `@anthropic-ai/claude-code` under the `agentSdkTypes` entry point. Control protocol types (prefixed `SDKControl`) are `@alpha` and subject to change.
</Note>

## How it works

<Steps>
  <Step title="Spawn a Claude Code process">
    Start `claude` with `--output-format stream-json` and `--print` (non-interactive mode). Pipe its stdin and stdout into your host process.

    ```bash  theme={null}
    claude --output-format stream-json --print --verbose
    ```

    For a persistent session that accepts multiple prompts over time, omit `--print` and instead send `SDKUserMessage` objects to stdin after the session initializes.
  </Step>

  <Step title="Send an initialize request">
    Write a `control_request` with `subtype: "initialize"` to stdin. The CLI responds with an `SDKControlInitializeResponse` containing available commands, models, agents, and account information.

    ```json  theme={null}
    {
      "type": "control_request",
      "request_id": "init-1",
      "request": {
        "subtype": "initialize",
        "systemPrompt": "You are a code reviewer.",
        "appendSystemPrompt": "Always suggest tests."
      }
    }
    ```
  </Step>

  <Step title="Stream messages from stdout">
    Read newline-delimited JSON from stdout. Each line is one of the `SDKMessage` union types — assistant turns, tool progress, system events, and result summaries.
  </Step>

  <Step title="Send user messages">
    Write `SDKUserMessage` objects to stdin to continue the conversation. Each message contains an Anthropic API-compatible `message` payload.
  </Step>
</Steps>

## Output formats

Pass `--output-format` to control what Claude Code writes to stdout.

| Format        | Description                                                                                         |
| ------------- | --------------------------------------------------------------------------------------------------- |
| `text`        | Plain text responses only. Default for interactive mode.                                            |
| `json`        | Single JSON object written at completion. Suitable for one-shot scripts.                            |
| `stream-json` | Newline-delimited JSON stream. One message per line, emitted as events occur. Required for SDK use. |

<Tip>
  Use `stream-json` when you need to render progress incrementally or handle tool events. Use `json` when you only care about the final result.
</Tip>

## Control protocol messages

The control protocol uses two top-level envelope types that flow bidirectionally over stdin/stdout.

### `SDKControlRequest`

Sent **to** the CLI process to configure the session or issue commands.

```json  theme={null}
{
  "type": "control_request",
  "request_id": "<unique-string>",
  "request": { "subtype": "...", ...payload }
}
```

<ResponseField name="type" type="literal: 'control_request'" required>
  Always `"control_request"`.
</ResponseField>

<ResponseField name="request_id" type="string" required>
  Unique identifier for this request. The CLI echoes it back in the corresponding `control_response`.
</ResponseField>

<ResponseField name="request" type="SDKControlRequestInner" required>
  The request payload. `subtype` identifies which operation to perform.
</ResponseField>

### `SDKControlResponse`

Emitted **from** the CLI process in response to a `control_request`.

```json  theme={null}
{
  "type": "control_response",
  "response": {
    "subtype": "success",
    "request_id": "<echoed-id>",
    "response": { ...payload }
  }
}
```

On error, `subtype` is `"error"` and the `error` field contains a human-readable message.

***

## Initialize request and response

The `initialize` request is the first control message you must send. It configures the session and returns available capabilities.

### `SDKControlInitializeRequest`

```json  theme={null}
{
  "type": "control_request",
  "request_id": "init-1",
  "request": {
    "subtype": "initialize",
    "systemPrompt": "You are a CI automation agent.",
    "appendSystemPrompt": "Always add test coverage.",
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "Bash",
          "hookCallbackIds": ["my-hook-id"]
        }
      ]
    },
    "agents": {
      "CodeReviewer": {
        "description": "Reviews code for quality and security.",
        "prompt": "You are an expert code reviewer...",
        "model": "opus"
      }
    }
  }
}
```

<ResponseField name="subtype" type="literal: 'initialize'" required>
  Identifies this as an initialize request.
</ResponseField>

<ResponseField name="systemPrompt" type="string">
  Replaces the default system prompt for this session.
</ResponseField>

<ResponseField name="appendSystemPrompt" type="string">
  Appended to the system prompt without replacing it. Use this to add context while keeping the default behavior.
</ResponseField>

<ResponseField name="hooks" type="Record<HookEvent, SDKHookCallbackMatcher[]>">
  Registers SDK-side hook callbacks. The CLI calls back into the SDK process when hook events fire. See [Hooks reference](/reference/sdk/hooks-reference).
</ResponseField>

<ResponseField name="sdkMcpServers" type="string[]">
  Names of in-process SDK MCP servers (created with `createSdkMcpServer`) to connect to this session.
</ResponseField>

<ResponseField name="agents" type="Record<string, AgentDefinition>">
  Custom subagent definitions available to the `Agent` tool during this session.

  <Expandable title="AgentDefinition fields">
    <ResponseField name="description" type="string" required>
      Natural language description of when Claude should invoke this agent.
    </ResponseField>

    <ResponseField name="prompt" type="string" required>
      System prompt for this agent.
    </ResponseField>

    <ResponseField name="model" type="string">
      Model alias (`"sonnet"`, `"opus"`, `"haiku"`) or full model ID. Defaults to inheriting the parent model.
    </ResponseField>

    <ResponseField name="tools" type="string[]">
      Allow-list of tool names. When omitted, the agent inherits all tools.
    </ResponseField>

    <ResponseField name="disallowedTools" type="string[]">
      Tools explicitly blocked for this agent.
    </ResponseField>

    <ResponseField name="maxTurns" type="number">
      Maximum agentic turns before the agent stops.
    </ResponseField>

    <ResponseField name="permissionMode" type="PermissionMode">
      Override permission mode for this agent. See [Permissions API](/reference/sdk/permissions-api).
    </ResponseField>
  </Expandable>
</ResponseField>

### `SDKControlInitializeResponse`

The CLI responds with the session's current capabilities.

```json  theme={null}
{
  "type": "control_response",
  "response": {
    "subtype": "success",
    "request_id": "init-1",
    "response": {
      "commands": [...],
      "agents": [...],
      "output_style": "stream-json",
      "available_output_styles": ["text", "json", "stream-json"],
      "models": [...],
      "account": {
        "email": "user@example.com",
        "organization": "Acme Corp",
        "apiProvider": "firstParty"
      }
    }
  }
}
```

<ResponseField name="commands" type="SlashCommand[]">
  Available slash commands (e.g., `/compact`, `/cost`). Each entry has `name`, `description`, and `argumentHint`.
</ResponseField>

<ResponseField name="agents" type="AgentInfo[]">
  Available subagent types. Each has `name`, `description`, and an optional `model`.
</ResponseField>

<ResponseField name="output_style" type="string">
  The active output format (`"stream-json"`, `"json"`, `"text"`).
</ResponseField>

<ResponseField name="models" type="ModelInfo[]">
  Available models for this account.

  <Expandable title="ModelInfo fields">
    <ResponseField name="value" type="string">
      Model identifier for API calls (e.g., `"claude-sonnet-4-6"`).
    </ResponseField>

    <ResponseField name="displayName" type="string">
      Human-readable name (e.g., `"Claude Sonnet 4.6"`).
    </ResponseField>

    <ResponseField name="supportsEffort" type="boolean">
      Whether this model supports effort levels.
    </ResponseField>

    <ResponseField name="supportsAdaptiveThinking" type="boolean">
      Whether this model supports adaptive thinking (Claude decides when and how much to think).
    </ResponseField>
  </Expandable>
</ResponseField>

<ResponseField name="account" type="AccountInfo">
  Logged-in account details.

  <Expandable title="AccountInfo fields">
    <ResponseField name="email" type="string">
      Account email address.
    </ResponseField>

    <ResponseField name="organization" type="string">
      Organization name.
    </ResponseField>

    <ResponseField name="subscriptionType" type="string">
      Subscription tier.
    </ResponseField>

    <ResponseField name="apiProvider" type="'firstParty' | 'bedrock' | 'vertex' | 'foundry'">
      Active API backend. Anthropic OAuth only applies when `"firstParty"`.
    </ResponseField>
  </Expandable>
</ResponseField>

***

## User messages

Send user messages to stdin to drive the conversation forward.

### `SDKUserMessage`

```json  theme={null}
{
  "type": "user",
  "message": {
    "role": "user",
    "content": "Refactor this function to use async/await."
  },
  "parent_tool_use_id": null
}
```

<ResponseField name="type" type="literal: 'user'" required>
  Always `"user"`.
</ResponseField>

<ResponseField name="message" type="APIUserMessage" required>
  An Anthropic API-compatible user message. `content` can be a string or a content block array (for images and other media).
</ResponseField>

<ResponseField name="parent_tool_use_id" type="string | null" required>
  Tool use ID this message is responding to, or `null` for top-level user messages.
</ResponseField>

<ResponseField name="uuid" type="string">
  Optional UUID to track this message. Echoed back in related events.
</ResponseField>

<ResponseField name="priority" type="'now' | 'next' | 'later'">
  Scheduling hint for async message queuing.
</ResponseField>

***

## SDK message stream types

Claude Code emits a stream of JSON messages to stdout. The `type` field identifies each message.

<AccordionGroup>
  <Accordion title="system — session initialization">
    Emitted once at session start with `subtype: "init"`. Contains the active model, tool list, MCP server statuses, permission mode, and session ID.

    ```json  theme={null}
    {
      "type": "system",
      "subtype": "init",
      "model": "claude-sonnet-4-6",
      "tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep"],
      "mcp_servers": [],
      "permissionMode": "default",
      "session_id": "abc123",
      "uuid": "..."
    }
    ```
  </Accordion>

  <Accordion title="assistant — model response">
    Emitted when the model produces a turn. Contains the full Anthropic API response object, including any `tool_use` blocks.

    ```json  theme={null}
    {
      "type": "assistant",
      "message": { "role": "assistant", "content": [...] },
      "parent_tool_use_id": null,
      "uuid": "...",
      "session_id": "abc123"
    }
    ```
  </Accordion>

  <Accordion title="stream_event — partial streaming tokens">
    Emitted during streaming with `RawMessageStreamEvent` payloads. Use these to render incremental output.
  </Accordion>

  <Accordion title="tool_progress — long-running tool status">
    Emitted periodically for tools that take more than a few seconds (e.g., Bash commands). Contains `tool_name`, `tool_use_id`, and elapsed time.
  </Accordion>

  <Accordion title="result — final turn summary">
    Emitted at the end of each turn. `subtype` is `"success"` or one of the error subtypes.

    ```json  theme={null}
    {
      "type": "result",
      "subtype": "success",
      "result": "The function has been refactored.",
      "duration_ms": 4200,
      "total_cost_usd": 0.0042,
      "num_turns": 3,
      "is_error": false,
      "stop_reason": "end_turn",
      "session_id": "abc123",
      "uuid": "..."
    }
    ```

    Error subtypes: `"error_during_execution"`, `"error_max_turns"`, `"error_max_budget_usd"`, `"error_max_structured_output_retries"`.
  </Accordion>

  <Accordion title="system — status updates">
    Emitted with `subtype: "status"` when the permission mode or session status changes (e.g., `"compacting"`).
  </Accordion>
</AccordionGroup>

***

## Other control requests

Beyond `initialize`, the control protocol exposes these operations.

| `subtype`             | Direction  | Description                                                           |
| --------------------- | ---------- | --------------------------------------------------------------------- |
| `interrupt`           | host → CLI | Interrupt the current turn.                                           |
| `set_permission_mode` | host → CLI | Change the active permission mode.                                    |
| `set_model`           | host → CLI | Switch to a different model mid-session.                              |
| `can_use_tool`        | CLI → host | Permission request for a tool call (requires SDK permission handler). |
| `mcp_status`          | host → CLI | Get MCP server connection statuses.                                   |
| `mcp_set_servers`     | host → CLI | Replace dynamically managed MCP servers.                              |
| `get_context_usage`   | host → CLI | Get context window usage breakdown.                                   |
| `get_settings`        | host → CLI | Read the effective merged settings.                                   |
| `apply_flag_settings` | host → CLI | Merge settings into the flag settings layer.                          |
| `rewind_files`        | host → CLI | Revert file changes made since a given message.                       |
| `hook_callback`       | CLI → host | Deliver a hook event for an SDK-registered hook callback.             |
| `reload_plugins`      | host → CLI | Reload plugins from disk.                                             |

***

## Session management API

For scripting scenarios, the SDK exports functions that operate on saved session transcripts stored in `~/.claude/`.

```typescript  theme={null}
import {
  query,
  listSessions,
  getSessionInfo,
  getSessionMessages,
  forkSession,
  renameSession,
  tagSession,
} from '@anthropic-ai/claude-code'
```

<AccordionGroup>
  <Accordion title="query — run a prompt">
    The primary SDK entry point. Accepts a `prompt` string or `AsyncIterable<SDKUserMessage>` and returns an async iterable of `SDKMessage`.

    ```typescript  theme={null}
    for await (const message of query({
      prompt: 'What files are in this directory?',
      options: { cwd: '/my/project' }
    })) {
      if (message.type === 'result') {
        console.log(message.result)
      }
    }
    ```
  </Accordion>

  <Accordion title="listSessions — list saved sessions">
    Returns session metadata for a project directory. Pass `dir` to scope to a specific project, or omit to list all sessions.

    ```typescript  theme={null}
    const sessions = await listSessions({ dir: '/my/project', limit: 50 })
    ```
  </Accordion>

  <Accordion title="getSessionMessages — read a transcript">
    Parses the JSONL transcript file for a session and returns messages in chronological order.

    ```typescript  theme={null}
    const messages = await getSessionMessages(sessionId, {
      dir: '/my/project',
      includeSystemMessages: false,
    })
    ```
  </Accordion>

  <Accordion title="forkSession — branch a conversation">
    Copies a session's transcript into a new session with remapped UUIDs. Supports `upToMessageId` to fork from a specific point.

    ```typescript  theme={null}
    const { sessionId: newId } = await forkSession(originalSessionId, {
      upToMessageId: 'msg-uuid',
      title: 'Experimental branch',
    })
    ```
  </Accordion>

  <Accordion title="renameSession and tagSession">
    ```typescript  theme={null}
    await renameSession(sessionId, 'My refactor session')
    await tagSession(sessionId, 'needs-review')
    await tagSession(sessionId, null) // clear tag
    ```
  </Accordion>
</AccordionGroup>

***

## Use cases

<Tabs>
  <Tab title="IDE integration">
    IDEs can spawn a persistent Claude Code process and route messages through the control protocol. Send the `initialize` request with a custom `systemPrompt` that describes the IDE context, then forward user messages from the editor's chat panel. Use `PreToolUse` hook callbacks to intercept file edits and display diffs in the IDE's native UI before they are applied.
  </Tab>

  <Tab title="CI/CD automation">
    In CI pipelines, use `--output-format json` and `--print` for one-shot tasks:

    ```bash  theme={null}
    result=$(echo "Review the diff and output pass/fail" | \
      claude --output-format json --print \
             --permission-mode bypassPermissions)
    ```

    Parse the `result` field from the JSON output to extract the agent's response programmatically.
  </Tab>

  <Tab title="Headless agents">
    For long-running background agents, use `query()` from the TypeScript SDK with a streaming output format. Combine with `watchScheduledTasks` (internal) to fire tasks on cron schedules while keeping a persistent daemon process that maintains the WebSocket connection to claude.ai.
  </Tab>
</Tabs>


Built with [Mintlify](https://mintlify.com).