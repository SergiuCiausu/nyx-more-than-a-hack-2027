# S5 — Ball state, possession and final labels

## Goal
Assign every frame one internal ball state (`controlled`, `in_flight`, `loose`, `dead`) and the team in possession, then produce the final `H`/`A`/`D`/`X` sequence and write the submission CSV.

## Input
`control.parquet`, `segments.parquet`, `frames.parquet`, `tracking.npz`, `tags.parquet`.

## Output
- `data/interim/ball_state.parquet`: `frame, ball_state, poss_team, dead_reason, label`
- The submission CSV at `paths.submission`

## Steps
1. Dead-ball evidence (vectorised, rolling windows). Signals per frame:
   - `out`: ball outside pitch ± `dead.out_margin_m`
   - `mass_decel`: share of tracked players below `dead.walk_kmh` > `dead.mass_share` for > 1 s
   - `stationary`: ball speed < 2 km/h for > `dead.stationary_s`
   - `restart`: ball and player positions match a restart shape
   - Dead needs at least `dead.min_signals` signals. A missing ball is never enough on its own.
2. Add Smart Tagging as weak evidence only: set pieces suggest restarts, and team tags suggest possession. Handle `N/A` teams explicitly.
3. Live frames: `controlled` from S3; inside a pass/long_pass segment → `in_flight`; inside a loose segment → `loose`.
4. Possession:
   - controlled → `ctrl_team`
   - in_flight → `release_team`
   - loose or long ball gap → stays with the last team
   - dead → −1
5. Choose the best `H`/`A`/`D` sequence with Viterbi, using `model.switch_penalty`, `model.dead_penalty` and `model.smart_tag_weight`. The pipeline runs offline, so it can use lookahead.
6. Place each hand-over at the frame the opponent gains control, not at the kick. Touches, deflections and aerial duels without subsequent control do not change possession.
7. Restart team after a dead ball: team of the first controller after the dead stretch.
8. Frames from the start of the second-half block until the kickoff detected in S2 are `D`.
9. Write `frame,label` with 199,319 rows. Use only `H`, `A` or `D` on graded frames, and `X` on ungraded frames.

## Config keys
`dead.out_margin_m: 0.75`, `dead.walk_kmh: 7`, `dead.mass_share: 0.8`, `dead.stationary_s: 3`, `dead.min_signals: 2`, `model.switch_penalty`, `model.dead_penalty`, `model.smart_tag_weight`.

## Tests
- Synthetic sequences for each state; a tracking gap alone (no other signal) must stay live.
- A deflection or touch without control does not flip possession.
- An interception flips possession at the control frame.
- Loose ball keeps the last team.
- Graded frames never contain `X`, and ungraded frames are all `X`.

## Acceptance
- `evaluation/validate_submission.py` passes on the written file.
- Possession counts and durations are plausible, with no tiny fragments.
- Ball-in-play time is in a plausible range for a professional match.
- Video may be used to debug, but what you see must not become labels, corrections or frame-specific rules.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S5 (or log it as a known limitation in the report).
