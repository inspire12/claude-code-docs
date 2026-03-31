> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Agent & Task Tools

> Spawn sub-agents for complex tasks and track work with a structured todo list.

Claude Code has two meta-tools for managing complex, multi-step work: `Task` (also called the Agent tool) for launching autonomous sub-agents, and `TodoWrite` for maintaining a structured task list within a session.

***

## Task (Agent tool)

Launches a specialized sub-agent to handle a complex task autonomously. The sub-agent runs independently, uses its own set of tools, and returns a single result message when it finishes.

### Parameters

<ParamField body="prompt" type="string" required>
  The full task description for the agent. Since sub-agents start with no knowledge of the parent conversation, the prompt must be self-contained — include relevant file paths, line numbers, context, and any constraints. Terse command-style prompts produce shallow results.
</ParamField>

<ParamField body="description" type="string" required>
  A short (3–5 word) label describing what the agent will do. Shown in the UI.
</ParamField>

<ParamField body="subagent_type" type="string">
  The type of specialized agent to use (e.g., `"code-reviewer"`, `"test-runner"`). When omitted, the general-purpose agent is used. Available agent types are listed in the system prompt.
</ParamField>

<ParamField body="model" type="string">
  Optional model override: `"sonnet"`, `"opus"`, or `"haiku"`. Takes precedence over the agent definition's configured model. Omit to inherit from the agent definition or parent.
</ParamField>

<ParamField body="run_in_background" type="boolean">
  When `true`, the agent runs in the background. Claude is notified when it completes and should not poll or sleep while waiting. Use background mode when you have genuinely independent work to continue in the meantime.
</ParamField>

<ParamField body="isolation" type="string">
  Isolation mode for the agent's filesystem context:

  * `"worktree"` — agent runs in a temporary git worktree (isolated copy of the repo). The worktree is cleaned up automatically if the agent makes no changes; otherwise the path and branch are returned.
</ParamField>

<ParamField body="cwd" type="string">
  Absolute path to use as the working directory for all filesystem and shell operations inside the agent. Mutually exclusive with `isolation: "worktree"`.
</ParamField>

<ParamField body="name" type="string">
  A name for the spawned agent. Makes it addressable via `SendMessage({ to: name })` while it is running.
</ParamField>

### How sub-agents work

Each agent invocation starts with zero conversation context. The agent:

1. Receives its own system prompt and the `prompt` you provide
2. Has access to a tool set defined by its `subagent_type` (or all tools for the general-purpose agent)
3. Runs autonomously until the task is complete
4. Returns a single result message back to the parent

The result is **not** shown directly to the user — Claude summarizes it in a follow-up message.

<Warning>
  Never write "based on your findings, fix the bug" or similar delegation phrases. Prompts that push synthesis back onto the agent produce generic work. Include concrete file paths, line numbers, and specific instructions in the prompt.
</Warning>

### When to use the Task tool

<Tabs>
  <Tab title="Good use cases">
    * Open-ended research that requires multiple rounds of searching and reading
    * Running a test suite and analyzing failures after code changes
    * Independent code review by a fresh agent with no prior context bias
    * Long-running operations that can proceed while Claude handles other requests
    * Tasks that would generate noisy intermediate output you don't need in the main context
  </Tab>

  <Tab title="When NOT to use it">
    * Reading a specific file — use the `Read` tool directly
    * Searching for a class definition — use `Grep` directly
    * Searching within 2–3 known files — use `Read` directly
    * Simple tasks unrelated to the available agent types
  </Tab>
</Tabs>

### Parallel agents

When tasks are independent, Claude launches multiple agents in a single message:

```
Task({ description: "Run tests", prompt: "Run the full test suite..." })
Task({ description: "Check types", prompt: "Run tsc --noEmit and report errors..." })
```

Both agents run concurrently. When both complete, Claude synthesizes the results.

### Resuming agents

To send a follow-up message to a running or completed agent, use `SendMessage` with the agent's `name` as the `to` field. The agent resumes with its full context preserved.

### Writing effective prompts

<Accordion title="Brief like a colleague joining mid-task">
  The agent knows nothing about your session. Treat the prompt like a complete briefing for a smart colleague who just arrived:

  * What you're trying to accomplish and why
  * What you've already tried or ruled out
  * The specific files, functions, or systems involved
  * What form the result should take ("report in under 200 words", "make the fix directly", etc.)
</Accordion>

<Accordion title="Lookups vs. investigations">
  * **Lookups**: hand over the exact command or query. The agent should not need to figure out what to run.
  * **Investigations**: hand over the question, not prescribed steps. Over-specifying steps for open-ended problems causes the agent to waste effort on a wrong path.
</Accordion>

***

## TodoWrite

Maintains a structured task list for the current session. Claude uses this tool proactively to track progress, organize multi-step work, and provide visibility into what it is doing.

### Parameters

<ParamField body="todos" type="TodoItem[]" required>
  The complete, updated todo list. Each call **replaces** the entire list — there is no incremental add or remove.
</ParamField>

### TodoItem schema

Each item in the `todos` array has the following fields:

<ParamField body="content" type="string" required>
  The task description in imperative form — what needs to be done. Example: `"Run tests"`, `"Fix authentication bug"`.
</ParamField>

<ParamField body="activeForm" type="string" required>
  The present-continuous form of the task description, shown while the task is in progress. Example: `"Running tests"`, `"Fixing authentication bug"`.
</ParamField>

<ParamField body="status" type="string" required>
  The current state of the task. One of:

  | Value         | Meaning                   |
  | ------------- | ------------------------- |
  | `pending`     | Not yet started           |
  | `in_progress` | Currently being worked on |
  | `completed`   | Finished successfully     |

  Exactly one task should be `in_progress` at any time.
</ParamField>

### How Claude uses TodoWrite

Claude calls `TodoWrite` proactively when:

* A task requires **3 or more distinct steps**
* Work is non-trivial and benefits from structured tracking
* The user provides a list of multiple things to do
* New sub-tasks are discovered mid-implementation

Claude does **not** use `TodoWrite` for:

* Single-step tasks
* Trivial operations (e.g., adding a comment, running one command)
* Purely informational or conversational responses

### Task lifecycle

<Accordion title="Starting work">
  Claude marks a task `in_progress` **before** beginning, not after. This ensures the UI reflects what is actively happening.

  ```json  theme={null}
  { "content": "Add dark mode toggle", "activeForm": "Adding dark mode toggle", "status": "in_progress" }
  ```
</Accordion>

<Accordion title="Completing work">
  Claude marks a task `completed` immediately when it finishes — it does not batch completions. A task is only marked complete when:

  * Implementation is fully done
  * No tests are failing
  * No unresolved errors remain
</Accordion>

<Accordion title="Handling blockers">
  If a task cannot be completed (missing file, failing test, unresolved error), Claude keeps it `in_progress` and creates a new `pending` task describing what needs to be resolved first.
</Accordion>

<Accordion title="Clearing the list">
  When all tasks are marked `completed`, Claude clears the list automatically. An empty list signals that all requested work is done.
</Accordion>

### Example list

```json  theme={null}
{
  "todos": [
    {
      "content": "Add dark mode CSS variables",
      "activeForm": "Adding dark mode CSS variables",
      "status": "completed"
    },
    {
      "content": "Implement theme context provider",
      "activeForm": "Implementing theme context provider",
      "status": "in_progress"
    },
    {
      "content": "Update existing components to use theme",
      "activeForm": "Updating existing components to use theme",
      "status": "pending"
    },
    {
      "content": "Run tests and build",
      "activeForm": "Running tests and build",
      "status": "pending"
    }
  ]
}
```


Built with [Mintlify](https://mintlify.com).