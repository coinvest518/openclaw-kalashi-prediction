# SKILL.md

name: kalshi-btc-agent
description: >
  Bitcoin prediction market trading agent for Kalshi. Learns from a 1-year
  historical JSON dataset, calls Kalshi's free public REST API for live data,
  and writes every trade outcome into memory so it improves over time.
  No WebSockets. No external price feeds. Just Kalshi data + memory.

requires:
  - requests
  - pandas
  - python-dotenv

env:
  KALSHI_KEY_ID=your-key-id-here
  KALSHI_PRIVATE_KEY_PATH=./kalshi.key
  PAPER_MODE=true
  PAPER_BALANCE=1000
  BASE_URL=https://demo-api.kalshi.co

---

## How the Data Flow Works

```
kalshi_data_final.json          (your 1-year dataset)
        ↓
  load_history.py               (reads it once, writes patterns to MEMORY.md)
        ↓
  MEMORY.md                     (permanent pattern storage — persists forever)
        ↓
  Every 15 min:
  scan_markets.py               (GET /markets — free public API, no auth)
        ↓
  compare live price             (current YES bid vs MEMORY.md historical rate)
        ↓
  if gap ≥ 7¢:
  risk_check.py → place_trade.py
        ↓
  trade resolves:
  write outcome → memory/YYYY-MM-DD.md
  update patterns → MEMORY.md
```

---

## Tools & Commands

### load_history.py
Run this once at setup. Reads `data/kalshi_data_final.json`, extracts BTC resolution stats, and writes findings into `MEMORY.md`. This is how the agent learns from the historical data before it ever places a trade.

```bash
python3 scripts/load_history.py
# Writes to MEMORY.md:
# - Resolution rates by price range
# - Best time windows
# - Historical spread data
```

---

### scan_markets.py
Pulls all open BTC hourly/15-min markets from Kalshi's free public API. Filters to markets closing in 10–50 minutes. Compares each YES price to historical rates in MEMORY.md. Prints signals ranked by gap size.

```bash
python3 scripts/scan_markets.py
python3 scripts/scan_markets.py --closes-in 45
```

Free API call (no auth):
```
GET https://api.elections.kalshi.com/trade-api/v2/markets
    ?status=open&series_ticker=KXBTC&limit=50
```

---

### get_market_data.py
Checks a single market — live YES/NO prices, orderbook depth, recent trades.

```bash
python3 scripts/get_market_data.py --ticker KXBTC-25FEB2826-T65249.99
python3 scripts/get_market_data.py --ticker KXBTC-25FEB2826-T65249.99 --orderbook
```

Orderbook to check:
```
YES bid + NO bid should be ~97–99¢ (normal)
If sum < 95¢ → liquidity pulled → signal
If depth < 20 contracts → skip, too thin
```

---

### risk_check.py
Always runs before place_trade.py. Validates gap, depth, position size, deployed capital, time to resolution. Exits code 0 (approved) or 1 (rejected).

```bash
python3 scripts/risk_check.py \
  --ticker       KXBTC-25FEB2826-T65249.99 \
  --side         YES \
  --contracts    10 \
  --price        47 \
  --gap-cents    14 \
  --depth        120 \
  --closes-in    28
```

---

### place_trade.py
Places a limit order. Uses RSA-PSS auth. Auto-cancels if unfilled after 90 seconds. Logs every trade to `logs/trades.jsonl`.

```bash
python3 scripts/place_trade.py \
  --ticker     KXBTC-25FEB2826-T65249.99 \
  --side       YES \
  --contracts  10 \
  --price      47 \
  --reasoning  "Historical rate 64%, current price 47¢, gap 17¢"
```

Order format (prices in CENTS):
```json
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

### get_portfolio.py
Balance, open positions, P&L, win rate.

```bash
python3 scripts/get_portfolio.py
```

---

### daily_report.py
End-of-day summary. Prints win/loss breakdown. Prompts agent to update MEMORY.md with new patterns learned today.

```bash
python3 scripts/daily_report.py
```

---

## Memory System

OpenClaw uses two memory layers. The agent reads and writes to both automatically.

**`MEMORY.md`** — permanent long-term memory. Curated. Only the most important patterns live here. Updated when a pattern is confirmed across multiple trades. Never gets noisy.

**`memory/YYYY-MM-DD.md`** — daily log. Every trade logged here on resolution. Running notes. Gets heavy over time but that's fine — it's a journal.

The agent auto-flushes to memory before context compaction so nothing important is lost.

---

## Usage Examples

```
"first time setup — load the historical data"
→ python3 scripts/load_history.py

"what BTC markets are open right now?"
→ python3 scripts/scan_markets.py

"check the orderbook on this market"
→ python3 scripts/get_market_data.py --ticker KXBTC-25FEB2826-T65249.99 --orderbook

"scan says there's a 14¢ gap on KXBTC-25FEB2826-T65249.99, YES at 47¢, 28 min left, 120 contracts available"
→ python3 scripts/risk_check.py --ticker KXBTC-25FEB2826-T65249.99 --side YES --contracts 10 --price 47 --gap-cents 14 --depth 120 --closes-in 28
→ (if approved) python3 scripts/place_trade.py --ticker KXBTC-25FEB2826-T65249.99 --side YES --contracts 10 --price 47 --reasoning "Historical 64%, priced at 47¢, 17¢ gap"

"show my portfolio"
→ python3 scripts/get_portfolio.py

"end of day"
→ python3 scripts/daily_report.py
```

---

## Common Errors

| Error | Fix |
|---|---|
| 401 Unauthorized | Check KALSHI_KEY_ID and key file path |
| 400 Bad Request | Price must be integer 1–99 (cents not decimals) |
| 409 Conflict | Duplicate order — generate fresh UUID4 |
| 429 Rate limit | Add 0.5s delay between requests |