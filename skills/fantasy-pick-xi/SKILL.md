---
name: pick-fantasy-xi
description: choose a legal fifa world cup fantasy xi from the daily game board by using only player_id values from game-board/players.json, dynamically reading whatever player metrics are actually present on the board, using organizer-approved research domains from team/README.md to detect current injuries, suspensions, unavailability, recent form, likely roles, and national-team context, and returning a valid lineup under the tournament answer contract.
---

# Pick Fantasy XI

## Goal

Choose the highest-projection legal XI from the current matchday board.

The `player_score` is a relative confidence score, not a predicted fantasy-point total. Use it only to rank players within the current matchday dataset.

## Source of truth

- Read the skills package `README.md` for package strategy notes.
- Read `team/README.md` for organizer-provided research constraints, including approved network domains, when present.
- Read `game-board/matchday.json`, `game-board/matches.json`, `game-board/players.json`, `game-board/teams.json`, `rules/fantasy-xi.md`, and `output-format/daily-submission.schema.json` first when present.
- Build the candidate pool only from player records that exist in `game-board/players.json`.
- Only use players whose `eligible_matchday_ids` include the current `matchday_id`.
- Never invent or infer a `player_id`, team, position, eligibility flag, stat, minute total, injury status, suspension status, or availability flag that is not explicitly present in the board, current conversation, daily prompt, or organizer-approved research.
- Ignore any player not present in `players.json`, even if outside knowledge or web research mentions them.
- Treat the board as the primary source for player pool, player IDs, positions, matchday eligibility, and explicit stats.
- Treat `team/README.md` as the source for organizer research rules and approved domains.

## Metric discovery

- Inspect the actual keys in each player record on every run.
- Read nested `prior_stats` and `prior_world_cup_record` when present.
- Use only metrics that are explicitly present in the board data.
- Do not assume a metric exists just because it exists on another player.
- If a metric is missing, omit that scoring term entirely.
- Do not estimate missing metrics or fill them in from memory.

## Organizer-approved research

Use public web research only when the run context allows it.

When research is available, read `team/README.md` and use only the organizer-approved network domains listed there.

Do not hard-code or invent additional approved domains. Do not use unapproved domains.

Research may be used only for eligible players already present in `game-board/players.json`.

Research may support:

- current injury, suspension, unavailability, squad status, or fitness concerns
- recent club or national-team form
- likely starting role
- national-team strength or FIFA ranking context
- explicit player role such as penalty taker, set-piece taker, captain, or key attacker
- current relevance of players with missing or limited `prior_stats`

Research must never:

- add a new player
- change a `player_id`
- change a player position
- override matchday eligibility
- invent stats or minutes
- use reputation without source evidence
- use unapproved domains

## Research-based availability check

When organizer-approved research is available, perform an availability check before final scoring.

Use only approved domains listed in `team/README.md`.

Research eligible players from `game-board/players.json`, prioritizing players who are likely to be selected by the board score and players with missing or limited `prior_stats`.

For each researched player, search using the player name, national team, matchday opponent, and current-availability terms such as:

- injury
- injured
- ruled out
- unavailable
- suspended
- doubtful
- withdrawn
- not in squad
- fitness
- recovering

If an approved source explicitly reports that a player is injured, suspended, unavailable, ruled out, withdrawn, or not in the squad for the matchday, remove that player from the candidate pool before scoring.

If an approved source reports that a player is doubtful, recovering, limited, or has a fitness concern but does not clearly rule them out, keep the player eligible but apply a `-4` research bonus unless stronger approved evidence confirms availability.

If approved sources conflict, prefer the most recent source from the organizer-approved domains. If conflict remains unresolved, do not exclude the player; apply a negative research bonus instead.

If research is unavailable or no approved evidence is found, do not infer injury or availability. Continue with the board-first scoring model.

## Projection model

Use a deterministic board-first scoring model.

For every eligible player not removed by the research-based availability check, compute `board_player_score` from fields explicitly present in that player record. Read `prior_stats` and `prior_world_cup_record` when present. If a field is absent, omit that scoring term entirely. Do not estimate missing fields.

Use position-specific board scoring:

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

## Research bonus

If organizer-approved research is available, compute a capped `research_bonus` between `-4` and `+4` for eligible players already present in `players.json`.

Use only approved domains listed in `team/README.md`.

Research bonus scale:

- `+4`: strong explicit evidence of current high form, likely starting role, or major attacking/goalkeeping importance
- `+3`: clear evidence of regular current role or strong recent club/national contribution
- `+2`: moderate evidence of current relevance, minutes, or positive form
- `+1`: weak but positive evidence from an approved source
- `0`: no approved evidence found, research unavailable, or evidence unclear
- `-4`: explicit doubtful status, fitness concern, strong playing-time concern, or unresolved conflicting availability evidence

Do not assign a research bonus from unapproved domains.

Do not let research bonuses dominate the board score for players with strong explicit board data.

When approved research provides current FIFA ranking or clear national-team strength context, apply only a small team-context adjustment within the `research_bonus`:

- `+1`: player belongs to a clearly stronger nation than the opponent
- `0`: match appears balanced or evidence is unclear
- `-1`: player belongs to a clearly weaker nation than the opponent

For goalkeepers and defenders, use team context as a small clean-sheet or defensive-reliability proxy only when no explicit clean-sheet metric exists in the board.

For forwards and midfielders, use team context only as a small chance-creation or attacking-upside proxy.

Final score:

`final_player_score = board_player_score + research_bonus`

If research is unavailable, use:

`final_player_score = board_player_score`

## Selection method

1. Filter the candidate pool to players whose `eligible_matchday_ids` include the current `matchday_id`.
2. If organizer-approved research is available, perform a research-based availability check for eligible players from the board.
3. Remove any player clearly reported by approved research as injured, suspended, unavailable, ruled out, withdrawn, or not in the squad.
4. Compute `board_player_score` for every remaining eligible player using the board-first scoring model.
5. If approved research is available, compute a capped `research_bonus` for eligible players only.
6. Compute `final_player_score = board_player_score + research_bonus`.
7. Enumerate all legal formations:
   - 3-4-3
   - 3-5-2
   - 4-3-3
   - 4-4-2
   - 4-5-1
   - 5-3-2
   - 5-4-1
8. For each formation, select:
   - the highest-scoring GK
   - the required number of highest-scoring DEF
   - the required number of highest-scoring MID
   - the required number of highest-scoring FWD
9. Sum the 11 final player scores for each formation.
10. Choose the formation with the highest total projected score.
11. If two formations are within 3 projected points, prefer the formation with more high-upside attackers, defined by explicit board goals/assists plus approved-source evidence of current attacking role.
12. Select exactly 11 unique `player_id` values:
   - exactly 1 GK
   - 3 to 5 DEF
   - 3 to 5 MID
   - 1 to 3 FWD
13. Validate the selected XI before finalizing.

## Tie-breakers

When projections are close, prefer:

1. player with more starts
2. player with more minutes
3. player with more explicit goals and assists
4. goalkeeper with more explicit saves
5. player with stronger approved-source evidence of current role or form
6. player with confirmed availability from approved research
7. player from a stronger national-team context, only when supported by approved research
8. player with lower card risk
9. player with fewer missing scoring fields
10. player from a position that improves the highest-scoring legal formation total

Use approved web research only as supporting evidence. Never use web research to override the board's player pool, player IDs, positions, or matchday eligibility.

## Validation

Before finalizing, verify:

- every selected `player_id` exists in `players.json`
- every selected player is eligible for the current `matchday_id`
- no selected player is explicitly injured, suspended, unavailable, ruled out, withdrawn, or not in the squad according to approved research
- no duplicate `player_id` values appear
- exactly 11 players are selected
- exactly 1 goalkeeper is selected
- 3 to 5 defenders are selected
- 3 to 5 midfielders are selected
- 1 to 3 forwards are selected
- the formation is legal under `rules/fantasy-xi.md`
- no invented details were used
- every Fantasy XI value is a `player_id`, not a score, name, position, or ranking value
- no decimal number appears as a Fantasy XI `player_id`
- output matches `output-format/daily-submission.schema.json`

## Output discipline

Return only the daily submission JSON required by the tournament schema.

Return only valid `player_id` values from `game-board/players.json` in the Fantasy XI output.

Never output player names, player scores, projected totals, positions, explanations, or decimal values inside the Fantasy XI array.

Before returning JSON, verify that every Fantasy XI entry is an exact `player_id` from `players.json`.