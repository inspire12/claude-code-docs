> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# CLAUDE.md

> How to write and organize CLAUDE.md files to give Claude persistent, project-specific instructions.

`CLAUDE.md` files let you encode project knowledge that Claude loads at the start of every session. Instead of explaining your project's conventions, build system, and architecture every time, you write them once and Claude reads them automatically.

## What belongs in CLAUDE.md

Write instructions that would cause mistakes if missing. Everything else is noise.

**Include:**

* Build, test, and lint commands (exact invocations, not just tool names)
* Architecture decisions that affect how code should be written or organized
* Coding conventions specific to your project (naming patterns, file structure rules)
* Environment setup requirements (required env vars, expected services)
* Common pitfalls or patterns Claude should know to avoid
* Monorepo structure and which packages own which responsibilities

**Omit:**

* Things Claude already knows (standard TypeScript syntax, common library APIs)
* Obvious reminders ("write clean code", "add comments")
* Sensitive data — API keys, passwords, tokens, or secrets of any kind
* Information that changes frequently and will become stale

<Tip>
  The test: would removing this line cause Claude to make a mistake on your codebase? If not, cut it.
</Tip>

## File locations

Claude discovers memory files by traversing from your current directory up to the filesystem root. Files closer to your current directory have higher priority (they are loaded later, so the model pays more attention to them).

| File                                | Type    | Purpose                                                       |
| ----------------------------------- | ------- | ------------------------------------------------------------- |
| `/etc/claude-code/CLAUDE.md`        | Managed | System-wide instructions for all users, set by administrators |
| `~/.claude/CLAUDE.md`               | User    | Your personal global instructions, applied to every project   |
| `~/.claude/rules/*.md`              | User    | Modular global rules, each file loaded separately             |
| `CLAUDE.md` (project root)          | Project | Shared team instructions, checked into source control         |
| `.claude/CLAUDE.md` (project root)  | Project | Alternative location for shared project instructions          |
| `.claude/rules/*.md` (project root) | Project | Modular project rules, organized by topic                     |
| `CLAUDE.local.md` (project root)    | Local   | Your private project-specific instructions, not checked in    |

<Note>
  `.claude/rules/` directories support subdirectories. All `.md` files found recursively are loaded as separate memory entries.
</Note>

## Loading order and priority

Files are loaded in the following order. Because the model attends more to content that appears later in context, later files have higher effective priority.

<Steps>
  <Step title="Managed memory">
    `/etc/claude-code/CLAUDE.md` and `/etc/claude-code/rules/*.md` — global policy set by administrators. Always loaded; cannot be excluded by users.
  </Step>

  <Step title="User memory">
    `~/.claude/CLAUDE.md` and `~/.claude/rules/*.md` — your personal global instructions for all projects.
  </Step>

  <Step title="Project memory (root-to-CWD)">
    `CLAUDE.md`, `.claude/CLAUDE.md`, and `.claude/rules/*.md` files in each directory from the filesystem root down to your current directory. Files in directories closer to your CWD are loaded later (higher priority).
  </Step>

  <Step title="Local memory">
    `CLAUDE.local.md` in each directory from root to CWD. Same traversal order. Gitignored by default.
  </Step>
</Steps>

## The @include directive

Memory files can include other files using `@` notation. Included files are processed recursively and inserted before the file that references them.

```markdown  theme={null}
# CLAUDE.md

@./docs/architecture.md
@~/shared/style-guide.md
@/etc/company-standards.md
```

**Supported path formats:**

| Syntax             | Resolves to                                                      |
| ------------------ | ---------------------------------------------------------------- |
| `@filename`        | Relative to the current file's directory (same as `@./filename`) |
| `@./relative/path` | Relative to the current file's directory                         |
| `@~/path/in/home`  | Path under your home directory                                   |
| `@/absolute/path`  | Absolute filesystem path                                         |

**Behavior:**

* Paths with a fragment (`#heading`) have the fragment stripped before resolving
* Non-existent files are silently ignored
* Circular references are detected and prevented
* Includes nest up to 5 levels deep
* Only text file formats are supported — binary files (images, PDFs) are skipped
* `@include` inside code blocks is not processed

<Warning>
  External includes (files outside the project directory) require explicit approval the first time they are loaded. Claude will prompt you to confirm before fetching content from outside your project.
</Warning>

## Frontmatter path targeting

Files in `.claude/rules/` can use YAML frontmatter to restrict which file paths they apply to. This lets you load rules conditionally based on the file Claude is working with.

```markdown  theme={null}
---
paths:
  - "src/api/**"
  - "*.graphql"
---

# API conventions

All API handlers must validate input using the shared `validate()` helper.
GraphQL resolvers must not perform direct database queries — use the data layer.
```

Rules without frontmatter apply unconditionally. Rules with `paths` frontmatter only apply when Claude is working in files matching the glob patterns.

## Example CLAUDE.md for a TypeScript project

```markdown  theme={null}
# Project: Payments API

## Build and test

- Build: `bun run build`
- Tests: `bun test` (uses Bun's built-in test runner — do not use Jest)
- Lint: `bun run lint` (biome, not eslint)
- Type check: `bun run typecheck`

Always run `bun run typecheck` before considering a change complete.

## Architecture

- `src/handlers/` — HTTP handlers, one file per route group
- `src/services/` — Business logic, no direct DB access
- `src/db/` — Database layer (Drizzle ORM); all queries live here
- `src/schemas/` — Zod schemas shared between handler validation and DB types

Handlers call services. Services call the DB layer. Never skip layers.

## Conventions

- Use `z.object().strict()` for all input validation schemas
- Errors propagate as `Result<T, AppError>` — never throw in service code
- All monetary values are integers in cents
- Timestamps are Unix seconds (number), not Date objects

## Environment

Required env vars: `DATABASE_URL`, `STRIPE_SECRET_KEY`, `JWT_SECRET`
Local dev: copy `.env.example` to `.env.local` and fill in values

## Common mistakes to avoid

- Do not use `new Date()` directly — use `getCurrentTimestamp()` from `src/utils/time.ts`
- Do not add `console.log` — use the `logger` from `src/utils/logger.ts`
- Do not write raw SQL — use the Drizzle query builder
```

## Generating CLAUDE.md with /init

Run `/init` in any Claude Code session to automatically generate a `CLAUDE.md` for your project. Claude analyzes your codebase and produces a file containing the commands and context most relevant to the project.

```
/init
```

The generated file is a starting point. Review it, trim anything that isn't genuinely useful, and add project-specific knowledge that Claude couldn't infer from the code.

## Editing CLAUDE.md with /memory

Run `/memory` to open the memory editor, which lists all currently loaded memory files and lets you edit them directly within the session.

```
/memory
```

Changes take effect immediately — Claude reloads the updated file and applies the new instructions to the current session.

## Excluding files

If you have CLAUDE.md files you want to prevent Claude from loading (for example, in vendored dependencies or generated code), add exclusion patterns to your settings:

```json  theme={null}
{
  "claudeMdExcludes": [
    "/absolute/path/to/vendor/CLAUDE.md",
    "**/generated/**",
    "**/third-party/.claude/rules/**"
  ]
}
```

Patterns are matched against absolute paths using picomatch. Only User, Project, and Local memory types can be excluded — Managed (administrator) files are always loaded.

See [Settings](/configuration/settings) for full documentation on the `claudeMdExcludes` option.


Built with [Mintlify](https://mintlify.com).