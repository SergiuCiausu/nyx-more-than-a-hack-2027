# 05 — Open Questions (verify on real data, then write answers here)

Each open item must be checked in the named stage and its answer recorded below. Do not build later stages on an unverified assumption. Checks run locally and must not print raw rows.

| # | Question | Check in | How to check | Answer |
|---|---|---|---|---|
| Q1 | How are missing values encoded in `atd.txt`? (empty, `NaN`, `-1`, …) | S1 | Count distinct non-numeric tokens locally | _todo_ (`DATA.md` says `NaN`) |
| Q2 | Does the TXT have exactly 199,319 lines, frames contiguous 1…199319? | S1 | Count lines, check diffs of frame column | _todo_ |
| Q3 | Frame offset between tracking, homography, tagging and video? | — | Settled by `DATA.md` | **Closed.** `video_frame = frame_id - 1`. Homography `frame` is already the video frame. Smart Tagging: `video_frame = round(seconds * 25)` |
| Q4 | Homography direction and pitch units | S2, only if video is used | Project pitch corners and centre spot | _todo_ (`DATA.md` says normalised pitch → image pixels) |
| Q5 | Video frame count and fps | — | Settled by `DATA.md`. Run `ffprobe` only if the video is used | **Closed.** 199,319 frames at 25 fps |
| Q6 | Where is half-time? | — | Settled by `DATA.md` | **Closed.** 1st half 11,185–90,651, halftime 90,652–110,807, 2nd half graded 110,808–190,197, 2nd-half kickoff at 113,742 |
| Q7 | Do teams switch sides in the 2nd half in the data (coordinates not normalised per half)? | S2 | Mean x of each team's deepest player per period | _todo_ |
| Q8 | Track identity: does each slot follow one player, and can a swap cross teams? | S1 | Per-slot coverage over time, position jumps, team-implausible jumps | _todo_ (`DATA.md` says slots are fragments and swap identity. Cross-team swaps are unconfirmed) |
| Q9 | Are unseen players NaN, or interpolated/extrapolated? | S1 | % NaN per frame, summary only | _todo_ |
| Q10 | Is the ball speed field reliable, and how do aerial balls look (jumps, spikes)? | S3 | Compare with speed computed from positions, summary only | _todo_ |
| Q11 | Are referees present in the data? | S1 | Count of slots active per frame, per team | _todo_ |
| Q12 | Is the speed column in m/s or km/h? | S1 | Compare with speed computed from positions | _todo_ (`DATA.md` says m/s, the broadcast XML says km/h) |
| Q13 | May aggregate statistics about the match be shared with a hosted LLM? | Setup | Ask the organisers | _todo_ (local-only until answered) |
