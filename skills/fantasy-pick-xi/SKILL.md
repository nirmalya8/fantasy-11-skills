---
name: pick-fantasy-xi
description: choose a legal fifa world cup fantasy xi from the daily game board by using only player_id values from game-board/players.json, dynamically reading whatever player metrics are actually present on the board, applying explicit manual exclusions such as injured or suspended players when provided in the skills package README, and using public web research only as an optional tie-breaker when available; use when the daily prompt asks for a valid lineup under the tournament answer contract.
---

# Pick Fantasy XI

## Goal

Choose the highest-projection legal XI from the current matchday board.

The `player_score` is a relative board-confidence score, not a predicted fantasy-point total. Use it only to rank players within the current matchday dataset.

## Source of truth

- Read the skills package `README.md`, `game-board/matchday.json`, `game-board/matches.json`, `game-board/players.json`, `game-board/teams.json`, `rules/fantasy-xi.md`, and `output-format/daily-submission.schema.json` first when present.
- Build the candidate pool only from player records that exist in `game-board/players.json`.
- Only use players whose `eligible_matchday_ids` include the current `matchday_id`.
- Never invent or infer a `player_id`, team, position, eligibility flag, stat, minute total, injury status, suspension status, or availability flag that is not explicitly present.
- Ignore any player not present in `players.json`, even if outside knowledge or web research mentions them.
- Treat the board as the primary source. Treat manual exclusions from the skills package `README.md`, the user, or the daily prompt as hard constraints.

## Metric discovery

- Inspect the actual keys in each player record on every run.
- Read nested `prior_stats` and `prior_world_cup_record` when present.
- Use only metrics that are explicitly present in the board data.
- Do not assume a metric exists just because it exists on another player.
- If a metric is missing, omit that scoring term entirely.
- Do not estimate missing metrics or fill them in from memory.

## Manual availability overrides

Before scoring players, check whether the skills package `README.md`, the user, or the daily prompt provides explicit player exclusions.

Treat the following words as explicit exclusion signals when attached to a player:

- injured
- suspended
- unavailable
- doubtful
- out
- do-not-pick
- excluded
- avoid

If an explicit exclusion is provided, remove that player from the candidate pool before scoring, even if the player is otherwise eligible in `players.json`.

Do not infer injuries, suspensions, or availability from memory or outside knowledge. Only apply an exclusion when it is explicitly stated in the skills package `README.md`, the current conversation, or the daily prompt.

Match manual exclusions by `player_id` first.

When a manual exclusion provides `player_id <id>`, find the matching record in `game-board/players.json` by exact `player_id` match and remove that player from the candidate pool before scoring. The player name is optional and must not be required.

If only a player name is provided, match it to `display_name` in `players.json` only when the match is clear and unambiguous.

If a player name could refer to multiple records, do not exclude anyone by guesswork. Prefer `player_id` for manual exclusions.

Never select a manually excluded player.

## Projection model

Use a deterministic board-only scoring model. The board is the only source of truth. Do not use reputation, memory, public knowledge, team popularity, or inferred player quality.

For every eligible non-excluded player, compute `player_score` from fields explicitly present in that player record. Read `prior_stats` and `prior_world_cup_record` when present. If a field is absent, omit that scoring term entirely. Do not estimate missing fields.

Use position-specific scoring:

- GK: `starts*5 + appearances*1 + minutes*0.02 + saves*2.0 - yellow_cards*1 - red_cards*6`
- DEF: `starts*4 + appearances*1 + minutes*0.02 + goals*8 + assists*5 - yellow_cards*1 - red_cards*6`
- MID: `starts*4 + appearances*1 + minutes*0.02 + goals*7 + assists*6 - yellow_cards*1 - red_cards*6`
- FWD: `starts*4 + appearances*1 + minutes*0.02 + goals*9 + assists*7 - yellow_cards*1 - red_cards*6`

Only apply a term when the corresponding field exists for that player.

Apply small age adjustment only when `age` exists:

- age 22-31: `+1`
- age 32-34: `0`
- age 35 or older: `-1`
- under age 22: `-0.5`

If `prior_stats` is absent, apply an uncertainty penalty:

- GK: `-3`
- DEF/MID/FWD: `-2`

This penalty represents lower confidence, not proof that the player is weak.

## Selection method

1. Filter the candidate pool to players whose `eligible_matchday_ids` include the current `matchday_id`.
2. Apply manual availability overrides and remove any explicitly excluded players.
3. Compute `player_score` for every remaining eligible player using the board-only scoring model.
4. Enumerate all legal formations:
   - 3-4-3
   - 3-5-2
   - 4-3-3
   - 4-4-2
   - 4-5-1
   - 5-3-2
   - 5-4-1
5. For each formation, select:
   - the highest-scoring GK
   - the required number of highest-scoring DEF
   - the required number of highest-scoring MID
   - the required number of highest-scoring FWD
6. Sum the 11 player scores for each formation.
7. Choose the formation with the highest total projected score.
8. If two formations are within 3 projected points, prefer the formation with more high-upside attackers, defined only by explicit goals and assists in the board data.
9. Select exactly 11 unique `player_id` values:
   - exactly 1 GK
   - 3 to 5 DEF
   - 3 to 5 MID
   - 1 to 3 FWD
10. Validate the selected XI before finalizing.

## Tie-breakers

When projections are close, prefer:

1. player with more starts
2. player with more minutes
3. player with more explicit goals and assists
4. goalkeeper with more explicit saves
5. player with lower card risk
6. player with fewer missing scoring fields
7. player from a position that improves the highest-scoring legal formation total

Use public web research only as a final tie-breaker when the run context allows it. Never use web research to override the board's player pool, player IDs, positions, matchday eligibility, or manual exclusions.

## Validation

Before finalizing, verify:

- every selected `player_id` exists in `players.json`
- every selected player is eligible for the current `matchday_id`
- no selected `player_id` appears in the manual exclusion list
- no duplicate `player_id` values appear
- exactly 11 players are selected
- exactly 1 goalkeeper is selected
- 3 to 5 defenders are selected
- 3 to 5 midfielders are selected
- 1 to 3 forwards are selected
- the formation is legal under `rules/fantasy-xi.md`
- no invented details were used
- output matches `output-format/daily-submission.schema.json`

## Output discipline

Return only the daily submission JSON required by the tournament schema.
