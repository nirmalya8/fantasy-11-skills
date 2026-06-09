---
name: pick-fantasy-xi
description: pick a legal fifa world cup fantasy xi from the daily game board using only player_id values from game-board/players.json, dynamically reading whatever player metrics the board exposes, and using public web data only as a fallback tie-breaker for the previous club season and completed world cup matches; use when the daily prompt asks for a valid lineup under the tournament answer contract.
---

# Pick Fantasy XI

## Goal
Choose the highest-projection legal XI.

## Hard source rules
- Build the candidate pool only from player records in `game-board/players.json`.
- Never invent or infer a player_id, team, position, eligibility flag, stat, or minute total that is not explicitly present.
- Ignore any player not present in `players.json`, even if external knowledge or web data mentions them.
- Treat `players.json` as the source of truth for player_id and eligibility.
- Use public web data only as a fallback signal when the board is incomplete, and never to override the player_id list from the board.

## Metric discovery
- Inspect the actual fields in `game-board/players.json` and any related team or match files on every run.
- Use only metrics that are explicitly present in the current board.
- If a metric is missing, do not estimate it; fall back to the next strongest available signal.
- Prefer direct fantasy-relevant fields over reputation or memory.

## Useful signals when present
Treat these as optional inputs, not requirements:
- starts
- minutes
- goals
- assists
- shots
- xG
- xA
- key passes
- clean sheets
- saves
- cards
- set-piece role
- penalty role
- recent tournament form
- previous club-season form

## Scoring model
Optimize expected fantasy points using:
- player starts: +2
- player plays 60+ minutes: +2
- goal: +6
- assist: +4
- defender or goalkeeper clean sheet: +4
- goalkeeper 3+ saves: +2
- yellow card: -1
- red card: -3
- own goal: -3

## Selection method
1. Score each eligible player by expected points.
2. Enumerate all legal formations.
3. Select exactly 11 unique players.
4. Use exactly 1 goalkeeper.
5. Use 3 to 5 defenders.
6. Use 3 to 5 midfielders.
7. Use 1 to 3 forwards.
8. Prefer the legal formation with the highest projected total.

## Tie-breakers
When projections are close, prefer:
1. more projected starters
2. more projected 60+ minute players
3. lower card and own-goal risk
4. stronger set-piece or penalty involvement
5. safer minutes profile

## Validation
Before finalizing, verify:
- every selected player_id exists in `players.json`
- no duplicate player_ids appear
- the formation is legal
- no invented details were used