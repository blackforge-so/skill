# Setting up access to BlackForge

This skill is a thin orchestration layer. It does not talk to the API directly — it drives one of
two clients that the user installs once. Read this when the user has neither the MCP tools nor the
`blackforge` CLI available, or asks how to get set up.

Every keyed call needs a **BlackForge API key**. Get one at **app.blackforge.so → Keys** (create a
key, copy the `bf_…` string). The key's plan decides which venues, columns and granularities are
returned; anything above the plan comes back empty with an `X-BlackForge-Columns-Omitted` note
rather than an error. See the pricing tiers at blackforge.so/pricing.

The public API base is `https://api.blackforge.so/v1`.

---

## Option A — MCP server (preferred)

The MCP server exposes the five `blackforge_*` tools this skill calls directly. Once configured,
the tools appear automatically and no shelling out is needed.

**Claude Desktop** — add to `claude_desktop_config.json` (Settings → Developer → Edit Config):

```json
{
  "mcpServers": {
    "blackforge": {
      "command": "npx",
      "args": ["-y", "@blackforge/mcp"],
      "env": { "BLACKFORGE_API_KEY": "bf_your_key_here" }
    }
  }
}
```

**Claude Code** — either add the same block to the project/user MCP config, or run:

```bash
claude mcp add blackforge --env BLACKFORGE_API_KEY=bf_your_key_here -- npx -y @blackforge/mcp
```

Restart the client. The tools become available as:

| tool | purpose | key params |
|---|---|---|
| `blackforge_catalog` | venues + 103 metric definitions | *(keyless — call first)* |
| `blackforge_symbols` | pairs a venue trades | `exchange` |
| `blackforge_latest` | latest closed 5-min bucket | `exchange`, `symbol`, `columns?` |
| `blackforge_series` | a metric over a time range | `exchange`, `symbol`, `metric`, `from`, `to`, `interval` |
| `blackforge_usage` | recent usage + rows remaining | *(none)* |

Optional env `BLACKFORGE_BASE_URL` overrides the API base for a local/dev server
(e.g. `http://localhost:3001/api`).

---

## Option B — `blackforge` CLI (fallback)

Use this when the MCP tools are not configured but a shell is available. No install step is
required — `npx` fetches it on demand:

```bash
npx -y @blackforge/cli catalog
```

Or install once for the bare `blackforge` binary:

```bash
npm install -g @blackforge/cli
blackforge auth set-key bf_your_key_here   # stored at ~/.blackforge/config.json (mode 0600)
```

The key is read from (in order) `--api-key`, `$BLACKFORGE_API_KEY`, then the stored config.

Commands mirror the MCP tools:

```bash
blackforge catalog                                              # keyless: venues + metrics
blackforge symbols --exchange binance
blackforge latest  --exchange binance --symbol BTCUSDT [--columns price,downDepth5,askLiqRemoved]
blackforge series  --exchange binance --symbol BTCUSDT \
                   --metric downDepth5 --interval 1h \
                   --from 2026-07-01T00:00:00Z --to 2026-07-08T00:00:00Z --output json
blackforge usage
```

Global options: `--output table|json|csv` (default table on a TTY, json when piped), `--api-key`,
`--base-url`, `--verbose`. Add `--output json` when the result will be parsed rather than read.

---

## Response headers worth surfacing

Both clients pass through BlackForge's accounting headers. When present, use them to explain results:

- `X-BlackForge-Columns-Omitted` — columns dropped because they sit above the caller's plan. Tell
  the user which tier includes them; do not report the data as missing.
- `X-BlackForge-Rows-Remaining` — rows left in the monthly quota for this key.
- `X-BlackForge-Rows-Served` / `X-BlackForge-Blocks-Billed` — what this call consumed.

A `403` on a venue/interval means the plan does not include it — point the user to blackforge.so/pricing.
