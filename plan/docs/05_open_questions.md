# 05 — Open Questions (verify on real data, then write answers here)

Each item must be checked in the named stage and its answer recorded below it. Do not build later stages on an unverified assumption.

| # | Question | Check in | How to check | Answer |
|---|---|---|---|---|
| Q1 | How are missing values encoded in `atd.txt`? (empty, `NaN`, `-1`, …) | S1 | `head -n 30`, count distinct non-numeric tokens | _todo_ |
| Q2 | Does the TXT have exactly 164,488 lines, frames contiguous 1…164488? | S1 | count lines, check diffs of frame column | _todo_ |
| Q3 | Frame offset: tracking frame 1 ↔ homography frame 0 or 1? ↔ video frame? | S2 | project players onto video frames at both offsets; pick the one that lines up | _todo_ |
| Q4 | Homography direction (pitch→image or image→pitch) and pitch units (metres vs normalised, origin corner) | S2 | project pitch corners/centre spot; check they land on the lines | _todo_ |
| Q5 | Video frame count and fps match 164,488 @ 25? | S2 | `ffprobe` | _todo_ |
| Q6 | Where is half-time? (not in XML) | S2 | long run of invalid homography / missing data mid-file + second kick-off (ball on centre spot, players in own halves) | _todo_ |
| Q7 | Do teams switch sides in the 2nd half in the data (coordinates not normalised per half)? | S2 | mean x of each team's GK per period | _todo_ |
| Q8 | Track identity: does each track follow one player all match, or are tracks fragments reused? | S1 | per-track coverage over time, jumps in position | _todo_ |
| Q9 | Are off-screen players NaN, or interpolated/extrapolated? | S1 | % NaN per frame vs camera zoom | _todo_ |
| Q10 | Is the ball speed field reliable, and how do aerial balls look (jumps, spikes)? | S3 | inspect 5 known long balls against video | _todo_ |
| Q11 | Are referees present in the data? | S1 | count of tracks active per frame (>22 regularly?) | _todo_ |
