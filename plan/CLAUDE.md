# Football Match Analysis — Project Guide for Claude Code

## Goal
Build a Python pipeline that analyses one football match (Barcelona vs Athletic Club, ~110 min broadcast) and produces, for every frame:
- which team (and, where possible, which player track) has the ball,
- the ball state (controlled / in flight / loose / dead),

and an events table with passes (incl. long balls), loose balls, interceptions, out-of-play, stoppages and tackles.

The match already comes with **optical tracking data** (Metrica Sports, FIFA EPTS format) and a **per-frame homography**. The core pipeline therefore works on tracking data with numpy/polars. Computer vision on the video is used only for verification and an optional ball-speed module.

## How to read these docs
Read in this order before writing code:
1. `docs/01_data_inputs.md` — what the input files contain and their exact format
2. `docs/02_architecture_and_stack.md` — repo layout, pipeline stages, libraries
3. `docs/03_definitions.md` — football rules/conventions the code must implement
4. `docs/04_data_model.md` — every array and table, with shapes and dtypes
5. `docs/05_open_questions.md` — assumptions that must be verified on the real data first
6. `docs/06_edge_cases.md` — 50 edge cases, each owned by a stage; re-read the ones for a stage before building it
7. `docs/stages/S1_…` to `S8_…` — one spec per pipeline stage, built in order

## Build order
S1 load → S2 align → S3 control → S4 segments → S5 ball state → S6 events → S7 verify & evaluate → (optional) S8 CV ball speed.
Each stage reads the previous stage's Parquet output and writes its own. Do not start a stage until the previous one passes its acceptance checks.

## Working rules
- **Verify before assuming.** The items in `05_open_questions.md` (frame offset, homography direction, missing-value encoding, track identity, half-time) must be checked against the real files in S1/S2 and the answers written back into that doc.
- **All thresholds live in `config.yaml`** (start by copying `config.example.yaml`), never hard-coded. Every threshold in these docs is a starting value to be tuned.
- **Vectorise per-frame logic with numpy.** Python loops only where state is inherently sequential (hysteresis, segment walking).
- **Every stage writes Parquet** to `data/interim/` and has a small CLI entry point.
- **Every stage has tests** (`pytest`) on small synthetic arrays plus a sanity report on the real match.
- **Integer codes for categories** (states, segment kinds, outcomes), with the name mappings defined once in `src/matchan/codes.py`.
- **Do not add detection/tracking models (YOLO etc.) to the core pipeline.** They belong only to S8.
- Keep the 13.9 GB video out of git; read it by path from `config.yaml`.
