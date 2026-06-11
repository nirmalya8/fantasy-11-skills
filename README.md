# World Cup Fantasy Picks

This package contains two skills:

- `pick-fantasy-xi` for the daily Fantasy XI answer
- `choose-risk-play` for the optional Risk Play decision

Tight rule: use `game-board/players.json` as the only source of `player_id` values. Never invent players, teams, positions, eligibility, stats, injuries, suspensions, or other details.

Use game-board metrics first. Public web data may be used only as an optional tie-breaker when available, and must never override the board's player pool, player IDs, eligibility, or explicit manual exclusions.

## Manual player exclusions

The following player IDs must not be selected by `pick-fantasy-xi`:

- player_id 276: injured
- player_id 545: injured

Manual exclusions are hard constraints. Excluded player IDs are matched against `game-board/players.json`, removed before scoring, and must not appear in the final XI.

Player names are optional when a `player_id` is provided.