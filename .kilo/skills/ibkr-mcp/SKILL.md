---
name: ibkr-mcp
description: Interactive Brokers MCP workflow — account state, positions, options chains, live quotes, and order management via ibkr_* tools. Use when checking the IBKR account, getting real bid/ask or greeks, or placing/canceling orders. Covers the canonical quote-resolution pattern (SMART contract details → unfiltered chain discovery → filtered chain → batch market data) and known quirks.
---

# IBKR MCP

Tools: `ibkr_get_connection_status`, `ibkr_get_account_summary`, `ibkr_get_account_values`, `ibkr_get_positions` / `ibkr_get_positions_detailed`, `ibkr_get_market_data`, `ibkr_get_contract_details`, `ibkr_get_options_chain`, `ibkr_get_filtered_options_chain`, `ibkr_get_historical_data`, `ibkr_place_order`, `ibkr_cancel_order`, `ibkr_get_open_orders`, `ibkr_reconnect`, `ibkr_get_ibkr_gateway_status` / `ibkr_get_ibkr_gateway_logs`.

Underlying REST API (what the MCP wraps): localhost:8000, FastAPI. Current surface discoverable at `GET /openapi.json` — check it after any server rebuild, endpoints get renamed.

## Canonical quote pattern (US stocks/ETFs, verified 2026-08-19)

**Step 1 — resolve the underlying** (`exchange=SMART` works for STK):

```
ibkr_get_contract_details(symbol="GDX", sec_type="STK", exchange="SMART", currency="USD")
  -> qualified_contract.con_id   (e.g. GDX = 229726316)
```

**Step 2 — unfiltered chain for expiry/strike discovery** (returns `candidate_chains` with all available expirations and strike ranges):

```
ibkr_get_options_chain(
  underlying_symbol="GDX", underlying_sec_type="STK", underlying_con_id=229726316,
  exchange="SMART")
  -> candidate_chains[0].expirations, candidate_chains[0].strikes
```

**Step 3 — filtered chain** (contract resolution; `exchange=SMART`):

```
ibkr_get_options_chain(
  underlying_symbol="GDX", underlying_sec_type="STK", underlying_con_id=229726316,
  exchange="SMART",
  filters='{"trading_class": ["GDX"], "expirations": ["20260918"], "rights": ["P"],
            "strikes": [84, 85, 86]}')
  -> con_id per contract (local_symbol, friendly_symbol, strike, right, expiry)
```

Filter keys: `trading_class`, `expirations`, `rights`, `strikes` (snake_case verified). **The chain returns contract fields only** — an older build joined market data (last_price, greeks, probability, costBase) into each entry; that join is gone, so get quotes in step 4.

**Step 4 — batch quotes for the shortlist** (MCP bridge now forwards arrays as repeated query params — one call for the whole ladder):

```
ibkr_get_market_data(contract_ids=[891797951, 812712858, 846346335, ...])
  -> close, bid/ask (null when market closed), greeks (often null post-restart)
```

**Underlying price** — realtime returns `[]` for stocks, use delayed:

```
ibkr_get_market_data(symbol="XLE", sec_type="STK", exchange="SMART", subscription_type="delayed")
```

**Delta ladder:** greeks in market data are intermittent (often `null` right after a server restart — model-value subscription lag). When greeks are missing, compute delta/P(OTM) manually via Black-Scholes from `close` + IV — see the `cash-secured-put-writing` skill.

## Orders

```
ibkr_place_order(
  contract={symbol, sec_type, exchange, currency, strike, right, expiry, ...},
  order={action: "SELL", total_quantity, order_type: "LMT", lmt_price, time_in_force: "DAY"})
  -> order_id
ibkr_cancel_order(order_id)
ibkr_get_open_orders()   # verify
```

New orders may sit in `PreSubmitted` until TWS/Gateway accepts them.

## Quirks

- `get_contract_details` with `exchange="SMART"` returned empty `candidate_contracts` for **OPT** at the MCP level — use the chain (steps 2–3) as the option con_id source instead of per-contract OPT resolution.
- Stock market data: `subscription_type=realtime` returns `[]` — use `delayed` with `exchange="SMART"`.
- Bid/ask are `null` when the US market is closed; `close` carries the previous close.
- `ibkr_get_filtered_options_chain` (REST `/ibkr/market_data/filtered_options_chain`) currently returns `[]` for valid filters/criteria — treat as broken until verified working again.
- Not every round strike exists (e.g. XLE puts step $2.50 in the lower range) — if a strike is missing from the chain, use the nearest available and say so.
- Endpoints get renamed between builds (`/ibkr/tickers` was removed in one rebuild, now `/ibkr/market_data`) — on a 404, fetch `/openapi.json` for the current surface.
- If all tools (even no-arg ones) return `-32602 Invalid request parameters`, the MCP bridge or the localhost:8000 server was restarted/misconfigured — check the server process, not the request.
- Connection problems: `ibkr_reconnect()`; gateway container health: `ibkr_get_ibkr_gateway_status`.
