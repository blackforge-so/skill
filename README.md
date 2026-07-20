# blackforge — Claude skill

A [Claude skill](https://docs.claude.com/en/docs/claude-code/skills) that lets Claude answer
crypto **market-data** questions by orchestrating the [BlackForge](https://blackforge.so) MCP
tools (preferred) or the `blackforge` CLI. It is a thin orchestration + interpretation layer —
it never reimplements the API.

BlackForge stores one wide row per `(exchange, symbol)` per closed 5-minute window — up to 117
columns built from 103 catalog metrics (order-book depth and depth walls, order-ladder rungs,
resting-liquidity add/withdraw, price-level lifetime, trade timing, outsized-trade counts, plus
market-cap and attention enrichment) across 9 spot exchanges and ~13,800 pairs. The skill knows
that vocabulary and the discover → pick → call → interpret playbook, and it frames every returned
column as a **measurement** with a definition, never as a trade call.

## Contents

| Path | What |
|---|---|
| `SKILL.md` | The skill: frontmatter trigger `description` + the playbook |
| `references/metrics-glossary.md` | All 103 metrics grouped by family, each with its measurement definition and min plan |
| `references/setup.md` | How to configure the MCP server or install the CLI, and where to get an API key |
| `scripts/latest-json.sh` | Optional CLI wrapper: dump the latest bucket for a pair as JSON |
| `evals/trigger-eval.json` | Trigger eval set (should / should-not queries) for description tuning |

## Install

Copy the skill into your Claude skills directory:

```bash
cp -R SKILL.md references scripts ~/.claude/skills/blackforge/
```

Then configure access — either the BlackForge MCP server (preferred) or the `blackforge` CLI.
See [`references/setup.md`](references/setup.md). You need a BlackForge API key from
app.blackforge.so → Keys.

## Source of truth

This repo is the versioned source. The installed copy under `~/.claude/skills/blackforge/` is an
install target — regenerate it from here.
