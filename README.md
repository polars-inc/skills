# Polars Skills

Official agent skill for [Polars](https://pola.rs) — the fast DataFrame
library for Python. It teaches coding agents to write fast, idiomatic Polars:

- Use the lazy API by default (`scan_*`, `.lazy()`, one `.collect()`)
- Write native expressions instead of `map_elements` or pandas-style code
- Choose the right context (`select`, `with_columns`, `filter`, `group_by`, `over`)
- Answer natural-language data questions in one shot
- Run SQL on DataFrames and translate pandas to Polars

The skill loads automatically when a task involves Polars or Python data
processing. It covers single-node Polars; it does not apply to Polars Cloud,
On-Prem, distributed, or GPU workloads.

## Install

**Claude Code**

```
/plugin marketplace add polars-inc/skills
/plugin install polars@polars
```

Type `/polars:polars` to invoke it explicitly.

**Codex**

```bash
codex plugin marketplace add polars-inc/skills
codex plugin add polars@polars
```

**GitHub Copilot CLI**

```bash
copilot plugin install polars-inc/skills
```

In VS Code, run **Chat: Install Plugin From Source** with
`https://github.com/polars-inc/skills`.

**Cursor, Gemini CLI, and other agents** (Amp, Cline, OpenCode, Windsurf, and more)

```bash
npx skills add polars-inc/skills
```

**Claude.ai and Claude Desktop**

Download `polars.zip` from the
[latest release](https://github.com/polars-inc/skills/releases/latest) and
upload it under **Customize > Skills**.

**Manual copy**

```bash
git clone https://github.com/polars-inc/skills polars-skills
cp -r polars-skills/skills/polars ~/.agents/skills/   # most agents
cp -r polars-skills/skills/polars ~/.claude/skills/   # Claude Code
cp -r polars-skills/skills/polars ~/.cursor/skills/   # Cursor
```

## Repository structure

```
plugin.json                       # Agent Plugins manifest (Codex, Cursor, Copilot, VS Code, ...)
.claude-plugin/                   # Claude Code plugin + marketplace
.codex-plugin/                    # Codex plugin metadata
.agents/plugins/marketplace.json  # Codex marketplace
.cursor-plugin/                   # Cursor plugin + marketplace
skills/polars/
├── SKILL.md                      # core skill: rules, patterns, gotchas
└── references/                   # loaded on demand
    ├── contexts.md               # select, with_columns, filter, group_by, over, join
    ├── expressions.md            # str, dt, list, struct, selectors, cast, when/then
    ├── insight-recipes.md        # natural-language question to query recipes
    ├── lazy-api.md               # scan options, query plans, larger-than-memory data
    ├── pandas-to-polars.md       # API differences and correctness traps
    └── sql.md                    # SQL interface, dialect notes, function map
```

To release, bump `version` in `plugin.json`, `.claude-plugin/plugin.json`,
`.codex-plugin/plugin.json`, and `.cursor-plugin/plugin.json`, then run the
Release workflow. It stops if the four versions differ.

## Companion: polars-mcp

The skill works alongside [`polars-mcp`](https://github.com/r-brink/polars-mcp),
a local MCP server that introspects your installed Polars package for live API
lookups (`polars_search_api`, `polars_browse`, `polars_get_docstring`). Without
it, the skill falls back to docs.pola.rs.

```bash
uvx --with polars polars-mcp            # latest Polars
uvx --with polars==2.0.0 polars-mcp     # pin to your project's version
```

```json
{
  "mcpServers": {
    "polars-mcp": {
      "command": "uvx",
      "args": ["--with", "polars==2.0.0", "polars-mcp"]
    }
  }
}
```

## License

MIT
