# S3 — Ball control per frame

## Goal
For every frame, decide whether a player controls the ball, and which one.

## Input
`tracking.npz`, `frames.parquet`.

## Output
`data/interim/control.parquet`: `frame, nearest, dmin, ball_v, raw_ctrl, controlled, ctrl_track, ctrl_team`.

## Steps
1. `dist (T,N)` = ball→slot distance in metres; NaN → inf. Speeds must already be in km/h, converted in S1 after Q12.
2. `nearest = argmin`, `dmin`, nearest player's speed.
3. `ball_v` = ball speed smoothed with Savitzky–Golay (`control.smooth_window`, default 7 frames, order 2). Compare with the provided speed column; also compute speed from positions as a cross-check.
4. `raw_ctrl = (dmin < control.max_dist_m) & (ball_v < near_speed + control.max_rel_speed_kmh)`.
5. Hysteresis (sequential loop, numba optional): control switches to a new track only after `control.min_frames` consecutive frames; short drops (< `control.max_gap_frames`) don't end control.
6. `ctrl_team = team[ctrl_track]`, −1 if none.
7. Compare the speed field with speed computed from positions, and summarise spikes and jumps for aerial artefacts → Q10. Video may be used to debug, but what you see must not become labels, corrections or frame-specific rules.

Reference implementation sketch:
```python
d = np.linalg.norm(Pxy - Bxy[:, None, :], axis=2)
d = np.where(np.isnan(d), np.inf, d)
near = d.argmin(1); dmin = d[np.arange(T), near]
near_speed = P[np.arange(T), near, 2]
raw = (dmin < MAX_D) & (ball_v < near_speed + MAX_REL)
```

## Config keys
`control.max_dist_m: 1.5`, `control.max_rel_speed_kmh: 8`, `control.min_frames: 6`, `control.max_gap_frames: 3`, `control.smooth_window: 7`.

## Tests
- Synthetic: player dribbling with ball → controlled; ball passing at 60 km/h past a player 1 m away → not controlled; brief 2-frame contact → filtered by hysteresis.

## Acceptance
- Runs in < 30 s for the whole match.
- Local report: % frames controlled per team and a histogram of control-episode lengths.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S3 (or log it as a known limitation in the report).
