# MEMORY.md

This file is the agent's long-term brain. It is read at the start of every session.
Keep it curated — only confirmed, durable insights live here.
Daily notes and raw trade logs go in `memory/YYYY-MM-DD.md`.

---

## BTC Historical Patterns
*Populated by load_history.py on first run. Updated as live trades resolve.*

[ Agent fills this in after running load_history.py on kalshi_data_final.json ]

Example format once populated:
- BTC YES markets priced 40–50¢ with 20–40 min to close → resolved YES X% of the time
- BTC range markets: "$64,750–$65,000" range → hit X% of the time historically
- Highest resolution accuracy window: X:00–X:00 UTC
- Average YES+NO spread at resolution: X¢

---

## Confirmed Patterns
*Only add a pattern here after it has won 3+ times.*

[ Empty until the agent starts trading and confirms patterns ]

---

## Patterns to Avoid
*Add here after a setup loses twice. Include why.*

[ Empty until the agent starts trading and identifies failing setups ]

---

## Best Trade Windows
*Times of day and minutes-to-close combos that produce the best results.*

[ Populated from historical data and live trade outcomes over time ]

---

## Current Stats
*Updated by daily_report.py*

- Total trades: 0
- Win rate: —
- Avg gap captured: —
- Best performing signal: —
- Last updated: —

---

## Notes
*Anything else worth remembering long-term. Agent writes here when it notices something important.*

[ Agent adds notes here over time ]