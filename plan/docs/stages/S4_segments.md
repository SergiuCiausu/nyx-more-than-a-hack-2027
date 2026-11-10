# S4 — Uncontrolled segments: pass, long pass, loose ball

## Goal
Cut every stretch between two control episodes into segments and classify each as `pass`, `long_pass` or `loose`, with an outcome (`complete`, `intercepted`, `out`, `stopped`). Rules in `03_definitions.md`.

## Input
`control.parquet`, `tracking.npz`, `frames.parquet`.

## Output
`data/interim/segments.parquet` with the `seg_dtype` columns from `04_data_model.md`.

## Steps
1. Walk the control sequence: each run of `controlled == False` is a candidate segment with `release_track` (last controller before) and `receive_track` (next controller after, −1 if none).
2. Split a segment at:
   - the frame the ball leaves the pitch (`out`),
   - a dead-ball signal (S5 detector may be called here, or segments re-cut in S5),
   - a sharp direction change near a player (deflection) → first part pass, rest loose.
3. Features per segment: `dist_m`, `path_length`, `straightness`, `kick_kmh` (speed jump within first 3 frames vs speed before release), `opp_dist_rel`, max/mean ball speed, number of distinct nearest players during segment, `aerial_flag` (position jump > `seg.jump_m` or speed spike inside a smooth fast run).
4. Classify with the rule function:
```python
if not contested and kick > MIN_KICK and straightness > MIN_STRAIGHT:
    kind = "long_pass" if dist > LONG_PASS_M else "pass"
    outcome = "complete" if receive_team == release_team else "intercepted"
else:
    kind = "loose"
```
   then overwrite outcome with `out` / `stopped` when the segment ended that way.
5. Clearance flag: kick under pressure, no teammate within `seg.clearance_teammate_m` of landing.
6. (Later) optional scikit-learn classifier trained on labelled segments from S7, same features.

## Config keys
`seg.min_kick_kmh: 15`, `seg.pressure_dist_m: 1.5`, `seg.min_straightness: 0.85`, `seg.long_pass_m: 25`, `seg.jump_m: 3`, `seg.clearance_teammate_m: 10`.

## Tests
- Synthetic straight 40 m fast ball, teammate receives → long_pass/complete.
- Same, opponent receives → long_pass/intercepted.
- Ball leaves y < 0 → outcome out.
- Low-speed zig-zag between two opponents → loose.

## Acceptance
- Report: counts per kind/outcome per team (a typical match: ~800–1000 passes total, ~10–15% long); 30 random segments of each kind checked on video, precision ≥ 85% for passes before moving on.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S4 (or log it as a known limitation in the report).
