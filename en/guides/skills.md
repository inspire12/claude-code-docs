> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Skills

> Create reusable on-demand capabilities that Claude invokes with a slash command.

Skills are markdown files that define reusable prompts and workflows. When you type `/skill-name` in Claude Code, Claude loads that skill's instructions and executes the described task. Skills are useful for any workflow you repeat across sessions — running a deployment, writing a changelog, reviewing a PR, or applying your team's specific coding conventions.

## How skills work

A skill is a directory inside `.claude/skills/` with a `SKILL.md` file. When you invoke `/skill-name`, Claude Code loads the skill's `SKILL.md` as the prompt for that action. The skill can include instructions, context, constraints, and even inline shell commands that run at invocation time.

Skills load lazily — they are only read when invoked, so having many skills defined doesn't affect startup time or context size.

## Creating a skill

<Steps>
  <Step title="Create the skill directory">
    Skills live in a `skills` subdirectory inside any `.claude/` directory:

    ```bash  theme={null}
    mkdir -p .claude/skills/my-skill
    ```

    You can put skills in:

    * `.claude/skills/` (project-level, relative to your working directory)
    * `~/.claude/skills/` (user-level, available in all projects)
  </Step>

  <Step title="Write SKILL.md">
    Create `.claude/skills/my-skill/SKILL.md` with your skill's instructions:

    ```markdown  theme={null}
    ---
    description: Run the full release process for this project
    argument-hint: version number (e.g. 1.2.3)
    ---

    Release the project at version $ARGUMENTS.

    Steps:
    1. Update the version in `package.json` to $ARGUMENTS
    2. Update CHANGELOG.md with a new section for this version
    3. Run `npm test` and confirm all tests pass
    4. Commit with message "chore: release v$ARGUMENTS"
    5. Create a git tag `v$ARGUMENTS`
    ```
  </Step>

  <Step title="Invoke the skill">
    ```
    /my-skill 1.2.3
    ```

    Claude loads the skill and executes the instructions, with `1.2.3` substituted for `$ARGUMENTS`.
  </Step>
</Steps>

## Skill frontmatter

The frontmatter at the top of `SKILL.md` configures how the skill behaves. All fields are optional.

```yaml  theme={null}
---
description: A short description shown in /skills and used by Claude to decide when to use it
argument-hint: what to pass as the argument (shown in autocomplete)
allowed-tools: Bash, Write, Read
when_to_use: Use this skill when the user asks to create a new component
model: claude-sonnet-4-6
user-invocable: true
context: fork
---
```

| Field            | Description                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------ |
| `description`    | Short description shown in `/skills` and used by Claude to decide when to invoke the skill |
| `argument-hint`  | Hint shown in slash command autocomplete describing what argument to pass                  |
| `allowed-tools`  | Comma-separated list of tools this skill is permitted to use (defaults to all)             |
| `when_to_use`    | Prose description of when Claude should use this skill proactively                         |
| `model`          | Model to use for this skill (e.g., `claude-sonnet-4-6`); defaults to the session model     |
| `user-invocable` | Set to `false` to hide the skill from the slash command list (still available to Claude)   |
| `context`        | `fork` to run the skill in an isolated subagent context                                    |
| `paths`          | Glob patterns; skill only activates when matching files are touched                        |
| `version`        | Skill version string                                                                       |
| `hooks`          | Hooks scoped to this skill's execution (same format as settings hooks)                     |

## Argument substitution

Use `$ARGUMENTS` anywhere in `SKILL.md` to insert the text passed after the slash command:

```markdown  theme={null}
Create a new React component named $ARGUMENTS following the project's conventions.
```

```
/new-component UserProfile
```

For named arguments, use `$ARG_NAME` syntax by listing arguments in frontmatter:

```yaml  theme={null}
---
arguments: [name, directory]
---
```

Then reference them as `$name` and `$directory` in the body.

## Inline shell commands

Skills can embed shell commands that execute at invocation time using backtick injection syntax. The output is inserted into the prompt before Claude sees it:

```markdown  theme={null}
---
description: Review recent changes
---

Here are the recent commits for context:

!`git log --oneline -20`

Review the changes above and summarize what was accomplished.
```

The `!` prefix followed by a backtick-quoted command runs the command and replaces the block with its output. This is useful for injecting live project state into the skill prompt.

<Warning>
  Inline shell commands execute with the same permissions as your shell. They run when you invoke the skill, not when the skill is loaded at startup.
</Warning>

## Listing skills

Run `/skills` to see all available skills:

```
/skills
```

This shows skills from all scopes (project, user, managed) along with their descriptions.

## Namespaced skills

Skills in subdirectories are namespaced with colons:

```
.claude/skills/
  deployment/
    SKILL.md      → /deployment
  database/
    migrate/
      SKILL.md    → /database:migrate
    seed/
      SKILL.md    → /database:seed
```

## Conditional skills (path-based activation)

Add a `paths` frontmatter field to activate a skill only when you work with matching files:

```yaml  theme={null}
---
description: Django model review
paths: "**/*.py"
when_to_use: Use when editing Django model files
---
```

The skill is loaded into Claude's context automatically when you read, write, or edit a file matching the glob pattern. This keeps skills out of context until they're relevant.

## Bundled skills

Claude Code ships with built-in skills that are always available. These are compiled into the binary and register themselves at startup. You can see them in `/skills` — they appear with source `bundled`.

Bundled skills include capabilities like:

* Project onboarding assistance
* Common code review workflows
* Agentic search and analysis patterns

Bundled skills follow the same interface as user-defined skills and can include reference files that are extracted to disk on first invocation.

## User-level skills

Skills in `~/.claude/skills/` are available in every project without needing to add them to each repository. This is a good place for personal workflows that span projects.

```bash  theme={null}
mkdir -p ~/.claude/skills/standup
cat > ~/.claude/skills/standup/SKILL.md << 'EOF'
---
description: Summarize what I worked on today for a standup update
---

Look at my git commits from today across this repository and summarize them in standup format: what I did, what I'm doing next, and any blockers. Keep it to 3-4 sentences.
EOF
```

## Skills vs. hooks

| Feature       | Skills                                                           | Hooks                                             |
| ------------- | ---------------------------------------------------------------- | ------------------------------------------------- |
| Invocation    | Explicit: `/skill-name` or by Claude when it recognizes the need | Automatic: fires on tool events                   |
| When to use   | Repeatable workflows you want to trigger intentionally           | Side effects, formatting, linting, blocking       |
| Configuration | `SKILL.md` in `.claude/skills/`                                  | `hooks` field in settings JSON                    |
| Context       | Can include files, shell output, and detailed instructions       | Receives event JSON, returns exit code and output |

Use skills when you want a named, repeatable action. Use hooks when you want something to happen automatically every time a specific event occurs.

## Example: custom component generator

```markdown  theme={null}
---
description: Generate a new React component with tests
argument-hint: ComponentName
allowed-tools: Write, Bash
---

Create a new React component named $ARGUMENTS.

1. Create `src/components/$ARGUMENTS/$ARGUMENTS.tsx` with:
   - A functional component using TypeScript
   - Props interface named `$ARGUMENTSProps`
   - JSDoc comment describing the component
   - Default export

2. Create `src/components/$ARGUMENTS/$ARGUMENTS.test.tsx` with:
   - At least one rendering test using React Testing Library
   - A snapshot test

3. Create `src/components/$ARGUMENTS/index.ts` that re-exports the component.

4. Run `npx tsc --noEmit` to confirm no type errors.
```

Invoke with:

```
/new-component Button
```


## Best practices

For detailed guidance on writing effective Skills — including structure, naming, progressive disclosure, workflows, evaluation, and common patterns — see [Skill authoring best practices](skills-best-practices.md).

---

Built with [Mintlify](https://mintlify.com).