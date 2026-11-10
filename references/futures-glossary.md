# BlackForge futures glossary

The futures data is a separate table from the spot rows: one row per **linear futures contract**
(perpetual or dated, quoted and margined in the quote currency, not coin-margined) per closed minute, on the major futures venues
(binance, bitget, bybit, gate, kucoin, mexc, okx). It is served by the `blackforge_futures_*` MCP
tools and the `blackforge futures` CLI commands, and described by the `futures` block of
`blackforge_catalog`. Like the spot columns, every futures column is a **measurement or a value
the venue published**, never a call.

**History is kept like spot, from 7 November 2026.** Futures rows are never deleted: each plan reads
them through its usual history window (Free the last 14 days, Pro and pay-per-call 90, Max 365, Ultra
the full archive), and nothing exists before recording began. `futures.availableFrom` in the catalog
is the live start; `futures.historyDays` is null (no retention limit).

**How to use this file.** Pick the `column key` that matches the question and pass it verbatim to
`blackforge_futures_series` / `blackforge_futures_top` (`metric`) or `blackforge_futures_latest`
(`columns`). The live `futures.metrics` in `blackforge_catalog` is canonical; this table is a fast
index. `min plan` is the lowest tier that includes the column.

**Units.** `price` = quote currency per base coin; `quote` = the contract's quote currency (USDT,
USDC...); `base` = coins, never contracts (venues that count in contracts are converted with the
contract multiplier); `ratio` = a plain fraction (`0.0001` = 0.01%); `ms` = epoch milliseconds UTC;
`count`; `text`; `bitmask`.

**Symbols are the venue's own.** binance, bybit and bitget `BTCUSDT`; okx `BTC-USDT-SWAP`; gate and
mexc `BTC_USDT`; kucoin `XBTUSDTM`; dated contracts carry the expiry (`BTCUSDT_261225`,
`BTCUSDT-25DEC26`). `baseAsset` is normalised (kucoin XBT reads BTC), so it is the key for "the same
coin on every venue". `blackforge_futures_symbols` lists what a venue has now.

## Keys and contract
_family keys: `keys`, `contract`_

| column key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `ts` | Minute end | ms | free | End of the minute the row covers, epoch ms UTC; a row covers (ts - 60 s, ts], the same convention as the spot rows, so a futures row and a spot row with the same `ts` describe the same minute. |
| `exchange` | Exchange | text | free | The futures venue, lower case. |
| `symbol` | Contract | text | free | The venue's own contract symbol, spelled as the venue spells it. |
| `baseAsset` | Base asset | text | free | The coin the contract prices, normalised across venues (kucoin XBT reads BTC). |
| `quoteAsset` | Quote asset | text | free | The currency the contract is quoted and margined in; prices, notionals and quote volumes are in it. |
| `contractType` | Contract type | text | free | `perpetual` (no expiry, funded) or `dated` (a delivery contract with an expiry; no funding). |

## Prices
_family key: `price` (a bucket reads the last minute's value)_

| column key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `markPrice` | Mark price | price | free | The venue's mark price at the minute's snapshot: the price it uses for unrealised PnL and for liquidations. |
| `indexPrice` | Index price | price | free | The venue's index price at the snapshot: its own weighted spot reference for the base coin. |
| `lastPrice` | Last price | price | free | The contract's last traded price at the snapshot. |
| `bidPrice` | Best bid | price | free | Best bid of the contract's order book at the snapshot. |
| `askPrice` | Best ask | price | free | Best ask of the contract's order book at the snapshot. |

## Funding
_family key: `funding` (a bucket reads the last minute's value)_

| column key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `fundingRate` | Funding rate | ratio | free | The venue's current funding rate for ONE funding interval, as a fraction (`0.0001` = 0.01%). Positive means longs pay shorts. Venues settle at different intervals (1 h, 4 h or 8 h), so the raw rate is not comparable across venues without saying so. Null on dated contracts. |
| `predictedFundingRate` | Predicted funding rate | ratio | free | The next interval's rate as the venue predicts it, only where the venue publishes one; null elsewhere (the catalog's `nullOn` names the venues). |
| `nextFundingTime` | Next funding time | ms | free | When the venue next settles funding, epoch ms. The spacing between settlements is the funding interval. Null on dated contracts. |

## Open interest
_family key: `openInterest` (a bucket reads the last minute's value)_

| column key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `openInterest` | Open interest | base | free | Open positions in coins (not contracts). |
| `openInterestValue` | Open interest value | quote | free | Open interest times the mark price, in the quote currency: the figure to compare across venues. |
| `openInterestAt` | Open interest read at | ms | free | When the open interest was read. Equal to the snapshot everywhere except binance, which has no all-contract endpoint: its most traded contracts are read every minute and the rest in rotation, so the value can be a few minutes old (`OI_STALE` above 10 minutes). |

## Trade flow
_family key: `tradeFlow` (a bucket sums its minutes)_

| column key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `volume` | Volume | base | free | Traded volume in the minute, in coins (binance: the venue's own closing one-minute bar). |
| `quoteVolume` | Quote volume | quote | free | Traded volume in the minute, in the quote currency. |
| `takerBuyVolume` | Taker buy volume | base | free | The part of the volume whose taker bought; `volume - takerBuyVolume` is the taker sell volume. |
| `takerBuyQuoteVolume` | Taker buy quote volume | quote | free | The part of the quote volume whose taker bought. |
| `tradeCount` | Trades | count | free | Number of trades in the minute. |

## Liquidations
_family key: `liquidations` (a bucket sums its minutes)_

| column key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `liqLongCount` | Long liquidations | count | max | LONG positions liquidated in the minute (closed by a forced sell), from the venue's public liquidation feed. |
| `liqLongNotional` | Long liquidation notional | quote | max | Quantity times price of the longs liquidated in the minute. |
| `liqShortCount` | Short liquidations | count | max | SHORT positions liquidated in the minute (closed by a forced buy). |
| `liqShortNotional` | Short liquidation notional | quote | max | Quantity times price of the shorts liquidated in the minute. |

Coverage is per venue, in `futures.venues[].liquidations`: `complete` (every liquidation is sent),
`lowerBound` (the venue throttles its feed, so in a cascade the figures undercount) or null (the
venue publishes no feed, and these columns read null there on every row). Read a total across
venues as "at least" whenever a lower-bound venue is in it. okx also delivers part of its feed
minutes late: a liquidation that arrives after its minute was written is in the event feed but not
in these columns, so for okx the event feed counts more than the minute columns over the same
window.

## Quality
_family key: `quality`_

| column key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `qualityFlags` | Quality flags | bitmask | free | The row's quality bitmask, decoded by the futures `qualityBits` table of the catalog; `0` is a clean row. Values are never hidden or altered on account of quality. A bucket ORs its minutes. |

The futures bits are their own table, **not** the spot `qualityFlags` bits: the same number means a
different thing on a futures row. Name the flag, never the number.

| bit | mask | flag | what it says |
|---|---|---|---|
| 0 | 1 | `QUOTE_MISSING` | No venue snapshot for the contract that minute: prices, funding and open interest are null. |
| 1 | 2 | `TRADES_PARTIAL` | The trade stream was not up for the whole minute (the first minute after a reconnect or a deploy): volume, taker volume and trade count are lower bounds. |
| 2 | 4 | `LIQ_PARTIAL` | The liquidation stream was not up for the whole minute: the liquidation columns are lower bounds. |
| 3 | 8 | `OI_STALE` | binance only: the open interest was read more than 10 minutes before the minute end. |
| 4 | 16 | `BAR_NOT_FINAL` | binance only: the venue's closing one-minute bar never arrived, so volume is its last in-progress bar. |

**Not sold, `internal: true`:** `snapshotAt`, `rowVersion` and `ingestedAt` measure our recorder
(when the snapshot was read, which copy of a minute won a deduplication, when the row was written),
not the market. They belong to no plan.

## Liquidation events

`blackforge_futures_liquidations` (CLI `blackforge futures liquidations`) returns the individual
events behind the liquidation columns, newest first, the last hour by default, on Max, Ultra and
PAYG (`futures.liquidationEvents.minPlan`). Each event has:

| field | what |
|---|---|
| `ts` | The venue's time of the liquidation, epoch ms. |
| `exchange`, `symbol` | The venue and its contract symbol. |
| `side` | `long` or `short`: the POSITION that was liquidated. A long is closed by a forced sell order, a short by a forced buy. |
| `price` | Fill (or order) price of the liquidation. |
| `quantity` | Size in coins. |
| `notional` | `quantity` times `price`, in the quote currency. |
