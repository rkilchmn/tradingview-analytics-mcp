---
name: cash-secured-put-writing
description: Cash-secured put (and covered call) trade planning — "tasty trade" style premium selling. Use when the user asks for a put/call selling setup, delta selection, strike selection, OTM probability, or a trade plan with entry/breakeven/management rules. Combines TradingView MCP data with the 30-35 delta framework, technical confluence, and Black-Scholes math.
---

# Cash-Secured Put Writing

Build premium-selling trade plans. Data comes from the `tradingview-mcp` skill (analysis, chain, Fibonacci) and optionally the `ibkr-mcp` skill (live bid/ask, greeks, order placement).

## 1. Context (parallel calls)

- `tradingview_yahoo_price` — spot
- `tradingview_coin_analysis` 1D + 4H — SMAs, RSI, MACD, support/resistance
- `tradingview_multi_timeframe_analysis` — weekly bias vs daily alignment
- `tradingview_fibonacci_retracement` — swing retracement levels

## 2. Expiry

- Prefer regular monthlies (3rd Friday) over weeklies (thin chains, wide spreads, missing strikes).
- Target **~45 DTE**; the nearest monthly at 58-60 DTE is acceptable (more theta runway).
- Compute DTE from today's date and state it explicitly.

## 3. Delta — compute yourself

The TradingView chain has no delta. Black-Scholes:

```
T   = DTE / 365
d1  = [ln(S/K) + (r + sigma^2/2) * T] / (sigma * sqrt(T))
d2  = d1 - sigma * sqrt(T)

put delta  = N(d1) - 1
P(put expires OTM)  = N(d2)
```

- Use **each strike's own IV** from the chain; assume r = 4% and state the assumption.
- `in_the_money` in the chain is spot-vs-strike only — don't confuse with ITM probability.
- If the IBKR MCP is available, option market data returns greeks for free — prefer those over manual math.
- Present a delta ladder table: strike, ~delta, P(OTM), bid/ask, mid, premium %, vol/OI, breakeven.

## 4. Strike selection (tasty-trade framework)

- **30-35 delta** is the sweet spot for premium selling (~60-70% OTM probability). If asked why: premium falls off a cliff below ~30 delta while convex tail risk does not; theta-per-dollar peaks in the 25-35 band.
- Cross-reference strikes with technical levels:
  - Fibonacci retracements of the current swing — 0.382 / 0.5 / 0.618
  - SMA200 (1D), 4H SMA200, S1/S2/S3, swing low of the current trend leg
- Prefer strikes where the **breakeven (strike - premium for puts) sits at or below a technical support** — assignment then happens at a price the chart supports.
- **Liquidity filter:** check OI, volume, and bid/ask spread. Flag spreads > $0.50 as "post a patient limit at mid, don't cross". Flag zero-OI far-OTM contracts as untradeable (stale quotes are common there).
- If the exact delta-zone strike has bad liquidity, offer the nearest liquid strike as the alternative with its trade-offs stated.

## 5. Trade plan output format

For the recommended strike (plus one fallback):

| Item | Value |
|------|-------|
| Action | Sell to open, N contracts |
| Entry | Limit at or below mid (never the ask on wide spreads) |
| Premium | $/contract and % of strike |
| Collateral | strike * 100 (cash-secured) |
| Breakeven | strike - premium (puts) |
| P(expire OTM) | from N(d2) |
| Return | % over DTE and annualized |
| Max profit | full premium |

Plus management rules:
1. **Early close — 2/3 in 1/3 rule:** if 2/3 of the max premium has been captured within the first 1/3 of the trade duration, close the position and redeploy capital (take the win early — don't wait for the 50% target if this condition is met)
2. **Take profit at 50%** of max premium (typical exit when the 2/3 rule hasn't triggered; usually reached in 2-4 weeks)
3. **Roll check at 21 DTE** if delta > 50 (roll down-and-back to restore ~30 delta)
4. **Assignment stance** — state whether the effective entry price is at a technically supported level (acceptable) or not (roll earlier)

Always end with the key risk (e.g. "strike sits $0.12 above the 0.5 fib — a normal correction tags it ITM").
