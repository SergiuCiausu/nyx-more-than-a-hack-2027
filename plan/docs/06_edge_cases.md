# 06 — Edge Cases

Cases already covered in `03_definitions.md`: deflected pass, clearance, second ball after a header, airborne-ball artefacts (no z), short tracking gap ≠ dead ball, offside vs foul not distinguishable, half boundaries (Q6) and frame offset (Q3), both settled by `DATA.md`.

Below are 50 more. Each has the **risk** (what goes wrong if ignored), the **handling**, and the **stage** that owns it. For each one, the owning stage must either implement the handling or log it as a known limitation in its report. Add a synthetic test wherever the handling is a rule.

**Scope:** cases owned only by S6 are deferred with that stage. Section B applies only if the broadcast feed is used. No case may be handled by hand on this match.

**New event types introduced here** (add to `codes.py` and the events schema only if S6 is built): `shot`, `goal`, `save`, `throw_in`, `aerial_duel`, `block`, `touch` (one-touch pass/flick). New ball-state reason: `unknown` (live play not observable).

---

## A. Tracking data quality

| # | Case | Risk | Handling | Stage |
|---|---|---|---|---|
| 1 | Ball off-screen for several seconds during live play (e.g. long switch to the far side, camera lags) | Called `dead`, or possession flips randomly | Keep state live; label frames `unknown` (state reason) and carry the last team; only end with a dead signal or the next observed control | S3/S5 |
| 2 | Ball position frozen (tracker repeats last value) while players keep moving | Fake stationary ball → false dead ball / fake control by whoever walks past | Detect runs of identical ball coordinates while median player speed > walking; treat as missing | S1/S3 |
| 3 | Ball "teleports" (tracker jumps to another ball-like object: head, spare ball, penalty spot) | Huge fake speed, broken segments | Reject frames where implied speed > `ball.max_kmh` (≈ 150) and the next frames don't continue the new path; interpolate if gap ≤ 5 frames | S1/S3 |
| 4 | A second (spare) ball appears near the touchline or is thrown on | Two plausible ball positions; restart detection confused | Trust continuity of the in-play trajectory; ignore ball positions outside the pitch that don't connect to the previous path | S3/S5 |
| 5 | Track ID swap: two close players exchange IDs | A Barcelona player's track suddenly sits with Athletic → wrong team on control | Detect team-implausible tracks (track's x-distribution / cluster changes); split into fragments; assign team by majority of fragment; report count | S1 |
| 6 | Same real player split across several track IDs | Player-level stats fragmented | Work at team level by default; player-level only per track fragment, documented | S1/S6 |
| 7 | Referees / linesmen in the data, or the ball hitting the referee | Referee becomes "nearest player", fake control or interception | Identify referee tracks (if present: outside team ids, or a third cluster); exclude from control; ball contact with referee keeps the segment going (play continues) | S1/S3 |
| 8 | Off-screen players are NaN, so the true nearest player is invisible | Visible player wrongly credited with control | Require `dmin` < threshold (not just argmin); if ball is near the edge of the camera view (via H) and no one is close, state `unknown` not `loose` | S3 |
| 9 | Substitutions (new tracks appear, old ones disappear) | Code assuming 11 per team fails | Never assume fixed counts; count active tracks per team per frame | S1 |
| 10 | Red card: a team plays with 10 | Same as above; pressure features shift | No fixed-count assumptions; report active players per team over time | S1 |
| 11 | GK replaced or an outfield player goes in goal; XML GK flag wrong for that player | Hand-handling rules (GK holding ball) applied to wrong track | Derive GK role from position (deepest player of team, inside own box most of the time) per period, not only from XML | S1/S2 |
| 12 | Player speed spikes from tracking jitter | Fake tackles (closing speed) and fake control mismatches | Smooth positions; recompute speed from smoothed positions; cap at `player.max_kmh` (≈ 37) | S1/S3 |

## B. Broadcast / camera (only if the broadcast feed is used)

| # | Case | Risk | Handling | Stage |
|---|---|---|---|---|
| 13 | Replay shown while live play continues (broadcast misses live action) | Live play gap looks like a stoppage | Gap + cut alone is not dead (needs ≥ 2 signals); use `unknown`; check where the ball/players are when live feed returns | S2/S5 |
| 14 | Replay frames tracked as if live (valid homography, real-looking motion) | The same action counted twice, time jumps backwards | Detect replay transitions (logo wipes, frame-difference spikes) and repeated trajectories; verify whether Metrica already excluded replays; mark `camera_cut` | S2 |
| 15 | Picture-in-picture / split screen / large graphics | Partial or wrong homography, ball occluded | Flag frames with high `reprojection_error`; treat positions as low confidence | S2 |
| 16 | Fast pan or zoom → homography error spikes | Position jumps → fake speeds/kicks | Threshold `reprojection_error` (`align.max_reproj_err`); smooth across; don't start segments on a single-frame jump | S2/S4 |
| 17 | Pre-match ceremony, half-time studio, post-match in the video | Non-match frames analysed | `period = 0` outside kick-off…final whistle; exclude everything | S2 |
| 18 | Variable frame rate / dropped frames in the MP4 | Tracking ↔ video drift over the match | Check pts with ffprobe; verify alignment at start, middle and end of each half, not just once | S2 |

## C. Ball control and possession

| # | Case | Risk | Handling | Stage |
|---|---|---|---|---|
| 19 | Shielding: attacker and defender equally close to a slow ball | Control flickers between teams | Hysteresis keeps the previous controller; if both within `control.max_dist_m` for long, treat as loose, so possession stays with the last team | S3 |
| 20 | Dribbler knocks the ball 3–5 m ahead and chases it | Control ends → fake pass/loose segment | Bridge gaps where the same player is the closest player and moving toward the ball at end of the gap (`control.knock_on_m`, ≈ 5 m, ≤ 1 s) | S3 |
| 21 | Dummy / step-over: ball rolls through a player's legs to a teammate | Dummying player credited with a touch | Proximity shorter than `min_frames` and no speed/direction change → no touch | S3/S4 |
| 22 | One-touch / first-time passes (control < 6 frames) | Hysteresis merges A→B→C into one pass A→C | Detect touches inside a segment: kick spike or direction change > 30° while a player is within 1 m → split segment, emit `touch` event | S4 |
| 23 | Player lets the ball run out for a goal kick while shielding | Counted as loose / possession loss | Live: controlling team keeps it until out; restart then decides | S5 |
| 24 | GK holding the ball in hands (ball speed ~0 for up to 6 s) | Stationary ball → dead ball | Stationary ball with GK of a team within 1 m inside own box → `controlled`, not dead | S5 |
| 25 | GK punt / drop-kick / throw | Release from hands may show no clean "kick spike" from feet level | Treat release from GK-in-hands control as a pass even with lower kick threshold; throws = short pass | S4 |
| 26 | Ball off the post or crossbar | Sharp direction change with nobody near → misread as deflection by a player | Direction change at goal frame coordinates (x≈0/105, y≈30.3–37.7) with no player near → segment continues as loose, flag `woodwork` | S4 |
| 27 | Ball in a crowded box (corners, scrambles) with 4+ players within 2 m | Control flips many times in a second | Raise `min_frames` locally or label `loose` while ≥ `n` players of both teams within 2 m; evaluate separately on the other-match data in S7 | S3/S7 |
| 28 | Advantage played after a foul | Players slow briefly → false dead | Dead requires ball also stationary/out or a cut; mass deceleration alone isn't enough | S5 |
| 29 | Fair play: ball kicked out for an injury, then returned to the other team | Restart team ≠ "expected" team; long dead stretch | Normal handling (restart decides); flag throw-ins where the thrower's team immediately kicks it to the opponent's GK | S5/S6 |

## D. Passes, shots and goals

| # | Case | Risk | Handling | Stage |
|---|---|---|---|---|
| 30 | **Shot vs pass**: a shot looks like a fast long pass | Shots counted as failed passes | New `shot` type: kick whose direction line crosses the goal mouth (± margin) within the attacking third; ends at goal line, GK save or block | S4/S6 |
| 31 | **Goal**: no goal events in the data | Possession/state after a goal wrong; no score | Ball crosses goal line between posts (y 30.34–37.66 m, ~2D) followed by kick-off at centre spot within ~2 min → `goal` | S6 |
| 32 | Ball crosses the goal line *over* the bar (no z) | Counted as a goal | Require the kick-off confirmation for a goal; otherwise it's a goal kick / corner by restart | S6 |
| 33 | Own goal / deflected shot into own net | Goal credited to wrong team | Credit goal by which goal the ball entered + attack direction, not last touch | S6 |
| 34 | GK save / catch | Looks like interception | Opponent control by GK inside own box after a shot → `save` | S6 |
| 35 | Block (defender's body stops a pass/shot at short range) | Read as tackle or interception | Ball speed drops sharply at a defender within 1 m, direction change, no control → `block`, segment continues loose | S4/S6 |
| 36 | Header / volley pass (aerial touch, no ground control) | No control episode → touch missed | Same as #22: direction/speed change near a player while in flight → `touch` | S4 |
| 37 | Aerial duel (two players jump for a header) | Read as tackle | Ball in flight (in_flight state) at contact → `aerial_duel`, never `tackle` | S6 |
| 38 | Chest / thigh control of a bouncing ball | Irregular path → loose, then control | Short loose segment ending in control by the receiver within 1 s → merge into the pass as `complete` | S4 |
| 39 | Through ball the receiver runs onto 2–3 s later | Long flight → misread as loose (ball slows, no one near) | Allow long segments if path is straight and decelerating smoothly (rolling) and the eventual receiver was running toward the landing point | S4 |
| 40 | Back-pass to GK | Fine as pass, but GK may then miscontrol | Normal handling; nothing special except keeping GK-in-box logic from #24 off (feet, not hands) | S4 |
| 41 | Pass to a player in an offside position | Pass "complete" then stoppage | If a stoppage starts within ~2 s of reception with the free kick to the opponent near the receiver → outcome `stopped (likely offside)` | S5/S6 |

## E. Restarts and match flow

| # | Case | Risk | Handling | Stage |
|---|---|---|---|---|
| 42 | Quick free kick / short corner taken before players settle | Dead stretch too short to be detected | Allow short dead stretches when the ball was out or stationary at a restart spot; `dead.min_s` low (≈ 1 s) when the out signal is present | S5 |
| 43 | Throw-in: ball enters from outside the pitch from the thrower's hands | Starts a segment out of bounds → filtered as noise | Segment starting outside the touchline with a player within 1 m → `throw_in` restart + pass | S4/S6 |
| 44 | Penalty kick | Treated as normal free kick; positions unusual | Ball stationary on penalty spot (11 m from goal line, centre) with players outside the box → `penalty` restart; following kick = shot | S6 |
| 45 | Dropped ball after a referee stoppage | Restart team ambiguous | Restart type `dropped_ball`, team = first controller | S6 |
| 46 | VAR check: long stoppage; a goal may be disallowed | Goal counted although not given | Goal only if confirmed by kick-off (#31) | S6 |
| 47 | Whistle for half/full time while the ball is in flight | Pass with no end | Segment ends at period end with outcome `stopped` | S5 |
| 48 | Goal celebration (players run to the corner, ball stationary in net) | Players' high speed → no mass deceleration; ball out of pitch | Ball out of pitch + stationary is enough (2 signals); state dead until kick-off | S5 |

## F. Tackles (deferred with S6)

| # | Case | Risk | Handling | Stage |
|---|---|---|---|---|
| 49 | Defender jockeying within 1 m without challenging | Many false tackle candidates | Require closing speed **and** a ball event within the window (speed/direction change or control change); otherwise `pressure`, not tackle | S6 |
| 50 | Tackle on a player who has only just received the ball (control < min_frames) | Attacker not yet "in control" → tackle missed | Use raw nearest-player intent (receiver = nearest after a pass for ≥ 2 frames) as the tackle target, not only hysteresis control | S6 |

Additional tackle notes (not counted above, handle with #49/#50 rules): a slide tackle that wins the ball but sends it out → outcome `won_out` (restart goes to the opponent); repeated tackles in a pressing sequence → merge candidates within `tackle.merge_s`; a 50/50 where both players touch simultaneously → `loose`.

## G. Coordinates and geometry (checks, also in Q4/Q7)

- Normalised y may run top-down: confirm which goal is at x = 0 and which touchline is y = 0 before any goal/corner logic (#26, #31, #44).
- Use the 105.3 × 68.0 m pitch from `games.py` for all line logic, not the 105 × 68 m in the XML metadata.
- Whole ball must cross the line: use ball radius 0.11 m plus `dead.out_margin_m`; the far touchline has the largest projection error.

## New config keys
```yaml
ball:     { max_kmh: 150 }
player:   { max_kmh: 37 }
align:    { max_reproj_err: 1.0 }
control:  { knock_on_m: 5, knock_on_s: 1.0, crowd_n: 4 }
dead:     { min_s: 1.0 }
shot:     { goal_margin_m: 2.0 }
tackle:   { merge_s: 0.5 }
touch:    { min_dir_change_deg: 30 }
```
