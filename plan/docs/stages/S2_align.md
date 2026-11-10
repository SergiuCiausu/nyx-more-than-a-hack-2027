# S2 — Align tracking, homography and video; find periods

## Goal
Make sure tracking frame N, homography row N and video frame N refer to the same moment; build per-frame flags for valid view, camera cuts and match period. Answer Q3–Q7.

## Input
S1 outputs, `homography.csv`, the video (path from `config.yaml`).

## Output
- `data/interim/frames.parquet`: `frame, view_ok, reproj_err, camera_cut, period, attack_dir_barca` (+1 = left→right)
- `data/interim/homography.npz`: `H (T,3,3)` aligned to tracking frames, in the verified direction (pitch metres → image pixels) and its inverse
- `reports/S2_alignment.md` with overlay images

## Steps
1. `ffprobe` the video: frame count, fps, resolution → Q5.
2. Load homography CSV; `view_ok = valid == 1`.
3. **Direction and units (Q4):** for a few valid frames, project pitch corners, centre spot and penalty spots with H and with H⁻¹, in metres and in normalised units; draw them on the video frame. Keep the combination that lands on the pitch markings.
4. **Offset (Q3):** at 5–10 well-spread frames, project all players and the ball onto the video frame for offsets −2…+2; choose the offset where points sit on players (score by eye or by detecting point-on-shirt). Store the offset in `config.yaml`.
5. **Camera cuts:** `camera_cut = ~view_ok` plus sudden large changes in H between consecutive frames (replay/scene change).
6. **Half-time (Q6):** find the long run of invalid view / missing ball in the middle of the file, then the second kick-off: ball within ~1 m of centre spot, ball speed ~0, all players in their own half. Set `period` 1/2 (0 outside play).
7. **Sides (Q7):** mean x of each GK per period; derive `attack_dir_barca` per period.

## Libraries
OpenCV, decord or PyAV, numpy, polars, ffmpeg/ffprobe.

## Tests
- Synthetic homography: project known points and invert round-trip within 1e-3.
- Period detector on a synthetic ball path with a gap and a centre-spot restart.

## Acceptance
- Overlay images for ≥ 5 frames (both halves) show players and ball on the right spots.
- Q3–Q7 answered; offset and H direction stored in config.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S2 (or log it as a known limitation in the report).
