> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# File Operations

> Read, edit, and write files on the local filesystem.

Claude Code has three dedicated tools for working with files: `Read` for reading content, `Edit` for precise in-place edits, and `Write` for creating or fully rewriting files.

<Tip>
  Prefer `Edit` over `Write` when modifying existing files — `Edit` sends only the changed fragment, saving context and reducing the chance of unintended overwrites.
</Tip>

***

## Read

Reads a file from the local filesystem and returns its contents with line numbers.

### Parameters

<ParamField body="file_path" type="string" required>
  Absolute path to the file. Relative paths are not accepted.
</ParamField>

<ParamField body="offset" type="number">
  Line number to start reading from (1-indexed). Use this with `limit` to read a specific section of a large file.
</ParamField>

<ParamField body="limit" type="number">
  Maximum number of lines to return. Defaults to **2,000** lines. For files longer than this, call `Read` again with a higher `offset` to page through the content.
</ParamField>

### Output format

Results use `cat -n` style formatting: each line is prefixed with its 1-indexed line number.

```
   1  import { z } from 'zod'
   2
   3  export const schema = z.object({
   4    name: z.string(),
   5  })
```

Claude preserves exact indentation from this output when constructing `Edit` calls.

### Supported file types

| Type                         | Behavior                                                                                                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Text files                   | Returns raw text with line numbers                                                                                             |
| Images (PNG, JPG, etc.)      | Displayed visually — Claude is multimodal                                                                                      |
| PDF files                    | Returned as document content. For PDFs over 10 pages, use the `pages` parameter (e.g., `"1-5"`). Maximum 20 pages per request. |
| Jupyter notebooks (`.ipynb`) | All cells and outputs are returned, combining code, text, and visualizations                                                   |

<Note>
  `Read` cannot list directories. To inspect a directory's contents, use `ls` via the `Bash` tool.
</Note>

### Caching

If a file has not changed since the last `Read` call in the same conversation, Claude receives a cached stub instead of re-reading the file. This avoids redundant reads on unchanged files.

***

## Edit

Performs exact string replacement within a file.

### Parameters

<ParamField body="file_path" type="string" required>
  Absolute path to the file to modify.
</ParamField>

<ParamField body="old_string" type="string" required>
  The exact string to find and replace. Must match the file content character-for-character, including whitespace and indentation.
</ParamField>

<ParamField body="new_string" type="string" required>
  The replacement string. Use an empty string to delete `old_string`.
</ParamField>

<ParamField body="replace_all" type="boolean">
  When `true`, replaces every occurrence of `old_string` in the file instead of only the first. Useful for renaming variables or symbols that appear multiple times.
</ParamField>

### How Edit works

`Edit` finds the literal text in `old_string` inside the file and substitutes it with `new_string`. No regex interpretation occurs — the match is a plain string comparison.

<Warning>
  The tool **fails** if `old_string` appears more than once in the file and `replace_all` is not set. Provide enough surrounding context in `old_string` to make it unique, or set `replace_all: true` to update all occurrences.
</Warning>

### Pre-read requirement

Claude must call `Read` on a file at least once in the conversation before calling `Edit` on it. The tool enforces this to prevent edits based on stale or assumed content.

### Indentation matching

When Claude derives `old_string` from `Read` output, it uses the content that appears **after** the line-number prefix. For example, given:

```
  12      const x = 1
```

The `old_string` is `      const x = 1` (six leading spaces), not `  12      const x = 1`.

***

## Write

Creates a new file or fully overwrites an existing one.

### Parameters

<ParamField body="file_path" type="string" required>
  Absolute path to the file to create or overwrite.
</ParamField>

<ParamField body="content" type="string" required>
  The complete content to write to the file. If the file already exists, it is replaced entirely.
</ParamField>

### When to use Write vs Edit

<Tabs>
  <Tab title="Use Edit">
    * Modifying one or more sections of an existing file
    * Renaming a variable or symbol across the file
    * Adding or removing a block of code

    Edit is preferred because it transmits only the changed fragment.
  </Tab>

  <Tab title="Use Write">
    * Creating a new file from scratch
    * Completely rewriting a file (e.g., regenerating a config from new requirements)
    * When the new content is entirely different from the original

    Write replaces the entire file in one call.
  </Tab>
</Tabs>

### Pre-read requirement

When overwriting an existing file, Claude must call `Read` first. Write enforces this to prevent unintentional data loss.

<Warning>
  Claude does not proactively create documentation files (`*.md`, `README.*`) unless explicitly instructed. Write follows the same principle.
</Warning>


Built with [Mintlify](https://mintlify.com).