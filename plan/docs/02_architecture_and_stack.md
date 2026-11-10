# 02 — Architecture and Tech Stack

## Principle
Metrica already did detection and tracking. The core pipeline is **numerical processing of tracking data** (numpy/polars), not computer vision. Video is optional, for debugging overlays or the deferred S8 module.

## Repo layout
```
matchan/
├── CLAUDE.md
├── config.yaml                 # paths + every threshold
├── pyproject.toml              # pinned dependencies
├── docs/                       # these specs
├── data/                       # gitignored
│   └── interim/                # Parquet output of each stage
├── reports/                    # gitignored, may contain match-derived output
├── submission/                 # the generated CSV
├── src/matchan/
│   ├── codes.py                # integer codes for states / kinds / outcomes
│   ├── config.py               # load config.yaml
│   ├── io_epts.py              # S1 parser
│   ├── clean.py                # S1 automatic repairs
│   ├── align.py                # S2
│   ├── control.py              # S3
│   ├── segments.py             # S4
│   ├── states.py               # S5
│   ├── verify.py               # S7 checks and tuning
│   └── cli.py                  # typer CLI: `matchan <stage>`
├── tests/                      # pytest, synthetic arrays per stage
└── notebooks/                  # local exploration, outputs cleared before commit
```

## Pipeline and data flow
| Stage | Input | Output |
|---|---|---|
| S1 load and clean | atd.txt + atd.xml + smart_tagging.xml | `tracking.npz`, `players.parquet`, `ball.parquet`, `tags.parquet`, `meta.json` |
| S2 align | S1 + published half boundaries | `frames.parquet` (period, graded, attack dir) |
| S3 control | S1 + S2 | `control.parquet` (nearest, dmin, controlled, ctrl_team) |
| S4 segments | S3 | `segments.parquet` (pass / long_pass / loose + outcome), internal signal only |
| S5 ball state and labels | S1 + S2 + S3 + S4 | `ball_state.parquet`, submission CSV |
| S7 verify and evaluate | everything | checks, tuning on other matches, reproducibility report |
| S6 events (deferred) | S4 + S5 + S1 | `events.parquet` |
| S8 CV ball speed (deferred) | mp4 + homography | `cv_ball.parquet` |

## Tech stack per feature
| Feature | Tech |
|---|---|
| Loading tracking data | Python 3.11, custom parser (or `kloppy` Metrica EPTS loader), numpy, polars, Parquet (pyarrow) |
| Possession / control | numpy (vectorised), scipy.signal (Savitzky–Golay smoothing) |
| Pass vs loose segmentation | numpy features + rules |
| Out of play | numpy boolean masks + restart-location rules |
| Stoppages / dead ball | numpy rolling windows |
| Label sequence | Viterbi over `H`/`A`/`D` with switching penalties |
| Debug overlays (optional) | OpenCV drawing, ffmpeg, mplsoccer pitch plots. Never published |
| Evaluation | the competition's `evaluation/possession_metrics.py`, `run_evaluation.py`, `validate_submission.py` |
| Pipeline / CLI | typer CLI + Makefile, one command to regenerate the submission |
| Tests | pytest |

## Not needed in the core pipeline
YOLO / detectors, ByteTrack / BoT-SORT, team colour clustering, SAHI, TensorRT, hand-labelling tools. Only S8 may use detection models, and hand-labelling tools must never be used on this match.

## Hardware
Any laptop with 16 GB RAM for the core pipeline. A GPU is only relevant for S8. Keep any video on a fast local disk.
