---
name: blackforge
description: >-
  Answer crypto market-data questions with BlackForge. Use this skill whenever the user asks about
  order-book or trade data for a coin on a spot venue — its latest stats on an exchange,
  order-book depth or resting/pulled liquidity for a pair, taker buy-vs-sell volume, order-ladder
  rungs, price-level lifetime, trade timing, outsized trades, or market-cap/attention enrichment;
  when they want to pull, chart or compare a metric over a time range; or when they ask which pairs
  a venue lists. Covers 9 spot exchanges (binance, bitget, bybit, coinbase, gate, kraken, kucoin,
  mexc, okx) and ~11,800 spot pairs — 119 measurement columns per pair per closed
  5-minute window. Trigger even when BlackForge is not named but the question is
  about crypto market data on a venue. Drives the BlackForge MCP tools (blackforge_catalog /
  blackforge_symbols / blackforge_latest / blackforge_series / blackforge_usage) or the `blackforge`
  CLI; never reimplements the API. Every returned column is a measurement, not a trade call.
---

# BlackForge market-data

BlackForge is a **raw market-data** product: for every `(exchange, symbol)` it stores one wide row
per closed **5-minute window**, **119 measurement columns** across ~11,800 spot pairs —
order-book depth, resting-liquidity dynamics, trade-flow and trade-timing measurements, plus
market-cap and attention enrichment, and a per-row quality bitmask. This skill lets you answer a
plain-language market-data question by calling BlackForge's own tools and reading the rows back
**as measurements**.

You are a thin orchestration + interpretation layer. **Never** build HTTP requests, curl the API,
or hardcode an endpoint URL. Always go through the MCP tools or the `blackforge` CLI. Your job is to
know the vocabulary (which metric answers which question), run the right call, and explain the
numbers correctly.

## What BlackForge is — and is not

It is a measurement feed. Each column has a precise definition (e.g. *"resting sell liquidity from
the best ask up to +100%"*, *"quote notional that left the bid side of the book"*, *"median
lifetime of a price level created and removed inside the window"*). Present results in exactly that
register: **a measurement with a definition and a unit**.

It describes what happened in the book and on the tape — it does not tell the user what will happen
next or what to trade. Do not describe any column, or the data as a whole, using the words **signal,
pump, anomaly, probability-scored, alpha, prediction, detection, or alert**, and do not imply
the data forecasts or recommends anything. Say what was *measured* ("bid depth within 5% fell from X
to Y"), not what it *means for a trade*. This framing is the whole point of the skill — hold it even
if the user's own question is phrased as a trading question; answer with the measurement.

("Flag" is the one exception, and only in its literal sense: `qualityFlags` is a real column and a
flagged *bucket* is a statement about data quality, never about the market.)

Also: never propose narrowing the venue or coin universe to save cost — the full universe is the
product.

## The playbook: discover → pick → call → interpret

### 1. Discover first — never guess identifiers

Before any keyed query, call **`blackforge_catalog`** (CLI: `blackforge catalog`). It is keyless and
returns the 9 venues (each with its `minPlan`) and all 119 metrics with `key`, `label`, `unit`,
`family`, `description`, `howToRead` and `minPlan`. Use it to resolve:

- the exact **`exchange`** identifier (lowercase: `binance`, `okx`, …), and
- the exact **`metric`** key the user's words map to (e.g. "spread"/"resting depth"/"sell wall" →
  the right `downDepth*` / `upDepth*` / `bidLiqRemoved` … key).

Never invent a metric key or a venue name. If you already hold a recent catalog in the conversation
you may reuse it, but when unsure, re-fetch — it is cheap and keyless. For a compact index of every
metric grouped by family with its one-line measurement definition, read
[`references/metrics-glossary.md`](references/metrics-glossary.md); the live catalog wording is
canonical when they differ.

To list the pairs a venue trades, call **`blackforge_symbols({exchange})`**
(CLI: `blackforge symbols --exchange <v>`). Symbol format is the venue's own
(`BTCUSDT` on binance, `BTC-USDT` on okx/coinbase) — confirm via symbols rather than assuming.

### 2. Pick the right tool for the shape of the question

| The user wants… | Call | Notes |
|---|---|---|
| a coin's **latest** stats on a venue (one snapshot) | `blackforge_latest({exchange, symbol, columns?})` | returns `{ ts, values }` for the last closed 5-min bucket. Pass `columns` (metric keys) to keep the answer focused; omit for the full row. |
| how a metric **moved over a time range** | `blackforge_series({exchange, symbol, metric, from, to, interval})` | returns `{ points: [{ ts, value }] }`, `ts` in epoch ms. One metric per call. |
| **which pairs** a venue lists | `blackforge_symbols({exchange})` | |
| **usage / quota** left | `blackforge_usage()` | recent daily usage + rows remaining this month. |

CLI fallback maps 1:1: `blackforge latest …`, `blackforge series …`, `blackforge symbols …`,
`blackforge usage`. Prefer `--output json` when you will parse the result.

**Choosing `interval` for a series** (guard the 50k-point cap — points ≈ span ÷ interval):

- hours to a few days → `5m` (native resolution)
- about a week to a month → `1h`
- multiple months → `1d`

`from`/`to` are ISO-8601 UTC. If the user says "last week", compute the range from today and state
the window you used. If a single call would exceed ~50k points, widen the interval or split the range.

### 3. Interpret the rows as measurements

When you present numbers, define each column with its catalog `description` / `howToRead` wording
(or the glossary). Convert quote-relative values to USD when helpful by multiplying by
`quoteUsdRate` (units are documented per metric). Anchor `ts` on the timeline. Compare windows in
plain measurement terms — "taker-buy volume was 2.3× taker-sell volume", "median resting-level
lifetime dropped from 4.1s to 0.6s" — and stop there. Do not translate a measurement into a buy/sell
call or label it with any banned word.

**Always read `qualityFlags`.** It is the one column that qualifies every other column on the row,
it is free on every plan, and it is deliberately queryable — request it alongside whatever else you
ask for. It is a **bitmask**: `0` means no known problem, and each set bit names one condition. The
full bit table ships on the catalog entry for `qualityFlags` as `bits`, and each bit carries a
`contaminates` list of the metric families it calls into question — so a broken order book leaves
the trade columns on the same row sound. Read the bit table from the catalog rather than hardcoding
bit numbers.

Nothing in a row is ever hidden, filtered or nulled. Every value is exactly as measured; the flags
tell you which of them to trust. Two companion columns are worth requesting with it:

- **`lastTradeAgeTime`** — how long before the window closed the pair last traded, `0` when the
  window itself contained a trade. About half of all windows contain no trade, and their candle
  carries the last traded price forward rather than inventing one. A large value means the price is
  real but old.
- **`bookObservedAt`** — the instant the book was actually read, which is later than the window
  close by a different amount on each venue. Use it, not `ts`, to line two venues up.

**Bit 15 (32768) is not a defect.** It means the row predates the quality rail and was never
assessed — **unchecked, not unreliable**. It is the ClickHouse column default, so the entire
pre-migration-006 archive carries it. Say "not assessed", never "bad data".

Where a chart draws this, the convention is: **wherever the mark is fainter or hollow, that bucket
is flagged; solid means final.**

**Three columns you must NOT request.** `bookAgeTime` and `seedDepth` are now `internal: true` —
they measure our collector, not the market, and the API accepts-and-ignores them. `bookSynced` is
non-queryable and naming it in `columns=` **400s the entire request**, the columns you actually
wanted included. Use `qualityFlags` for all three concerns.

### 4. Handle entitlements gracefully — omitted ≠ nonexistent

Entitlements (venues, columns, granularity, history depth) are enforced **server-side by plan**.
Two things to recognise and explain:

- A response header **`X-BlackForge-Columns-Omitted`** (or simply missing expected columns) means
  those columns sit **above the caller's plan** and were dropped — the data exists, the key just
  doesn't include it. Tell the user which tier includes them and point to **blackforge.so/pricing**.
  Never report it as "there is no data for that".
- A **`403`** on a venue or interval means the same at the request level (e.g. a `pro`-only venue on
  a free key, or `1m` granularity the plan lacks). Explain the plan gap and the upgrade path.

`blackforge_usage` / `X-BlackForge-Rows-Remaining` tell you the monthly quota left; if a call fails
for quota, say so plainly.

### 5. Prefer MCP, fall back to CLI, else help them set up

1. If the **`blackforge_*` MCP tools** are available, use them — this is the primary path.
2. Otherwise, if the **`blackforge` CLI** is installed (or `npx -y @blackforge-so/cli` is usable), shell
   out to it and parse `--output json`.
3. If neither exists, don't hand-roll API calls — tell the user how to set one up and point them to
   [`references/setup.md`](references/setup.md) (MCP config block, CLI install, and where to get a
   key at app.blackforge.so → Keys).

## Worked examples

**"What's the resting depth for ETH on Binance right now?"**
→ `blackforge_catalog` to confirm `binance` and the depth metric keys → `blackforge_symbols` if
unsure of the symbol (`ETHUSDT`) → `blackforge_latest({exchange:'binance', symbol:'ETHUSDT',
columns:['price','downDepth5','downDepth10','upDepth30','upDepth100','qualityFlags']})`. Report each as its
measurement: "bid depth within −5% of top-of-book: \$X; ask depth to +30%: \$Y", noting they are
resting-liquidity sums in the quote currency at the last closed 5-min window, and reading
`qualityFlags` before trusting them.

**"Chart the 5-minute spread proxy for BTC-USDT on OKX last week."**
→ catalog → pick the metric → `blackforge_series({exchange:'okx', symbol:'BTC-USDT',
metric:'<key>', interval:'5m', from:'<7d ago>', to:'<now>'})`. State the window used; if 5m over 7
days risks the point cap, switch to `1h`. Describe the line as the measured quantity over time, not
as a trade cue.

**"Compare taker buy vs sell volume for SOL on Binance today."**
→ two `blackforge_series` calls (`buyTradeVol`, `sellTradeVol`) or one `blackforge_latest` with both
columns → present the ratio as measured aggressor balance.

## References

- [`references/metrics-glossary.md`](references/metrics-glossary.md) — all 119 metrics grouped by
  family, each with its one-line measurement definition and `min plan`. Read it to map the user's
  words to the right `metric` key and to explain a column.
- [`references/setup.md`](references/setup.md) — how to configure the MCP server or install the CLI,
  and where to get an API key.
