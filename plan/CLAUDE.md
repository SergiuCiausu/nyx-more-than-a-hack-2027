# Football Possession Pipeline — Project Guide for Claude Code

## Goal
Build a Python pipeline that labels every one of the 199,319 frames of the **tactical video** of Barcelona vs Athletic Club with `H`, `A`, `D` or `X`, for More than a Hack 2027 Phase 1. Ranking is on `spellF1`, with `macroF1` as the tie-break.

The match comes with automatic tracking (Metrica Sports, FIFA EPTS format), Smart Tagging and a per-frame homography. The core pipeline works on tracking data with numpy/polars. Video is optional and used for debugging or your own ball detection.

## How to read these docs
Read in this order before writing code:
1. `masterplan.md` — the full plan, schedule and competition rules
2. `docs/01_data_inputs.md` — what the input files contain and their exact format
3. `docs/02_architecture_and_stack.md` — repo layout, pipeline stages, libraries
4. `docs/03_definitions.md` — football rules/conventions the code must implement
5. `docs/04_data_model.md` — every array and table, with shapes and dtypes
6. `docs/05_open_questions.md` — assumptions that must be verified on the real data first
7. `docs/06_edge_cases.md` — edge cases, each owned by a stage; re-read the ones for a stage before building it
8. `docs/stages/S1_…` to `S5_…` and `S7_…` — one spec per core pipeline stage, built in order

S6 and S8 are deferred. Build them only if they raise the score.

## Build order
S1 load → S2 align → S3 control → S4 segments → S5 ball state and labels → S7 verify and evaluate.
Each stage reads the previous stage's Parquet output and writes its own. Do not start a stage until the previous one passes its acceptance checks.

## Competition rules (these override everything else here)
- **No hand labels or hand corrections on this match**, in any amount or for any purpose, including validation.
- **No rules or constants tied to specific frames of this match** that come from watching it. Published half boundaries are allowed.
- **No human-produced data about this match** from any source, including event data, corrected tracking and known match facts.
- **The public 5 minutes are for checking correctness only.** Never train on them, tune on them or use them to choose between versions.
- **Tune only on other matches.** Hand-labelled data from other matches is allowed and must be documented.
- **Never expose match data to a hosted LLM, including this assistant.** Do not open, print or paste raw tracking rows, tags, homography rows, video frames or extracts. Scripts run locally and report pass/fail. Whether aggregate statistics may be shared is an open question for the organisers, so treat them as local-only until answered.
- **Never commit or publish** the data, the download link, clipped video or frame-level exports.
- **The submission must be byte-for-byte reproducible** from the public repository.

## Working rules
- **Verify before assuming.** The open items in `05_open_questions.md` must be checked against the real files in S1/S2 and the answers written back into that doc.
- **All thresholds live in `config.yaml`** (start by copying `config.example.yaml`), never hard-coded. Every threshold in these docs is a starting value, tuned only as described in S7.
- **Vectorise per-frame logic with numpy.** Python loops only where state is inherently sequential (hysteresis, segment walking, Viterbi).
- **Every stage writes Parquet** to `data/interim/` and has a small CLI entry point.
- **Every stage has tests** (`pytest`) on small synthetic arrays plus a local sanity report on the real match.
- **Integer codes for categories** (states, segment kinds, outcomes), with the name mappings defined once in `src/matchan/codes.py`.
- **Do not add detection/tracking models (YOLO etc.) to the core pipeline.** They belong only to S8.
- **Determinism:** fix seeds and avoid anything that changes the output between runs.
