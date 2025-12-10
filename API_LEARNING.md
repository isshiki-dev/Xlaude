# Claude Code API Learning

This document provides a comprehensive analysis of the Claude Code API, including the CLI interface and the Plugin system structure.

## 1. CLI API Structure

The `claude` CLI is the main entry point. It supports interactive sessions, non-interactive execution (using `--print`), and various configuration options.

### General Usage
```bash
claude [options] [command] [prompt]
```

### Core Options
- **Prompt**: The main argument is the prompt text.
- **Output Control**:
    - `-p, --print`: Non-interactive mode. Prints response and exits.
    - `--output-format`: `text` (default), `json`, or `stream-json`.
    - `--input-format`: `text` or `stream-json`.
    - `--json-schema`: Enforce JSON output structure.
- **Session Management**:
    - `-c, --continue`: Continue last session.
    - `-r, --resume [id]`: Resume specific session.
    - `--session-id <uuid>`: Use specific session ID.
    - `--fork-session`: Create new session from resumed state.
    - `--no-session-persistence`: Don't save session.
- **Context & Permissions**:
    - `--add-dir`: Allow access to additional directories.
    - `--permission-mode`: `acceptEdits`, `bypassPermissions`, `default`, `dontAsk`, `plan`.
    - `--dangerously-skip-permissions`: Bypass checks (sandbox use).
- **Tools**:
    - `--tools`: Specify available tools (e.g. `Bash,Edit,Read`).
    - `--allowed-tools`: Whitelist tools.
    - `--disallowed-tools`: Blacklist tools.
- **MCP (Model Context Protocol)**:
    - `--mcp-config`: Load MCP servers config.
    - `--strict-mcp-config`: Exclusive MCP config.
- **Model Configuration**:
    - `--model`: Specify model (e.g. `sonnet`, `opus`).
    - `--agent`: Override agent setting.
    - `--system-prompt`: Set system prompt.
    - `--append-system-prompt`: Append to system prompt.

### Commands
- `mcp`: Manage MCP servers.
- `plugin`: Manage plugins.
- `setup-token`: Auth setup.
- `doctor`: Health check.
- `update`: Self-update.
- `install`: Install native build.

## 2. Plugin API Structure

Claude Code uses a directory-based plugin system. Plugins are typically located in a `plugins` directory or installed via `claude plugin`.

### Directory Structure
A plugin is a directory containing a `.claude-plugin/plugin.json` file and other resources.

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Metadata (name, version, description)
├── commands/                # Slash commands definitions (.md files)
├── agents/                  # Agent definitions (.md files)
├── skills/                  # Skills (specialized capabilities)
├── hooks/                   # Event hooks (.json and scripts)
└── README.md
```

### Metadata (`plugin.json`)
Located in `.claude-plugin/plugin.json`.
```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "Description of plugin",
  "author": { "name": "...", "email": "..." }
}
```

### Commands
Defined in `commands/*.md`. The file frontmatter defines the command properties.

**Frontmatter Fields:**
- `description`: Help text for the command.
- `argument-hint`: Usage hint for arguments.
- `allowed-tools`: List of tools the command can use.
- `hide-from-slash-command-tool`: Boolean.

**Implementation:**
The content of the markdown file serves as the prompt or logic for the command. It can execute scripts using the `Bash` tool with `${CLAUDE_PLUGIN_ROOT}` variable.

Example (`commands/example.md`):
```markdown
---
description: "Run an example command"
argument-hint: "[args]"
allowed-tools: ["Bash"]
---

# Command Logic

Execute the script:
```!
"${CLAUDE_PLUGIN_ROOT}/scripts/script.sh" $ARGUMENTS
```
```

### Agents
Defined in `agents/*.md`. Agents are specialized personas or workflows.

**Frontmatter Fields:**
- `name`: Agent name.
- `description`: Agent description.
- `model`: Model to use (e.g., `inherit`).
- `color`: UI color.
- `tools`: Allowed tools.

**Implementation:**
The markdown content acts as the system prompt for the agent.

Example (`agents/analyzer.md`):
```markdown
---
name: analyzer
description: Analyzes code
tools: ["Read", "Grep"]
---

You are an analysis agent...
```

### Hooks
Defined in `hooks/hooks.json` and implementation scripts. Hooks allow plugins to intercept events.

**`hooks/hooks.json` Structure:**
```json
{
  "description": "Plugin hooks",
  "hooks": {
    "Stop": [ ... ],
    "PreToolUse": [ ... ]
  }
}
```
Hooks can trigger commands or scripts (e.g., `${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh`).

### Skills
Skills seem to be reusable capabilities or knowledge bases, often referenced by agents or commands. Structure typically involves a directory in `skills/` with definition files.

## 3. Parameter Learning

### CLI Parameters
- **Boolean flags**: `false` by default, `true` if present (e.g., `--print`).
- **Value options**: Require an argument (e.g., `--model sonnet`).
- **Lists**: Comma or space separated (e.g., `--tools Bash,Edit`).

### Plugin Parameters
- **Environment Variables**:
    - `CLAUDE_PLUGIN_ROOT`: Absolute path to the plugin directory.
    - `ARGUMENTS`: Arguments passed to a slash command.
- **Frontmatter**: YAML frontmatter in `.md` files configuration.

## 4. Workflows

1.  **Command Execution**: User types `/command`. Claude looks up `commands/command.md`, loads the context/prompt defined there, and executes it.
2.  **Agent Invocation**: Agents can be invoked by name or usage context. They inherit tools and have specific prompts.
3.  **Hooks**: Triggered automatically on events (e.g., `Stop`, `SessionStart`). Useful for enforcing rules (like `hookify`) or loops (`ralph-wiggum`).
