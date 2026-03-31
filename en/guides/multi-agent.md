> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Multi-agent workflows

> Parallelize complex tasks by having Claude spawn and coordinate sub-agents.

Claude Code can spawn sub-agents — separate Claude instances that run independently to complete tasks in parallel. This lets you tackle large, multi-step work by dividing it across specialized agents that run concurrently, rather than doing everything sequentially in a single conversation.

## How sub-agents work

When Claude uses the `Agent` tool, it launches a new Claude instance with its own context, system prompt, and tool permissions. The parent Claude waits for the agent to complete (or continues other work if the agent is running in the background), then receives the agent's result as a single message.

Each agent:

* Starts with a fresh context window (unless it's a fork)
* Gets a specialized system prompt based on its agent type
* Has its own tool permissions (configurable per agent type)
* Can itself spawn further sub-agents, though nesting is limited

## The Agent tool

Claude uses the `Agent` tool to spawn sub-agents. As a user, you don't call this tool directly — Claude decides when to use it. You can see when Claude spawns agents: they appear in the terminal with their own progress indicators.

The tool accepts:

* `description` — a 3-5 word summary of what the agent will do (shown in the UI)
* `prompt` — the full task description for the agent
* `subagent_type` — which specialized agent type to use (optional; defaults to general-purpose)
* `run_in_background` — whether to run asynchronously so Claude can continue other work
* `isolation` — `"worktree"` to give the agent an isolated git worktree

## When Claude uses sub-agents

Claude spawns agents when a task benefits from parallelism or specialization:

* **Independent parallel tasks** — writing tests while also updating documentation
* **Specialized work** — using a code-reviewer agent for a security audit
* **Long-running tasks** — background research while Claude works on something else
* **Isolated exploration** — forking itself to explore a solution without polluting the main context

Claude does not spawn agents for simple tasks, small file reads, or anything it can complete directly in a few tool calls.

## Requesting multi-agent workflows

You can ask Claude to use multiple agents explicitly:

```
Run the linter and the test suite in parallel.
```

```
Use separate agents to research how three competing libraries handle this problem,
then summarize the findings.
```

```
Have an agent review the security implications of this code change while you
continue implementing the feature.
```

<Tip>
  When asking Claude to run agents "in parallel", it will send a single message with multiple Agent tool calls, launching all of them simultaneously. Be explicit about what should run in parallel vs. what must be sequential.
</Tip>

## Foreground vs. background agents

By default, agents run in the **foreground**: Claude waits for each agent to complete before proceeding. Use foreground agents when Claude needs the results to continue its work.

Background agents run asynchronously. Claude launches them and continues with other work. You receive a notification when the agent completes.

```
Run the integration tests in the background while you implement the next feature.
```

Claude will not poll or check on background agents — it continues working and receives the result as a notification when the agent finishes.

<Note>
  Background agents are shown in a tasks panel alongside the main conversation. You can see their progress and cancel them if needed.
</Note>

## Coordinator mode

In coordinator mode, Claude acts as an orchestrator that delegates all implementation work to sub-agents and focuses on planning, routing, and synthesis. This is useful for very large tasks where you want Claude to manage the overall workflow while sub-agents do the hands-on work.

Coordinator mode is primarily used in multi-agent team setups. In a standard single-user session, Claude decides for itself when to delegate.

## Agent memory and context isolation

Each sub-agent starts with a **clean context window**. The parent Claude provides the full task description and any relevant background in the agent's prompt — the agent does not automatically inherit the parent's conversation history.

This means:

* Agents are independent; they cannot read the parent's conversation
* The parent must provide sufficient context in the prompt for the agent to succeed
* Results from agents are returned as a single response, not streamed turn by turn

### Persistent agent memory

Some agent types have persistent memory across invocations. Memory is stored in markdown files:

* **User scope**: `~/.claude/agent-memory/<agent-type>/MEMORY.md` — shared across all projects
* **Project scope**: `.claude/agent-memory/<agent-type>/MEMORY.md` — shared with your team
* **Local scope**: `.claude/agent-memory-local/<agent-type>/MEMORY.md` — local only, not committed

When an agent type has memory configured, it reads and writes this file to remember things across sessions. This is useful for agents that learn your preferences over time.

## Worktree isolation

Set `isolation: "worktree"` to give an agent its own git worktree — an isolated copy of the repository. Changes the agent makes do not affect your working directory until you merge them.

```
Implement this feature in an isolated worktree so I can review the changes before merging.
```

If the agent makes changes, the worktree path and branch are returned in the result so you can inspect them. If the agent makes no changes, the worktree is cleaned up automatically.

## Monitoring sub-agent progress

While agents run, you can see:

* The agent's description and elapsed time in the progress display
* Tool calls the agent is making (shown in transcript mode with Ctrl+O)
* A notification when the agent completes

For background agents, progress updates appear in the tasks panel. Claude summarizes the agent's findings when reporting back to you.

<Warning>
  Do not ask Claude to check on a running background agent's output file directly. Claude receives a completion notification automatically. Polling the output file mid-run pulls the agent's internal tool noise into Claude's context, which is counterproductive.
</Warning>

## Writing effective agent prompts

Sub-agents start with zero context from the parent conversation. Claude should — and you can prompt Claude to — write agent prompts that are self-contained briefs.

A good agent prompt includes:

* What the agent is trying to accomplish and why
* Relevant file paths, function names, or data
* What the agent should report back (format, length, specific questions to answer)
* What the agent should *not* do (scope constraints)
* What has already been tried or ruled out

A weak prompt:

```
Based on your findings, fix the bug.
```

A stronger prompt:

```
Fix the null reference bug in UserService.getProfile() in src/services/user.ts:247.
The bug occurs when the user has no associated profile record — getProfile() calls
profile.preferences without checking if profile is null first. Add a null check and
return a default preferences object { theme: 'light', notifications: true }.
Run npm test afterward to confirm the fix passes existing tests.
```

## Best practices

<AccordionGroup>
  <Accordion title="Give agents independent, parallelizable tasks">
    Agents are most valuable when their work does not depend on each other's output. If task B requires the result of task A, they must be sequential — but tasks A and C can run in parallel while B waits for A.
  </Accordion>

  <Accordion title="Provide complete context in the prompt">
    Paste relevant code snippets, file paths, function signatures, or error messages directly into the agent prompt. Never assume the agent will infer what it needs from minimal instructions.
  </Accordion>

  <Accordion title="Be specific about the expected output format">
    If you need a concise summary, say "report in under 200 words." If you need structured output, describe the format. Vague prompts produce vague results.
  </Accordion>

  <Accordion title="Use worktree isolation for risky changes">
    When an agent will make significant file changes, use `isolation: "worktree"` so you can review before merging. This is especially useful for refactoring or large feature work.
  </Accordion>

  <Accordion title="Don't over-parallelize">
    Spawning many agents for small tasks adds overhead without benefit. Claude decides when agents are worthwhile; trust its judgment unless you have a specific parallel structure in mind.
  </Accordion>
</AccordionGroup>

## Limitations and safety

* Sub-agents have their own permission modes. By default, they use `acceptEdits` mode.
* Agents can be denied by permission rules using `Agent(AgentName)` syntax in deny rules.
* Background agents are not linked to the parent's abort controller — pressing Escape cancels the parent turn but not running background agents. Use the tasks panel to cancel background agents explicitly.
* Sub-agents cannot spawn other teammates (the team roster is flat). They can spawn further sub-agents using the Agent tool.
* Fork agents (agents that inherit the parent's context) cannot themselves fork — Claude prevents recursive forking.
* Agent results are capped at 100,000 characters before being returned to the parent.


Built with [Mintlify](https://mintlify.com).