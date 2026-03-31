> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Web

> Fetch web pages and search the internet for current information.

Claude Code has two web tools: `WebFetch` for retrieving and analyzing specific URLs, and `WebSearch` for querying the web when information may be beyond Claude's training cutoff.

***

## WebFetch

Fetches a URL, converts the HTML to markdown, and then processes the content using a small fast model with a prompt you provide. Returns the model's response — not the raw page content.

### Parameters

<ParamField body="url" type="string" required>
  A fully-formed, valid URL. HTTP URLs are automatically upgraded to HTTPS.
</ParamField>

<ParamField body="prompt" type="string" required>
  A natural-language prompt describing what to extract or analyze from the page content. The fetched content is passed to a secondary model along with this prompt, and the model's response is returned.
</ParamField>

### Behavior

1. Claude fetches the URL and converts HTML to markdown.
2. The markdown and the `prompt` are sent to a small, fast model.
3. That model's response is returned as the tool result.

Because a secondary model processes the content, the result is always a synthesized answer — not a raw dump of the page. If the content is very large, it may be summarized automatically.

<Info>
  WebFetch includes a 15-minute in-memory cache. Repeated calls to the same URL within that window return cached content without a network request.
</Info>

### Redirects

When a URL redirects to a different host, the tool returns a special message indicating the redirect destination rather than following it automatically. Claude then makes a new `WebFetch` call with the redirect URL.

### Limitations

* Read-only — does not submit forms or interact with pages
* Not suitable for pages requiring authentication (login walls, paywalls)
* For GitHub resources, prefer the `gh` CLI via `Bash` (e.g., `gh pr view`, `gh issue view`, `gh api`)

<Tip>
  If an MCP-provided web fetch tool is available in your environment, it may have fewer restrictions and should be preferred over this built-in tool.
</Tip>

### Example usage

<Tabs>
  <Tab title="Extract API details">
    ```json  theme={null}
    {
      "url": "https://docs.example.com/api/authentication",
      "prompt": "What authentication methods are supported and what headers are required?"
    }
    ```
  </Tab>

  <Tab title="Check library version">
    ```json  theme={null}
    {
      "url": "https://github.com/some-org/some-lib/blob/main/CHANGELOG.md",
      "prompt": "What changed in the latest version?"
    }
    ```
  </Tab>

  <Tab title="Read documentation">
    ```json  theme={null}
    {
      "url": "https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API",
      "prompt": "Summarize the Fetch API and show a basic usage example."
    }
    ```
  </Tab>
</Tabs>

***

## WebSearch

Searches the web and returns results including titles, URLs, and a model-synthesized summary. Useful when information may be newer than Claude's training data.

<Info>
  Web search is only available in the United States. It requires a compatible API provider (Anthropic first-party, Vertex AI with Claude 4.0+ models, or Foundry).
</Info>

### Parameters

<ParamField body="query" type="string" required>
  The search query to execute. Minimum length: 2 characters.
</ParamField>

<ParamField body="allowed_domains" type="string[]">
  When set, only return results from these domains. Cannot be combined with `blocked_domains` in the same request.

  Example: `["docs.python.org", "peps.python.org"]`
</ParamField>

<ParamField body="blocked_domains" type="string[]">
  Exclude results from these domains. Cannot be combined with `allowed_domains` in the same request.

  Example: `["w3schools.com"]`
</ParamField>

### Output

The tool returns search results as structured blocks containing:

* Result titles and URLs
* A synthesized summary produced by a model that has access to the search results

Claude is required to cite sources from web searches by appending a `Sources:` section with markdown hyperlinks to its response.

### Domain filtering

<Accordion title="Restricting to specific domains (allowed_domains)">
  Use `allowed_domains` when you want results exclusively from trusted or authoritative sources:

  ```json  theme={null}
  {
    "query": "Python asyncio best practices 2024",
    "allowed_domains": ["docs.python.org", "realpython.com"]
  }
  ```
</Accordion>

<Accordion title="Excluding domains (blocked_domains)">
  Use `blocked_domains` to suppress low-quality or paywalled sources:

  ```json  theme={null}
  {
    "query": "React server components tutorial",
    "blocked_domains": ["medium.com"]
  }
  ```
</Accordion>

### Search limits

Each `WebSearch` invocation performs up to **8 individual web searches** internally. Claude automatically refines queries as needed within that budget.

### Current date awareness

When searching for recent information, Claude includes the current year in search queries to avoid returning outdated results — for example, appending the year when searching for library documentation or current events.


Built with [Mintlify](https://mintlify.com).