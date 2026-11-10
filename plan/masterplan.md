# Master plan: More than a Hack 2027, Phase 1

Written 2026-11-10. Submission deadline: **2026-11-17**. Target submission date: **2026-11-16**.

## The goal

Build an algorithm that labels all 199,319 frames of the tactical video of Barcelona vs Athletic Club, and submit:

1. `<team_name>.csv`, with the header `frame,label` and one row per frame, frames `0` to `199318` in order.
2. A public code repository that regenerates that exact file.
3. `<team_name>.pdf` or `.md`, a short technical document.

| Label | Meaning |
|---|---|
| `H` | Home (Barcelona) in possession, ball alive |
| `A` | Away (Athletic Club) in possession, ball alive |
| `D` | Dead ball |
| `X` | No answer |

Ranking is on `spellF1`, with `macroF1` as the first tie-break. The score to beat is `spellF1` 0.2433. Differences below about 0.04 are noise.

## What "validated" can mean here

The full-match answers are private and there is no live leaderboard, so the final labels cannot be fully validated before submission.

| Can be validated | How |
|---|---|
| Input files | Published counts, baseline reproduction, file hashes |
| Cleaning | Hide known positions and measure how well they are rebuilt |
| File format | `evaluation/validate_submission.py` |
| Reproducibility | Same SHA-256 hash from a clean clone |
| Pipeline correctness | Public 5 minutes, for correctness only |
| Expected accuracy | An estimate from other matches, degraded to match this data's noise |

## Rules that constrain the work

- Labels must come from an algorithm. No hand labels or hand corrections on this match, in any amount or for any purpose, including validation.
- No rules or constants tied to specific frames of this match that come from watching it.
- No human-produced data about this match from any source, such as event data or corrected tracking.
- The public 5 minutes are for checking correctness. They are not for training, tuning or choosing between versions.
- Do not make the videos, datasets or extracts available to any third-party hosted LLM. Share code with LLMs, not match data.
- Do not commit or publish the data, clipped video or frame-level exports.
- Everything used must be accessible to the organisers and explained in the technical document.
- Hand-labelled data from other matches is allowed, and must be documented: what, how, by whom and how much.

## Changes to the existing plan docs

Applied on 2026-11-10.

1. **Remove hand labelling from S7.** Remove the keyboard tagger, `data/labels/`, the hand-labelled test set and the tuning loop that depends on it.
2. **Switch the inputs to the tactical files.** Use the 46-slot, 199,319-frame data and add Smart Tagging. `docs/01_data_inputs.md` currently describes only the broadcast files.
3. **Cut unscored work.** Keep S1 to S5, with passes and loose balls as internal signals only. Drop tackles, the events table and S8 unless they raise the score.
4. **Fix the definitions.** Loose balls always stay with the last team, so remove the `none` option for `possession.loose_credit`. Homography cannot signal stoppages on the tactical feed, because it is valid on every frame.
5. **Close answered questions.** `DATA.md` answers Q3, Q5 and Q6 for the tactical files: the frame offset, frame count and half boundaries.
6. **Limit video checks to debugging.** Watching video is allowed, but what you see must not become labels, corrections or frame-specific rules.

## Step-by-step plan

### Setup and data (Nov 10)

1. Confirm the team has 3 to 5 members, the terms are accepted, and someone has a Google account for the submission form.
2. Download these four files directly to your own machine, using the link the organisers provided. Do not fetch them through an LLM connector.
   - `BAR-ATH_tactical_atd.txt`
   - `BAR-ATH_tactical_atd.xml`
   - `BAR-ATH_tactical_homography.csv`
   - `BAR-ATH_tactical_smart_tagging.xml`
3. Leave the videos and broadcast files for later. The broadcast feed is optional and not synchronised, and video is only needed for your own ball detection or debugging overlays.
4. Put the files in `more-than-a-hack-2027/data/tactical/`, keeping their original names. To store them elsewhere, set `MTAH_DATA_DIR` and keep the `tactical/` subfolder.
5. Record a SHA-256 hash of each file.
6. Use Python 3.11, since the scorer is tested on 3.8 and 3.11. Create a virtual environment and pin all dependencies.
7. Add `reports/`, Parquet files and video clips to `.gitignore`. Keep the data download link out of the repository, README and technical document.
8. Ask the organisers whether aggregate statistics may be shared with a hosted LLM. Until they answer, run everything locally and share only code and pass/fail results.

### Validate the inputs (Nov 10)

9. Run the scorer from `more-than-a-hack-2027` and confirm it reproduces the table in its README.

   ```powershell
   pip install -r requirements.txt
   python evaluation/run_evaluation.py
   ```

10. Regenerate the baselines and confirm they match the committed ones. No changed files means your data and setup match the organisers'.

    ```powershell
    python starter/baseline.py
    git status --short evaluation/results
    ```

11. Write one check script that fails loudly unless all of these hold:
    - 199,319 contiguous tracking frames, numbered 1 to 199319, each with 46 slots.
    - The ball present on 113,488 frames overall and 94,214 graded frames.
    - A mean of about 20.5 tracked objects per graded frame.
    - 638 Smart Tagging instances.
    - 199,319 homography rows, all with `valid` equal to 1.
12. Confirm the speed unit. `DATA.md` says m/s and the broadcast notes say km/h. Compare the speed column with speed computed from positions, and set thresholds to match.
13. Confirm the team of each slot from the XML, and which way each team attacks in each half.
14. Report data quality as summary numbers: ball-gap lengths, slot jumps, frames with more than 22 objects, and positions off the pitch.

### Clean the data with code (Nov 11)

15. Remove impossible ball and player jumps, then merge track fragments.
16. Interpolate short ball gaps only. Leave long gaps missing.
17. Test the repairs by hiding known ball positions and measuring how well they are rebuilt. This needs no human labels.

### Build the labeller (Nov 12 to 14)

18. Compute per-frame evidence: who is near the ball, relative speed, sustained control, kicks, ball out of bounds, players slowing together, and restart shapes.
19. Add Smart Tagging as weak evidence only, and handle its `N/A` teams, which include every throw-in.
20. Choose the best `H`/`A`/`D` sequence with a state model, such as Viterbi with a penalty for switching. This stops flicker and can use lookahead, because the pipeline runs offline.
21. Flip possession only at the moment the opponent gains control. Test that touches, deflections and aerial duels without control do not flip it. Being 1 second late drops `spellF1` to about 0.65.
22. Never call a ball gap dead on its own. A missed stoppage merges two possessions, so give dead-ball detection real attention.
23. Detect the second-half kickoff from the data, and use the published frame 113,742 only as a check. Frames 110,808 to 113,741 are graded and all `D`.
24. Write only `H`, `A` or `D` on graded frames, since `X` counts as a miss in `macroF1`. Use `X` only on ungraded frames.

### Tune without breaking the rules (Nov 15)

25. Get tracking with possession labels from other matches. Candidates are the DFL/Sportec open dataset and Metrica's sample data. Check their access terms and possession definitions first, as these were named from memory.
26. Degrade that data to match the noise measured in step 14, then tune on `spellF1` using `evaluation/possession_metrics.py`.
27. Run on the public 5 minutes only to confirm correctness, such as alignment and label meaning. Never use it to choose between versions.
28. Check that the output is plausible: possession counts, durations, and no tiny fragments.

### Freeze and submit (Nov 16)

29. From a clean clone, run one command twice and confirm both CSVs have the same SHA-256 hash. Fix random seeds and any other source of non-determinism.
30. Run `python evaluation/validate_submission.py <team_name>.csv` on the exact file you will upload.
31. Confirm no match data is in the repository or its git history, then make it public.
32. Write the technical document:
    - The approach, and why it was chosen.
    - Assets used and assets dropped.
    - All external labelled data: source, who labelled it and how much.
    - The possession rules implemented for loose balls, challenges and restarts.
    - Limitations, and what you would fix with more time.
    - How to reproduce the file.
33. Name the files `<team_name>.csv` and `<team_name>.pdf`, then submit through the form emailed to participants, with the team name and the public repository URL. You can edit the response until the deadline, and the last response is the one scored.

## Status

| Item | State on 2026-11-10 |
|---|---|
| Match data | Not yet in `more-than-a-hack-2027/data/` |
| Organiser question on aggregate statistics | Not yet asked |
| Changes to existing plan docs | Applied |
| Pipeline code | Not started |
