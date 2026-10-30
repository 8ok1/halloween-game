# Technical Specification: Spooky Night Theme

- **Functional Specification:** `context/spec/002-spooky-night-theme/functional-spec.md`
- **Status:** Draft
- **Author(s):** AWOS

---

## 1. High-Level Technical Approach

The theme is implemented as a **sprite re-skin plus a stack of cosmetic features**, so that no gameplay rule, no public contract from architecture section 3, and no determinism guarantee changes.

- The pumpkin re-skin is a bitmap swap inside `src/sprites.ts`: the two cactus bitmaps become two pumpkin bitmaps of the same width, height and hitbox. `ObstacleKind = 'small' | 'large'` is unchanged, so `obstacles.ts`, `collision.ts` and every test keyed on obstacle kinds still work.
- The night sky, intensification, and event reactions are added as **new feature modules** under `src/features/`, each installed from `main.ts` with the `installX({ game, renderer })` shape already modelled by `features/puddle.ts`. They subscribe to the typed event bus for triggers and draw on the layer that fits their meaning (`background` for sky and moon, `parallax` for fog and world-anchored dust, `world` for pumpkin shards, `hud` for milestone sparks).
- The one "new logic" the spec requires a unit test for lives in a new pure module `src/atmosphere.ts`, which maps `state.tick` to a deterministic atmosphere record (sky colour, fog alpha, cloud target count). The intensification features read it; the test exercises the pure function directly without a renderer.
- New tunable numbers go into `src/tuning.ts` as named `as const` exports. The frozen `PALETTE` keeps its three semantic entries (`bg`, `fg`, `accent`) and gains spooky-only entries (`sky`, `skyDeep`, `moon`, `pumpkin`, `pumpkinShadow`, `stem`, `fog`, `bat`), so the puddle (which still reads `PALETTE.accent`) is untouched.

**Interesting bits deliberately chosen** (so "make it interesting" is a design decision, not a maybe):

- **Fog that rises with the run.** A grey band drifts up from the ground over roughly ninety seconds (same tick budget as the speed ramp), capped at a modest alpha so pumpkins stay readable.
- **Sky that deepens with the run.** The sky lerps from midnight blue to near-black over the same ramp, so a long run looks genuinely different at a glance.
- **Pumpkin shatter on death.** The pumpkin the dinosaur hits bursts into shards that fall for about half a second on the frozen screen (animation-already-in-flight, so the "no new reactions on game-over" rule in the functional spec is honoured).
- **Milestone bats.** Every hundred points, a small flock of silhouette bats flaps across the sky from right to left for about one and a half seconds. The silhouettes alone sell "spooky" harder than a HUD pulse ever would.
- **Dust puffs on jump and land, plus a one-tick screen shiver on landing** — the four reactions are all delivered, exceeding the "at least two" floor the functional spec sets.

---

## 2. Proposed Solution & Implementation Plan (The "How")

### 2.1 File-level changes

| Path | Change | Responsibility |
|---|---|---|
| `src/sprites.ts` | **Edit.** Replace `cactusSmall` and `cactusLarge` bitmaps with `pumpkinSmall` and `pumpkinLarge` bitmaps of identical outer dimensions and identical hitboxes. Rename the exports to `pumpkinSmall` / `pumpkinLarge`? **No** — see the determinism / contract note below. The export names stay `cactusSmall` / `cactusLarge` or whatever the obstacle sprite registry currently keys off; only the pixel data changes. | Pure data, no DOM. |
| `src/tuning.ts` | **Edit.** Extend `PALETTE` with new named entries; add new `as const` exports (listed in 2.3). | Pure data. |
| `src/atmosphere.ts` | **New.** Pure module exporting `atmosphereAt(tick: number): Atmosphere`. | Pure logic. |
| `src/features/nightSky.ts` | **New.** Draws sky gradient, moon, drifting clouds on the `background` layer. | Cosmetic feature. |
| `src/features/fog.ts` | **New.** Draws rising ground fog on the `parallax` layer. | Cosmetic feature. |
| `src/features/dust.ts` | **New.** Spawns short-lived dust puffs on `jump` and `land`; owns the one-tick landing shiver via `renderer.setCamera`. Draws on `parallax`. | Cosmetic feature. |
| `src/features/pumpkinShatter.ts` | **New.** Listens to `die`, captures the last obstacle from `state.obstacles` nearest the dino, spawns falling shards. Draws on `world`. Also drives a brief screen flash on `background`. | Cosmetic feature. |
| `src/features/milestoneBats.ts` | **New.** Listens to `milestone`, spawns a flock that flaps across the sky over ~90 render frames. Draws on `foreground` (above fog, under HUD). | Cosmetic feature. |
| `src/main.ts` | **Edit.** Install the new features alongside `installPuddle`. Each returns `dispose()`, collected and called on teardown as today. Order of installation matters only insofar as draw-layer registration order within a layer is paint order — list sky before fog, dust before shatter. | Browser shell. |
| `src/style.css` | **Edit.** Change the canvas CSS `background-color` (currently near-white) to a dark fallback that matches the sky, so brief first-frame flashes are not white. | Browser shell. |
| `tests/atmosphere.test.ts` | **New.** Unit tests for `atmosphereAt`. Includes a dedicated RED-validation case. | Vitest. |
| `tests/features/*.test.ts` | **New (optional, recommended for shatter and bats).** Drive the install function with a recording `DrawContext` fake, dispatch events via a stub `on()`, assert `fillRect` is called with the expected pixel runs. | Vitest. |

### 2.2 `atmosphere.ts` (the testable new logic)

```ts
export interface Atmosphere {
  readonly intensity: number;      // 0 at tick 0, 1 at FOG_RAMP_TICKS, clamped to [0, 1]
  readonly skyColor: string;       // lerp(PALETTE.sky, PALETTE.skyDeep, intensity), hex
  readonly fogAlpha: number;       // 0 to FOG_MAX_ALPHA
  readonly cloudTarget: number;    // integer in [CLOUDS_MIN, CLOUDS_MAX]
}

export function atmosphereAt(tick: number): Atmosphere;
```

- `intensity = clamp01(tick / FOG_RAMP_TICKS)` where `FOG_RAMP_TICKS` is a new tuning export (default 5400, matching `MAX_SPEED_AT_TICK` so the look lands with the pace).
- `skyColor` interpolates hex channels linearly between `PALETTE.sky` and `PALETTE.skyDeep` using `intensity`.
- `fogAlpha = intensity * FOG_MAX_ALPHA`. `FOG_MAX_ALPHA` default `0.35`.
- `cloudTarget = Math.round(lerp(CLOUDS_MIN, CLOUDS_MAX, intensity))`. Defaults: `CLOUDS_MIN = 2`, `CLOUDS_MAX = 6`.
- The function is pure, synchronous, allocation-light, and depends only on `tick` and `tuning.ts` constants. No events, no RNG, no DOM. This is the module the new unit tests exercise.

### 2.3 New `tuning.ts` exports

| Name | Default | Purpose |
|---|---|---|
| `PALETTE.sky` | `#1b2342` | Early-run sky colour. |
| `PALETTE.skyDeep` | `#070914` | Late-run sky colour. |
| `PALETTE.moon` | `#f3ecd1` | Moon disc. |
| `PALETTE.cloud` | `#2a3658` | Clouds against the sky. |
| `PALETTE.pumpkin` | `#d87f27` | Pumpkin body orange. |
| `PALETTE.pumpkinShadow` | `#8a4a16` | Pumpkin ridges / shadow pixels. |
| `PALETTE.stem` | `#3d6b2e` | Pumpkin stem. |
| `PALETTE.fog` | `#6a7294` | Ground mist colour (alpha applied at draw). |
| `PALETTE.bat` | `#0b0c14` | Bat silhouette, near-black so bats read on any sky colour. |
| `MOON` | `{ x: 680, y: 60, radius: 22 }` | Logical-coord moon position; fixed, as the functional spec requires. |
| `CLOUD` | `{ minY: 20, maxY: 110, minWidth: 36, maxWidth: 90, baseSpeed: 0.25, speedJitter: 0.15 }` | Cloud bounds and drift speed (logical px per tick while running). |
| `CLOUDS_MIN` / `CLOUDS_MAX` | `2` / `6` | Cloud count bounds, interpolated by `intensity`. |
| `CLOUD_FEATURE_SALT` | `0x5c0ud` | XOR salt for the night-sky feature's own RNG (`createRng(state.seed ^ SALT)`). |
| `FOG_RAMP_TICKS` | `5400` | Ticks at which `intensity` reaches 1. |
| `FOG_MAX_ALPHA` | `0.35` | Cap on fog alpha. |
| `FOG` | `{ bandHeight: 56, speed: 0.4 }` | Logical-px band above the ground line and its horizontal drift speed. |
| `DUST` | `{ jumpParticles: 4, landParticles: 6, lifeFrames: 18, speedPxPerFrame: 0.6 }` | Dust puff tuning. |
| `LANDING_SHAKE` | `{ amplitudePx: 2, frames: 3 }` | One-tick landing shiver amplitude (via `renderer.setCamera`). |
| `SHATTER` | `{ shardCount: 10, lifeFrames: 32, gravityPxPerFrame2: 0.35, spreadPxPerFrame: 1.6, flashFrames: 4 }` | Pumpkin shatter particle budget. |
| `BATS` | `{ flockSize: 3, lifeFrames: 90, speedPxPerFrame: 2.2, flapFrames: 6, salt: 0xba75 }` | Milestone-bat flock. |

All values are tuning numbers; tests must assert against the names, not the literals, per `CLAUDE.md`.

### 2.4 Draw layer plan

Layers run `background` → `parallax` → `world` → `foreground` → `hud`. Within a layer, registration order is paint order.

| Layer | Painted (back to front) |
|---|---|
| `background` | sky gradient fill (one `fillRect` across the canvas, colour from `atmosphereAt(state.tick).skyColor`), moon disc, clouds, death flash overlay. |
| `parallax` | puddle (existing), ground fog band, dust puffs. |
| `world` | pumpkin shards (new), then existing world drawers (dinosaur, pumpkins, ground). |
| `foreground` | milestone bats. |
| `hud` | existing HUD only. |

Fog, clouds and bats never move with the camera (fog uses its own horizontal drift; clouds too; bats fly in their own screen-space path). The `background` layer is unaffected by `setCamera` per architecture 2.2, so sky and moon are stable during the landing shiver.

### 2.5 Features in detail

- **`features/nightSky.ts`.** On `start` / `restart`, builds `clouds[]` positions from a feature-local `createRng(state.seed ^ CLOUD_FEATURE_SALT)`, so the same seed replays the same sky. On each draw frame, advances each cloud's `x` by `-CLOUD.baseSpeed` (plus per-cloud jitter) per **tick elapsed** while running, so cloud drift is deterministic by tick count. The drift reads `state.tick` and compares to its last-seen tick; it does **not** use the render frame counter for its position so a seed replays identically. Clouds that leave the left edge wrap back to the right with a fresh jitter sampled from the same RNG. Target cloud count comes from `atmosphereAt(state.tick).cloudTarget`; the feature spawns new clouds (one every N ticks) until it reaches the target. Moon is drawn once from `MOON`.
- **`features/fog.ts`.** Fills a horizontal band `FOG.bandHeight` px tall just above the ground. Alpha from `atmosphereAt(state.tick).fogAlpha`. Horizontal offset drifts by `FOG.speed * elapsedTicks`, independent of the camera. Fully transparent in `idle`.
- **`features/dust.ts`.** On `jump` and `land` events, pushes `DUST.jumpParticles` / `DUST.landParticles` particles into a feature-local list. Each particle has an initial x/y (from `state.dino.hitbox` at event time), a `vx`/`vy`, and a spawn render-frame. Each draw frame, particles age by render-frame delta; those older than `DUST.lifeFrames` are evicted; survivors render as 1 or 2 px squares in `PALETTE.fg`, fading via `globalAlpha`. On `land`, additionally registers a camera function that returns a `{x: 0, y}` with `y = sin(progress * π) * LANDING_SHAKE.amplitudePx` for `LANDING_SHAKE.frames` render frames, then clears itself with `setCamera(() => ({ x: 0, y: 0 }))`. Only one shiver active at a time (overwrites previous).
- **`features/pumpkinShatter.ts`.** On `die`, finds the collided obstacle (`state.obstacles` entry whose hitbox the dino hitbox intersects) and spawns `SHATTER.shardCount` shards from it with randomised velocities (feature RNG seeded from `state.seed ^ SHATTER.salt`). Each shard is a 2x2 px square alternating `PALETTE.pumpkin` and `PALETTE.pumpkinShadow`. Shards age by render frame (because the simulation is frozen in `dead`), falling under `SHATTER.gravityPxPerFrame2`, and disappear after `SHATTER.lifeFrames`. Also paints a semi-transparent white overlay on `background` for `SHATTER.flashFrames` right after `die`. On `start` / `restart`, all shards are cleared. The shatter animation is "already specified for the game-over screen" (functional spec 2.4 last criterion), not a new reaction fired after.
- **`features/milestoneBats.ts`.** On `milestone`, spawns a flock of `BATS.flockSize` silhouettes at the right edge with slight y jitter and per-bat phase offsets. Each bat moves at `BATS.speedPxPerFrame` leftward **per render frame while running**, flapping between two sprite poses every `BATS.flapFrames` render frames. Bats whose life exceeds `BATS.lifeFrames` are evicted. The flock does **not** continue to render on the game-over screen: on `die`, the active flock is cleared, honouring the "frozen screen stays visually calm" rule.

### 2.6 Pumpkin sprite data

The two pumpkin bitmaps are drawn to match the current cactus footprints exactly: identical outer `width`/`height` and identical `hitbox` rects. The pixel art is three-colour (`PALETTE.pumpkin` for the body, `PALETTE.pumpkinShadow` for ridges, `PALETTE.stem` for the stem) and uses the existing `readonly string[]` encoding in `sprites.ts`:

- `#` → body
- `=` → ridge/shadow (new palette index in the sprite decoder; see 2.7)
- `+` → stem
- `.` → transparent

### 2.7 Multi-colour sprite support

The current sprite decoder in `sprites.ts` paints every `#` with a single `fillStyle`. Pumpkins need three colours. The minimal change is to extend the sprite encoding and the renderer helper so each sprite row maps each character to its colour. Two options:

- **A. Per-sprite palette map.** Each sprite declares `{ bitmap, hitbox, palette: Record<string, string> }`; the decoder picks the palette entry per non-`.` character. Backwards-compatible: the dino and ground sprites get a `{ '#': PALETTE.fg }` palette. **Chosen.**
- B. Multiple single-colour layers per sprite. Rejected — doubles the sprite table and the drawer loop count.

The frozen contract in architecture section 3 does not fix the sprite internals; only `sprites.ts`'s module responsibility and the `Rect` / hitbox shapes are contracted. Adding a palette map does not change any exported type.

### 2.8 Determinism and reproducibility

- `atmosphereAt` is a pure function of `tick`.
- `nightSky` cloud positions and `pumpkinShatter` shard velocities use **feature-local** `createRng(state.seed ^ SALT)`, never the game's `Rng`, so a seeded page (`?seed=42`) replays the same course and the same clouds and the same shatter pattern.
- Dust and landing shiver are driven by **events** (`jump` / `land`), which fire on the same ticks for a seeded run. The visual animation itself uses render frames, so it may look slightly different on 120 Hz vs 60 Hz — this is acceptable and consistent with the existing puddle, which also animates via render frames on the frozen screen.
- Bats trigger on `milestone`, which the game already fires at the same ticks for a seeded run.

---

## 3. Impact and Risk Analysis

- **System Dependencies.**
  - `sprites.ts` palette-map change affects the renderer loop in `render.ts` that paints obstacle / dino / ground sprites. Dino, ground and the existing cactus/pumpkin fillers must still render with `PALETTE.fg` after the change — covered by existing render tests driven by the recording `DrawContext` fake.
  - `main.ts` wires five new features; each returns a `dispose()` collected in the existing teardown path.
  - The new `PALETTE` keys are additive; `puddle.ts` keeps reading `PALETTE.accent`.
  - No change to the frozen contract (architecture section 3): no renamed exports, no changed signatures, no removed events, no new `GameState` fields.

- **Potential Risks & Mitigations.**
  - **Readability of pumpkins against a dark sky and fog.** Risk: late-run runs become visually hard. Mitigation: `FOG_MAX_ALPHA` cap of `0.35`; pumpkins use a high-contrast orange body and dark ridge; the dino stays `PALETTE.fg = #535353` on fog, which stays readable.
  - **Determinism drift from render-frame-driven cosmetics.** Risk: a learner thinks the cosmetics change the course. Mitigation: this spec and `CLAUDE.md` already draw the line — the test for `atmosphereAt` and the seed-replay smoke covers "same course"; features drawing from render frames do not touch `GameState`.
  - **Game-over screen "stays calm" vs. the shatter animation.** Risk: the functional spec's last reaction criterion could be read strictly. Mitigation: the shatter is scheduled by the `die` event — i.e. the same frame the game ends — so it is "already specified for the game-over screen", not a reaction triggered after it. Bats and dust are cleared on `die`.
  - **Palette compatibility of the dino sprite.** Risk: existing dino sprite drawer breaks when switching to a palette map. Mitigation: every existing sprite gets a `{ '#': PALETTE.fg }` palette so the one-colour behaviour is preserved byte-for-byte; a unit test asserts the dino is still painted in `PALETTE.fg`.
  - **Performance of many particles on low-end laptops.** Risk: dust + shards + bats + clouds all at once on the frozen screen. Mitigation: tuning budgets keep totals under ~50 particles at any instant; all drawing is `fillRect`, matching the existing renderer's style; no per-frame allocations beyond list pushes and splices.
  - **CSS fallback flash.** Risk: before the first `clearRect` on page load, the canvas shows the CSS background (currently near-white), which would flash bright against the new sky. Mitigation: change the canvas CSS `background-color` to the sky colour (hard-coded hex matching `PALETTE.sky`).

---

## 4. Testing Strategy

- **Unit test for the new logic (functional spec 2.5).** `tests/atmosphere.test.ts`:
  - `atmosphereAt(0)` returns `intensity === 0`, `skyColor === PALETTE.sky`, `fogAlpha === 0`, `cloudTarget === CLOUDS_MIN`.
  - `atmosphereAt(FOG_RAMP_TICKS)` returns `intensity === 1`, `skyColor === PALETTE.skyDeep`, `fogAlpha === FOG_MAX_ALPHA`, `cloudTarget === CLOUDS_MAX`.
  - `atmosphereAt(FOG_RAMP_TICKS / 2)` returns `intensity === 0.5`, a channel-wise midpoint colour, half the fog alpha, and a cloud target between the bounds.
  - `atmosphereAt(FOG_RAMP_TICKS * 10)` is clamped to the same values as `atmosphereAt(FOG_RAMP_TICKS)`.
  - **RED validation:** at least one of these cases is checked out against the current repo (`git stash`) and run — the test file does not resolve `src/atmosphere` and fails. Captured in the commit message / PR description.

- **Acceptance-level tests already present stay green.** `createGame` tests do not read `sprites.ts` palette keys; obstacle hitboxes and spawn ticks are unchanged. Collision tests pass unchanged because `hitbox` on the pumpkin kinds is identical to the cactus kinds.

- **Rendering tests (recommended).** `tests/sprites.test.ts` gains a case asserting the dino is painted in `PALETTE.fg` after the palette-map change (regression guard for section 2.7).

- **Feature smoke tests (recommended, not required).** `tests/features/*.test.ts` for `nightSky` cloud spawn count on a known seed and for `pumpkinShatter` shard count after a stubbed `die` event, using the recording `DrawContext` fake.

- **End-to-end.** The Playwright smoke test is unchanged. It does not inspect pixel colours; it only checks wiring, canvas size, phase transitions, score growth and the `dino.hi` round-trip, all of which continue to hold.

- **Type checking.** New `atmosphere.ts` and new feature modules carry explicit `: Atmosphere` / `: () => void` return types and no `any`, no `as`, as per `CLAUDE.md`.

---

## 5. Open Assumptions (challenge any of these to redirect the plan)

- **Assumption:** Rename of exported sprite names is avoided — the exports stay `cactusSmall` / `cactusLarge` even though they now draw pumpkins. Rationale: the frozen testing contract (architecture section 3) does not list them, but tests outside this repository could reasonably import them. Keeping the names is a one-line decision reversible later; a rename would be a `/awos:spec`-level change if any downstream relies on the symbol.
- **Assumption:** The required-reactions pair chosen in the functional spec resolves to **die** and **milestone**, with **jump** and **land** delivered as bonus. All four are included in the plan.
- **Assumption:** Milestones remain every `MILESTONE_EVERY = 100` points, which the functional spec also recommends. No change to `score.ts`.
- **Assumption:** The dinosaur sprite stays as-is; no ghost / cat re-skin. The spooky mood comes from the world around it.
- **Assumption:** The intensification uses `FOG_RAMP_TICKS = 5400` (90 s at 60 Hz), matching `MAX_SPEED_AT_TICK` so a run that reaches full pace also reaches full atmosphere.
- **Assumption:** Clouds drift opposite the dinosaur's running direction (right-to-left), reinforcing forward motion.
- **Assumption:** The moon is a plain disc, no face. The jack-o'-lantern character is carried by the pumpkins themselves and the bats.
