# INTERCEPTOR — Driver's Seat

A first-person, mobile-first homage to the 1983 arcade classic **Spy Hunter**, rebuilt
from the driver's point of view. Plain HTML + JS + Canvas in a single file — no
libraries, no build step, no external assets. Full 8–16-bit pixel aesthetic: the whole
game renders into a low-resolution buffer (~427×240) with a hard-edged integer upscale,
banded sky, chunky sprites, and CRT scanlines.

> **Taking over the code?** Start with [DEVELOPERS.md](DEVELOPERS.md) — a full
> walkthrough of the architecture, the systems, and the day-one recipes.

## Premise

You're behind the wheel of the G-6155 Interceptor on a desert highway at dusk.
Hostiles ahead of you can be shot. Hostiles **behind** you — visible only in your
rearview mirror — shoot back. Watch the mirror, manage your countermeasures, and
dock with the weapons van to escalate your arsenal.

## How to run

Open `index.html` in any modern browser, or serve the folder with any static file
server. Designed for **phones in landscape** (a rotate prompt appears in portrait);
fully playable on desktop with a keyboard. The low-res render buffer keeps fill rate
low on mobile GPUs, and the game degrades draw distance automatically if a device
can't hold 60 fps.

## Controls

| Action | Touch | Keyboard |
|---|---|---|
| Steer | Left thumb — drag anywhere on the left half (analog stick appears where you land) | ← → or A / D |
| Brake | Pull the stick down past the brake line | ↓ or S |
| Speed boost | Push the stick up (needs a stocked charge) | ↑ or W |
| Machine guns (unlimited) | FIRE (hold) | Space or Z |
| Missiles / heat-seekers | MSL | X |
| Smoke screen | SMK (count shown) | C |
| Oil slick | OIL (count shown) | V |
| Pause | ❚❚ button | P or Esc |
| Mute | — | M |

## Objective & systems

- **Survive and score.** Enemy kill 100, missile kill 150, oil spin-out 200,
  losing a pursuer 50, van dock 250, supply pickup 50. Hitting civilians
  costs you 300.
- **Green supply trucks are your lifeline.** A garbage truck wanders the
  highway every 20–30 seconds (first one arrives early). Shoot it and it
  leaves a glowing pickup where it died — drive through it. Drops rotate:
  **smoke screen → speed boost → oil slick → missiles +2 → armor +30**, so a
  countermeasure is always coming. This is how you survive pursuers before
  your first van dock.
- **Speed boost** — orange "B" pickup from supply trucks, holds up to 2
  charges. Push ▲ to light it: ~4.5s at 235 MPH with FOV kick and speed
  lines. Pursuers top out around 165-175 MPH, so a lit boost opens real
  daylight — great for escaping a tail you can't shake. The dash shows
  "PUSH ▲ = BOOST ×n" when stocked and "BOOSTING" while lit.
- **The mirror matters.** Pursuers close in from behind and open fire at close
  range (watch for the ▲ TAIL ▲ alert). Outrun them, or:
  - **Brake-check** — pursuers have weak brakes. Slam yours with one on your
    tail and he overshoots, popping out ahead of you as a regular enemy car:
    target locked, guns free. ("PURSUER OVERRAN — take the shot.")
  - **Smoke screen** — max 3 charges. A smoked pursuer is shoved hard back on
    the hit and keeps tumbling back until the smoke clears — usually far
    enough to lose him for good (+50). 7s cooldown between charges.
  - **Oil slick** — slides back along the road; the first pursuer to run over it
    spins out. Swerving after you drop it changes the slick's lane.
- **Pursuer tiers** — every sector brings a new color in the mirror, each
  harder to put down. Sector announcements name the new hostile and its trick:
  | Sector | Color | Behavior |
  |---|---|---|
  | 1 | Crimson | standard |
  | 2 | Ember | faster, 15% smoke-proof |
  | 3 | Honey | 2 slicks to spin out, 25% smoke-proof |
  | 4 | Steel | 2 slicks, 40% smoke-proof, faster |
  | 5 | Violet | 2 slicks, 55% smoke-proof, tougher after overshoot |
  | 6+ | Blackout | 3 slicks, 65% smoke-proof, fastest |
  Armored pursuers shrug a slick off ("ARMORED PURSUER — N more slicks to
  spin him") — they slide wide and drop back, but keep coming. After a
  brake-check overshoot, the car ahead keeps its tier color and extra HP.
- **Weapons van** — appears periodically (VAN distance shown top-right). Pull up
  behind it to dock. It's friendly — your guns and missiles ignore it. Upgrade
  sequence: missiles → smoke screen (3 charges) → oil slick → heat-seeking
  missiles → resupply (ammo + armor).
- **Heat-seekers** lock the nearest hostile ahead; plain missiles fly straight
  with splash damage.
- **Armor** is lost to enemy fire and collisions. At zero, the Interceptor goes
  down. Sectors (difficulty) escalate every 3.5 km.
- High score and mute preference persist in `localStorage` (this browser only).

## Key techniques

- Pseudo-3D road renderer (segment projection with hills, back-face
  culling, distance haze) on a 2D canvas — the classic top-down racer turned
  first-person with a cockpit overlay (dash, working steering wheel, gun ports).
- Pixel world, crisp text: gameplay renders into a low-res buffer with a
  hard-edged integer upscale for the 8–16-bit look, while title-screen text
  draws on the visible canvas at full resolution for legibility.
- Independent rearview-mirror renderer: a second miniature scene drawn to an
  offscreen canvas each frame (chasers with headlights, smoke clouds, oil slicks,
  muzzle flashes).
- All sprites are generated procedurally at boot (no image files); all audio is
  synthesized with Web Audio (engine, gunfire, explosions, lock-on, docking
  chimes, bass sequencer) — autoplay-safe, starting on first user gesture.
- Fixed-timestep simulation (60 Hz) decoupled from rendering, with render
  interpolation (camera and every entity lerped between sim snapshots) so the
  view stays glass-smooth on 90/120 Hz phone displays. Object pools for
  particles, deterministic track layout, adaptive quality (drops draw distance
  if a device can't hold frame rate).
- Multi-touch pointer input with per-button tracking and an analog left-thumb
  joystick (anchors where the thumb lands, pull-down to brake), keyboard
  support, and auto-pause when the tab is hidden.
