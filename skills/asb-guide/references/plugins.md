# Plugin System Reference

## Source Configuration

### Auto-discovery

Directories in `~/.asb/plugins/` auto-discovered as plugin sources. No config needed. Symlinks supported.

### Explicit Sources in config.toml

```toml
[plugins.sources]
# Local absolute path
my-lib = "/absolute/path/to/library"

# Bare name -> resolves to ~/.asb/plugins/<name>/
my-lib = "my-lib"

# Home-relative
my-lib = "~/some/path"

# Git URL (shallow clone to ~/.asb/.source-cache/<name>/)
team = "https://github.com/org/team-library"

# GitHub tree URL (auto-parsed into ref + subdir)
sub = "https://github.com/org/monorepo/tree/main/plugins/sub"

# Structured form with explicit ref/subdir
pinned = { url = "https://github.com/org/repo.git", ref = "v1.0", subdir = "plugins/foo" }
```

### Source Kind Detection

| Kind          | Detection                                | Structure                          |
|:--------------|:-----------------------------------------|:-----------------------------------|
| `marketplace` | `.claude-plugin/marketplace.json` exists | Multiple plugins in subdirectories |
| `plugin`      | Everything else                          | Single plugin (formal or informal) |

## Plugin Directory Structure

### Minimal (informal plugin)

```
my-plugin/
├── rules/
│   └── coding-style.md
└── commands/
    └── deploy.md
```

No manifest needed. Any directory with component subdirectories works.

### Full (formal plugin)

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json
├── .mcp.json
├── rules/          # .md / .markdown files
├── commands/       # .md / .markdown files
├── agents/         # .md / .markdown files
├── skills/         # subdirs, each containing SKILL.md
└── hooks/          # .json files or subdirs with hook.json
```

### plugin.json Fields

```json
{
  "name": "my-plugin",
  "description": "What this plugin provides",
  "version": "1.0.0"
}
```

| Field         | Type                  | Required | Notes                                 |
|:--------------|:----------------------|:---------|:--------------------------------------|
| `name`        | `string`              | Yes      | Plugin name (used as namespace)       |
| `description` | `string`              | No       | Plugin description                    |
| `version`     | `string`              | No       | Version string                        |
| `commands`    | `string \| string[]`  | No       | Custom command paths (override scan)  |
| `agents`      | `string \| string[]`  | No       | Custom agent paths                    |
| `hooks`       | `unknown`             | No       | Hooks config                          |
| `mcpServers`  | `unknown`             | No       | MCP server declarations               |

## Plugin MCP (.mcp.json)

Two formats accepted (auto-detected):

```jsonc
// Flat (servers at root)
{
  "server-name": {
    "command": "npx",
    "args": ["-y", "my-mcp-server"]
  }
}

// Wrapped (under mcpServers key)
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "my-mcp-server"]
    }
  }
}
```

Plugin MCP servers follow plugin lifecycle: enable = available, disable = removed. User-defined servers in `~/.asb/mcp.json` win on ID conflict.

## Component Namespacing

All plugin components prefixed with plugin name:

```
{pluginName}:{componentBaseName}

Examples:
  context7:context7          # MCP server
  my-plugin:deploy           # command from commands/deploy.md
  my-plugin:reviewer         # agent from agents/reviewer.md
  my-plugin:my-skill         # skill from skills/my-skill/
  my-plugin:coding-style     # rule from rules/coding-style.md
```

### Plugin References

Used in `plugins.enabled`, `plugins.exclude`, and per-app overrides:

| Syntax              | When to use                              |
|:--------------------|:-----------------------------------------|
| `pluginName`        | Default, unique plugin name              |
| `pluginName@source` | Disambiguate same name across multiple sources |

### Granular Exclude

Cherry-pick within enabled plugin:

```toml
[plugins]
enabled = ["context7"]

[plugins.exclude]
commands = ["context7:docs"]
rules = ["context7:use-context7"]
mcp = ["context7:context7"]
```

Sections: `commands`, `agents`, `skills`, `hooks`, `rules`, `mcp`.

## Marketplace Format

Collection of plugins, identified by `.claude-plugin/marketplace.json`:

```
my-marketplace/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/                 # pluginRoot (configurable)
    ├── plugin-a/
    └── plugin-b/
```

### marketplace.json Fields

```json
{
  "name": "Team Marketplace",
  "owner": { "name": "Team", "email": "team@example.com" },
  "metadata": { "pluginRoot": "plugins" },
  "plugins": [
    {
      "name": "plugin-a",
      "source": "./plugin-a",
      "description": "...",
      "version": "1.0.0"
    }
  ]
}
```

Plugin entries support multiple source types:

| Source format                          | Resolution                                |
|:--------------------------------------|:------------------------------------------|
| `"./relative/path"`                   | Relative to marketplace root + pluginRoot |
| `{ path: "./local" }`                 | Same as above                             |
| `{ github: "org/repo" }`             | Clone from github.com                     |
| `{ git: "git@host:repo.git" }`       | Clone any git URL                         |

### Strict Mode

Marketplace entries default `strict: true`. In strict mode, marketplace entry metadata (commands, agents, mcpServers paths) overrides plugin's own plugin.json. Set `strict: false` to let plugin.json be authoritative.
