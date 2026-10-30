<!-- not-user-reviewed -->

# Tasks: Spooky Night Theme

Target: `context/spec/002-spooky-night-theme/`. Each slice leaves the app in a runnable, visibly-improved state. Verification uses `npm test` for pure-logic slices and the dev server (`npm run dev`) driven by `verify-ui` for visual slices. Clean up every verification artifact at the end of the Verify task.

---

- [ ] **Slice 1: Pumpkins replace cacti (course unchanged)**

  > Every obstacle is drawn as a pumpkin; dimensions, hitboxes and the seeded spawn course are identical to today.
  - [ ] Extend `src/tuning.ts` `PALETTE` with `pumpkin`, `pumpkinShadow` and `stem` named entries (defaults from technical-considerations §2.3). Do not change existing `bg`, `fg`, `accent` keys. **[Agent: typescript-game-logic]**
  - [ ] In `src/sprites.ts`, add a `palette: Record<string, string>` field to each sprite record and update the sprite type accordingly. Give every existing sprite the palette `{ '#': PALETTE.fg }` so one-colour behaviour is preserved. **[Agent: typescript-game-logic]**
  - [ ] In `src/render.ts`, change the sprite-painting helper to read the per-character colour from the sprite's `palette` instead of a single `fillStyle`. The dino, ground and HUD paths remain pixel-identical. **[Agent: canvas-rendering]**
  - [ ] Replace the two cactus bitmaps in `src/sprites.ts` with pumpkin bitmaps that use `#` (body), `=` (ridge) and `+` (stem). Keep the exact outer `width` / `height` and the exact `hitbox` of today's small and large cacti. Keep the export names (`cactusSmall`, `cactusLarge`) unchanged — only the pixel data and palette change. **[Agent: typescript-game-logic]**
  - [ ] Verify: run `npm test` — all type-checks and existing Vitest suites pass unchanged (same hitboxes, same spawn ticks). Then run `npm run dev`, open the game in a browser with `verify-ui`, play one short run, confirm the obstacles are visibly pumpkins in two distinct sizes and the dino/ground still render in `PALETTE.fg`. Reload with `?seed=42`, run again, confirm obstacle positions match the pre-change recording. Stop the dev server by its recorded PID and delete any screenshots or recordings produced. **[Agent: canvas-rendering]**

- [ ] **Slice 2: Night sky backdrop (moon and drifting clouds)**

  > The scene is a dark sky with a fixed moon and a few clouds drifting from right to left. No gameplay change.
  - [ ] Add new palette and layout tunables to `src/tuning.ts`: `PALETTE.sky`, `PALETTE.skyDeep`, `PALETTE.moon`, `PALETTE.cloud`, plus `MOON`, `CLOUD`, `CLOUDS_MIN`, `CLOUDS_MAX`, `CLOUD_FEATURE_SALT` (defaults from technical-considerations §2.3). **[Agent: typescript-game-logic]**
  - [ ] Create `src/features/nightSky.ts` exporting `installNightSky({ game, renderer })`. It subscribes to `start` and `restart` to (re)seed a feature-local `createRng(state.seed ^ CLOUD_FEATURE_SALT)` and build `CLOUDS_MIN` initial cloud positions. It registers one draw callback on `background` that paints: a full-canvas sky fill in `PALETTE.sky` (fixed, Slice 3 makes it tick-driven), the moon disc from `MOON`, then each cloud as a `fillRect` cluster. Clouds advance left by `CLOUD.baseSpeed * (state.tick - lastTick)` per draw; wrap back to the right edge when off-screen. Must never write to `GameState` and never call `Math.random`. Returns a `dispose()`. **[Agent: canvas-rendering]**
  - [ ] Wire `installNightSky` into `src/main.ts` alongside `installPuddle`; collect its dispose in the existing teardown path. **[Agent: canvas-rendering]**
  - [ ] Update `src/style.css` so the canvas CSS `background-color` matches `PALETTE.sky` (hard-coded hex). **[Agent: canvas-rendering]**
  - [ ] Verify: `npm test` stays green. Run `npm run dev` and use `verify-ui` to confirm (a) the scene is dark before pressing space, (b) a moon disc is visible in the top-right area, (c) at least one cloud is visible, (d) running the game for ~5 s shows clouds drifting leftwards while the moon stays put, (e) `?seed=42` reload produces the same cloud starting positions as a second visit. Stop the dev server by PID and delete any screenshots. **[Agent: canvas-rendering]**

- [ ] **Slice 3: Scene intensifies the longer the player survives**

  > Over roughly 90 s the sky deepens, fog rolls up from the ground, and more clouds appear. All reset on restart. Pure driver lives in a new unit-tested module.
  - [ ] Add `FOG_RAMP_TICKS`, `FOG_MAX_ALPHA`, `FOG`, and `PALETTE.fog` to `src/tuning.ts`. **[Agent: typescript-game-logic]**
  - [ ] Create `tests/atmosphere.test.ts` **before** the implementation. Cover: `atmosphereAt(0)`, `atmosphereAt(FOG_RAMP_TICKS)`, `atmosphereAt(FOG_RAMP_TICKS / 2)`, clamping at `FOG_RAMP_TICKS * 10`, and that `skyColor` at midpoint is the channel-wise midpoint hex of `PALETTE.sky` and `PALETTE.skyDeep`. Run the test — it must fail (import of `src/atmosphere.js` unresolved). Record the RED output in the slice's verification notes. **[Agent: typescript-game-logic]**
  - [ ] Create `src/atmosphere.ts` exporting `interface Atmosphere` and `atmosphereAt(tick: number): Atmosphere` per technical-considerations §2.2. Pure, no imports beyond `./tuning.js`. **[Agent: typescript-game-logic]**
  - [ ] Update `src/features/nightSky.ts` so the sky fill colour and the cloud target count come from `atmosphereAt(state.tick)` on each draw. When the current cloud count is below the target, spawn one new cloud (feature RNG) every `CLOUD.spawnEveryTicks` (add this to `CLOUD` tunable, default 180). **[Agent: canvas-rendering]**
  - [ ] Create `src/features/fog.ts` exporting `installFog({ game, renderer })`. On the `parallax` layer it paints a `FOG.bandHeight`-tall band just above `GROUND_Y` in `PALETTE.fog`, with `globalAlpha = atmosphereAt(state.tick).fogAlpha`, drifting by `FOG.speed * elapsedTicks`. Fully transparent in `idle`. Returns `dispose()`. **[Agent: canvas-rendering]**
  - [ ] Install `installFog` in `src/main.ts` after `installNightSky`. **[Agent: canvas-rendering]**
  - [ ] Verify: `npm test` — new `atmosphere.test.ts` passes GREEN, all existing tests still pass. Run `npm run dev` with `verify-ui`: at t ≈ 0 scene matches early-run look; after ~90 s of play (use the browser clock) sky is visibly darker and fog is visible along the ground; after a `die` → space restart the scene returns to the early look. Stop the dev server by PID and delete screenshots. **[Agent: canvas-rendering]**

- [ ] **Slice 4: Jump dust, land dust and landing shiver**

  > Visible reactions to the `jump` and `land` events: a short dust puff at the dino's feet, plus a one-tick camera shiver when landing.
  - [ ] Add `DUST` and `LANDING_SHAKE` tunables to `src/tuning.ts`. **[Agent: typescript-game-logic]**
  - [ ] Create `src/features/dust.ts` exporting `installDust({ game, renderer })`. Subscribes to `jump` (spawns `DUST.jumpParticles` behind the dino's feet) and `land` (spawns `DUST.landParticles` + registers a camera function returning `{ x: 0, y: sin(progress * π) * LANDING_SHAKE.amplitudePx }` for `LANDING_SHAKE.frames` render frames, then calls `renderer.setCamera(() => ({ x: 0, y: 0 }))`). Particles are 1–2 px squares in `PALETTE.fg`, aged by render-frame, evicted after `DUST.lifeFrames`, drawn on `parallax`. Resets on `start` / `restart`. Returns `dispose()`. **[Agent: canvas-rendering]**
  - [ ] Install `installDust` in `src/main.ts`. **[Agent: canvas-rendering]**
  - [ ] Verify: `npm test` stays green (no new logic to break it). `npm run dev` + `verify-ui`: press space to jump, confirm puff at feet; after landing confirm a second puff and a one-tick camera shiver (ground line briefly offset vertically). Stop the dev server by PID and delete screenshots. **[Agent: canvas-rendering]**

- [ ] **Slice 5: Pumpkin shatter and screen flash on die**

  > Hitting a pumpkin bursts it into shards that fall for ~0.5 s under gravity, with a brief white flash on `background`. Shards are the "animation already specified" on the game-over screen; everything else goes calm.
  - [ ] Add `SHATTER` tunable to `src/tuning.ts`. **[Agent: typescript-game-logic]**
  - [ ] Create `src/features/pumpkinShatter.ts` exporting `installPumpkinShatter({ game, renderer })`. On `die`, locates the obstacle whose hitbox intersects the dino hitbox in `state.obstacles` and spawns `SHATTER.shardCount` shards from its centre using feature RNG `createRng(state.seed ^ SHATTER.salt)`. Add `SHATTER.salt` to the tunable. Shards age by render frame, falling under `SHATTER.gravityPxPerFrame2`, drawn on `world` as 2x2 px squares alternating `PALETTE.pumpkin` / `PALETTE.pumpkinShadow`. Also paints a `rgba(255,255,255,alpha)` overlay on `background` for `SHATTER.flashFrames` render frames with alpha fading from 0.5 to 0. Cleared on `start` / `restart`. Returns `dispose()`. **[Agent: canvas-rendering]**
  - [ ] Install `installPumpkinShatter` in `src/main.ts`. **[Agent: canvas-rendering]**
  - [ ] Verify: `npm test` green. `npm run dev` + `verify-ui`: start a run with `?seed=42`, let the dino collide, confirm (a) a brief white flash, (b) shards burst from the pumpkin, (c) shards fall under gravity, (d) after shards settle the game-over screen is calm (puddle still growing as today, no new effects). Press space; confirm shards clear and a new run starts. Stop the dev server by PID and delete screenshots. **[Agent: canvas-rendering]**

- [ ] **Slice 6: Milestone bats**

  > Every multiple of `MILESTONE_EVERY` points, a small flock of silhouette bats flaps across the sky from right to left over ~1.5 s.
  - [ ] Add `BATS` tunable to `src/tuning.ts` and `PALETTE.bat`. **[Agent: typescript-game-logic]**
  - [ ] Add a two-pose bat sprite set (wings up / wings down) to `src/sprites.ts`, with a `palette: { '#': PALETTE.bat }`. **[Agent: typescript-game-logic]**
  - [ ] Create `src/features/milestoneBats.ts` exporting `installMilestoneBats({ game, renderer })`. On `milestone`, spawns `BATS.flockSize` bats at the right edge with jittered `y` and per-bat phase offsets (feature RNG seeded from `state.seed ^ BATS.salt`). Each draw frame, each bat advances `-BATS.speedPxPerFrame` while `state.phase === 'running'`; pose toggles every `BATS.flapFrames` render frames. Evicted after `BATS.lifeFrames`. On `die`, cleared immediately. Drawn on `foreground`. Returns `dispose()`. **[Agent: canvas-rendering]**
  - [ ] Install `installMilestoneBats` in `src/main.ts`. **[Agent: canvas-rendering]**
  - [ ] Verify: `npm test` green. `npm run dev` + `verify-ui` with `?seed=42` and `collisions=false`-style long run (play with intent until score crosses 100): confirm a bat flock crosses the sky right-to-left at the milestone and the score HUD blink still behaves as today. Deliberately collide; confirm no bats continue to render after `die`. Stop the dev server by PID and delete screenshots. **[Agent: canvas-rendering]**

- [ ] **Slice 7: Feature Testing & Regression**

  > Verifies the whole feature end-to-end against functional-spec.md, run after all implementation slices are complete.
  - [ ] Read functional-spec.md acceptance criteria in full. Generate acceptance-level tests that verify the entire feature as a whole — not individual slices. Cover applicable layers (unit for pure logic in `atmosphere.ts` and sprite palettes, integration for feature install / dispose and bus subscriptions using a stub `game.on` and a recording `DrawContext`, e2e for the seeded-replay guarantee via the existing Playwright smoke harness extended with a `?seed=42` round-trip). Write tests with RED validation (must fail before implementation is confirmed done). Annotate each test with `@spec: 002-spooky-night-theme` and `@regression` if suitable for long-term regression. **[Agent: testing-expert]**
  - [ ] Run all generated tests. All must pass. Fix any failures before proceeding. **[Agent: testing-expert]**
