---
name: pick-fantasy-xi
description: choose a legal fifa world cup fantasy xi from the daily game board by using only player_id values from game-board/players.json, dynamically reading whatever player metrics are actually present on the board, applying explicit manual exclusions such as injured or suspended players when provided in the skills package README, using organizer-approved research domains from team/README.md only as capped supporting evidence, and returning a valid lineup under the tournament answer contract.
---


# Pick Fantasy XI


## Goal


Choose the highest-projection legal XI from the current matchday board.


The `player_score` is a relative confidence score, not a predicted fantasy-point total. Use it only to rank players within the current matchday dataset.


## Source of truth


- Read the skills package `README.md` for manual exclusions and strategy notes.
- Read `team/README.md` for organizer-provided research constraints, including approved network domains, when present.
- Read `game-board/matchday.json`, `game-board/matches.json`, `game-board/players.json`, `game-board/teams.json`, `rules/fantasy-xi.md`, and `output-format/daily-submission.schema.json` first when present.
- Build the candidate pool only from player records that exist in `game-board/players.json`.
- Only use players whose `eligible_matchday_ids` include the current `matchday_id`.
- Never invent or infer a `player_id`, team, position, eligibility flag, stat, minute total, injury status, suspension status, or availability flag that is not explicitly present in the board, skills package `README.md`, daily prompt, current conversation, or approved research.
- Ignore any player not present in `players.json`, even if outside knowledge or web research mentions them.
- Treat the board as the primary source for player pool, player IDs, positions, matchday eligibility, and explicit stats.
- Treat manual exclusions from the skills package `README.md`, the user, or the daily prompt as hard constraints.
- Treat `team/README.md` as the source for organizer research rules, not as a place for user-controlled exclusions.


## Metric discovery


- Inspect the actual keys in each player record on every run.
- Read nested `prior_stats` and `prior_world_cup_record` when present.
- Use only metrics that are explicitly present in the board data.
- Do not assume a metric exists just because it exists on another player.
- If a metric is missing, omit that scoring term entirely.
- Do not estimate missing metrics or fill them in from memory.

## FIFA Ranking Table (Official FIFA Ranking - 11 June 2026)

Use these rankings as a fixed strength signal.

Lower rank number = stronger team.

Argentina=1
Spain=2
France=3
England=4
Portugal=5
Brazil=6
Morocco=7
Netherlands=8
Belgium=9
Germany=10
Croatia=11
Mexico=13
Colombia=14
USA=15
Senegal=16
Uruguay=17
Japan=18
Switzerland=19
Iran=20
South Korea=22
Türkiye=23
Ecuador=24
Austria=25
Australia=27
Algeria=28
Egypt=29
Norway=30
Canada=31
Ivory Coast=33
Panama=34
Scotland=40
Paraguay=42
Czechia=43
Tunisia=45
DR Congo=46
Uzbekistan=50
Qatar=56
Iraq=57
Saudi Arabia=60
South Africa=61
Bosnia and Herzegovina=63
Jordan=64
Cape Verde=67
Ghana=73
Curacao=82
Haiti=83
New Zealand=85

## Match team mapping and rank-based stack rule

For each match:

1. Read `home_team_id` and `away_team_id` from `game-board/matches.json`.
2. Map each `team_id` to its team name using `game-board/teams.json`.
3. Map each team name to its hardcoded FIFA rank.
4. Treat the lower rank number as the stronger team.
5. Compute `rank_gap = abs(rank_home - rank_away)`.

## Rank-based player quota for the two match teams

Use the two teams in the selected match as the main stack pool.

Strong team quota:
- rank_gap <= 10  -> choose 1-2 players from both teams
- rank_gap 11-25 -> choose 3-4 players from the stronger team
- rank_gap > 25   -> choose 4-5 players from the stronger team

Weak team quota:
- choose at least 1 player from the weaker team when legal
- never choose more than 4 players from the weaker team unless needed to satisfy position legality

Keep the flexibility to accomodate more or less players from a team to satisfy position legality.


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


Do not infer injuries, suspensions, or availability from memory or outside knowledge. Only apply an exclusion when it is explicitly stated in the skills package `README.md`, the current conversation, the daily prompt, or approved research.


Match manual exclusions by `player_id` first.


When a manual exclusion provides `player_id <id>`, find the matching record in `game-board/players.json` by exact `player_id` match and remove that player from the candidate pool before scoring. The player name is optional and must not be required.


If only a player name is provided, match it to `display_name` in `players.json` only when the match is clear and unambiguous.


If a player name could refer to multiple records, do not exclude anyone by guesswork. Prefer `player_id` for manual exclusions.


Never select a manually excluded player.


## Organizer-approved research


Use public web research only when the run context allows it.


When research is available, read `team/README.md` and use only the organizer-approved network domains listed there.


Do not hard-code or invent additional approved domains. Do not use unapproved domains.


Research may be used only for eligible, non-excluded players already present in `game-board/players.json`.


Research may support:


- recent club or national-team form
- likely starting role
- current injury, suspension, or unavailability
- national-team strength or FIFA ranking context
- explicit player role such as penalty taker, set-piece taker, captain, or key attacker
- current relevance of players with missing or limited `prior_stats`


Research must never:


- add a new player
- change a `player_id`
- change a player position
- override matchday eligibility
- override manual exclusions
- invent stats or minutes
- use reputation without source evidence
- use unapproved domains


## Projection model


Use a deterministic board-first scoring model.


For every eligible non-excluded player, compute `board_player_score` from fields explicitly present in that player record. Read `prior_stats` and `prior_world_cup_record` when present. If a field is absent, omit that scoring term entirely. Do not estimate missing fields.


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


If organizer-approved research is available, compute a capped `research_bonus` between `-4` and `+4` for eligible, non-excluded players already present in `players.json`.


Use only approved domains listed in `team/README.md`.


Research bonus scale:


- `+4`: strong explicit evidence of current high form, likely starting role, or major attacking/goalkeeping importance
- `+3`: clear evidence of regular current role or strong recent club/national contribution
- `+2`: moderate evidence of current relevance, minutes, or positive form
- `+1`: weak but positive evidence from an approved source
- `0`: no approved evidence found, research unavailable, or evidence unclear
- `-4`: explicit injury, suspension, unavailability, or strong playing-time concern from an approved source


If approved research explicitly says a player is injured, suspended, unavailable, or out, remove that player from the candidate pool before scoring.


When approved research provides current FIFA ranking or clear national-team strength context, apply only a small team-context adjustment within the `research_bonus`:


- `+1`: player belongs to a clearly stronger nation than the opponent
- `0`: match appears balanced or evidence is unclear
- `-1`: player belongs to a clearly weaker nation than the opponent


For goalkeepers and defenders, use team context as a small clean-sheet or defensive-reliability proxy only when no explicit clean-sheet metric exists in the board.


For forwards and midfielders, use team context only as a small chance-creation or attacking-upside proxy.


Do not let research bonuses dominate the board score for players with strong explicit board data.


Final score:


`final_player_score = board_player_score + research_bonus`


If research is unavailable, use:


`final_player_score = board_player_score`


## Selection method


1. Filter the candidate pool to players whose `eligible_matchday_ids` include the current `matchday_id`.
2. Apply manual availability overrides and remove any explicitly excluded players.
3. Use approved research, when available, to remove any player explicitly reported as injured, suspended, unavailable, or out.
4. Compute `board_player_score` for every remaining eligible player using the board-first scoring model.
5. If approved research is available, compute a capped `research_bonus` for eligible, non-excluded players only.
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
6. player from a stronger national team context, only when supported by approved research
7. player with lower card risk
8. player with fewer missing scoring fields
9. player from a position that improves the highest-scoring legal formation total


Use approved web research only as supporting evidence. Never use web research to override the board's player pool, player IDs, positions, matchday eligibility, or manual exclusions.


## Validation


Before finalizing, verify:


- every selected `player_id` exists in `players.json`
- every selected player is eligible for the current `matchday_id`
- no selected `player_id` appears in the manual exclusion list
- no selected player is explicitly injured, suspended, unavailable, or out according to approved research
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