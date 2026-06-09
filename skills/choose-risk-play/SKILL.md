---
name: choose-risk-play
description: choose an optional fifa world cup risk play from the current claim catalog by using only ids from the game board and match relationships that are consistent with the board; use when the daily prompt asks for a risk play decision.
---

# Choose Risk Play

## Goal
Select the best optional Risk Play or return `null`.

## Source of truth
- Read `game-board/claim-catalog.json`, `game-board/matches.json`, `game-board/players.json`, `game-board/teams.json`, and `game-board/standings-before.json`.
- Use only `claim_id` values present in the catalog.
- Use only `match_id`, `team_id`, and `player_id` values that exist in the current game board.
- Never invent a match, team, player, or claim.
- For team- or player-based claims, keep the referenced team or player tied to the same match row in `matches.json`.

## Selection rules
- Prefer Green claims over Yellow claims over Red claims.
- Prefer low-variance claims over exact-score or high-volatility claims.
- Return `null` when no claim is clearly stronger than skipping.
- Choose a player-based claim only when the player is a strong starter with a realistic path to scoring.
- Choose a team-based claim only when the match context clearly supports it.

## Optional web use
If public web research is available in the run context, use it only to confirm or refine a claim that is already supported by the board. Do not rely on web data to invent ids or override the claim catalog.

## Validation
Before finalizing, verify:
- the claim exists in the catalog
- every required field is present and valid
- all ids come from the current board
- the claim and match relationship is consistent

## Output
Return a valid claim object or `null` as the `risk_play` field in the final daily JSON answer.
Do not include stake, stake percent, or bet points.