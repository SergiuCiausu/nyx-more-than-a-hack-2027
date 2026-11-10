# 04 — Data Model

Two layouts:
- **Dense per-frame arrays** (one row per frame) for computation — vectorised numpy.
- **Small per-event tables** (one row per segment/event) for results — polars / structured arrays.

T = 164,488 frames, 51 tracks. **Array index `i` ↔ frame `i + 1`** — keep this mapping in one helper (`frame_to_idx`, `idx_to_frame`).

## Codes (`src/matchan/codes.py`)
```python
STATE = {"controlled": 0, "in_flight": 1, "loose": 2, "dead": 3}
SEG   = {"pass": 0, "long_pass": 1, "loose": 2}
OUT   = {"complete": 0, "intercepted": 1, "out": 2, "stopped": 3, "none": -1}
TEAM  = {"barcelona": 0, "athletic": 1, "none": -1}
```

## 1. Raw inputs (S1)
| Name | Shape | dtype | Contents |
|---|---|---|---|
| `frame` | (T,) | int32 | frame number 1…164488 |
| `P` | (T, 51, 3) | float32 | per track x (m), y (m), speed (km/h); NaN if missing (~100 MB) |
| `B` | (T, 3) | float32 | ball x (m), y (m), speed (km/h) |
| `team` | (51,) | int8 | 0 for tracks 1–26, 1 for 27–51 |
| `is_gk` | (51,) | bool | True for tracks 26 and 51 |
| `H` | (T, 3, 3) | float32 | homography per frame (S2) |
| `view_ok` | (T,) | bool | homography `valid == 1` (S2) |

Parquet (long format) for `players.parquet`: `frame, track, team, x, y, speed`.

## 2. Per-frame derived arrays (S2–S5)
| Name | Shape | dtype | Stage | Contents |
|---|---|---|---|---|
| `period` | (T,) | int8 | S2 | 1 / 2 (0 = before kick-off/after end) |
| `camera_cut` | (T,) | bool | S2 | invalid view or replay |
| `dist` | (T, 51) | float32 | S3 | ball→track distance (inf if missing) |
| `nearest` | (T,) | int16 | S3 | nearest track index, −1 if no ball |
| `dmin` | (T,) | float32 | S3 | distance to nearest |
| `ball_v` | (T,) | float32 | S3 | smoothed ball speed |
| `controlled` | (T,) | bool | S3 | after hysteresis |
| `ctrl_track` | (T,) | int16 | S3 | track in control, −1 if none |
| `ctrl_team` | (T,) | int8 | S3 | team in control, −1 if none |
| `ball_state` | (T,) | int8 | S5 | STATE code |
| `poss_team` | (T,) | int8 | S5 | team credited, −1 if dead |

Illustrative frames around a long pass:
```
frame nearest dmin ball_v controlled ctrl_team ball_state poss_team
 5001    0    0.6  14.8   True        0        0          0
 5002    0    0.9  46.3   False      -1        1          0   <- kick
 5003    0    2.4  61.0   False      -1        1          0
 5040   30    0.8  18.2   True        1        0          1   <- opponent controls
```

## 3. Segments table (S4)
```python
seg_dtype = np.dtype([
    ("seg_id",        "i4"),
    ("start",         "i4"),   # first uncontrolled frame
    ("end",           "i4"),   # last frame before control / dead
    ("kind",          "i1"),   # SEG
    ("outcome",       "i1"),   # OUT
    ("release_track", "i2"),
    ("release_team",  "i1"),
    ("receive_track", "i2"),   # -1 if none
    ("receive_team",  "i1"),
    ("dist_m",        "f4"),
    ("straightness",  "f4"),
    ("kick_kmh",      "f4"),
    ("opp_dist_rel",  "f4"),   # nearest opponent at release
    ("aerial_flag",   "?"),    # suspected airborne artefacts
    ("clearance",     "?"),
])
```

## 4. Events table (S6)
| Column | Type | Notes |
|---|---|---|
| `event_id` | int32 | |
| `type` | str/cat | pass, long_pass, interception, loose_ball, out_of_play, stoppage, tackle |
| `frame_start`, `frame_end` | int32 | for clip cutting |
| `team` | int8 | −1 if n/a |
| `track` | int16 | −1 if n/a |
| `outcome` | str/cat | complete / intercepted / out / stopped / throw_in / corner / goal_kick / free_kick / won / loose / failed / foul_likely |
| `x`, `y` | float32 | event start location (m) |
| `seg_id` | int32 | link to segment, −1 if none |

## Storage
- numpy for per-frame computation; polars for joins/aggregations.
- Every stage writes Parquet to `data/interim/` so later stages never re-parse the 142 MB TXT.
