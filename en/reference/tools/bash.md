> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Bash

> Execute shell commands in a persistent bash session.

The `Bash` tool runs shell commands and returns their output. Commands execute in a persistent bash session — the working directory is preserved between calls, though the shell state (variables, aliases, functions) is re-initialized from the user's profile on each invocation.

<Note>
  Claude prefers dedicated tools (`Read`, `Edit`, `Glob`, `Grep`) over Bash equivalents for file operations. Bash is reserved for tasks where no purpose-built tool applies.
</Note>

## Parameters

<ParamField body="command" type="string" required>
  The shell command to execute. Supports all standard bash syntax including pipes, redirections, and compound operators (`&&`, `||`, `;`).
</ParamField>

<ParamField body="timeout" type="number">
  Maximum execution time in milliseconds. Defaults to **120,000 ms (2 minutes)**. Maximum allowed value is **600,000 ms (10 minutes)**.

  Both values can be overridden at runtime via `BASH_DEFAULT_TIMEOUT_MS` and `BASH_MAX_TIMEOUT_MS` environment variables.
</ParamField>

<ParamField body="description" type="string">
  A short human-readable description of what the command does. Shown in the UI when Claude explains its actions.
</ParamField>

<ParamField body="run_in_background" type="boolean">
  When `true`, the command runs in the background. Claude is notified when it completes and does not need to poll for results. Do not append `&` to the command when using this parameter — background execution is handled internally.
</ParamField>

## Session behavior

The bash session persists working directory across calls but does **not** persist:

* Shell variables set in previous commands
* Aliases or functions
* `cd` changes (Claude uses absolute paths instead)

Claude initializes each session from the user's shell profile (`bash` or `zsh`).

## Timeouts

| Setting         | Default             | Max |
| --------------- | ------------------- | --- |
| Default timeout | 120,000 ms (2 min)  | —   |
| Maximum timeout | 600,000 ms (10 min) | —   |

Pass a `timeout` value (in milliseconds) to override the default for a specific command.

## Permission model

Every Bash command passes through a permission check before execution. The outcome is one of:

* **Allow** — command matches an existing allow rule or is classified as read-only
* **Ask** — command requires user approval before running
* **Deny** — command matches an explicit deny rule and is blocked

### Allow rules

Rules are matched by exact command, prefix (`git commit:*`), or wildcard pattern. Safe wrapper commands (`timeout`, `time`, `nice`, `nohup`) and safe environment variable assignments (`NODE_ENV=prod`, `GOARCH=arm64`) are stripped before matching so prefix rules work correctly.

### Read-only auto-allow

Commands that only read state (e.g., `ls`, `cat`, `git log`, `grep`) are automatically allowed without a prompt.

### Security restrictions

Claude's security layer checks each command for patterns that commonly indicate injection or obfuscation attempts. Commands containing the following require explicit user approval:

* Command substitution: `` `...` ``, `$()`, `${}`
* Process substitution: `<()`, `>()`
* Input/output redirections (`<`, `>`) in suspicious contexts
* Zsh-specific constructs: `zmodload`, `emulate -c`, `ztcp`, `zsocket`
* ANSI-C quoting (`$'...'`) or locale quoting (`$"..."`)
* IFS variable manipulation (`$IFS`)
* Access to `/proc/*/environ`
* Newlines or carriage returns in non-quoted positions

<Warning>
  Commands are split into subcommands before permission checking. A rule like `Bash(ls:*)` will **not** match `ls /tmp && rm -rf /` — the `rm -rf /` subcommand is checked separately.
</Warning>

## Background execution

<Tabs>
  <Tab title="Foreground (default)">
    ```json  theme={null}
    {
      "command": "npm run build",
      "description": "Build the project"
    }
    ```

    Claude waits for the command to finish and receives its output immediately.
  </Tab>

  <Tab title="Background">
    ```json  theme={null}
    {
      "command": "npm run dev",
      "description": "Start dev server",
      "run_in_background": true
    }
    ```

    The command starts and Claude continues without blocking. A notification arrives when the process finishes.
  </Tab>
</Tabs>

## Examples

<Accordion title="Running tests">
  ```bash  theme={null}
  pytest tests/ -x --tb=short
  ```

  Run a test suite and stop on the first failure.
</Accordion>

<Accordion title="Git operations">
  ```bash  theme={null}
  git status && git diff --staged
  ```

  Check working tree status and staged changes in a single compound command.

  <Note>
    Claude follows a strict protocol for `git commit` and `git push` — it never amends pushed commits, never skips hooks with `--no-verify`, and never force-pushes to `main`/`master` without explicit user instruction.
  </Note>
</Accordion>

<Accordion title="Installing dependencies">
  ```bash  theme={null}
  npm install
  ```

  Standard package installation. Claude stages related commands in parallel when they are independent.
</Accordion>

<Accordion title="Long-running server with timeout override">
  ```json  theme={null}
  {
    "command": "cargo build --release",
    "timeout": 300000,
    "description": "Release build (up to 5 min)"
  }
  ```

  Override the 2-minute default for commands known to take longer.
</Accordion>

## Tool preference

Claude uses Bash only when no purpose-built tool is available:

| Task                          | Preferred tool |
| ----------------------------- | -------------- |
| Find files by name            | `Glob`         |
| Search file contents          | `Grep`         |
| Read a file                   | `Read`         |
| Edit a file                   | `Edit`         |
| Write a new file              | `Write`        |
| Shell commands, builds, tests | **Bash**       |


Built with [Mintlify](https://mintlify.com).