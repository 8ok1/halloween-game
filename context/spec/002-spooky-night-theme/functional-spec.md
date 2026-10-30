# Functional Specification: Spooky Night Theme

- **Topic:** A Halloween-flavoured visual re-skin of the running game — pumpkins instead of cacti, a night sky backdrop, something that intensifies the longer the player survives, and small reactions to events the player causes on screen.
- **Status:** Draft
- **Author:** AWOS

---

## 1. Overview and Rationale (The "Why")

The core game is complete and plays well, but it still looks like the day-one placeholder: a daylight scene with cacti and no feedback beyond the score counter. The product definition flags "make it shine" as the natural next version, and the game-over screen currently gives the player nothing to react to besides a number.

This change dresses the game in a Halloween night theme and adds subtle, reactive touches that respond to what the player is doing. The intent is twofold:

- **Make the game feel alive.** A night sky with a visible moon and slowly drifting clouds, obstacles that look like pumpkins instead of cacti, and small visual reactions when the player jumps, lands, hits a pumpkin, or crosses a milestone score give the run a mood and a sense of consequence without changing a single gameplay rule.
- **Make longer runs look different from shorter ones.** Something in the scene changes the longer the player survives, so a thirty-second run and a two-minute run are visually distinguishable at a glance. This rewards persistence with a visible payoff.

Success looks like a player who plays one short run and one long run, and can describe — unprompted — at least one way the scene changed in the longer run, and at least one visible reaction that happened when they jumped, landed, hit a pumpkin, or passed a milestone.

---

## 2. Functional Requirements (The "What")

### 2.1 Pumpkins replace cacti

Every obstacle the player must jump over is drawn as a pumpkin instead of a cactus. Pumpkin size variants map one-to-one onto the existing small / medium / large cactus variants, so the shapes a player has to clear are unchanged — only the look changes.

- **Acceptance Criteria:**
  - [ ] Given a run is in progress, when any obstacle appears on screen, then it is drawn as a pumpkin and no cactus artwork is visible anywhere in the scene.
  - [ ] Given the game has been running long enough for several obstacles to pass, when the player observes the obstacles, then they see pumpkins in more than one visibly distinct size.
  - [ ] Given the run is replayed with the same seed, when the sequence of obstacles plays out, then the pumpkins appear in the same positions and the same sizes as the cacti did before the re-skin — the course is unchanged.

### 2.2 Night sky backdrop with moon and clouds

The scene behind the running dinosaur is a night sky instead of a plain daylight background. The night sky shows a single moon in a fixed position on screen, and a small number of clouds that drift slowly across the sky.

- **Acceptance Criteria:**
  - [ ] Given the player opens the page, when they look at the scene before pressing space, then they see a dark night sky, a moon, and at least one cloud visible against it.
  - [ ] Given the run is in progress, when the player watches the sky, then the clouds visibly drift horizontally over several seconds, while the moon stays in place.
  - [ ] Given the run restarts, when the new run begins, then the night sky, moon and clouds are present from the first frame of the new run.
  - [ ] [NEEDS CLARIFICATION: do the clouds drift in the same direction the dinosaur is running, the opposite direction, or should they look wind-blown independent of the dinosaur? Recommended: opposite direction, so they reinforce the sense of forward motion.]
  - [ ] [NEEDS CLARIFICATION: should the moon be a plain disc, a crescent, or a jack-o'-lantern moon face? Recommended: plain full moon for readability.]

### 2.3 The scene intensifies the longer the player survives

Something in the scene changes visibly the longer a single run lasts, so that a long run looks different from a short one. The change is purely cosmetic — it does not change speed, scoring, obstacles, or any gameplay rule — and it resets completely when the run ends and a new run starts.

- **Acceptance Criteria:**
  - [ ] Given a fresh run has just started, when the player looks at the scene, then it is in its "early run" state (baseline look).
  - [ ] Given the player has survived long enough to cross at least one milestone, when they look at the scene, then at least one visible element has changed compared to the first few seconds — the scene is noticeably more intense, dense, or dramatic.
  - [ ] Given the player hits an obstacle and the game-over screen appears, when they press space to start a new run, then the scene returns to its early-run state and begins to intensify again from scratch.
  - [ ] [NEEDS CLARIFICATION: what specifically should intensify? Candidates: more clouds appearing, the sky deepening in colour, mist rolling in near the ground, a bat or two flying across occasionally, the moon reddening. Recommended: more clouds appear and the sky darkens, keeping the effect subtle and non-distracting.]
  - [ ] [NEEDS CLARIFICATION: at what points in the run should intensifications kick in? Recommended: step up at score milestones that already matter in-game (every 100 points), so visual change and milestone reactions line up.]

### 2.4 Visible reactions to game events

The scene reacts visibly to things the player causes. At minimum, two of the following four events must produce a short, visible on-screen reaction: jumping, landing, hitting a pumpkin (dying), and crossing a score milestone. Each reaction is a brief visual flourish — not a change to how the game plays.

- **Acceptance Criteria:**
  - [ ] Given a run is in progress, when the player presses space to jump, then a short visible reaction appears on screen within the same moment as the jump (for example, a small puff of dust under the dinosaur, or a brief flash near the moon).
  - [ ] Given the dinosaur is airborne, when it lands back on the ground, then a short visible reaction appears on screen at the landing spot (for example, a dust puff or a small shake of the ground line).
  - [ ] Given a run is in progress, when the player's score crosses a milestone, then a short visible reaction appears on screen that is clearly tied to the milestone (for example, the score briefly pulses, or sparkles appear around it) and the player can tell a milestone just happened without reading the number.
  - [ ] Given a run is in progress, when the dinosaur collides with a pumpkin, then a short visible reaction appears on screen at the point of impact (for example, the pumpkin shatters into pieces, or the screen briefly flashes) before the game-over screen settles.
  - [ ] Given the game-over screen is showing, when the player looks at it, then no new reactive effects continue to trigger — the frozen screen stays visually calm apart from any animation already specified for it.
  - [ ] [NEEDS CLARIFICATION: which two (or more) of the four events — jump, land, die, milestone — are the required set, and which are optional nice-to-haves? Recommended required pair: **die** and **milestone**, as they carry the most player meaning; **jump** and **land** are welcome additions if they are easy.]
  - [ ] [NEEDS CLARIFICATION: what counts as a "milestone" score? Recommended: every 100 points, matching the natural reading of the three-digit score counter.]

### 2.5 At least one new automated test

The new logic that drives the visual theme — specifically, whatever logic decides when the scene intensifies or when a milestone fires — is covered by at least one new automated test. The test exercises the logic directly; it does not depend on anyone looking at the screen.

- **Acceptance Criteria:**
  - [ ] Given the project's test command runs, when it completes, then at least one new test is reported that did not exist before this change and that covers the new theme logic.
  - [ ] Given the new test is run against the code as it was before this change, when it executes, then it fails — proving the test actually covers the new behaviour (RED validation).
  - [ ] Given the new test is run against the code after this change, when it executes, then it passes.

---

## 3. Scope and Boundaries

### In-Scope

- A single visual theme — pumpkins, night sky with moon and clouds, something that intensifies with survival time, and visible reactions to at least two of the four named game events — applied as the game's default look.
- Full visual reset of the theme on every restart, consistent with the current restart contract.
- At least one new automated test covering the new logic that drives the theme.

### Out-of-Scope

- Any change to gameplay rules: speed, jump arc, obstacle sizes and gaps, scoring, collision detection, restart behaviour, and the seeded-replay guarantee are all unchanged.
- Sound effects or music of any kind, including spooky ambience.
- A theme toggle or switcher — the game ships with the Halloween night theme as the only look.
- New obstacle types, flying obstacles, ducking, power-ups, lives, or difficulty settings.
- Mobile/touch controls and responsive layout work beyond what already exists.
- Localisation of any on-screen text.
- [NEEDS CLARIFICATION: should the dinosaur itself be re-skinned (e.g. as a ghost or a cat), or does it stay as the current dinosaur? Recommended: keep the dinosaur as-is, so the re-skin focuses attention on the world around it.]

---

## Change Log

- [YYYY-MM-DD] — [source reference] — [what behavior changed and why]
