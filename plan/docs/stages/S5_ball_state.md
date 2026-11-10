# S5 — Ball state and team possession per frame

## Goal
Assign every frame one ball state (`controlled`, `in_flight`, `loose`, `dead`) and the team credited with possession.

## Input
`control.parquet`, `segments.parquet`, `frames.parquet`, `tracking.npz`.

## Output
`data/interim/ball_state.parquet`: `frame, ball_state, poss_team, dead_reason` (out / stoppage / camera_cut / pre_match / half_time).

## Steps
1. Dead-ball detector (vectorised, rolling windows). Signals per frame:
   - `out`: ball outside pitch ± `dead.out_margin_m`
   - `mass_decel`: share of tracked players below `dead.walk_kmh` > `dead.mass_share` for > 1 s
   - `stationary`: ball speed < 2 km/h for > `dead.stationary_s`
   - `cut`: `camera_cut` from S2
   - dead if ≥ `dead.min_signals` signals hold (window-smoothed). Outside `period` 1/2 → dead.
2. Live frames: `controlled` from S3; inside a pass/long_pass segment → `in_flight`; inside a loose segment → `loose`.
3. Possession:
   - controlled → `ctrl_team`
   - in_flight → `release_team`
   - loose → per `possession.loose_credit`
   - dead → −1
4. Restart team after dead ball: team of the first controller after the dead stretch; record restart type (throw-in / corner / goal kick / kick-off / free kick) from the restart location (rules in `03_definitions.md`).
5. Optional: HMM smoothing (`hmmlearn`) of state sequence to remove 1–2-frame flickers.

## Config keys
`dead.out_margin_m: 0.75`, `dead.walk_kmh: 7`, `dead.mass_share: 0.8`, `dead.stationary_s: 3`, `dead.min_signals: 2`, `possession.loose_credit: last_team`.

## Tests
- Synthetic sequences for each state; tracking gap alone (no other signal) must stay live.

## Acceptance
- Ball-in-play time in a plausible range (~50–60 min of 90+ min match).
- Possession % per team reported both with loose → none and loose → last_team.
- 20 dead stretches and 20 random live frames checked on video.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S5 (or log it as a known limitation in the report).
