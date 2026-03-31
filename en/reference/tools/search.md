> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Search

> Find files by name pattern and search file contents with regular expressions.

Claude Code has two dedicated search tools: `Glob` for finding files by name and `Grep` for searching file contents. Both are preferred over running `find` or `grep` as Bash commands — they have correct permission integration, return results sorted by modification time, and produce structured output Claude can act on directly.

***

## Glob

Finds files whose paths match a glob pattern. Results are sorted by modification time (most recently modified first).

### Parameters

<ParamField body="pattern" type="string" required>
  A glob pattern to match file paths against. Supports `*` (any characters within a path segment), `**` (any number of path segments), and `?` (single character).
</ParamField>

<ParamField body="path" type="string">
  The directory to search in. Defaults to the current working directory. Must be an existing directory path — omit the parameter entirely to use the default; do not pass `null` or `"undefined"`.
</ParamField>

### Output

Returns a list of matching file paths, relative to the working directory, sorted by modification time. Results are capped at **100 files**. If the result is truncated, consider narrowing the pattern or specifying a more specific `path`.

### Pattern examples

| Pattern                      | Matches                                         |
| ---------------------------- | ----------------------------------------------- |
| `**/*.ts`                    | All TypeScript files anywhere in the tree       |
| `src/**/*.tsx`               | All TSX files under `src/`                      |
| `*.json`                     | JSON files in the current directory only        |
| `**/{package,tsconfig}.json` | `package.json` and `tsconfig.json` at any depth |
| `tools/*/prompt.ts`          | `prompt.ts` one level inside `tools/`           |

<Tip>
  Use `Glob` when you know the filename or extension you're looking for. For open-ended exploration that requires multiple rounds of searching, use the `Task` (Agent) tool instead.
</Tip>

***

## Grep

Searches file contents using regular expressions, powered by ripgrep. Returns matching file paths, line numbers, and (optionally) matching lines.

### Parameters

<ParamField body="pattern" type="string" required>
  A regular expression to search for in file contents. Supports full regex syntax: character classes, quantifiers, lookaheads, alternation, etc.

  Ripgrep syntax differs from POSIX `grep` in one notable way: literal braces must be escaped. Use `interface\{\}` to find `interface{}` in Go code.
</ParamField>

<ParamField body="path" type="string">
  File or directory to search in. Defaults to the current working directory. Can be a single file path to restrict the search.
</ParamField>

<ParamField body="glob" type="string">
  Glob pattern to filter which files are searched (e.g., `"*.js"`, `"*.{ts,tsx}"`). Maps to `rg --glob`.
</ParamField>

<ParamField body="type" type="string">
  Ripgrep file type to restrict the search (e.g., `"js"`, `"py"`, `"rust"`, `"go"`, `"java"`). More efficient than `glob` for standard file type filtering. Maps to `rg --type`.
</ParamField>

<ParamField body="output_mode" type="string">
  Controls what is returned. Options:

  * `"files_with_matches"` *(default)* — returns file paths only
  * `"content"` — returns matching lines with context support
  * `"count"` — returns match counts per file
</ParamField>

<ParamField body="-i" type="boolean">
  Case-insensitive search. Maps to `rg -i`.
</ParamField>

<ParamField body="-n" type="boolean">
  Show line numbers in output. Applies to `output_mode: "content"` only. Defaults to `true`.
</ParamField>

<ParamField body="-A" type="number">
  Lines of context to show after each match. Applies to `output_mode: "content"` only. Maps to `rg -A`.
</ParamField>

<ParamField body="-B" type="number">
  Lines of context to show before each match. Applies to `output_mode: "content"` only. Maps to `rg -B`.
</ParamField>

<ParamField body="context" type="number">
  Lines of context to show before and after each match (equivalent to `rg -C`). Takes precedence over `-A` and `-B`. Applies to `output_mode: "content"` only.
</ParamField>

<ParamField body="multiline" type="boolean">
  Enable multiline mode where `.` matches newlines and patterns can span multiple lines (`rg -U --multiline-dotall`). Default: `false`.
</ParamField>

<ParamField body="head_limit" type="number">
  Limit output to the first N lines or entries (equivalent to `| head -N`). Applies across all output modes. Defaults to **250**. Pass `0` for unlimited results.
</ParamField>

<ParamField body="offset" type="number">
  Skip the first N entries before applying `head_limit` (equivalent to `| tail -n +N`). Useful for paginating large result sets. Defaults to `0`.
</ParamField>

### Output modes

<Tabs>
  <Tab title="files_with_matches (default)">
    Returns a list of file paths that contain at least one match, sorted by modification time.

    ```
    Found 3 files
    src/utils/permissions/bashClassifier.ts
    src/tools/BashTool/BashTool.tsx
    src/tools/BashTool/bashPermissions.ts
    ```
  </Tab>

  <Tab title="content">
    Returns the matching lines themselves, with optional surrounding context.

    ```
    src/utils/permissions/bashClassifier.ts:42:export function classifyBashCommand(
    src/utils/permissions/bashClassifier.ts:43:  command: string,
    ```
  </Tab>

  <Tab title="count">
    Returns match counts per file.

    ```
    src/utils/permissions/bashClassifier.ts:7
    src/tools/BashTool/bashPermissions.ts:3

    Found 10 total occurrences across 2 files.
    ```
  </Tab>
</Tabs>

### Pattern syntax notes

<Accordion title="Escaping braces for Go / TypeScript">
  Ripgrep treats `{` and `}` as literal characters by default, unlike some regex engines that use them for quantifiers. To search for literal braces, escape them:

  ```
  interface\{\}    →  finds  interface{}
  map\[string\]    →  finds  map[string]
  ```
</Accordion>

<Accordion title="Multiline patterns">
  By default, `.` does not match newlines. To search across line boundaries:

  ```json  theme={null}
  {
    "pattern": "struct \\{[\\s\\S]*?field",
    "multiline": true
  }
  ```
</Accordion>

<Accordion title="Combining glob and type filters">
  `glob` and `type` can be used independently or together:

  ```json  theme={null}
  { "pattern": "TODO", "type": "ts" }
  ```

  ```json  theme={null}
  { "pattern": "TODO", "glob": "src/**/*.ts" }
  ```

  Use `type` for standard language types; use `glob` when the file set doesn't align with a named type.
</Accordion>

### Excluded directories

Grep automatically excludes version-control metadata directories: `.git`, `.svn`, `.hg`, `.bzr`, `.jj`, `.sl`.


Built with [Mintlify](https://mintlify.com).