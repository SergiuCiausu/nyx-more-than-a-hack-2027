# S1 — Load tracking data

## Goal
Parse `atd.txt` using the spec in `atd.xml` into numpy arrays and Parquet, and answer Q1, Q2, Q8, Q9, Q11 in `05_open_questions.md`.

## Input
`data/raw/BAR-ATH_broadcast_atd.txt`, `data/raw/BAR-ATH_broadcast_atd.xml`

## Output
- `data/interim/tracking.npz`: `frame (T,)`, `P (T,51,3)`, `B (T,3)`, `team (51,)`, `is_gk (51,)` — positions already in metres
- `data/interim/players.parquet` (long: frame, track, team, x, y, speed)
- `data/interim/ball.parquet` (frame, x, y, speed)
- `data/interim/meta.json` (fps, pitch size, team names/ids, track→team, GK tracks, attack direction 1st half)
- `reports/S1_sanity.md`

## Steps
1. Parse the XML (lxml): fps, field size, teams, players (id, teamId, position_type), channel order from `DataFormatSpecification`. Do **not** hard-code 51 tracks or the channel order — read them from the XML.
2. First, print the first 30 lines of the TXT and list the distinct non-numeric tokens → answer Q1.
3. Parse the TXT line by line (`frame:players:ball`; players split `;`; values split `,`). Missing token → NaN. Optionally a faster path: read with polars/pyarrow as a single string column and split vectorised.
4. Convert x × 105, y × 68 (from XML field size).
5. Check: line count, contiguous frames, value ranges (0–1 before scaling, speeds 0–40 km/h players, 0–150 km/h ball) → Q2.
6. Sanity report: % non-NaN per track over time (plot), number of active tracks per frame (histogram), max per-frame position jump per track (teleports → track fragments) → Q8, Q9, Q11.

## Libraries
lxml, numpy, polars/pyarrow, matplotlib for the report. Optionally compare with `kloppy`'s Metrica EPTS loader.

## Tests (synthetic)
- A 3-line fake TXT + minimal XML with 2 players → exact arrays.
- Missing values in player and ball blocks → NaN.
- Scaling to metres.

## Acceptance
- T == 164,488 (or a documented reason why not); no parse errors.
- Q1, Q2, Q8, Q9, Q11 answered in `05_open_questions.md`.
- Full parse in < 2 min on a laptop.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S1 (or log it as a known limitation in the report).
