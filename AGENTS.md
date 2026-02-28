# AGENTS.md

## Goal
Trade Bitcoin hourly and 15-minute prediction markets on Kalshi. Use the 1-year historical dataset to understand what prices tend to resolve YES or NO. Use Kalshi's free public API for live data. Learn from every trade and write findings into memory so decisions get better over time.

---

## Data Sources

### Historical Data (Training)
Your 1-year Kalshi dataset lives at `data/kalshi_data_final.json`.
On first run, read this file and extract:
- What % of BTC hourly markets resolved YES vs NO at different price levels
- Which price ranges hit most often (e.g. BTC $64,750–$65,000 resolved YES 68% of the time)
- Which times of day had the most mispriced markets
- Average spread between YES and NO bids at resolution

Write your findings into `MEMORY.md` under the section `## BTC Historical Patterns` so they persist forever.

### Live Data (Every Cycle)
Pull fresh data from Kalshi's free public REST API — no authentication required for market data:

```
Base URL: https://api.elections.kalshi.com/trade-api/v2

Get open BTC markets:
GET /markets?status=open&series_ticker=KXBTC&limit=50

Get specific market:
GET /markets/{ticker}

Get orderbook:
GET /markets/{ticker}/orderbook

Get recent trades:
GET /trades?ticker={ticker}&limit=20
```

No API key needed for any of the above. Just call them directly.

---

## Trading Strategy

### The Core Idea
Kalshi's BTC hourly markets often misprice contracts relative to what the historical data says they should be worth. When the current YES price is significantly lower than the historical resolution rate for similar conditions — that's the trade.

### How to Spot a Trade
Every cycle, for each open BTC market:

1. **Pull the current YES bid price** from the live API (in cents)
2. **Look up in MEMORY.md**: what % of similar historical markets resolved YES?
3. **Calculate the gap**: `historical_resolution_rate - current_yes_price_as_pct`
4. **If gap ≥ 7 cents** and market closes in 10–50 minutes → run risk check → trade

### Example
Memory says: "BTC markets where YES was priced 45–50¢ with 20–40 min to close resolved YES 64% of the time."
Current market: YES at 46¢, closes in 28 minutes.
Gap = 64 - 46 = 18 cents. That's a strong signal. BUY YES.

### Liquidity Check
Before trading, check the orderbook:
- YES bid + NO bid should be close to 100¢ (normal market)
- If sum < 95¢, liquidity was pulled — one side is mispriced, buy the cheaper side
- If depth on your side < 20 contracts, skip — too thin to fill cleanly

---

## Workflow (Every 15 Minutes)

```
1. SCAN
   → GET /markets?status=open&series_ticker=KXBTC
   → Filter to markets closing in 10–50 minutes

2. ANALYZE
   → For each market, pull YES price from API
   → Compare to historical resolution rates in MEMORY.md
   → Calculate gap

3. CHECK LIQUIDITY
   → GET /markets/{ticker}/orderbook
   → Confirm YES + NO sum > 95¢
   → Confirm depth ≥ 20 contracts on entry side

4. RISK CHECK
   → Run risk_check.py before any trade
   → All rules must pass — no exceptions

5. TRADE
   → If gap ≥ 7¢ and risk check passes → place_trade.py
   → Limit orders only. Never market orders.

6. MONITOR
   → Check open orders every 15 min
   → Cancel any unfilled orders after 90 seconds

7. LEARN (after every trade resolves)
   → Write outcome to memory/YYYY-MM-DD.md
   → Update win/loss patterns in MEMORY.md
   → If a pattern keeps winning → increase confidence weight
   → If a pattern keeps losing → flag it in MEMORY.md and avoid
```

---

## Memory Rules — How to Learn

### After Every Trade Resolves, Write to `memory/YYYY-MM-DD.md`:
```
## Trade: [TICKER]
- Side: YES/NO
- Entry price: X¢
- Historical rate from MEMORY: X%
- Gap at entry: X¢
- Outcome: WIN/LOSS
- Market closed at: X¢
- What I learned: [one sentence]
```

### Update `MEMORY.md` When You See Patterns:
- If the same setup wins 3+ times in a row → promote it to `## Confirmed Patterns` in MEMORY.md
- If a setup loses twice → add it to `## Patterns to Avoid` in MEMORY.md
- Update `## BTC Historical Patterns` with new resolution stats as more trades resolve
- Keep MEMORY.md curated — remove outdated entries, keep it clean

### MEMORY.md Structure (maintain this):
```
## BTC Historical Patterns
[Stats from the 1-year dataset + updated from live trades]

## Confirmed Patterns
[Setups that have proven profitable with win rate]

## Patterns to Avoid
[Setups that have lost — with reason]

## Best Trade Windows
[Times of day / minutes-to-close combos that work best]

## Current Stats
[Running win rate, avg gap captured, total trades]
```

---

## Risk Rules — Never Break These

| Rule | Limit |
|---|---|
| Min gap to trade | 7¢ (confirmed patterns can go to 5¢) |
| Min book depth | 20 contracts on entry side |
| Market entry window | 10–50 minutes before resolution |
| Max per trade | 5% of portfolio |
| Max deployed at once | 40% of portfolio |
| Fill timeout | 90 seconds — cancel if not filled |
| Stop-loss | −40% of position value |

---

## API Reference

All market data endpoints are free — no auth required.

```
Base: https://api.elections.kalshi.com/trade-api/v2

Scan BTC markets:   GET /markets?status=open&series_ticker=KXBTC
Get market:         GET /markets/{ticker}
Get orderbook:      GET /markets/{ticker}/orderbook
Get trades:         GET /trades?ticker={ticker}&limit=20
Get candlesticks:   GET /markets/{ticker}/candlesticks?period_seconds=3600
```

For placing trades, auth is required (RSA-PSS SHA256 signing):
```
KALSHI-ACCESS-KEY        → your Key ID
KALSHI-ACCESS-TIMESTAMP  → unix time in milliseconds
KALSHI-ACCESS-SIGNATURE  → base64(RSA-PSS-SHA256(timestamp + METHOD + path))

Prices in CENTS (1–99). $0.47 = 47.

POST /trade-api/v2/portfolio/orders
{
  "ticker": "KXBTC-25FEB2826-T65249.99",
  "action": "buy",
  "side": "yes",
  "count": 10,
  "type": "limit",
  "yes_price": 47,
  "client_order_id": "uuid4"
}
```

---

## Tools

| Command | What it does |
|---|---|
| `python3 scripts/load_history.py` | Reads kalshi_data_final.json, writes patterns to MEMORY.md |
| `python3 scripts/scan_markets.py` | Gets open BTC markets from live API |
| `python3 scripts/get_market_data.py --ticker T` | Live price + orderbook for one market |
| `python3 scripts/risk_check.py [params]` | Validates trade before execution |
| `python3 scripts/place_trade.py [params]` | Places limit order via Kalshi API |
| `python3 scripts/get_portfolio.py` | Balance, positions, P&L |
| `python3 scripts/daily_report.py` | End-of-day summary + memory update prompt |

---

## Sub-Agents (if using multi-agent mode)

| Agent ID | Job |
|---|---|
| `scanner` | Runs scan + analyze every 15 min, posts signals to coordinator |
| `executor` | Receives signals, runs risk check, places trades |
| `memory-keeper` | After each resolved trade, writes outcome to memory files |
| `reporter` | Daily summary at 11PM, updates MEMORY.md patterns |