# Changelog

All notable changes to INTERCEPTOR are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project
uses [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [0.1.0] — 2026-11-09

First public release. The Interceptor leaves the lab.

### Added

**Game**
- First-person, mobile-first driving game inspired by the 1983 arcade classic *Spy Hunter*:
  an endless desert highway at dusk, rendered pseudo-3D from the driver's seat.
- Player vehicle the G-6155 Interceptor with armor, scoring, and sector escalation
  (difficulty steps up every 3.5 km).
- Traffic, hostiles, and a friendly weapons van (`hp: 999` — your guns and missiles
  deliberately ignore it).

**Rearview-mirror pursuit system**
- Independent rearview-mirror renderer drawn to its own offscreen canvas each frame:
  pursuers, headlights, smoke clouds, oil slicks, muzzle flashes.
- Pursuer tiers, one new color per sector — CRIMSON → EMBER → HONEY → STEEL → VIOLET →
  BLACKOUT — each with its own `oilHp`, `smokeRes`, `speedBonus`, `hpBonus`, and
  per-tier front/rear sprites generated at boot.
- Brake-check escapes: pursuers have weak brakes (~14 m/s² vs the Interceptor's 58 m/s²),
  so a hard stop makes a glued-on tail overshoot and pop out ahead as a shootable
  enemy car ("PURSUER OVERRAN").

**Weapons and countermeasures**
- Unlimited machine guns plus optional heat-seeking missiles that lock the nearest
  hostile ahead.
- Missiles with splash damage.
- Smoke screen (max 3 charges, 7 s cooldown) with a persistent `blindT` timer, so a
  pursuer blinded inside 100 m stays blinded after crossing back out.
- Oil slick, which decrements a pursuer's `oilHp` (armored pursuers stagger and need
  multiple slicks).
- Speed boost (2 charges, 105 m/s ≈ 235 MPH for 4.5 s) with FOV kick and speed lines —
  outruns every pursuer's speed cap. Edge-triggered on stick push.
- Weapons-van docking upgrade sequence: missiles → smoke → oil → heat-seekers →
  resupply.

**Supply trucks**
- Green supply trucks periodically cross the highway; destroying one leaves a glowing
  pickup. Drops rotate deterministically through
  `smoke → boost → oil → missiles → armor`, so a countermeasure is always coming.

**Presentation**
- Full 8–16-bit aesthetic: low-resolution render buffer with hard-edged nearest-neighbour
  upscale, banded sky, procedural sprites, CRT scanlines.
- All sprites generated procedurally at boot and all audio synthesized with Web Audio
  (engine tied to speed, gunfire, explosions, lock-on, docking chimes, bass sequencer).
  No image files, no audio files, no external assets. Autoplay-safe.
- Synthesized chiptune music loop.
- In-game announcement system for tier changes, armored pursuers, overshoots, and
  boost availability.

**Input**
- Analog left-thumb joystick that anchors where the thumb lands (steer; pull down to
  brake; push up to boost).
- Multi-touch right-hand buttons with per-button tracking, plus full keyboard parity.
- Portrait rotate-device screen, auto-pause on tab hide.

**Persistence**
- High score and mute preference via `localStorage`.

**Technical foundations**
- Fixed 1/60 s timestep decoupled from rendering, with render interpolation of the
  camera and every entity for smooth motion on 90/120 Hz displays.
- Pseudo-3D segment projector with painter's algorithm, back-face culling, distance
  haze, and the near-road fill invariant.
- Object pools for particles; adaptive quality that lowers draw distance
  (150 → 110 → 85 segments) if frame time exceeds ~22 ms.
- `window.__test` automation hooks for scripted verification.

**Documentation**
- `README.md` — player-facing: controls, scoring, systems, pursuer tier table, techniques.
- `DEVELOPERS.md` — developer-facing: architecture tour, `CFG`/`G` reference, golden
  rules, day-one recipes, gotchas, performance budget, roadmap hooks.
- `CHANGELOG.md` — this file.
- `cline_instructions.md` — lab assistant operating protocol.

### Verified

Smoke-tested in headless Chrome (900×480) against a local static server:

| Check | Result |
|---|---|
| Console / exceptions | clean (0 errors, 0 warnings) |
| Frame rate during gameplay | 61 fps |
| `__test` hooks | present and driving |
| Smoke screen vs. pursuer at 77.9 m | shoved back 77.9 → 308.6 m, then lost (+50) |
| Smoke screen vs. pursuer beyond 100 m | correctly **not** caught (by design) |
| Speed boost | charge consumed 2 → 1, 235 MPH peak, matching `CFG.boostSpeed` |
| Oil slick / missile / truck spawn | ammo decrements and entities spawn correctly |

### Known limitations

- **The road is straight.** `buildTrack()` ships with curvature and elevation scaffolding
  in place but zeroed. Adding curves is the largest planned feature — the projection and
  the `centrifugal` force in `update()` are already wired for it. See
  *Roadmap hooks* in `DEVELOPERS.md`.
- Pursuer tiers stop escalating visually after BLACKOUT (the tier table extends cleanly;
  the open design question is what escalation means after max resists).
- On-ramp/exit ramps are not implemented.
- No build step, no multiplayer, no accounts — by design, not by omission.

[0.1.0]: https://github.com/DevCabin/interceptor/releases/tag/v0.1.0
