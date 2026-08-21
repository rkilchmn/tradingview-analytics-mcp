---
name: tradingview-mcp
description: Knowledge of the TradingView MCP server — multi-market screener for crypto, stocks, and futures. Use when fetching prices, technical analysis, options chains, Fibonacci levels, or market scans via tradingview_* tools. Covers tool selection, exchange conventions, and known quirks.
---

# TradingView MCP

Multi-market screener backed by TradingView. Supports crypto (KuCoin, Binance, Bybit, MEXC...), stocks (EGX, BIST, NASDAQ, NYSE, HKEX, SSE, SZSE, TWSE, TPEX...), and futures (CME, COMEX, NYMEX, CBOT — equity index, energy, metals, agriculture, rates, forex, crypto).

## Exchange conventions

- `tradingview_coin_analysis`, `tradingview_multi_timeframe_analysis`, `tradingview_fibonacci_retracement`: bare ticker + `exchange` param (e.g. `GDX` + `AMEX`, `AAPL` + `NASDAQ`).
- `tradingview_stock_prices`: `EXCHANGE:TICKER` format (e.g. `NASDAQ:AAPL`).
- GDX trades on AMEX. If unsure which exchange lists a symbol, read the `exchanges://list` resource.
- `tradingview_coin_analysis` works for stocks despite the "coin" name.

## Common workflows

### Context gathering (parallel calls)

- `tradingview_yahoo_price(symbol)` — spot (Yahoo symbols: `AAPL`, `BTC-USD`, `^GSPC`)
- `tradingview_coin_analysis(symbol, exchange, timeframe="1D")` — SMA20/50/200, EMA, RSI, MACD, `support_resistance` (pivot, R1-R3, S1-S3), trend state
- `tradingview_coin_analysis(symbol, exchange, timeframe="4h")` — finer structure
- `tradingview_multi_timeframe_analysis(symbol, exchange)` — weekly vs daily alignment
- `tradingview_combined_analysis` — TA + sentiment + news in one call (sentiment may be unconfigured)

### Fibonacci

`tradingview_fibonacci_retracement(symbol, exchange, lookback, timeframe)` — exchange-agnostic. `lookback`: `1M`, `3M`, `6M`, `52W` (default), `ALL`. Falls back to pivot R3/S3 as the swing when screener columns are unavailable — state which swing was used.

Manual fallback: swing low/high from `support_resistance` + `price_data` in the analysis calls; levels = `high - ratio * (high - low)` for 0.236 / 0.382 / 0.5 / 0.618 / 0.786. Report which technical levels confluence with each fib.

### Options chain (TradingView)

`tradingview_stock_options_chain(symbol, expiry?)` — no expiry returns the nearest + `available_expiries`. Returns `strike, last_price, bid, ask, volume, open_interest, implied_volatility, in_the_money` — **no delta** (compute it, or use IBKR MCP greeks).

- Prefer regular monthlies (3rd Friday) over weeklies: weeklies have thin chains (e.g. GDX weekly: 16 calls/12 puts, $5 gaps near the money) vs monthlies (68/55, $1-wide strikes). Thin chains = wide spreads, missing strikes, stale quotes.
- `tradingview_stock_options_unusual_activity(symbol)` — V/OI screener for institutional positioning.

## Known quirks

- `tradingview_stock_prices` can time out — a retry usually succeeds.
- Large chain outputs get **truncated** by the harness; the full output is saved to a file path in the error. Delegate extraction to the `explore` subagent (Grep + Read with offset/limit) instead of reading the whole file.
- `combined_analysis` sentiment may return `MARKETAUX_API_TOKEN not configured` — proceed without sentiment, don't block on it.
- Far-OTM puts often have dirty quotes (last price far from bid/ask, zero OI) — exclude from recommendations.
- Some last prices in chains are stale prints — trust bid/ask over last for entry math.
