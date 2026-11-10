# S6 — Events table (incl. tackles)

> **Deferred.** Events are not scored and this stage is not in the build order. Build it only if it raises `spellF1` on other matches. Any labels must come from other matches, never this one.

## Goal
Turn segments and states into one tidy events table (schema in `04_data_model.md`).

## Input
`segments.parquet`, `ball_state.parquet`, `control.parquet`, `tracking.npz`.

## Output
`data/interim/events.parquet`, `reports/S6_summary.md`.

## Event types
| Type | Source |
|---|---|
| `pass`, `long_pass` | segments of that kind; outcome from segment |
| `interception` | segment outcome `intercepted` → event at receive frame by receive team |
| `loose_ball` | loose segments (flag clearance) |
| `out_of_play` | dead stretches with reason `out`; outcome = restart type |
| `stoppage` | dead stretches with reason `stoppage`; outcome = free_kick / unknown |
| `tackle` | see below |

## Tackles (`src/matchan/tackles.py`)
1. Candidates: frames where a player controls the ball and an opponent is within `tackle.contact_m` with closing speed > `tackle.min_closing_kmh` (relative velocity along the line between them, from smoothed positions). Merge candidate frames within 0.5 s into one candidate.
2. Exclude: ball arrived at pass speed (candidate starts within an in_flight segment) → that's an interception.
3. Outcome in the next `tackle.window_s`:
   - possession flips to defender's team → `won`
   - state becomes loose/out → `loose`
   - attacker keeps control → `failed`
   - dead stoppage starts → `foul_likely`
4. Optional (later): cut 3 s clips around each candidate with ffmpeg and train/apply a PyTorch clip classifier (SoccerNet action-spotting baseline) to reject false candidates. Only GPU part of the core.

## Config keys
`tackle.contact_m: 1.0`, `tackle.min_closing_kmh: 5`, `tackle.window_s: 1.5`.

## Acceptance
- Summary per team: passes (complete %), long passes (complete %), interceptions, loose balls, outs by restart type, stoppages, tackles by outcome.
- Every event has frame_start/frame_end usable for clip cutting.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S6 (or log it as a known limitation in the report).
