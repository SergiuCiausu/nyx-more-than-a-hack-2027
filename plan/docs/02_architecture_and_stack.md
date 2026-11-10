# 02 — Architecture and Tech Stack

## Principle
Metrica already did detection and tracking. The core pipeline is **numerical processing of tracking data** (numpy/polars), not computer vision. The video is used for verification clips and an optional ball-speed CV module (S8).

## Repo layout
```
matchan/
├── CLAUDE.md
├── config.yaml                 # paths + every threshold
├── pyproject.toml
├── docs/                       # these specs
├── data/
│   ├── raw/                    # atd.txt, atd.xml, homography.csv (video by path only)
│   ├── interim/                # Parquet output of each stage
│   └── labels/                 # hand-labelled test set (S7)
├── src/matchan/
│   ├── codes.py                # integer codes for states / kinds / outcomes
│   ├── config.py               # load config.yaml
│   ├── io_epts.py              # S1 parser
│   ├── align.py                # S2
│   ├── control.py              # S3
│   ├── segments.py             # S4
│   ├── states.py               # S5
│   ├── events.py               # S6
│   ├── tackles.py              # S6 (tackle candidates + outcomes)
│   ├── verify.py               # S7 overlays, clips, metrics
│   ├── cv_ball/                # S8 optional
│   └── cli.py                  # typer CLI: `matchan <stage>`
├── tests/                      # pytest, synthetic arrays per stage
└── notebooks/                  # exploration / sanity reports
```

## Pipeline and data flow
| Stage | Input | Output |
|---|---|---|
| S1 load | atd.txt + atd.xml | `tracking.npz` / `players.parquet`, `ball.parquet`, `meta.json` |
| S2 align | S1 + homography.csv + mp4 | `frames.parquet` (view_ok, camera cut, period, attack dir) |
| S3 control | S1 + S2 | `control.parquet` (nearest, dmin, controlled, ctrl_team) |
| S4 segments | S3 | `segments.parquet` (pass / long_pass / loose + outcome) |
| S5 ball state | S3 + S4 + S2 | `ball_state.parquet` (state + poss_team per frame) |
| S6 events | S4 + S5 + S1 | `events.parquet` (passes, interceptions, outs, stoppages, tackles) |
| S7 verify | everything + mp4 | overlay clips, metrics report |
| S8 CV ball speed (optional) | mp4 + homography | `cv_ball.parquet`, comparison with Metrica speed |

## Tech stack per feature
| Feature | Tech |
|---|---|
| Loading tracking data | Python 3.11+, custom parser (or `kloppy` Metrica EPTS loader), numpy, polars, Parquet (pyarrow) |
| Frame alignment & homography | OpenCV (`perspectiveTransform`), decord or PyAV for single frames, `ffprobe` |
| Possession / control | numpy (vectorised), scipy.signal (Savitzky–Golay smoothing) |
| Pass vs loose segmentation | numpy features + rules; later optional scikit-learn (gradient boosting) on labelled segments |
| Out of play | numpy boolean masks + restart-location rules |
| Interception | logic over segment/control tables (polars) |
| Stoppages / dead ball | numpy rolling windows, join homography `valid`; optional `hmmlearn` smoothing |
| Half-time / periods | numpy (long gap/invalid run, ball on centre spot) |
| Tackles | numpy rules; optional PyTorch clip classifier (SoccerNet action-spotting baseline) on 3 s clips — only GPU use in core |
| Verification | OpenCV drawing, ffmpeg clip cutting, mplsoccer pitch plots |
| Labelling | Label Studio or CVAT, or a small OpenCV keyboard tagger |
| Evaluation | scikit-learn metrics, Jupyter |
| Pipeline / CLI | typer CLI + Makefile; Prefect only if many matches |
| Dashboard (optional) | Streamlit |
| Tests | pytest |

## Not needed in the core pipeline
YOLO / detectors, ByteTrack / BoT-SORT, team colour clustering, SAHI, TensorRT. Only S8 may use them.

## Hardware
Any laptop with 16 GB RAM for S1–S7. GPU only for the optional tackle classifier and S8. Keep the video on a fast local disk.
