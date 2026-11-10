# S7 — Verification and evaluation

## Goal
Check that the pipeline is correct, tune it without using human labels on this match, and prove the submission can be reproduced.

**No hand labels on this match, for any purpose.** The full-match answers are private, so the final labels cannot be fully validated before submission.

## Tools to build
1. **Input check script:** the S1 acceptance checks, reporting pass or fail only.
2. **Repair test:** hide known ball positions and measure how well S1 rebuilds them.
3. **Other-match evaluator:** run the pipeline on other matches and score with `evaluation/possession_metrics.py`.
4. **Noise simulator:** degrade clean tracking from other matches to match the noise measured in S1, including bursty ball loss, track fragments and duplicates.
5. **Debug overlay (optional):** draw tracking and predicted labels on video for local debugging. Never publish the output.

## Tuning
1. Get tracking with possession labels from other matches. Candidates are the DFL/Sportec open dataset and Metrica's sample data. Check their access terms and possession definitions first.
2. Degrade that data using the noise simulator.
3. Tune thresholds and `model.*` values on `spellF1`, starting with the worst failure mode.
4. Hold out at least one other match to confirm the tuned values generalise.
5. Log each config change and its effect.
6. Document every external dataset in the technical document: source, who labelled it and how much.

## Public 5 minutes
Run `python evaluation/run_evaluation.py` only to confirm correctness: frames aligned, file well-formed and labels meaning what you think. Never use it to choose between versions or settings.

## Final checks
1. Compare possession counts and durations with expected values for professional football. Do not tune to match the public window.
2. Run `python evaluation/validate_submission.py <team_name>.csv` on the exact file to upload.
3. From a clean clone, run the one regenerate command twice and confirm both files have the same SHA-256 hash.
4. Confirm no match data, clips, frame-level exports or the download link are in the repository or its git history.

## Acceptance
- All input checks pass.
- `spellF1` on the held-out other match clearly beats 0.2433, with the caveat that it is only an estimate for this match.
- `validate_submission.py` passes.
- Two clean runs produce byte-identical files.

## Edge cases
Before building, read `docs/06_edge_cases.md` and handle every case whose Stage column includes S7 (or log it as a known limitation in the report).
