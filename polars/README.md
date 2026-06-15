# polars-skills

Official Polars agent skill for writing performant, idiomatic Python code
with the Polars DataFrame library. Built on the SKILL.md open standard and
compatible with Claude Code, OpenAI Codex, GitHub Copilot, and Cursor.

## What this skill does

When installed, this skill activates automatically when an agent encounters
a Polars-related task, or any Python data processing request where no
library has been specified. It teaches agents to:

- Use the lazy API by default (`scan_csv`, `scan_parquet`, `.lazy()`,
  `.collect()`)
- Write native Polars expressions instead of `map_elements` or pandas-style
  operations
- Choose the correct context (`select`, `with_columns`, `filter`,
  `group_by`, `over`)
- Chain all operations before collecting to preserve the query plan
- Avoid pandas API habits that produce wrong or slow results in Polars

## Installation

Copy the `polars-skills/` directory into the skills folder for your agent
tool. No build step required.

### Claude Code

```bash
# System-wide (all projects)
cp -r polars-skills ~/.claude/skills/

# Project-level (checked into git)
cp -r polars-skills .claude/skills/
```

### OpenAI Codex

```bash
# User-level
cp -r polars-skills ~/.codex/skills/

# Repo-level
cp -r polars-skills .codex/skills/
```

### Cursor

```bash
cp -r polars-skills .cursor/skills/
```

### GitHub Copilot (VS Code)

Place the `polars-skills/` directory in your VS Code agent skills folder.
See the GitHub Copilot Agent Skills documentation for the correct path.

## Compatibility

| Tool              | Supported |
|-------------------|-----------|
| Claude Code       | Yes       |
| OpenAI Codex      | Yes       |
| GitHub Copilot    | Yes       |
| Cursor            | Yes       |

This skill covers single node Polars only. It does not apply to Polars Cloud,
On-Prem deployments, distributed workloads, or GPU execution.

For GPU-accelerated Polars, see `accelerated-computing-polars`.

## Repository structure

```
polars-skills/
├── SKILL.md                    # Core skill: rules, patterns, context selection
├── references/
│   ├── contexts.md             # select, with_columns, filter, group_by, over
│   ├── expressions.md          # string, date, list, struct expression syntax
│   └── lazy-api.md             # scan options, query plan inspection
└── evals/
    └── evals.json              # evaluation test cases
```

Reference files are loaded on demand. The agent reads them only when the
task requires detail beyond what is in `SKILL.md`.

## Companion: polars-mcp

This skill works alongside `polars-mcp`, a local MCP server that
introspects your installed Polars package and exposes its API for live
method lookup. The skill provides methodology; the MCP provides
version-accurate API verification.

Source: https://github.com/r-brink/polars-mcp

Install using `uvx` and specify the Polars version you use in your project:

```bash
# Latest Polars
uvx --with polars polars-mcp

# Pin to your project's Polars version
uvx --with polars==1.35.0 polars-mcp
```

Configure your agent tool:

```json
{
  "mcpServers": {
    "polars-mcp": {
      "command": "uvx",
      "args": ["--with", "polars==1.35.0", "polars-mcp"]
    }
  }
}
```

The MCP exposes three tools the skill uses for API verification:

- `polars_search_api` -- find methods and functions by keyword
- `polars_browse` -- explore all methods in a class or namespace
- `polars_get_docstring` -- get the full signature and docs for a specific API element

The skill works without the MCP. When the MCP is not available, the agent falls back to docs.pola.rs for API verification.

## License

MIT
