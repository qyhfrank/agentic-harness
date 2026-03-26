# Distribution Targets Reference

## Output Paths by Target

### claude-code

| Section  | Global Path                     | Project Path                          |
|:---------|:--------------------------------|:--------------------------------------|
| MCP      | `~/.claude/settings.json`      | `<project>/.mcp.json`                 |
| Rules    | `~/.claude/CLAUDE.md`           | `<project>/.claude/CLAUDE.md`         |
| Commands | `~/.claude/commands/{id}.md`    | `<project>/.claude/commands/{id}.md`  |
| Agents   | `~/.claude/agents/{id}.md`      | `<project>/.claude/agents/{id}.md`    |
| Skills   | `~/.claude/skills/{id}/`        | `<project>/.claude/skills/{id}/`      |
| Hooks    | Merged into `~/.claude/settings.json` | --                              |

### codex

| Section  | Global Path                   | Project Path                         |
|:---------|:------------------------------|:-------------------------------------|
| MCP      | `~/.codex/config.json`        | `<project>/.codex/config.json`       |
| Rules    | `~/.codex/AGENTS.md`          | `<project>/AGENTS.md`                |
| Commands | `~/.codex/prompts/{id}.md`    | -- (global only, deprecated)         |
| Agents   | Custom distribution logic     | --                                   |
| Skills   | `~/.codex/skills/{id}/`       | `<project>/.codex/skills/{id}/`      |

### cursor

| Section  | Global Path                          | Project Path                            |
|:---------|:-------------------------------------|:----------------------------------------|
| MCP      | `~/.cursor/mcp.json`                 | `<project>/.cursor/mcp.json`            |
| Rules    | `~/.cursor/rules/asb-rules.mdc`      | `<project>/.cursor/rules/asb-rules.mdc` |
| Commands | `~/.cursor/commands/{id}.md`          | `<project>/.cursor/commands/{id}.md`    |
| Agents   | `~/.cursor/agents/{id}.md`            | `<project>/.cursor/agents/{id}.md`      |
| Skills   | `~/.cursor/skills/{id}/`              | `<project>/.cursor/skills/{id}/`        |

Cursor rules are composed into a single `.mdc` file with `alwaysApply: true` frontmatter. Agent frontmatter supports: `name`, `description`, `model` (default `inherit`), `readonly`, `is_background`.

### gemini

| Section  | Global Path                        | Project Path                           |
|:---------|:-----------------------------------|:---------------------------------------|
| MCP      | `~/.gemini/settings.json`          | `<project>/.gemini/settings.json`      |
| Rules    | `~/.gemini/AGENTS.md`              | `<project>/AGENTS.md`                  |
| Commands | `~/.gemini/commands/{id}.toml`     | `<project>/.gemini/commands/{id}.toml` |
| Skills   | `~/.gemini/skills/{id}/`           | `<project>/.gemini/skills/{id}/`       |

Commands are output in TOML format (not Markdown).

### opencode

| Section  | Global Path                       | Project Path                            |
|:---------|:----------------------------------|:----------------------------------------|
| MCP      | `~/.config/opencode/config.toml`  | `<project>/.opencode/config.toml`       |
| Rules    | `~/.config/opencode/AGENTS.md`    | `<project>/AGENTS.md`                   |
| Commands | `~/.opencode/command/{id}.md`     | `<project>/.opencode/command/{id}.md`   |
| Agents   | `~/.opencode/agent/{id}.md`       | `<project>/.opencode/agent/{id}.md`     |
| Skills   | `~/.opencode/skill/{id}/`         | `<project>/.opencode/skill/{id}/`       |

Note: OpenCode uses **singular** directory names (`command`, `agent`, `skill`).

### trae / trae-cn

| Section | Global Path                              | Project Path                          |
|:--------|:-----------------------------------------|:--------------------------------------|
| MCP     | `<TraeDataDir>/mcp.json`                 | `<project>/.trae/mcp.json`            |
| Rules   | `<TraeDataDir>/user_rules/asb-rules.md`  | `<project>/.trae/rules/asb-rules.md`  |
| Skills  | `<TraeDataDir>/skills/{id}/`             | `<project>/.trae/skills/{id}/`        |

Rules use MDC frontmatter. MCP distribution strips the `type` field (Trae auto-infers transport). Has install detection.

### claude-desktop

| Section | Global Path                              |
|:--------|:-----------------------------------------|
| MCP     | `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) |

Only supports MCP. Has install detection.

## Custom Targets (DSL)

Define targets purely in config.toml without writing code:

```toml
[targets.my-agent.mcp]
format = "json"                          # json | yaml
config_path = "~/.my-agent/config.json"
# project_config_path = ...             # optional project-level path
# root_key = "mcpServers"               # JSON key for server map (default)
# structure = "record"                  # record | keyed-array
# key_field = "name"                    # field name when keyed-array
# defaults = { ... }                    # default values for each server
# env_transform = true                  # convert env map to kv-array

[targets.my-agent.rules]
file_path = "~/.my-agent/rules.md"
# format = "markdown"                   # markdown | mdc
# project_file_path = ...

[targets.my-agent.commands]
target_dir = "~/.my-agent/commands"
# project_target_dir = ...
# filename_pattern = "{id}.md"
# platform_key = "my-agent"            # key for extras lookup
# frontmatter = { rename = {}, omit = [], include = [], defaults = {} }

[targets.my-agent.agents]
target_dir = "~/.my-agent/agents"
# Same options as commands

[targets.my-agent.skills]
parent_dir = "~/.my-agent/skills"
# project_parent_dir = ...
```

DSL targets are registered as extension targets and override same-ID builtins. All paths support `~/` expansion.

### Frontmatter Transform Spec

For commands/agents sections, control how frontmatter is adapted:

| Field      | Type                        | Effect                                     |
|:-----------|:----------------------------|:-------------------------------------------|
| `rename`   | `Record<string, string>`    | Rename frontmatter keys                    |
| `omit`     | `string[]`                  | Remove these keys from output              |
| `include`  | `string[]`                  | Only keep these keys (allowlist)            |
| `join`     | `Record<string, string>`    | Join array values with given separator     |
| `defaults` | `Record<string, unknown>`   | Default values for missing keys            |

Transform pipeline order: defaults -> join -> omit/include -> rename.
