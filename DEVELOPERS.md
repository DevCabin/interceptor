# DEVELOPERS.md — INTERCEPTOR: Driver's Seat

Welcome aboard. This is the document I wish someone had handed me on day one. Read it top to bottom once, then keep it open in a tab while you work in the code. Everything in here matches `index.html` as it ships today — if the code and this doc ever disagree, the code is right and this doc needs a PR.

---

## The 60-second version

**INTERCEPTOR** is a mobile-first, landscape browser driving game — a loving homage to the 1983 arcade classic *Spy Hunter* — rendered as a pseudo-3D endless highway. You drive the G-6155 Interceptor, an armed supercar. Enemy cars try to wreck you, colored pursuers hunt you in your rearview mirror, a supply truck drops weapons pickups, and a friendly weapons van lets you dock for resupply.

The whole game is **one file**: `index.html`, ~2,290 lines, hand-written HTML/CSS/JS with **zero dependencies, no build step, no bundler, no framework**. You can serve it with any static file server and it runs. That's a deliberate constraint, not laziness — see Golden Rules below.

Target device: a phone held sideways. Desktop works too (keyboard + mouse), and we develop/test there, but mobile landscape is the product.

---

## Golden rules

1. **One file, no dependencies.** Do not add a library, a build tool, a package.json, or a second source file without a conversation about why the constraint no longer serves us. The whole asset budget is generated at boot (sprites) or synthesized at runtime (audio).
2. **Mobile landscape is the product.** Portrait shows a rotate-overlay by design. Every UI decision is thumb-first. If a feature doesn't work with two thumbs on a phone, it doesn't ship.
3. **The fixed timestep is sacred.** Game logic advances in fixed 1/60s steps with a render interpolation factor. Never multiply gameplay logic by raw `requestAnimationFrame` deltas.
4. **Readable text goes through the text queue.** World art lives on the pixel buffer; legible UI text is drawn crisp at full resolution afterward. Don't `fillText` directly onto the buffer for anything a player needs to read. (Details below — this one bites everyone once.)
5. **Test hooks are part of the codebase.** `window.__test` at the bottom of the file is how we drive automated verification with Playwright. When you add a system, extend the hooks. Don't delete them.

---

## Repo layout

```
index.html      The entire game: markup, CSS, and all JS in one <script>
README.md       Player-facing readme (controls, systems, techniques)
DEVELOPERS.md   You are here
```

That's it. If you're looking for the rest of the project, there isn't any.

---

## Running it

```bash
cd <project dir>
python3 -m http.server 8901
# open http://localhost:8901
```

Any static server works (`npx serve`, nginx, GitHub Pages, Vercel static). There is no build. What you save is what runs. Use the browser devtools device toolbar in a landscape phone emulation for a decent approximation of the real experience, but check a real phone before you call touch work done.

---

## Architecture tour

The `<script>` in `index.html` is organized into clearly bannered sections, in this order. Line numbers are approximate and will drift — search for the banner comment, not the number.

| Approx. line | Section | What lives there |
|---|---|---|
| 60 | Config | `CFG` — every tuning constant in the game |
| 92 | Utils | `clamp`, `lerp`, easing, RNG helpers |
| 108 | Audio | `AudioSys` — Web Audio synth: engine, guns, explosions, music |
| 238 | Sprite factory | All car/van/truck/pickup art, generated into offscreen canvases at boot |
| 497 | Track | `buildTrack()` — the road's curvature/elevation data (currently straight — see Roadmap hooks) |
| 549 | Game state | `G` — the single global state object, and `resetGame()` |
| 599 | Storage | High score + mute persistence via localStorage |
| 614 | Input | Touch joystick, buttons, keyboard, pause-on-hide |
| 764 | Weapons | Fire functions: guns, missiles, seekers, oil, smoke, boost |
| 874 | Spawning | Traffic, enemies, trucks, pursuers, pickups |
| 956 | Van docking | The weapons van resupply sequence |
| 977 | Update | `update(dt)` — the entire simulation |
| 1324 | Rendering | Road renderer, world drawing, sprites, effects |
| 1650 | Mirror view | The rearview mirror mini-renderer |
| 1752 | Cockpit | Dashboard, A-pillars, steering wheel |
| 1871 | Crisp text queue | `qtext()` / `flushText()` — read this section before touching any text |
| 1903 | HUD | Score, MPH, weapons chips, announcements |
| 2024 | Joystick overlay | Visual thumb-stick feedback |
| 2058 | Overlays | Title, pause, game over, rotate-device screens |
| 2156 | Frame render | `render(alpha)` — composes buffer → visible canvas |
| 2218 | Main loop | Fixed-timestep accumulator, adaptive quality, resize |
| 2259 | Boot | Init, `window.__test` hooks, first frame |

Two objects to memorize before anything else:

**`CFG`** — every knob in the game: road geometry (`segLen`, `roadHalfW`, `laneW`, `drawDist`), camera (`camHeight`, `fov`), player physics (`maxSpeed`, `accel`, `brakePow`, `steerSpeed`, `centrifugal`), weapons (`gunCooldown`, `gunRange`, `enemyHp`, ammo counts), boost (`boostSpeed`, `boostDuration`, `boostCap`), and pacing (`chaserMax`, `vanInterval`). If you're tuning feel, you're editing this object. The comments on each line tell you the unit (meters, m/s, m/s², seconds).

**`G`** — the entire live game state: `state` (`"title" | "play" | "pause" | "over"`), position/speed (`playerZ`, `playerX`, `speed`), resources (`armor`, `score`, `boost`, `weapons`), and the entity lists (`cars[]`, `chasers[]`, `missiles[]`, `slicks[]`, `particles[]`, `pickups[]`). There is no hidden state outside `G` except input and audio. When a bug report says "the game did X," your first question is "what did `G` look like" — and `__test` lets you answer that from a script.

---

## Core concepts, in the order you'll trip over them

### 1. Fixed timestep with render interpolation (main loop, ~2218)

Logic runs at exactly 60 Hz via an accumulator: real frame time accumulates into `acc`, we consume it in `STEP = 1/60` chunks calling `update(STEP)`, and we render with `alpha = clamp(acc / STEP, 0, 1)`. Entities snapshot their previous position (`pz`/`px`) each step, and the renderer lerps between previous and current by `alpha`. That's why motion is butter-smooth on a 120 Hz phone while the simulation stays deterministic.

**Rule:** anything that changes gameplay goes in `update(dt)` and uses `dt`. Anything that only *draws* goes in `render(alpha)` and uses the interpolated positions.

### 2. The pixel pipeline + the crisp text queue (rendering ~1324, text queue ~1871)

The world is drawn into a small offscreen canvas (`buf`) at roughly 380 px wide (`PIX` is computed in `resize()` and clamped 2–4), then blitted up to the visible canvas with `imageSmoothingEnabled = false` for hard pixel edges, plus a CRT scanline overlay. That pipeline is the entire 8–16-bit aesthetic.

The catch: text drawn at 380 px wide and upscaled is mush. So all *legible* text is captured by `qtext(str, x, y, size, opts)` into a queue during the world/HUD pass, then `flushText()` replays it onto the visible canvas at full resolution after the blit (coordinates are multiplied by the buffer→screen scale). Deliberate exceptions: pickup glyphs (S/B/O/M/+) and the G-6155 door decal are *supposed* to be pixel art and stay on the buffer.

**If your text looks blurry, you drew it on the buffer. If your pixel art looks wrong, you queued it.** That's the whole debugging story for this system.

### 3. The pseudo-3D road renderer (rendering ~1324)

Classic outrun-style projection. The road is a list of segments each `segLen` (10 m) long. For each segment we project its two endpoints into screen space using the camera height, FOV-derived `camDepth`, and the player's Z position, then draw quads far-to-near (painter's algorithm) with a `maxy` backface cull. Lateral position is just an X offset scaled by depth — there is no real 3D math, and that's why it runs at 60 fps on a phone.

One subtlety you'll find if you stare at the code: **the near-road fill.** The nearest valid projection anchor sits 0–10 m in front of the car and wraps as you cross segment boundaries, which made the strip of road right in front of the windshield visibly "snap" several times a second. The fix extrapolates linearly in scale space from the two nearest valid anchors (`pts[1]`/`pts[2]`) and fills solid asphalt and rumble down to the bottom of the screen. If you touch the projection code, keep that fill working — the comment above it explains the invariant.

Speed boost also kicks the camera here: `zoom = 1 - 0.16 * boostEase()` tightens the FOV while boosting, which is most of the *feeling* of speed.

### 4. The rearview mirror is a second renderer (mirror view ~1650)

The mirror is not a camera trick — it's an independent miniature projection that maps "meters behind the player" (`chaser.dist`) to mirror Y and scale, with its own horizon, rumble strips, and lane dashes. Pursuer sprites come from per-tier front-view variants (`Sprites["chaserF" + tierIdx]`) generated in the sprite factory. When you add anything the player should see coming from behind, it goes in this renderer, not the main one.

### 5. Input, and the mobile bug that taught us `touch-action` (input ~614)

Input is two-thumb: left side of screen is a floating joystick (steer; push down to brake; **push up past a threshold to fire the boost** — `input.joyBoost` is an edge trigger consumed in `update`), right side has FIRE plus the MSL/SMK/OIL weapon buttons and pause. Keyboard mirrors everything (arrows/WASD, space, 1–4, P/Esc, ↑/W for boost).

The CSS on the canvas sets `touch-action: none` and we handle pointer events ourselves. If you ever see the page scroll/zoom fighting the joystick on a real phone, that's the property you broke. Test touch on real hardware.

### 6. Audio is synthesized, not sampled (audio ~108)

`AudioSys` builds everything from the Web Audio API at runtime: engine (oscillator pitch tracks speed), gunfire, explosions, pickups, and a chiptune music loop. There are no audio files. It must be resumed from a user gesture (mobile autoplay policy) — that's why `resume()` is called on every pointer/key handler. Mute persists via localStorage.

### 7. Entities and systems

Everything moving is a plain object in a list on `G`:

- **`cars[]`** — civilian traffic (`civ`, harmless, you crash into them), `enemy` (hostile, `hp: CFG.enemyHp + tier.hpBonus`), `truck` (supply truck, drops pickups), `van` (the weapons van).
- **The van is friendly.** `hp: 999`, guns skip it in `findGunTarget`, missiles and splash skip it, heat-seekers only lock `type === "enemy"`. A player shooting the van is a player having a bad day by design. Don't "fix" this.
- **`chasers[]`** — pursuers in the mirror. See the tier system below.
- **`missiles[]`, `slicks[]`, `particles[]`, `pickups[]`** — exactly what they sound like. Pickups cycle through `DROP_CYCLE = ["smoke", "boost", "oil", "missiles", "armor"]` so a countermeasure is always coming.

**Pursuer tiers (`CHASER_TIERS`):** one new color per sector, escalating: CRIMSON → EMBER → HONEY → STEEL → VIOLET → BLACKOUT. Each tier carries `oilHp` (slicks needed to spin it out — armored pursuers *stagger* on early slicks), `smokeRes` (0–0.65 fraction of smoke they shrug off), `speedBonus`, and `hpBonus`. Per-tier sprites are generated once at boot (`Sprites["chaserF"+i]` front, `Sprites["enemyR"+i]` rear), and `spriteFor()` resolves variants generically — new tiers need a table entry and a sprite color, nothing else.

**Pursuit physics:** a chaser has its own speed and weak brakes; it wants to sit ~30 m behind you, capped near ~165–175 MPH. If you brake-check hard, the clamp releases and the pursuer overshoots — at `dist < -4` it converts into an ordinary enemy car ahead of you ("PURSUER OVERRAN"). That conversion is why the speed boost is a real escape tool: boosting to ~235 MPH outruns every pursuer's cap.

**Smoke and oil:** smoke is charge-based (0–3); a caught pursuer gets a persistent `blindT` timer, a shove back, and tumbles until it loses you (+50 past 330 m) — scaled down by its tier's `smokeRes`. Oil decrements a pursuer's `oilHp`; at zero it spins out (+200). These resist curves are the entire difficulty ramp, so tune them before touching raw speeds.

### 8. State, pause, and persistence

`G.state` gates everything; `update()` early-outs for title/pause/over. The tab hiding itself auto-pauses (`visibilitychange`). High score and mute live in localStorage — nothing else is persisted, on purpose.

---

## Testing: `window.__test` + Playwright

The boot section exposes `window.__test` with handles a script can drive:

```js
{
  G,                    // live state
  start,                // startRun()
  spawnChaser(dist),    // pursuer n meters behind
  spawnEnemy(dz),       // enemy ahead
  spawnTruck(dz),
  giveAll(),            // full loadout: missiles 6, seekers 8, oil 4, smoke 3, docks 4
  setArmor(v),
  smoke, oil, missile, boost,  // the fire functions
  pickups()
}
```

Workflow we use (the `playwright-cli` skill's `run-code --raw` against a locally served copy):

1. Serve the folder (`python3 -m http.server 8901`).
2. `__test.start()`, spawn the scenario, call the fire function.
3. Step assertions by polling `__test.G` — e.g. chaser `dist` growing after smoke, `hp` dropping after missile impact, speed reaching boost target.
4. Screenshot for the visual pass.
5. **Console errors fail the test, full stop.** We ship at ~61 fps with a clean console.

Add a hook whenever you add a system you'd otherwise have to test by hand for five minutes.

---

## Performance budget

60 fps on a mid-range phone in landscape is the bar. The mechanisms that keep us there, in order of how much they cost us in elegance:

- **Adaptive quality:** the main loop tracks average frame time and steps `perfTier` down (draw distance 150 → 110 → 85 segments) if we exceed ~22 ms. Never remove this to chase a visual.
- **Everything pre-rendered:** all sprites are drawn once at boot. Per-frame canvas work is blits and fills only.
- **The small buffer:** we rasterize ~380 px wide and upscale. Fill-rate problems mostly don't exist at that size.
- **Far-field simplification:** past ~90 segments the ground/rumble go solid and lane dashes stop, with a fog ramp to hide it. Distant cars get a bottom-anchored scale boost so they read without extra segments.

If you add per-frame allocation (new arrays/objects in the hot loop), expect frame spikes on garbage collection. Reuse objects.

---

## Conventions

- **Units are SI in the sim** (meters, m/s), converted to MPH only for display.
- **Tuning lives in `CFG`, never inline.** If you write a magic number in a system, move it up.
- **Announce, don't surprise.** Player-facing events (new pursuer tier, armored pursuer, overshoot, boost availability) go through the announcement system so the player learns the game from the game.
- **Comments explain *why*.** The code says what; the comment says why the invariant exists (see the near-road fill).
- **`README.md` is player-facing; this file is dev-facing.** Update both when a system changes.

---

## Day-one recipes

**Add a new supply-truck drop kind**
1. Add the kind string to `DROP_CYCLE` (spawning ~874).
2. Handle it in the pickup-collect branch (same section — follow the `kind === "smoke"` / `"boost"` chain).
3. Give it a glyph + color in the pickup sprite drawing (sprite factory) — pickups stay pixelated by design.
4. If it's a weapon: fire function in weapons (~764), ammo on `G.weapons`, button in `computeButtons`, count rendering in the HUD chips, and a `__test` hook.
5. Update README's drop-rotation line and this doc.

**Add a new pursuer tier**
1. Add an entry to `CHASER_TIERS` (name, colors, `oilHp`, `smokeRes`, `speedBonus`, `hpBonus`).
2. The sprite factory loops the table — your front/rear sprites generate automatically. Check the mirror on a real phone: if the color reads as mud at that scale, pick a punchier palette entry.
3. `tierHint()` composes the level-up announcement — make sure your tier's hint is accurate ("needs 2 slicks" etc.).
4. Extend the automated tier test: spawn it, hit it with smoke/oil, assert the resist math.

**Tune difficulty**
Resist touching raw speeds first. The levers, in order of blast radius: `smokeRes`/`oilHp` per tier → `speedBonus`/`hpBonus` → `CFG.chaserMax` and spawn rates → and only then `CFG.maxSpeed`/`accel`. Feel lives in `accel`/`brakePow`/`steerSpeed` — change those in 10% steps and drive for a minute each.

---

## Gotchas (each of these cost real time)

- **The bumper clamp:** chasers only lock to your speed when very close *and* matched in speed (`dist < 1.5 && |Δspeed| < 2`). A naive "clamp whenever close" makes brake-check escapes impossible. Don't simplify it.
- **Smoke blindness must be persistent:** a chaser blinded at 99 m who crosses 100 m must *stay* blind — that's the `blindT` timer. A distance-only check lets pursuers re-engage through the cloud.
- **Edge-trigger the boost:** `input.joyBoost` fires once per stick push, tracked with `G.boostHeld`. Level-triggered, it burns both charges in one push.
- **Offroad speed cap will fake out your tests:** pinning the player off-road caps speed at `offroadMax` (58 MPH). If your boost test tops out at 58, you're in the grass, not in a bug.
- **Weapon buttons get counts; pause doesn't.** The two-line label/count render is guarded (`hasCnt`) — don't let counts leak onto every small button.
- **The van dock timing test:** the van passes you quickly; any test that docks it needs tight polling (~0.5 s) or it'll be gone before your assertion runs.
- **Text queue ordering:** `flushText()` runs once in `render()` after the blit. Queueing text after that point draws it next frame — fine for a frame, weird if you queue every frame conditionally.

---

## Roadmap hooks

- **`buildTrack()` (track ~497) currently produces a straight road** — the curvature/elevation scaffolding is in place and zeroed. Adding curves means filling in the track data and letting the existing projection (including `centrifugal` force in the update) do its job. This is the largest planned feature; everything else is additive.
- Difficulty past the BLACKOUT tier: the table extends cleanly; the design question is what escalation means after max resists (variants? pairs?).
- No plans for a build step, multiplayer, or accounts. If you find yourself wanting them, that's a project-level conversation, not a PR.

---

## Where to start

1. Run it. Play three runs on your phone.
2. Read `CFG` and `G` until you can recite what each field does.
3. Read `update()` top to bottom — it's the whole game in one function.
4. Read the renderer with this doc open at the road section.
5. Pick a recipe above and ship it.

Welcome to the team. The Interceptor is fueled and the road is straight — for now.
