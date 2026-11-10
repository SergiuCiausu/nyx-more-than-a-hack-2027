# 03 — Definitions and Conventions

These are football conventions the code must implement. Thresholds are starting values in `config.yaml`.

## Team vs player possession
- **Player control:** a player has the ball when they are the nearest player, within `control.max_dist_m` (1.5 m), the ball is not moving much faster than them (`ball_speed < player_speed + control.max_rel_speed_kmh`, 8 km/h), for at least `control.min_frames` (6 frames ≈ 0.25 s).
- **Team possession:** a team keeps possession until the opponent gains control or the ball goes dead. Time in flight of a pass is credited to the **passer's team**, even if the pass fails (most common convention).
- **Dead time** (ball out, stoppage) is labelled `D` and counts for nobody.
- **Loose-ball time:** always stays with the last team in possession, as the competition defines it.

## Ball states (one per frame)
| Code | State | Possession credited to |
|---|---|---|
| 0 | `controlled` | controlling player's team |
| 1 | `in_flight` | passer's team |
| 2 | `loose` | last team in possession |
| 3 | `dead` | nobody |

## Pass vs loose ball
An uncontrolled stretch (segment) is classified by how it **started**, how the ball **travelled**, and how it **ended**.

**Pass** — all of:
- release from a controlled state with a **kick**: ball speed jumps by > `seg.min_kick_kmh` (15) within 1–3 frames,
- no opponent within `seg.pressure_dist_m` (1.5 m) at release,
- ball path roughly straight: `straightness = |end-start| / path_length > seg.min_straightness` (0.85).

**Long pass**: a pass with distance > `seg.long_pass_m` (25 m).

**Loose ball** — anything else: release under opponent pressure (tackle/block), low-speed miscontrol, irregular path (bounces), multiple short touches by different players.

### Edge cases
- **Deflected pass:** clean pass, then a sharp direction change near an opponent → pass ends at the deflection (outcome `intercepted` if opponent then controls, else segment continues as loose).
- **Clearance:** hard kick under pressure, no teammate near landing → `loose` with flag `clearance`.
- **Second ball after a header:** the pass ends at the header; a new loose segment starts.
- **No ball height:** airborne balls projected onto the ground plane can show position jumps and speed spikes. Do not let these flip a segment to "loose"; a smooth high-speed run ending with a jump back to a plausible landing spot is likely aerial.

## Pass outcomes
| Code | Outcome | Detection |
|---|---|---|
| 0 | `complete` | next control by a teammate |
| 1 | `intercepted` | next control by an opponent; possession flips at that frame, not at the kick |
| 2 | `out` | ball crosses a pitch line (outside 0–105 / 0–68 with `dead.out_margin_m` ≈ 0.5–1 m) |
| 3 | `stopped` | game stops before control (offside, foul, injury) |
| -1 | `none` | not a pass |

A header/touch by an opponent **without control** starts a loose segment; possession stays with the last team until the opponent gains control.

## Out of play and restarts
- Pass ends at the frame the ball crosses the line → ball state `dead`.
- Who restarts is read from the **restart**, not from touches (not visible in tracking):
  - stationary ball on the touchline → **throw-in**
  - ball in a corner arc → **corner**
  - ball in the six-yard box → **goal kick**
  - player standing over the ball decides the team.

## Stoppages (dead ball)
Require **at least 2** of these signals (`dead.min_signals`):
1. ball out of bounds,
2. mass deceleration: most tracked players below walking speed (~7 km/h) simultaneously,
3. ball stationary > 3 s (`dead.stationary_s`),
4. ball and player positions match a restart shape.

Homography is valid on every tactical frame, so it cannot signal a stoppage.

Never call a tracking gap "dead" on its own — the ball is often simply undetected during live play.

Offside vs foul cannot be told from tracking, and the label does not need it, since both are `D`. Never label them by hand on this match.

## Tackles (deferred with S6)
An **event**, built on top of possession. Not part of the scored output.
- **Candidate:** an opponent of the player in control gets within `tackle.contact_m` (≈1 m) while closing at speed (`tackle.min_closing_kmh`).
- **Outcome** over the next `tackle.window_s` (1–2 s):
  - `won` — possession flips to the defender's team
  - `loose` — ball goes loose / out
  - `failed` — attacker keeps control
  - `foul_likely` — ball and players go stationary (stoppage)
- **Not a tackle:** ball arrived at pass speed → interception.
- Rules over-detect in crowded areas (set pieces, box); optional clip classifier refines.
