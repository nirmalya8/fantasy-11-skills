---
name: bracket-result-composer
description: compose compliant knockout bracket results from bracket and standings inputs for bracket-only runs. use when given game-board bracket files, team files, and standings files to produce one winner for every required match, keep every later-round pick reachable from earlier winners, and set champion_team_id to the final winner.
---

# bracket result composer

## purpose

Produce a knockout bracket result that is internally consistent from the opening knockout round through the champion. Use only the supplied bracket and standings inputs. Do not invent a path that cannot be reached from earlier winners.

## files to read first

Read the available bracket and standings files before choosing winners:

- `game-board/bracket.json`
- `game-board/matches.json`
- `game-board/teams.json`
- `game-board/world-cup-standings.json`
- `game-board/bracket-context.json`
- `current-standings/leaderboard.json` when present

Treat `standings-before.json` and `current-standings/leaderboard.json` as fantasy leaderboard context only if they are provided. Do not use them as World Cup group tables.

## scope

Use this skill only for bracket-only runs.

Do not use projection-first reasoning. Do not prioritize forecast markers, provisional labels, or invented bracket narratives. Use the actual football signals in the provided data.

## ranking signals

Use only these football signals for winner selection:

- `form`
- `goal_difference`
- `goals_for`
- `goals_against`

Ignore projection fields such as:

- `qualification_projection`
- `projection_status`
- `provisional`
- `official` labels tied to predicted advancement
- any derived future-round placeholder state

## scoring model

Convert each team into a deterministic strength score before choosing winners.

### 1) normalize inputs

Use the candidate teams available for the current match.

If `form` is a string such as `WLD`, convert it into recent-form points:

- `W` = win = 3
- `D` = draw = 1
- `L` = loss = 0

Then divide by the maximum possible points in that sample to get a 0 to 1 form score.

Normalize the other fields within the current candidate pool:

- higher `goal_difference` = better
- higher `goals_for` = better
- lower `goals_against` = better

### 2) weighted score

Compute:

`team_score = (0.40 * form_score) + (0.30 * gd_score) + (0.20 * gf_score) + (0.10 * ga_score)`

where `ga_score` is inverted so fewer goals conceded produces a higher score.

### 3) tie-break order

If scores are tied, break ties in this order:

1. higher `goal_difference`
2. higher `goals_for`
3. fewer `goals_against`
4. stable identity tie-breaker: `team_id`
5. then `display_name`

Do not use randomness.

## bracket selection workflow

1. Identify all required matches in round order.
2. For each match, select exactly one winner.
3. Choose winners only from teams eligible in that match.
4. Carry each winner forward into the next round.
5. For every later round, verify that both entrants are reachable from earlier winners.
6. Never introduce a team into a later round unless it already appeared as a winner on the path to that round.
7. Set `champion_team_id` to the winner of the final match.

## compliance guardrails

Before finalizing the bracket, verify all of the following:

- Every required match has exactly one winner.
- Every semifinalist was already a quarterfinal winner.
- Every finalist was already a semifinal winner.
- The champion was already a finalist and is the final winner.
- No match references a team that was not present in the previous round path.
- No later-round pick contradicts an earlier declared winner.
- No team is used in a round unless that team is reachable from the winners already selected.

If a mismatch appears, backtrack and repair the earlier pick first, then re-run the downstream picks so the full chain remains consistent.

## round sensitivity

Use the same scoring model for all rounds, but apply it carefully:

- In early knockout rounds, prefer the stronger direct statistical profile.
- In later rounds, preserve the same winner chain and do not replace an earlier winner with a new team that was never reached.
- When two reachable teams are close, choose the one with the better score rather than forcing an upset.

The bracket must close using teams already declared as winners in earlier rounds.

## outputs

Return one winner for every required match. Later-round picks must be reachable from earlier winners, and champion_team_id must match the final winner.

Requirements:

- Pick exactly one winner for every required match.
- Set `champion_team_id` to the final match winner.
- For any match with `source_match_ids`, the picked winner must be one of the winners already selected for those source matches.
- Build the bracket from early rounds to later rounds.
- Advance winners forward at each step.
- Do not choose a later-round team that was eliminated in your own earlier picks.

The final result must be internally consistent from the opening knockout round through the champion.

## output structure

Return a clean bracket result structure that includes:

- `champion_team_id`
- a complete per-match winner list covering every required match
- enough match identifiers or round labels to show how each winner advances

Do not omit any required match. Do not invent a winner chain after the fact. The final structure must be traceable from the opening knockout round through the champion without gaps.

## decision style

Prefer the most defensible team on the available football evidence. When teams are close, use the scoring model above rather than projection factors.

Keep the result deterministic, compliant, and self-consistent from round to round.