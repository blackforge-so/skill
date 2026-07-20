# BlackForge metrics glossary

Every column BlackForge returns is a **measurement of the order book or trade tape** over one
closed 5-minute window for one `(exchange, symbol)`. There are **103 catalog metrics**; a single
returned row can carry **up to 117 columns** once keys, quote/USD conversion fields and quality
markers are included. Describe each column to the user using the measurement wording below — never
as a score, a call, or an event to act on.

**How to use this file.** Pick the `metric key` that matches what the user asked for and pass it
verbatim to `blackforge_series` (or `blackforge_latest`'s `columns`). Always reconcile against the
live `blackforge_catalog` output — plans, units and wording there are canonical; this table is a
fast index. `min plan` tells you the lowest tier that includes the column: if the caller's key is
below it the column comes back empty with an `X-BlackForge-Columns-Omitted` note (see SKILL.md).

**Units.** `quote` = the pair's quote currency (multiply by `quoteUsdRate` for USD); `base` = the
base coin; `price` = quote per base; `ms` = milliseconds; `count`/`index`/`ratio`/`bool`/`usd` as
named. Columns marked quote-relative are raw in the quote currency.

**Families at a glance.** candle (price OHLC) · tradeFlow (taker buy/sell aggression) · bookWalls
(cumulative resting depth within a % band of top-of-book) · orderLadders (resting depth in fixed
3%-wide slices) · bookMicro (liquidity added, liquidity withdrawn beyond trade-explained volume,
level flicker, level lifetime) · tradeTiming (silences, repeating intervals, same-size / same-instant
groupings) · strong (counts and value of outsized trades at size multiples) · enrichment
(market-cap, rank, market-wide liquidity/volume) · context (BTC/ETH reference price, sentiment
index, search interest) · quality (sync state, book age, unit conversion).

> **bookWalls vs orderLadders.** A *wall* is cumulative depth from top-of-book out to a band edge
> (e.g. `upDepth100` = all resting asks up to +100%). A *ladder* rung is the depth inside one
> discrete 3% slice (e.g. `buyOrderVol6` = bids 3–6% below top). Walls measure total thickness;
> ladder rungs measure how that depth is distributed across price.

## Keys & timestamps  
_family key: `keys` · 4 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `exchange` | Exchange | index | free | The exchange the snapshot came from. |
| `symbol` | Symbol | index | free | The trading pair the snapshot describes. |
| `ts` | Snapshot time | ms | free | The time the 5-minute window closed. |
| `ingestedAt` | Ingested time | ms | free | The time the snapshot was written to storage. |

## Candle (price)  
_family key: `candle` · 4 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `priceOpen` | Open price | price | free | The price at the start of the window. |
| `priceHigh` | High price | price | free | The highest price reached during the window. |
| `priceLow` | Low price | price | free | The lowest price reached during the window. |
| `price` | Close price | price | free | The last price of the window. |

## Trade flow (taker aggression)  
_family key: `tradeFlow` · 10 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `buyTradeVol` | Buy trade volume | quote | free | Total value of taker-buy trades in the window. |
| `sellTradeVol` | Sell trade volume | quote | free | Total value of taker-sell trades in the window. |
| `buyTradeCount` | Buy trade count | count | free | Number of taker-buy trades in the window. |
| `sellTradeCount` | Sell trade count | count | free | Number of taker-sell trades in the window. |
| `buyTradePriceAvg` | Buy trade average price | price | free | Plain average price of taker-buy trades in the window. |
| `sellTradePriceAvg` | Sell trade average price | price | free | Plain average price of taker-sell trades in the window. |
| `buyTradeSizeAvg` | Buy trade average size | base | free | Average size of taker-buy trades in base coin units. |
| `sellTradeSizeAvg` | Sell trade average size | base | free | Average size of taker-sell trades in base coin units. |
| `buyTradeMax` | Largest buy trade | quote | free | Value of the single largest taker-buy trade in the window. |
| `sellTradeMax` | Largest sell trade | quote | free | Value of the single largest taker-sell trade in the window. |

## Book walls — cumulative depth bands  
_family key: `bookWalls` · 12 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `upDepth30` | Ask depth to +30% | quote | free | Resting sell liquidity from the best ask up to 30% above it. |
| `upDepth60` | Ask depth to +60% | quote | pro | Resting sell liquidity from the best ask up to 60% above it. |
| `upDepth100` | Ask depth to +100% | quote | pro | Resting sell liquidity from the best ask up to 100% above it. |
| `upDepth200` | Ask depth to +200% | quote | pro | Resting sell liquidity from the best ask up to 200% above it. |
| `upDepth300` | Ask depth to +300% | quote | max | Resting sell liquidity from the best ask up to 300% above it. |
| `upDepth400` | Ask depth to +400% | quote | max | Resting sell liquidity from the best ask up to 400% above it. |
| `upDepthFull` | Total ask depth | quote | max | All resting sell liquidity across the entire order book. |
| `downDepth5` | Bid depth to -5% | quote | free | Resting buy liquidity from the best bid down to 5% below it. |
| `downDepth10` | Bid depth to -10% | quote | pro | Resting buy liquidity from the best bid down to 10% below it. |
| `downDepth15` | Bid depth to -15% | quote | pro | Resting buy liquidity from the best bid down to 15% below it. |
| `downDepth20` | Bid depth to -20% | quote | pro | Resting buy liquidity from the best bid down to 20% below it. |
| `downDepthFull` | Total bid depth | quote | max | All resting buy liquidity across the entire order book. |

## Order ladders — fixed 3% slices  
_family key: `orderLadders` · 14 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `buyOrderVol3` | Bid volume 0 to 3% | quote | free | Resting buy liquidity in the slice from 0 to 3% below top of book. |
| `buyOrderVol6` | Bid volume 3 to 6% | quote | free | Resting buy liquidity in the slice from 3 to 6% below top of book. |
| `buyOrderVol9` | Bid volume 6 to 9% | quote | pro | Resting buy liquidity in the slice from 6 to 9% below top of book. |
| `buyOrderVol12` | Bid volume 9 to 12% | quote | pro | Resting buy liquidity in the slice from 9 to 12% below top of book. |
| `buyOrderVol15` | Bid volume 12 to 15% | quote | pro | Resting buy liquidity in the slice from 12 to 15% below top of book. |
| `buyOrderVol18` | Bid volume 15 to 18% | quote | max | Resting buy liquidity in the slice from 15 to 18% below top of book. |
| `buyOrderVol21` | Bid volume 18 to 21% | quote | max | Resting buy liquidity in the slice from 18 to 21% below top of book. |
| `sellOrderVol3` | Ask volume 0 to 3% | quote | free | Resting sell liquidity in the slice from 0 to 3% above top of book. |
| `sellOrderVol6` | Ask volume 3 to 6% | quote | free | Resting sell liquidity in the slice from 3 to 6% above top of book. |
| `sellOrderVol9` | Ask volume 6 to 9% | quote | pro | Resting sell liquidity in the slice from 6 to 9% above top of book. |
| `sellOrderVol12` | Ask volume 9 to 12% | quote | pro | Resting sell liquidity in the slice from 9 to 12% above top of book. |
| `sellOrderVol15` | Ask volume 12 to 15% | quote | pro | Resting sell liquidity in the slice from 12 to 15% above top of book. |
| `sellOrderVol18` | Ask volume 15 to 18% | quote | max | Resting sell liquidity in the slice from 15 to 18% above top of book. |
| `sellOrderVol21` | Ask volume 18 to 21% | quote | max | Resting sell liquidity in the slice from 18 to 21% above top of book. |

## Book microstructure — liquidity add / remove / flicker / lifetime  
_family key: `bookMicro` · 11 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `bestBid` | Best bid | price | free | The highest bid price at the moment the window closed. |
| `bestAsk` | Best ask | price | free | The lowest ask price at the moment the window closed. |
| `bookUpdateCount` | Book update count | count | pro | Number of order-book change events during the window. |
| `bidLevelCount` | Bid level count | count | pro | Number of price levels on the bid side at window close. |
| `askLevelCount` | Ask level count | count | pro | Number of price levels on the ask side at window close. |
| `bidLiqAdded` | Bid liquidity added | quote | max | Buy-side resting liquidity placed into the book during the window, in quote units. |
| `bidLiqRemoved` | Bid liquidity removed | quote | max | Buy-side resting liquidity withdrawn from the book during the window — the portion of size decrease not explained by executed trades (i.e. cancellations), in quote units. |
| `askLiqAdded` | Ask liquidity added | quote | max | Sell-side resting liquidity placed into the book during the window, in quote units. |
| `askLiqRemoved` | Ask liquidity removed | quote | max | Sell-side resting liquidity withdrawn from the book during the window — the portion of size decrease not explained by executed trades (i.e. cancellations), in quote units. |
| `levelFlickerCount` | Level flicker count | count | max | Count of price levels that appeared, vanished, then reappeared within the window (a placed-removed-placed cycle). |
| `levelLifetimeMedianTime` | Median level lifetime | ms | max | Median lifetime, in ms, of price levels that were both created and removed inside the window. |

## Trade timing & cadence  
_family key: `tradeTiming` · 8 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `tradeSilenceMaxTime` | Longest trade silence | ms | max | The longest gap between trades during the window. |
| `tradeGapModeTime` | Common trade interval | ms | max | The most frequent gap between consecutive trades. |
| `tradeGapModeCount` | Common interval count | count | max | How many trades followed the most common interval. |
| `sameQtyTradeCount` | Same-size trade count | count | max | Trades sharing an exact quantity in groups of three or more. |
| `sameQtyMaxCount` | Largest same-size group | count | max | The biggest group of trades sharing an exact quantity. |
| `atc` | Same-instant trade count | count | max | Trades sharing the same millisecond timestamp in clusters of three or more. |
| `atcMaxCluster` | Largest same-instant cluster | count | max | The biggest cluster of trades sharing one millisecond timestamp. |
| `ltc` | Loser round-trip count | count | max | Same-quantity buy then sell round-trips closed at a loss within 30 minutes. |

## Strong (outsized) trades  
_family key: `strong` · 19 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `stc50` | Strong trade count 1.5x | count | pro | Trades at least 1.5 times the average trade size in the window. |
| `stc100` | Strong trade count 2x | count | pro | Trades at least 2 times the average trade size in the window. |
| `stc200` | Strong trade count 3x | count | max | Trades at least 3 times the average trade size in the window. |
| `sbc50` | Strong buy count 1.5x | count | pro | Taker-buy trades at least 1.5 times the average trade size. |
| `sbc100` | Strong buy count 2x | count | pro | Taker-buy trades at least 2 times the average trade size. |
| `sbc200` | Strong buy count 3x | count | max | Taker-buy trades at least 3 times the average trade size. |
| `sbc500` | Strong buy count 6x | count | max | Taker-buy trades at least 6 times the average trade size. |
| `ssc50` | Strong sell count 1.5x | count | pro | Taker-sell trades at least 1.5 times the average trade size. |
| `ssc100` | Strong sell count 2x | count | pro | Taker-sell trades at least 2 times the average trade size. |
| `ssc200` | Strong sell count 3x | count | max | Taker-sell trades at least 3 times the average trade size. |
| `ssc500` | Strong sell count 6x | count | max | Taker-sell trades at least 6 times the average trade size. |
| `sbcVol50` | Strong buy volume 1.5x | quote | pro | Total value of taker-buy trades at least 1.5 times the average size. |
| `sbcVol100` | Strong buy volume 2x | quote | pro | Total value of taker-buy trades at least 2 times the average size. |
| `sbcVol200` | Strong buy volume 3x | quote | max | Total value of taker-buy trades at least 3 times the average size. |
| `sbcVol500` | Strong buy volume 6x | quote | max | Total value of taker-buy trades at least 6 times the average size. |
| `ssVol50` | Strong sell volume 1.5x | quote | pro | Total value of taker-sell trades at least 1.5 times the average size. |
| `ssVol100` | Strong sell volume 2x | quote | pro | Total value of taker-sell trades at least 2 times the average size. |
| `ssVol200` | Strong sell volume 3x | quote | max | Total value of taker-sell trades at least 3 times the average size. |
| `ssVol500` | Strong sell volume 6x | quote | max | Total value of taker-sell trades at least 6 times the average size. |

## Enrichment (market-cap / attention)  
_family key: `enrichment` · 9 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `cgMarketCap` | CoinGecko market cap | usd | free | The coin market capitalisation reported by CoinGecko. |
| `cgRank` | CoinGecko rank | index | free | The coin market-cap rank on CoinGecko. |
| `cgAprox` | CoinGecko match confidence | usd | pro | How closely the pair price matched a CoinGecko candidate. |
| `cmcMarketCap` | CoinMarketCap market cap | usd | pro | The coin market capitalisation reported by CoinMarketCap. |
| `cmcDilutedMc` | CoinMarketCap diluted cap | usd | pro | The fully diluted market cap reported by CoinMarketCap. |
| `cmcSelfMc` | CoinMarketCap self-reported cap | usd | pro | The self-reported market cap from CoinMarketCap. |
| `cmcRank` | CoinMarketCap rank | index | pro | The coin market-cap rank on CoinMarketCap. |
| `cmcLiquidity` | CoinMarketCap liquidity | usd | pro | The effective liquidity for the pair from CoinMarketCap. |
| `cmcVolume` | CoinMarketCap volume | usd | pro | The 24-hour trading volume for the pair from CoinMarketCap. |

## Market context  
_family key: `context` · 5 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `btcPriceUsd` | Bitcoin price | price | free | The reference Bitcoin price in USD at the snapshot time. |
| `ethPriceUsd` | Ethereum price | price | free | The reference Ethereum price in USD at the snapshot time. |
| `fearGreed` | Fear and greed index | index | max | The market-wide crypto fear and greed reading. |
| `fearGreedCmc` | Fear and greed index (CMC) | index | max | The CoinMarketCap version of the fear and greed reading. |
| `searchInterest` | Search interest | index | max | Relative search interest in the coin. |

## Quality & units (data-integrity fields)  
_family key: `quality` · 7 metrics_

| metric key | label | unit | min plan | measurement |
|---|---|---|---|---|
| `quoteAsset` | Quote asset | index | free | The currency the pair is quoted in. |
| `quoteUsdRate` | Quote to USD rate | ratio | free | The rate to convert the quote asset into USD. |
| `enrichmentTs` | Enrichment time | ms | free | The time the enrichment data was captured. |
| `bookSynced` | Book synced | bool | free | Whether the order book was in sync when the snapshot was taken. |
| `bookAgeTime` | Book age | ms | free | How long the order book had been running at snapshot time. |
| `seedDepth` | Seed depth | count | free | The number of levels the order book was seeded with. |
| `missingTrades` | Missing trades | bool | free | Whether some trades may have been missed in the window. |
