---
name: man-asb
description: >
  Agent Switchboard (ASB) usage guide covering CLI commands, configuration files (config.toml,
  .asb.toml, mcp.json), library entry formats (rules, commands, agents, skills, hooks), plugin
  system, and distribution targets. Use when the user asks about ASB configuration, wants to
  write or debug ASB config files, manage MCP servers/rules/commands/agents/skills/hooks via ASB,
  set up plugins, or understand how ASB distributes to different AI coding agents. Trigger on
  mentions of "asb", "agent-switchboard", "config.toml" (in ASB context), ".asb.toml",
  "asb sync", "asb mcp", "asb rule", "asb plugin", or any ASB CLI subcommand.
---

# Agent Switchboard (ASB) Guide

ASB manages MCP servers, rules, commands, agents, skills, and hooks from a single source of truth (`~/.agent-switchboard/`), then syncs them to every AI coding agent (Claude Code, Codex, Cursor, Gemini, OpenCode, Trae, Claude Desktop).

Alias: `asb`. Install: `npm i -g agent-switchboard`.

## Directory Layout

```
~/.agent-switchboard/          # ASB_HOME (configurable via env)
├── config.toml                # User-level config (primary)
├── <profile>.toml             # Profile config layer (activated with -p)
├── mcp.json                   # MCP server definitions (JSONC)
├── rules/                     # Rule snippet .md files
├── commands/                  # Command .md files
├── agents/                    # Agent .md files
├── skills/                    # Skill directories (each with SKILL.md)
├── hooks/                     # Hook .json files or bundle dirs
├── plugins/                   # Auto-discovered local plugins
├── extensions/                # Extension modules (.mjs/.js)
└── .source-cache/             # Git-cloned plugin source cache

<project>/.asb.toml            # Project-level config layer
```

## CLI Commands

```
asb init                        # Interactive project setup (.asb.toml + AGENTS.md)
asb init --force                # Overwrite existing .asb.toml
asb mcp                         # Interactive MCP server selector
asb rule                        # Interactive rule selector with ordering
asb command                     # Interactive command selector
asb agent                       # Interactive agent selector
asb skill                       # Interactive skill selector
asb hook                        # Interactive hook selector (Claude Code only)
asb sync                        # Push all libraries + MCP to applications
asb sync --dry-run              # Preview changes without writing
asb sync -u / --update          # Update remote plugin sources before syncing
asb sync --no-update            # Skip remote source updates (default)
asb <lib> load <platform>       # Import from a platform into library
asb <lib> list [--json]         # Show inventory and sync status
asb plugin list                 # List discovered plugins
asb plugin info <ref>           # Show plugin details + components
asb plugin enable <ref>         # Enable a plugin
asb plugin disable <ref>        # Disable a plugin
asb plugin marketplace add <path-or-url>   # Add plugin source
asb plugin marketplace remove <name>       # Remove plugin source
asb plugin marketplace list                # List sources
asb plugin update [name]                   # Pull latest for remote sources
```

`<lib>` = `rule | command | agent | skill | hook`. `<ref>` = `plugin` or `plugin@source`.

Shared flags: `-p, --profile <name>`, `--project <path>`, `--json`.

## Configuration

### config.toml (Full Reference)

All sections are optional. Defaults apply when omitted.

```toml
# --- Target Applications ---
[applications]
enabled = ["claude-code", "codex", "cursor"]
assume_installed = []  # force distribution to apps whose dirs don't exist yet
# Supported IDs: claude-code, claude-desktop, codex, cursor,
#                gemini, opencode, trae, trae-cn

# Per-app overrides (3 modes, pick one per section):
# Replace:  codex.skills.enabled = ["only-this"]
# Append:   gemini.commands.add = ["extra-cmd"]
# Remove:   codex.rules.remove = ["not-this"]
# Sections: mcp, rules, commands, agents, skills, hooks, plugins

# --- Library Sections ---
# Array order = composition/priority order
[mcp]
enabled = ["filesystem", "github"]

[rules]
enabled = ["prompt-hygiene", "code-style"]
includeDelimiters = false   # wrap each rule in <!-- id:start/end --> comments

[commands]
enabled = ["docs", "deploy"]

[agents]
enabled = ["reviewer", "planner"]

[skills]
enabled = ["visual-reader", "orchestrate"]

[hooks]
enabled = ["pre-commit-lint"]

# --- Plugins ---
[plugins]
enabled = ["context7", "my-plugin@team-lib"]
auto_update = false  # true: auto-pull remote sources on every sync

[plugins.sources]
team-lib = "https://github.com/org/team-library"
local-lib = "/absolute/path/to/library"
mono-sub = "https://github.com/org/monorepo/tree/main/plugins/sub"
# Also accepts structured form:
# remote = { url = "https://...", ref = "v1.0", subdir = "plugins/foo" }

[plugins.exclude]
commands = ["context7:docs"]       # cherry-pick: exclude specific components
rules = ["context7:use-context7"]

# --- Extensions ---
[extensions]
my-extension = true    # opt-in/out for auto-discovered modules
disabled-one = false

# --- Custom Targets (DSL) ---
# See references/targets.md for full DSL spec
[targets.my-agent.mcp]
format = "json"
config_path = "~/.my-agent/config.json"

[targets.my-agent.rules]
file_path = "~/.my-agent/rules.md"

# --- Distribution ---
[distribution]
use_agents_dir = false   # true: skills go to 2 targets instead of 4

# --- UI ---
[ui]
page_size = 20   # items shown in interactive selectors (5-50)
```

### Layered Config

Three TOML layers merge in priority order (higher wins):

| Layer   | File                      | Activated by             |
|:--------|:--------------------------|:-------------------------|
| User    | `~/.asb/config.toml`      | Always                   |
| Profile | `~/.asb/<name>.toml`      | `-p <name>`              |
| Project | `<project>/.asb.toml`     | `--project <path>`       |

Profile and project layers use the same schema but fields are all optional (partial overlay). With `--project`, outputs target the project directory.

### Per-Application Override Resolution

Given global `enabled = ["a", "b", "c"]`:

| Override                          | Result              |
|:----------------------------------|:--------------------|
| `app.section.enabled = ["x"]`     | `["x"]` (replace)   |
| `app.section.add = ["d"]`         | `["a","b","c","d"]`  |
| `app.section.remove = ["b"]`      | `["a","c"]`          |

`enabled` takes full precedence; `add`/`remove` are ignored when `enabled` is set.

Plugin components are auto-expanded into section lists during `asb sync`. You don't need to manually add plugin entries to `[mcp].enabled` etc.

## MCP Configuration (mcp.json)

File: `~/.agent-switchboard/mcp.json` (JSONC, comments allowed).

Stores server **definitions** only. Enabled state is managed in `config.toml [mcp].enabled`.

```jsonc
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-filesystem"],
      "env": { "HOME": "/Users/me" }
    },
    "remote-api": {
      "url": "https://api.example.com/mcp",
      "type": "http"
    }
  }
}
```

Server fields: `command`, `args`, `env`, `url`, `type` (`stdio | sse | http`). Type is auto-inferred from `url`/`command` if omitted. Unknown fields are preserved.

Plugin MCP servers (from `.mcp.json` inside plugins) are automatically merged when the plugin is enabled. User-defined servers win on ID conflict.

## Library Entry Formats

### Rules

Markdown files in `~/.asb/rules/`. YAML frontmatter is optional.

```markdown
---
title: Code Style
description: Enforce consistent style
tags: [style, lint]
requires: [claude-code]
extras:
  cursor:
    alwaysApply: false
    globs: "*.ts"
---
Use 2-space indentation. Prefer const over let.
```

| Field         | Type       | Required | Notes                              |
|:--------------|:-----------|:---------|:-----------------------------------|
| `title`       | `string`   | No       | Section heading in composed output |
| `description` | `string`   | No       | Rule description                   |
| `tags`        | `string[]` | No       | Classification tags                |
| `requires`    | `string[]` | No       | Dependency on other rule IDs       |
| `extras`      | `object`   | No       | Platform-specific options          |

Rules are composed into a single document per target. Order in `enabled` array controls composition order.

### Commands & Agents

Markdown files in `~/.asb/commands/` or `~/.asb/agents/`.

```markdown
---
description: Generate project documentation
extras:
  cursor:
    model: claude-sonnet-4-5-20250514
---
Scan the codebase and produce comprehensive docs covering architecture,
API surface, and usage examples.
```

| Field         | Type     | Required | Notes                      |
|:--------------|:---------|:---------|:---------------------------|
| `description` | `string` | No       | Entry description          |
| `extras`      | `object` | No       | Platform-specific options  |

Import: `asb command load claude-code`, `asb agent load cursor`.

### Skills

Directory bundles in `~/.asb/skills/<id>/`, must contain `SKILL.md`:

```
my-skill/
├── SKILL.md          # Entry file (frontmatter required)
├── scripts/
│   └── helper.py
└── references/
    └── guide.md
```

SKILL.md frontmatter:

```yaml
---
name: my-skill
description: What this skill does and when to use it
extras:
  cursor:
    model: claude-sonnet-4-5-20250514
---
```

| Field         | Type     | Required | Notes                    |
|:--------------|:---------|:---------|:-------------------------|
| `name`        | `string` | **Yes**  | Skill identifier         |
| `description` | `string` | **Yes**  | Trigger description      |
| `extras`      | `object` | No       | Platform-specific config |

Entire directories are copied to each agent's skill location. Deactivated skills are cleaned up.

### Hooks (Claude Code only)

JSON files or bundles in `~/.asb/hooks/`.

**Single file**: `~/.asb/hooks/my-hook.json`
**Bundle**: `~/.asb/hooks/my-hook/hook.json` + script files

```json
{
  "name": "Pre-commit lint",
  "description": "Run linter before commits",
  "hooks": {
    "PreCommit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${HOOK_DIR}/lint.sh",
            "timeout": 30000
          }
        ]
      }
    ]
  }
}
```

`${HOOK_DIR}` is resolved to the absolute distribution path at sync time.

Hook handler types: `command`, `http`, `prompt`, `agent`. Additional fields: `url`, `prompt`, `model`, `timeout`, `statusMessage`, `async`, `once`, `headers`, `allowedEnvVars`.

## Plugins

A plugin bundles related components (rules, commands, agents, skills, hooks, MCP) into a single enable/disable unit.

### Quick Setup

```bash
# Add a source
asb plugin marketplace add /path/to/my-plugin
asb plugin marketplace add https://github.com/org/repo

# Enable
asb plugin enable my-plugin

# Sync to distribute all plugin components
asb sync
```

### Plugin Structure

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       # Optional: { name, description, version }
├── .mcp.json              # Optional: MCP server definitions
├── rules/
├── commands/
├── agents/
├── skills/
└── hooks/
```

No manifest is required. A bare directory with component subdirectories works. Plugin components are namespaced: `pluginName:componentId` (e.g., `context7:docs-researcher`).

For detailed plugin configuration (sources, marketplace format, namespacing, exclude), see `references/plugins.md`.

## Distribution Targets

ASB distributes to 8 built-in targets. Each target supports a subset of sections:

| Target           | MCP | Rules  | Commands | Agents | Skills | Hooks |
|:-----------------|:---:|:------:|:--------:|:------:|:------:|:-----:|
| `claude-code`    |  +  |   +    |    +     |   +    |   +    |   +   |
| `claude-desktop` |  +  |        |          |        |        |       |
| `codex`          |  +  |   +    |    +     |   +    |   +    |       |
| `cursor`         |  +  |   +    |    +     |   +    |   +    |       |
| `gemini`         |  +  |   +    |    +     |        |   +    |       |
| `opencode`       |  +  |   +    |    +     |   +    |   +    |       |
| `trae`           |  +  |   +    |          |        |   +    |       |
| `trae-cn`        |  +  |   +    |          |        |   +    |       |

Custom targets can be defined via `[targets.<id>]` in config.toml using a DSL. See `references/targets.md` for output paths and DSL spec.

## Common Workflows

### First-time setup

```bash
npm i -g agent-switchboard

# Create config
cat > ~/.agent-switchboard/config.toml << 'EOF'
[applications]
enabled = ["claude-code", "cursor"]
EOF

# Select MCP servers interactively
asb mcp

# Import existing rules from Claude Code
asb rule load claude-code

# Select and order rules
asb rule

# Push everything
asb sync
```

### Add a plugin from GitHub

```bash
asb plugin marketplace add https://github.com/org/my-plugin
asb plugin enable my-plugin
asb sync
```

### Project Setup

```bash
asb init                        # Answer prompts to generate .asb.toml
asb sync -P .                   # Distribute project config to agents
```

### Per-project config

Create `<project>/.asb.toml`:

```toml
[rules]
enabled = ["project-conventions"]

[applications]
claude-code.rules.add = ["claude-specific-note"]
```

Then: `asb sync --project /path/to/project`

### Import from multiple platforms

```bash
asb command load claude-code      # ~/.claude/commands/
asb command load cursor           # ~/.cursor/commands/
asb skill load claude-code        # ~/.claude/skills/
asb agent load codex              # ~/.codex/agents/
```

## Environment Variables

| Variable          | Default                | Purpose                             |
|:------------------|:-----------------------|:------------------------------------|
| `ASB_HOME`        | `~/.agent-switchboard` | Library, config, and state location |
| `ASB_AGENTS_HOME` | User home directory    | Base path for agent config files    |
