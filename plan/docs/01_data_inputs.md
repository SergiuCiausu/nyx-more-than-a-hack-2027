# 01 — Data Inputs

All four files share the prefix `BAR-ATH_broadcast`.

| File | Size | What it is |
|---|---|---|
| `BAR-ATH_broadcast_atd.xml` | 65 KB | Metadata + format spec for the tracking data (FIFA EPTS) |
| `BAR-ATH_broadcast_atd.txt` | 142 MB | Tracking data: one line per frame |
| `BAR-ATH_broadcast_homography.csv` | 15 MB | One 3×3 homography per frame |
| `BAR-ATH_broadcast.mp4` | 13.9 GB | Broadcast video the tracking was derived from |

## Tracking metadata (`atd.xml`)
- Provider: **Metrica Sports**, tracking type **Optical**, `data_type = atd`, version 1
- **Frame rate: 25.0 fps**
- Frames: `startFrame=1`, `endFrame=164488` (≈ 6579.5 s ≈ 109.7 min — the whole video)
- Pitch: **105 × 68 m**
- Position unit: **normalised** (0–1) → multiply x by 105, y by 68 for metres
- Speed unit: **km/h**
- Teams:
  - `MET_user_team_o4Hossvb8M` = **Barcelona** (local), attack direction first half: left_to_right
  - `MET_user_team_x7C4FFX2HF` = **Athletic Club** (visiting), attack direction first half: right_to_left
- Players: **51 tracks**
  - Tracks 1–26 → Barcelona; track 26 is the goalkeeper
  - Tracks 27–51 → Athletic; track 51 is the goalkeeper
  - Names are only "Track N"; shirt numbers equal the track id (placeholders, not real numbers)
- Period parameters: `first_half_start=0`, `first_half_end=164488`, all other periods **empty** → half-time is NOT marked (see S2).

## Tracking data (`atd.txt`)
One line per frame, defined by `DataFormatSpecification` in the XML:

```
frame:p1x,p1y,p1s;p2x,p2y,p2s;…;p51x,p51y,p51s:ballx,bally,balls
```

- Separator between the three blocks: `:`
- Players separated by `;`, in track order 1…51
- Within a player / the ball: `x,y,speed` separated by `,`
- Ball has **no z (height)**.
- Missing values: encoding not yet confirmed (empty string, `NaN`, or similar) — check in S1.

## Homography (`homography.csv`)
Columns: `frame,h00,h01,h02,h10,h11,h12,h20,h21,h22,reprojection_error,valid`

- `frame` starts at **0** (tracking starts at 1 → offset to confirm in S2)
- `h22` = 1 (normalised matrix)
- `valid` = 0 with all-NaN matrix and `reprojection_error = -100` when no pitch view (frames 0–843 at the start; expect more around replays/close-ups/half-time)
- Typical `reprojection_error` ≈ 0.1–0.7 when valid
- Translation terms (`h02 ≈ 1700`, `h12 ≈ 660`) suggest the matrix maps **pitch → image pixels**; direction and pitch units (metres vs normalised) must be confirmed in S2.

## Video (`.mp4`)
Broadcast feed: camera pans/zooms, cuts to replays and close-ups. Frame count should match ~164,488 at 25 fps — confirm with `ffprobe` in S2.
