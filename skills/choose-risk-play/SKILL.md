---
name: choose-risk-play
description: choose an optional fifa world cup risk play from the current claim catalog by dynamically reading claim_id, risk_type, and required_fields from game-board/claim-catalog.json, validating match/team/player relationships from the board, applying matchday.json phase constraints, and using organizer-approved research from team/README.md to select the strongest evidence-adjusted claim rather than always playing safe; use when the daily prompt asks for a risk play decision.
---

# Choose Risk Play

## Goal

Select the best optional Risk Play or return `null`.

The goal is to maximize evidence-adjusted expected value, not to always choose the safest claim.

## Source of truth

- Read `game-board/matchday.json`, `game-board/claim-catalog.json`, `game-board/matches.json`, `game-board/players.json`, `game-board/teams.json`, `game-board/standings-before.json`, `rules/risk-play.md`, `output-format/daily-submission.schema.json`, and `team/README.md` first when present.
- Use only `claim_id` values present in `claim-catalog.json`.
- Use each claim's `required_fields` exactly as listed in `claim-catalog.json`.
- Use each claim's `risk_type` exactly as listed in `claim-catalog.json`.
- Use only `match_id`, `team_id`, and `player_id` values that exist in the current game board.
- Never invent a match, team, player, score, claim, `risk_type`, required field, or ID.
- For team-based claims, the selected `team_id` must be the home or away team for the submitted `match_id`.
- For player-based claims, the selected `player_id` must exist in `players.json` and belong to one of the teams in the submitted match.
- Do not combine a match from one row in `matches.json` with a team or player from another row.
- Treat `team/README.md` as the source for organizer-approved research domains when research is available.


## Matchday phase constraints


Read `game-board/matchday.json` before evaluating claims.


Use the `phase` field to determine which claim types are possible for the current matchday.


If `phase` is `league`, group stage, or any non-knockout phase:


- do not generate candidates for claims whose outcome requires extra time
- do not generate candidates for claims whose outcome requires penalties


Only generate extra-time or penalties candidates when `phase` is `knockout` or another explicitly knockout-style phase where extra time or penalties are possible.


If the phase is missing or unclear, treat extra-time and penalties claims as invalid rather than guessing.


## Catalog-driven candidate generation


Build candidate Risk Plays directly from `claim-catalog.json`.


For each claim in the catalog:


1. Read `claim_id`, `risk_type`, and `required_fields`.
2. Generate only claim objects whose required fields can be validly filled from the current board and whose `claim_id` is possible for the current `matchday.json` phase.
3. If a claim requires only `match_id`, create candidates for each valid match in `matches.json`.
4. If a claim requires `match_id` and `team_id`, create candidates only for the home and away teams in that match.
5. If a claim requires `match_id` and `player_id`, create candidates only for players from the home or away team in that match.
6. If a claim requires `home_score` and `away_score`, generate only non-negative integer score candidates that are reasonable under the evidence. Do not generate unrealistic scores without support.
7. Do not create a candidate if any required field cannot be filled from the current board.


Do not submit fields that are not required by the selected claim.


Use claim-specific strategy based on each claim's `required_fields`, `risk_type`, phase validity, and evidence. Do not rely on a hard-coded list of available claims.


## Organizer-approved research


Use public web research only when the run context allows it.


When research is available, read `team/README.md` and use only the organizer-approved network domains listed there.


Do not hard-code or invent additional approved domains. Do not use unapproved domains.


Use research only to evaluate candidate claims that are already valid under the board and claim catalog.


Research may support:


- historical match result or match report when the fixture maps to a known prior match
- recent team form
- current squad availability
- likely starting attackers
- player scoring form
- FIFA ranking or national-team strength
- team attacking strength
- defensive weakness
- card or red-card tendency
- match phase context


Research must never:


- add a new claim
- add a new match
- add a new team
- add a new player
- change any ID
- override required-field validation
- use unapproved domains


## Claim evaluation


Evaluate every valid candidate claim generated from `claim-catalog.json`.


For each candidate, assign:


- `risk_type`: read from `claim-catalog.json`
- `stake_risk`: 15 for Green, 25 for Yellow, 35 for Red
- `evidence_strength`: weak, moderate, strong, or very strong
- `volatility`: low, medium, high, or extreme
- `validity`: whether all required fields are valid and board-consistent


Choose the claim with the strongest evidence-adjusted expected value.


Do not automatically prefer Green over Yellow or Red.


A higher-stake Yellow or Red claim may be selected when board data and approved research provide strong enough support.


Return `null` only when no valid claim has enough support to justify the downside. Try to keep null returns to a minimum.


## Risk thresholds


Use these minimum standards:


- Green claims may be selected with moderate or stronger evidence.
- Yellow claims require strong evidence.
- Red claims require very strong and specific evidence.


Do not choose a Red claim only because it has high upside. Choose Red only when the evidence is unusually specific, such as:


- a known historical result from approved research
- a clearly dominant team mismatch
- a player with strong scoring evidence and likely starting role
- a match context that specifically supports the Red claim


## Dynamic claim strategy


Use `claim-catalog.json` as the only source for available `claim_id` values, `risk_type`, and `required_fields`.


Do not assume a claim exists unless it appears in `claim-catalog.json`.


Evaluate claims according to their required fields, risk type, phase validity, and evidence.


### Match-only claims


For claims whose `required_fields` contain only `match_id`, evaluate the match context directly.


Prefer match-only claims when the evidence is about the whole match, such as:


- expected total goals
- timing of goals
- card volume
- red-card likelihood
- extra-time or penalties phase context


Use match-only claims when research and board data support the match-level event more strongly than any individual team or player event.


### Team-based claims


For claims whose `required_fields` include `team_id`, evaluate only teams that are home or away in the selected `match_id`.


Prefer team-based claims when the evidence supports one team-specific outcome, such as:


- stronger team likely to score first
- clearly superior team likely to win heavily
- comeback outcome supported by very specific historical or match-context evidence


Do not select team-based claims from general team reputation alone.


### Player-based claims


For claims whose `required_fields` include `player_id`, evaluate only players who:


- exist in `players.json`
- belong to one of the teams in the selected match
- are not clearly injured, suspended, unavailable, ruled out, or absent from the squad according to approved research
- have board data or approved research supporting likely starting role or meaningful scoring involvement


Prefer player-based claims only when player-specific evidence is stronger than match-level or team-level alternatives.


Do not select player-based claims from name recognition alone.


### Scoreline claims

For claims whose `required_fields` include `home_score` and `away_score`, require very strong and specific evidence.

Only submit scoreline claims when approved research strongly supports that exact scoreline, such as a known historical result or unusually clear match evidence.

Do not submit scoreline claims when evidence only supports a general winner, goal total, or team advantage.

### Phase-limited claims

For claims whose `claim_id` or scoring meaning depends on extra time or penalties, use `game-board/matchday.json` phase constraints.

If the matchday phase is not knockout, treat extra-time and penalties claims as invalid.

## Expected-value preference

Use this preference order only after validity is confirmed:

1. Prefer the highest evidence-adjusted expected value.
2. Prefer Yellow over Green when both have strong evidence, because Yellow has higher upside.
3. Prefer Red over Yellow only when Red has very strong and specific evidence.
4. Prefer lower-volatility claims when evidence strength is similar.
5. Prefer match-level claims over player-specific claims unless the player evidence is strong.
6. Prefer `null` over weak or speculative claims.


## Required-field validation

For the selected claim, include all and only the fields required by that claim.

Validate:

- `claim_id` exists in `claim-catalog.json`
- selected claim's `risk_type` matches `claim-catalog.json`
- selected claim uses exactly the required fields listed in `claim-catalog.json`
- `match_id` exists in `matches.json`
- required `team_id`, if any, exists in `teams.json`
- required `team_id`, if any, is home or away for the selected `match_id`
- required `player_id`, if any, exists in `players.json`
- required `player_id`, if any, belongs to one of the teams in the selected `match_id`
- required score fields, if any, are non-negative integers
- selected claim is valid for the current `matchday.json` phase
- no invented IDs, scores, fields, or assumptions were used
## Output

Return a valid claim object or `null` as the `risk_play` field in the final daily JSON answer.

Do not include stake, stake percent, bet points, confidence score, rationale, citations, `risk_type`, or non-required fields in the final JSON.