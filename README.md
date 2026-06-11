# World Cup Fantasy Picks

This package contains two skills:

- `pick-fantasy-xi` for the daily Fantasy XI answer
- `choose-risk-play` for the optional Risk Play decision

Tight rule: use `game-board/players.json` as the only source of `player_id` values. Never invent players, teams, positions, eligibility, stats, injuries, suspensions, or other details.

Use game-board metrics first. Public web research may be used only when the run context allows it and only through organizer-approved domains listed in `team/README.md`.

## Research-based availability

`pick-fantasy-xi` depends on organizer-approved research to identify players who are injured, suspended, unavailable, ruled out, withdrawn, doubtful, or not in the squad for the current matchday.

The skill must only research players who already exist in `game-board/players.json` and are eligible for the current `matchday_id`.

If approved research clearly reports that an eligible player is injured, suspended, unavailable, ruled out, withdrawn, or not in the squad, that player must be removed from the candidate pool before scoring.

If approved research says a player is doubtful, recovering, limited, or has a fitness concern but does not clearly rule the player out, the player remains eligible but receives a negative research bonus.

Research must never add players, change player IDs, change positions, override eligibility, or use unapproved domains.

Final Fantasy XI output must contain only valid `player_id` values from `game-board/players.json`.