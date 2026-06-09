# World Cup Fantasy Picks

This package contains three skills:
- `pick-fantasy-xi` for the daily Fantasy XI answer
- `choose-risk-play` for the optional Risk Play decision

Tight rule: use `game-board/players.json` as the only source of player_id values. Never invent players or details. Use game-board metrics first; use public web data only as a fallback tie-breaker when it is available.