# S1 — Load, check and clean the tracking data

## Goal
Parse the tactical `atd.txt` using the spec in `atd.xml`, parse Smart Tagging, repair the tracking with code, and answer Q1, Q2, Q8, Q9, Q11 and Q12 in `05_open_questions.md`.

## Input
`BAR-ATH_tactical_atd.txt`, `BAR-ATH_tactical_atd.xml`, `BAR-ATH_tactical_smart_tagging.xml`, using paths from `config.yaml`.

## Output
- `data/interim/tracking.npz`: `frame (T,)`, `P (T,N,3)`, `B (T,3)`, `team (N,)`, `is_gk (N,)`, positions in metres and speeds in km/h
- `data/interim/players.parquet` (long: frame, slot, team, x, y, speed)
- `data/interim/ball.parquet` (frame, x, y, speed, repaired flag)
- `data/interim/tags.parquet` (video_start, video_end, code, team, half, type)
- `data/interim/meta.json` (fps, pitch size, team names/ids, slot→team)
- `reports/S1_sanity.md`, kept local

Array index `i` is the submission row and the video frame. It corresponds to ATD `frame_id = i + 1`.

## Steps
1. Parse the XML (lxml): fps, teams, players (id, teamId, position_type if present), channel order from `DataFormatSpecification`. Do **not** hard-code the number of slots or the channel order.
2. Count the distinct non-numeric tokens and print only the counts → Q1. Never print raw lines.
3. Parse the TXT line by line (`frame:players:ball`; players split `;`; values split `,`). Missing token → NaN.
4. Convert x and y to metres using the `pitch` values in `config.yaml`. Convert speeds to km/h after answering Q12.
5. Fail loudly unless these match: 199,319 contiguous frames from 1, 46 slots, the ball present on 113,488 frames overall and 94,214 graded frames, and about 20.5 mean tracked objects on graded frames → Q2.
6. Parse Smart Tagging, convert times with `round(seconds * 25)`, and confirm 638 instances. Handle `N/A` teams explicitly and match strings case-insensitively.
7. Record a quality summary: ball-gap lengths, per-slot coverage, position jumps, frames with more than 22 objects, positions off the pitch → Q8, Q9, Q11.
8. Clean with code only:
   - Remove impossible ball and player jumps using `ball.max_kmh` and `player.max_kmh`.
   - Treat frozen ball positions as missing.
   - Merge track fragments where continuity supports it.
   - Interpolate ball gaps up to `clean.max_ball_gap_frames`, and leave longer gaps missing.
9. Test the repairs by hiding known ball positions and measuring how well they are rebuilt.

## Libraries
lxml, numpy, polars/pyarrow, matplotlib for local reports. Optionally compare with `kloppy`'s Metrica EPTS loader.

## Tests (synthetic)
- A 3-line fake TXT + minimal XML with 2 players → exact arrays.
- Missing values in player and ball blocks → NaN.
- Scaling to metres and speed conversion.
- Fake Smart Tagging XML with an `N/A` team → correct frames and team.
- A ball teleport is removed, and a short gap is interpolated while a long gap stays missing.

## Acceptance
- All step 5 and step 6 checks pass, with no parse errors.
- Regenerated baseline files match the committed ones.
- Q1, Q2, Q8, Q9, Q11 and Q12 answered in `05_open_questions.md`.
- Full parse in < 2 min on a laptop.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S1 (or log it as a known limitation in the report).
