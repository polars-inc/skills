# Polars Plugin Marketplace

Official Claude Code plugin marketplace for [Polars](https://pola.rs) — the fast DataFrame library for Python.

## Install

Add this marketplace, then install the Polars plugin:

```
/plugin marketplace add polars-inc/skills
/plugin install polars@polars
```

Start a session. Claude Code loads the skill whenever a task involves Polars or Python data processing. To invoke it explicitly, type `/polars:polars`.

## Plugins

| Plugin | Description |
|--------|-------------|
| `polars` | Write fast, idiomatic Polars and answer natural-language data questions with one-shot lazy queries |

## Manual install (no marketplace)

Clone the repo and copy the `polars/` directory into your skills folder:

```bash
# Personal (all projects)
git clone https://github.com/polars-inc/skills
cp -r skills/polars ~/.claude/skills/

# Project-level (checked into git)
cp -r skills/polars .claude/skills/
```

## Companion: polars-mcp

This skill works alongside `polars-mcp`, a local MCP server for live API
lookup against your installed Polars version:

```bash
uvx --with polars polars-mcp
```

See [r-brink/polars-mcp](https://github.com/r-brink/polars-mcp) for setup.

## License

MIT
