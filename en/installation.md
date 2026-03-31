> ## Documentation Index
> Fetch the complete documentation index at: https://vineetagarwal-code-claude-code.mintlify.app/llms.txt
> Use this file to discover all available pages before exploring further.

# Installation

> Install Claude Code on macOS, Linux, or Windows (via WSL). Requires Node.js 18 or higher.

## Requirements

* **Node.js 18 or higher** — Claude Code checks the Node.js version at startup and exits with an error if it is below 18.
* **npm** — included with Node.js.

Check your current version:

```bash  theme={null}
node --version
npm --version
```

***

## Install Claude Code

Install globally with npm:

```bash  theme={null}
npm install -g @anthropic-ai/claude-code
```

After installation, verify it works:

```bash  theme={null}
claude --version
```

***

## Platform-specific notes

<Tabs>
  <Tab title="macOS">
    npm global installs work out of the box on macOS. If you get a permissions error when running `npm install -g`, you have two options:

    **Option A: Fix npm permissions (recommended)**

    Configure npm to use a directory in your home folder:

    ```bash  theme={null}
    mkdir -p ~/.npm-global
    npm config set prefix ~/.npm-global
    ```

    Add the following to your shell profile (`~/.zshrc` or `~/.bash_profile`):

    ```bash  theme={null}
    export PATH=~/.npm-global/bin:$PATH
    ```

    Then reload your profile and install:

    ```bash  theme={null}
    source ~/.zshrc
    npm install -g @anthropic-ai/claude-code
    ```

    **Option B: Use a Node version manager**

    Tools like [nvm](https://github.com/nvm-sh/nvm) or [fnm](https://github.com/Schniz/fnm) install Node.js in your home directory, which avoids global permission issues entirely:

    ```bash  theme={null}
    # Using nvm
    nvm install --lts
    nvm use --lts
    npm install -g @anthropic-ai/claude-code
    ```
  </Tab>

  <Tab title="Linux">
    On most Linux distributions, `npm install -g` requires either `sudo` or a corrected npm prefix. Using `sudo` is not recommended because it can create permission problems later.

    **Recommended: Use a Node version manager**

    ```bash  theme={null}
    # Install nvm
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash

    # Reload your shell, then install Node.js
    nvm install --lts
    nvm use --lts

    # Install Claude Code
    npm install -g @anthropic-ai/claude-code
    ```

    **Alternative: Fix npm global prefix**

    ```bash  theme={null}
    mkdir -p ~/.npm-global
    npm config set prefix ~/.npm-global
    echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
    source ~/.bashrc
    npm install -g @anthropic-ai/claude-code
    ```
  </Tab>

  <Tab title="Windows (WSL)">
    Claude Code runs on Windows through the Windows Subsystem for Linux (WSL). Running it directly in Command Prompt or PowerShell is not supported.

    **Step 1: Install WSL**

    Open PowerShell as Administrator and run:

    ```powershell  theme={null}
    wsl --install
    ```

    Restart your machine when prompted. This installs WSL 2 with Ubuntu by default.

    **Step 2: Open a WSL terminal**

    Launch Ubuntu from the Start menu, or run `wsl` from PowerShell.

    **Step 3: Install Node.js inside WSL**

    ```bash  theme={null}
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.0/install.sh | bash
    source ~/.bashrc
    nvm install --lts
    nvm use --lts
    ```

    **Step 4: Install Claude Code**

    ```bash  theme={null}
    npm install -g @anthropic-ai/claude-code
    ```

    <Note>
      Always run `claude` from within your WSL terminal, not from Windows CMD or PowerShell. Your project files should live inside the WSL filesystem (e.g., `~/projects/`) for best performance. Accessing Windows files via `/mnt/c/...` works but is slower.
    </Note>
  </Tab>
</Tabs>

***

## Updating

Update Claude Code to the latest version with:

```bash  theme={null}
npm update -g @anthropic-ai/claude-code
```

Or use Claude Code's built-in update command:

```bash  theme={null}
claude update
```

To check what version you're running:

```bash  theme={null}
claude --version
```

***

## Uninstalling

Remove Claude Code with:

```bash  theme={null}
npm uninstall -g @anthropic-ai/claude-code
```

This removes the `claude` binary. Your configuration files in `~/.claude/` are not removed automatically. To delete them:

```bash  theme={null}
rm -rf ~/.claude
```

***

## Troubleshooting

<AccordionGroup>
  <Accordion title="'claude' command not found after installation">
    This usually means the npm global bin directory is not on your `PATH`.

    Find where npm installs global binaries:

    ```bash  theme={null}
    npm config get prefix
    ```

    The `bin` subdirectory of that path needs to be on your `PATH`. For example, if the output is `/home/you/.npm-global`, add this to your shell profile:

    ```bash  theme={null}
    export PATH=/home/you/.npm-global/bin:$PATH
    ```

    Reload your shell (`source ~/.zshrc` or open a new terminal) and try again.
  </Accordion>

  <Accordion title="Node.js version is below 18">
    Claude Code requires Node.js 18 or higher. If you see this error at startup:

    ```
    Error: Claude Code requires Node.js version 18 or higher.
    ```

    Upgrade Node.js using your version manager:

    ```bash  theme={null}
    # nvm
    nvm install --lts
    nvm use --lts

    # fnm
    fnm install --lts
    fnm use --lts
    ```

    Or download the latest LTS release from [nodejs.org](https://nodejs.org).
  </Accordion>

  <Accordion title="Permission denied when running npm install -g">
    Do not use `sudo npm install -g` — it can leave files owned by root and cause further issues. Instead, fix your npm prefix to point to a user-writable directory:

    ```bash  theme={null}
    mkdir -p ~/.npm-global
    npm config set prefix ~/.npm-global
    export PATH=~/.npm-global/bin:$PATH
    npm install -g @anthropic-ai/claude-code
    ```

    Add the `export PATH` line to your shell profile so it persists across sessions.
  </Accordion>

  <Accordion title="Authentication fails on first run">
    If the browser-based OAuth flow fails or you cannot use a browser, set your API key directly as an environment variable instead:

    ```bash  theme={null}
    export ANTHROPIC_API_KEY=sk-ant-...
    claude
    ```

    You can add this to your shell profile to make it permanent. API keys are available in the [Anthropic Console](https://console.anthropic.com).
  </Accordion>

  <Accordion title="Running inside Docker or a CI environment">
    In non-interactive environments, authenticate with an API key via the environment variable:

    ```bash  theme={null}
    export ANTHROPIC_API_KEY=sk-ant-...
    ```

    Use the `-p` flag to run non-interactively:

    ```bash  theme={null}
    claude -p "run the test suite and report any failures"
    ```

    If you need Claude Code to operate without permission prompts in a sandboxed container, use the `--dangerously-skip-permissions` flag. This flag only works in environments that pass Claude Code's sandbox safety checks (no internet access and not running as root outside a container).
  </Accordion>
</AccordionGroup>


Built with [Mintlify](https://mintlify.com).