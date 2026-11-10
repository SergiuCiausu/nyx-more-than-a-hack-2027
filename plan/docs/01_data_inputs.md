# 01 — Data Inputs

Do not open these files with a hosted LLM or paste their contents into one. Everything below comes from the competition's public docs or starter code.

## Core inputs: tactical recording
All files share the prefix `BAR-ATH_tactical` and live in `more-than-a-hack-2027/data/tactical/`, or under `MTAH_DATA_DIR`.

| File | What it is |
|---|---|
| `BAR-ATH_tactical_atd.xml` | Metadata and format spec for the tracking data (FIFA EPTS) |
| `BAR-ATH_tactical_atd.txt` | Tracking data: one line per frame |
| `BAR-ATH_tactical_homography.csv` | One 3×3 homography per frame |
| `BAR-ATH_tactical_smart_tagging.xml` | 638 automatic tactical phases and set pieces |
| `BAR-ATH_tactical.mp4` | Optional. 199,319 frames, 25 fps, 1920×1080 |

Row *i* of the submission is frame *i* of the tactical video.

## Tracking metadata (`atd.xml`)
- Provider: Metrica Sports, automatic and uncorrected
- Frame rate: 25 fps
- Records: 199,319, with `frame_id` 1 to 199319
- Track slots: 46, for 22 players on the pitch. Slots are tracker fragments, not people
- Each `<Player>` entry has a `teamId`. The order of entries matches the order of slots in the TXT
- The first `<Team>` listed is home, which is Barcelona
- Positions are normalised 0–1. Use a 105.3 × 68.0 m pitch for metres, from `games.py`
- Speed unit: `DATA.md` says m/s, while the broadcast XML says km/h. Confirm it (Q12)

## Tracking data (`atd.txt`)
One line per frame:

```
frame_id:p1x,p1y,p1s;p2x,p2y,p2s;…;p46x,p46y,p46s:ballx,bally,balls
```

- Separator between the three blocks: `:`
- Slots separated by `;`, in slot order
- Within a slot or the ball: `x,y,speed` separated by `,`
- The ball has no height
- Missing observations are `NaN`
- The ball is present on 113,488 of all frames and 94,214 of the 158,857 graded frames
- The mean number of tracked objects on graded frames is 20.5, and it can exceed 22 when the tracker duplicates someone

## Smart Tagging (`smart_tagging.xml`)
SportsCode XML. `<start>` and `<end>` are seconds from the first frame of the tactical video.

- Each instance has a code and tags: `Team`, `Half`, and sometimes `Type`, `Side`, `Direction of ball entry` and `Max Players in the box`
- `Team` values are `Barcelona`, `Atletic Club` (spelled that way) and `N/A`. All throw-ins are `N/A`
- Match codes and tags case-insensitively
- Treat it as weak evidence. It contains errors, and its intervals are phases rather than possessions

## Homography (`homography.csv`)
Columns: `frame,h00,h01,h02,h10,h11,h12,h20,h21,h22,reprojection_error,valid`

- `frame` starts at 0, the same index as a submission row
- Maps normalised pitch coordinates (0–1) to image pixels
- Valid on every tactical frame, so it cannot signal stoppages
- Only needed if the video is used

## Frame alignment
| Source | Index | To get a submission row |
|---|---|---|
| Submission | `0 … 199318` | — |
| ATD | `frame_id` `1 … 199319` | `video_frame = frame_id - 1` |
| Smart Tagging | seconds from video start | `video_frame = round(seconds * 25)` |
| Homography | `frame` `0 … 199318` | already a submission row |

## Graded frames
| | First frame | Last frame | Graded |
|---|---|---|---|
| Before the match | 0 | 11,184 | no |
| 1st half | 11,185 | 90,651 | yes |
| Halftime | 90,652 | 110,807 | no |
| 2nd half | 110,808 | 190,197 | yes |
| After the final whistle | 190,198 | 199,318 | no |

The second half kicks off at frame 113,742, so frames 110,808 to 113,741 are graded and all `D`.

## Public ground truth
`evaluation/gt/BAR-ATH_possession_gt_public.csv` has labels for frames 11,185 to 18,684 and `X` elsewhere. It is for checking correctness only.

## Optional: broadcast recording
Files use the prefix `BAR-ATH_broadcast` in `data/broadcast/`. It has 164,488 frames, 51 track slots and valid homography on 65.8% of frames. It is **not synchronised** to the tactical video and no mapping is provided, so it is not part of the core pipeline.
