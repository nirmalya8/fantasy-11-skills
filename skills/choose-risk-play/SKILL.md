---
name: choose-risk-play
description: choose an optional fifa world cup risk play from the daily claim catalog using the current match board and standings, respecting match/team/player id constraints, using only ids from the board, and returning null when no claim is clearly strong; use when the daily prompt asks for a risk play decision.
---

# Choose Risk Play

## Goal
Select the best optional Risk Play or return `null`.

## Hard source rules
- Use only claim IDs present in `game-board/claim-catalog.json`.
- Use only match_id, team_id, and player_id values from the current game board.
- Never invent a player_id, team_id, or match_id.
- Keep any team or player tied to the same match row in `game-board/matches.json`.
- Use public web data only as a secondary signal when it improves confidence and does not conflict with the board.

## Selection rules
- Prefer Green claims over Yellow claims over Red claims.
- Prefer lower-variance claims over exact-score or high-volatility claims.
- Return `null` when no claim is clearly stronger than skipping.
- Choose a player-based claim only when the player is a strong starter with a realistic path to scoring.
- Choose a team-based claim only when the match context clearly supports it.

## Validation
Before finalizing, verify:
- the claim exists in the catalog
- every required field is present and valid
- all ids come from the current board
- the claim and match relationship is consistent

## Output
Return a valid claim object or `null` as the `risk_play` field in the final daily JSON answer.
Do not include stake, stake percent, or bet points.