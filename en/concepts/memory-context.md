> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Memory and context (CLAUDE.md)

> How Claude Code discovers, loads, and prioritises CLAUDE.md memory files to give Claude persistent, project-aware instructions.

Claude Code supports a layered memory system based on plain Markdown files. By writing instructions into `CLAUDE.md` files at different levels of your filesystem, you can customise Claude's behaviour globally, per-project, or per-user — and keep some instructions private while committing others to source control.

## The 4-level memory hierarchy

Memory files are loaded in the following order (from lowest to highest priority). Files loaded **later** take precedence because the model pays more attention to instructions that appear later in the context window.

<Steps>
  <Step title="Managed memory (lowest priority)">
    **Path:** `/etc/claude-code/CLAUDE.md` (or the platform equivalent configured by your organisation)

    System-wide instructions set by an administrator or deployment tool. Applies to all users on the machine. Also supports a `rules/` subdirectory at `managed claude rules dir`. Cannot be overridden by user or project files when enforced via policy settings.
  </Step>

  <Step title="User memory">
    **Path:** `~/.claude/CLAUDE.md` and `~/.claude/rules/*.md`

    Your private global instructions that apply to every project. A good place for personal preferences like preferred code style, default language, or your git username. This file is never committed to any repository.
  </Step>

  <Step title="Project memory">
    **Paths (checked in each ancestor directory, from root down to CWD):**

    * `CLAUDE.md`
    * `.claude/CLAUDE.md`
    * `.claude/rules/*.md`

    Instructions checked into the codebase and shared with the whole team. Ideal for project conventions, architecture notes, testing commands, and anything that every contributor should follow. Committed to source control.
  </Step>

  <Step title="Local memory (highest priority)">
    **Path:** `CLAUDE.local.md` (checked in each ancestor directory)

    Private project-specific overrides. This file should be added to `.gitignore`. Use it for personal workflow preferences that apply only to this project — local environment paths, personal shortcuts, or instructions you don't want to share.
  </Step>
</Steps>

<Info>
  Files closer to the current working directory are loaded **later** and therefore have **higher priority**. A `CLAUDE.md` in your project root outweighs one in a parent directory.
</Info>

## File discovery algorithm

When Claude Code starts, it walks the filesystem from the current working directory **up to the filesystem root**, collecting memory files at each level. The walk is performed by `getMemoryFiles()` in `utils/claudemd.ts`.

The discovery order ensures lower-priority files appear first in the assembled context:

1. Managed files are loaded first.
2. User files are loaded next.
3. Project and local files are loaded by iterating the path from the **root downward to CWD** — so ancestor directories come before child directories.

<Note>
  The full list of discovered files is memoised for the duration of the conversation. Use `/memory` to open the memory editor and force a reload, or restart the session to pick up changes made outside Claude Code.
</Note>

## The `@include` directive

Memory files can reference other files using `@` notation. The referenced file is inserted into the context as a separate entry before the including file.

```markdown  theme={null}
# My project CLAUDE.md

@./docs/architecture.md
@./docs/conventions/typescript.md

Always run `bun test` before committing.
```

**Supported path forms:**

| Syntax             | Resolves to                                       |
| ------------------ | ------------------------------------------------- |
| `@filename`        | Relative path from the including file's directory |
| `@./relative/path` | Explicit relative path                            |
| `@~/home/path`     | Path relative to the user's home directory        |
| `@/absolute/path`  | Absolute path                                     |

**Rules:**

* `@include` paths inside fenced code blocks and inline code spans are **ignored** — only plain text nodes are processed.
* Circular references are detected and skipped.
* Non-existent files are silently ignored.
* Maximum include depth is **5 levels**.
* Only text-based file types are included (`.md`, `.ts`, `.py`, `.json`, etc.). Binary files such as images and PDFs are skipped.

<Warning>
  By default, `@include` paths that point outside the current project directory require explicit approval. Claude Code will display a warning and ask for confirmation the first time an external include is detected.
</Warning>

## `.claude/rules/*.md` — granular rule files

Instead of putting everything in one large `CLAUDE.md`, you can split instructions across multiple Markdown files inside `.claude/rules/`:

```
my-project/
├── CLAUDE.md
└── .claude/
    └── rules/
        ├── testing.md
        ├── typescript-style.md
        └── git-workflow.md
```

All `.md` files in `.claude/rules/` (and subdirectories) are loaded automatically for the project and user memory levels. Rules files also support **path-scoped frontmatter**:

```markdown  theme={null}
---
paths:
  - "src/api/**"
  - "src/services/**"
---

Always use dependency injection. Never import concrete implementations directly.
```

When `paths` is set, the rule file is only injected into context when Claude is working on a file that matches one of the glob patterns. This keeps context lean for large projects.

## Maximum file size

The recommended maximum for any single memory file is **40 000 characters** (`MAX_MEMORY_CHARACTER_COUNT`). Files that exceed this limit are flagged, and Claude may not read the full content. Keep memory files focused and concise.

## How memory affects Claude's behaviour

When memory files are loaded, they are assembled into a single context block prefixed with:

> *"Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written."*

This means instructions in CLAUDE.md files take precedence over Claude's built-in defaults. Use this to enforce project conventions, restrict certain operations, or inject domain knowledge.

## When to use each level

<AccordionGroup>
  <Accordion title="Use managed memory for…">
    Organisation-wide policies, security guardrails, or deployment-specific configuration that every user on a machine must follow. Managed by system administrators, not individual developers.
  </Accordion>

  <Accordion title="Use user memory (~/.claude/CLAUDE.md) for…">
    Personal preferences that apply across all projects: your preferred language for responses, commit message style, editor shortcuts, or personal aliases. This is private and never committed.
  </Accordion>

  <Accordion title="Use project memory (CLAUDE.md) for…">
    Project conventions shared with your team: how to run tests, build commands, architecture decisions, naming conventions, PR checklist items, or links to internal documentation. Commit this file.
  </Accordion>

  <Accordion title="Use local memory (CLAUDE.local.md) for…">
    Your personal overrides for a specific project: local environment paths, personal debugging notes, or workflow preferences that differ from the team defaults. Add this to `.gitignore`.
  </Accordion>
</AccordionGroup>

## Example CLAUDE.md

```markdown  theme={null}
# My Project

## Commands

- Build: `bun run build`
- Test: `bun test`
- Lint: `bun run lint`
- Type check: `bun run typecheck`

## Architecture

This is a monorepo. Core logic lives in `packages/core`. The CLI entrypoint is `src/index.ts`.

## Conventions

- Use `zod/v4` for all schema validation.
- Never use `any` — prefer `unknown` with a type guard.
- All async functions must handle errors explicitly; never swallow exceptions.
- Tests go in `__tests__/` next to the source file they test.

## Git

- Branch names: `feat/<ticket>-short-description`
- Always run `bun test && bun run typecheck` before committing.
- Do not include `console.log` statements in committed code.

@./docs/api-conventions.md
```

## The `/memory` command

Run `/memory` inside the Claude Code REPL to open the memory file editor. It shows you which memory files are currently loaded, lets you edit them directly, and reloads the context when you save.

<Tip>
  You can ask Claude directly: *"Add a rule to CLAUDE.md that we always use 2-space indentation."* Claude will locate the appropriate memory file and write the instruction for you.
</Tip>

## Disabling memory loading

| Method                             | Effect                                                                                                                    |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` | Disables all memory file loading entirely                                                                                 |
| `--bare` flag                      | Skips auto-discovery of memory files from the CWD walk; only loads files from explicitly provided `--add-dir` directories |
| `claudeMdExcludes` setting         | Glob patterns of memory file paths to skip (e.g., to exclude a noisy ancestor `CLAUDE.md`)                                |


Built with [Mintlify](https://mintlify.com).