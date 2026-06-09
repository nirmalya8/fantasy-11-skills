---
name: pick-fantasy-xi
description: choose a legal fifa world cup fantasy xi from the daily game board by using only player_id values from game-board/players.json, dynamically reading whatever player metrics are actually present on the board, and using public web research only as an optional tie-breaker when available; use when the daily prompt asks for a valid lineup under the tournament answer contract.
---

# Pick Fantasy XI

## Goal
Choose the highest-projection legal XI from the current matchday board.

## Source of truth
- Read `game-board/matchday.json`, `game-board/matches.json`, `game-board/players.json`, `game-board/teams.json`, `rules/fantasy-xi.md`, `output-format/daily-submission.schema.json`, and `team/README.md` first.
- Build the candidate pool only from player records that exist in `game-board/players.json`.
- Only use players whose `eligible_matchday_ids` include the current `matchday_id`.
- Never invent or infer a `player_id`, team, position, eligibility flag, stat, or minute total that is not explicitly present.
- Ignore any player not present in `players.json`, even if outside knowledge or web research mentions them.

## Metric discovery
- Inspect the actual keys in each player record on every run.
- Read nested `prior_stats` and `prior_world_cup_record` when present.
- Use only metrics that are explicitly present in the board data.
- Do not assume a metric exists just because it exists on another player.
- If a metric is missing, do not estimate it or fill it in from memory.

## Projection model
Rank players by expected fantasy points using only available evidence.
Prioritize, when present:
- starts
- minutes
- goals
- assists
- saves for goalkeepers
- clean-sheet indicators for defenders and goalkeepers
- card risk as a negative signal
- own-goal risk as a negative signal if any explicit field exists

Use the board as the primary signal. If public web research is available and the run context allows it, use it only as a tie-breaker between close players, never to override the board’s player pool or eligibility.

## Selection method
1. Score every eligible player from the board.
2. Enumerate all legal formations.
3. Prefer the legal formation with the highest projected total.
4. Select exactly 11 unique `player_id` values.
5. Use exactly 1 goalkeeper.
6. Use 3 to 5 defenders.
7. Use 3 to 5 midfielders.
8. Use 1 to 3 forwards.

## Tie-breakers
When projections are close, prefer:
1. likely starter
2. higher projected minutes
3. stronger attacking involvement
4. lower card risk
5. stronger set-piece or penalty role, if explicitly present

## Validation
Before finalizing, verify:
- every selected `player_id` exists in `players.json`
- no duplicate `player_id` values appear
- the formation is legal
- no invented details were used

## Output discipline
Return only the daily submission JSON required by the tournament schema.