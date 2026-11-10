# S7 — Verification and evaluation

## Goal
Check results against the video and measure accuracy on a hand-labelled test set; feed findings back into `config.yaml` thresholds.

## Tools to build
1. **Overlay renderer** (`verify.py`): for a frame range, read video frames (decord/PyAV), project players (team colours), ball and ball state/possession text via H (OpenCV), write MP4 with ffmpeg.
2. **Event clip cutter:** for any events row → 3 s clip with overlay (1 s before, 2 s after).
3. **2D pitch animation** with mplsoccer for any segment (useful when the camera cut).
4. **Labelling:** a simple OpenCV keyboard tagger (or Label Studio/CVAT): play the video, keys for `1` Barcelona control, `2` Athletic control, `f` in flight, `l` loose, `d` dead, `p` pass start, `t` tackle. Store to `data/labels/*.csv` as frame, label.

## Test set
- 10 minutes of open play + one set-piece-heavy period: per-frame ball state & possession.
- All passes, loose balls and tackles in one half (or 20 min).

## Metrics
- Per-frame ball-state accuracy and confusion matrix; possession-team accuracy (live frames).
- Events: precision/recall with a ±10-frame matching tolerance, per type.
- Pass outcome accuracy.
- Possession % vs labels.
(scikit-learn metrics, notebook `notebooks/S7_eval.ipynb`, report `reports/S7_eval.md`.)

## Loop
Tune thresholds for the worst failure mode first (usually aerial balls and crowded boxes), re-run S3–S6, re-evaluate. Log each config change and its metric effect in the report.

## Acceptance
- Ball-state accuracy ≥ 90% on live frames, possession-team accuracy ≥ 95%, pass precision/recall ≥ 85% (targets — adjust after first measurement).

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S7 (or log it as a known limitation in the report).
