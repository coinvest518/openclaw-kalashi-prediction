# Kalshi BTC Agent

[![GitHub Stars](https://img.shields.io/github/stars/coinvest518/openclaw-kalashi-prediction?style=social)](https://github.com/coinvest518/openclaw-kalashi-prediction/stargazers)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.8%2B-blue)](#)

An OpenClaw agent that trades Bitcoin hourly prediction markets on Kalshi. It learns from a 1-year historical dataset, uses Kalshi's free public API for live data, and writes what it learns from every trade into memory so it gets better over time.

Disclaimer: Trading on Kalshi involves risk. This is not financial advice. Always run in `PAPER_MODE=true` before going live. Past performance is not indicative of future results. Kalshi is regulated by the CFTC.

## What's in This Repo

```
SOUL.md      ← who the agent is
AGENTS.md    ← strategy, workflow, API calls, memory rules
SKILL.md     ← tools and commands
MEMORY.md    ← long-term memory (agent fills this in over time)
README.md    ← this file
LICENSE      ← MIT license
```

Drop all five files into your OpenClaw workspace folder and you're set.

## Quick Setup

1. Get a Kalshi API key (for placing trades). Market data is free without one.
   - kalshi.com → Account Settings → API Keys → Create New API Key
   - Save your Key ID and download the `.key` file

2. Install dependencies

```bash
pip install requests pandas python-dotenv
```

3. Set environment variables (example)

```bash
KALSHI_KEY_ID=your-key-id-here
KALSHI_PRIVATE_KEY_PATH=./kalshi.key
PAPER_MODE=true
PAPER_BALANCE=1000
```

4. Place your historical data file at:

```
data/kalshi_data_final.json
```

5. Load historical patterns into memory:

```bash
python3 scripts/load_history.py
```

6. Start the scanner/trader (runs every 15 minutes):

```bash
python3 scripts/scan_markets.py
```

## API Endpoints Used

Purpose | Endpoint
---|---
Scan markets | GET /trade-api/v2/markets?status=open&series_ticker=KXBTC
Get orderbook | GET /trade-api/v2/markets/{ticker}/orderbook
Get balance | GET /trade-api/v2/portfolio/balance
Place order | POST /trade-api/v2/portfolio/orders
Cancel order | DELETE /trade-api/v2/portfolio/orders/{order_id}

Kalshi WebSocket: `wss://api.elections.kalshi.com/trade-api/ws/v2`

Full API docs: https://docs.kalshi.com

## How It Works (summary)

- Learn from history: `scripts/load_history.py` reads `kalshi_data_final.json` and extracts resolution patterns into `MEMORY.md`.
- Find live opportunities: `scripts/scan_markets.py` calls Kalshi's public API every 15 minutes and checks open BTC hourly markets.
- Spot the gap: A trade is considered when the current YES price is at least 7¢ below what history indicates it should be worth.
- Trade with rules: `risk_check.py` validates position size, liquidity, time to resolution, and available capital.
- Learn from outcomes: After each resolution, the agent writes a daily note to `memory/YYYY-MM-DD.md` and updates `MEMORY.md` over time.

## Memory System

- `MEMORY.md` — long-term curated memory of confirmed patterns and stats.
- `memory/YYYY-MM-DD.md` — daily logs of trades and observations.

## Risk Rules (core)

- Only trade markets 10–50 minutes from resolution
- Minimum 7¢ gap between live price and historical resolution rate
- Minimum 20 contracts available on entry side
- Maximum 5% of portfolio per single trade
- Maximum 40% of portfolio deployed at once
- Cancel unfilled orders after 90 seconds
- Stop-loss at −40% of position value

## Environment & Safe Mode

Always test in paper mode:

```bash
export PAPER_MODE=true
```

Set `PAPER_BALANCE` to simulate starting capital.

## Contributing & Stars

If you find this project useful, please consider starring the repo and opening issues or PRs with improvements.

[![Star on GitHub](https://img.shields.io/github/stars/coinvest518/openclaw-kalashi-prediction?style=social)](https://github.com/coinvest518/openclaw-kalashi-prediction/stargazers)

## License

MIT — see the `LICENSE` file.

---

Trading on Kalshi involves risk. Not financial advice.
