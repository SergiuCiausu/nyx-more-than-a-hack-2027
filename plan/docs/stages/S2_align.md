# S2 — Periods, graded frames and attack direction

## Goal
Build the per-frame timeline that later stages use: match period, graded frames and attack direction. Answer Q7. Frame alignment and half boundaries are already settled by `DATA.md`.

## Input
S1 outputs and `games.half_video_blocks("BAR-ATH")`. The homography and video are only needed for optional debugging overlays.

## Output
- `data/interim/frames.parquet`: `frame, period, graded, attack_dir_barca` (+1 = left→right)
- `reports/S2_alignment.md`, kept local

## Steps
1. Read the graded half blocks from `games.half_video_blocks`. Set `period` to 1 or 2 inside them and 0 elsewhere, and set `graded` to match.
2. Confirm that the kept frames total 158,857.
3. **Sides (Q7):** use the mean x of each team's deepest player per period to derive `attack_dir_barca` per period.
4. Detect the second-half kickoff from the data: ball near the centre spot, low ball speed, players in their own halves. Use the published frame 113,742 only as a check, and store the detected frame for S5.
5. Optional, only if the video is used: confirm the homography direction (Q4) by drawing pitch markings. Overlays are for local debugging only.

## Libraries
numpy, polars. OpenCV only for optional overlays.

## Tests
- Synthetic blocks produce the right `period` and `graded` values.
- Kickoff detector on a synthetic ball path with a centre-spot restart.
- Side detector on synthetic team positions that switch ends at halftime.

## Acceptance
- 158,857 graded frames.
- Detected second-half kickoff agrees with the published frame within a small tolerance.
- Q7 answered.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S2 (or log it as a known limitation in the report).
