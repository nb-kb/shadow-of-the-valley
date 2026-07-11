# Known Bugs — Shadow of the Valley

Living list of open issues, worst-first-ish. Fixed items move to the changelog
in commit history. Line numbers drift — treat them as hints, re-grep before
editing.

Last reviewed: 2026-07-11 (build beta 1.0.1).

## Open

### 0. Carve-height twin drift family
- **Found by the v22e hideout probe.** Same pattern as the fixed gate lintel:
  FENCE gate carve JS y-center 1.2 vs GLSL 1.3, FACADE door recess JS 1.2 vs
  GLSL 1.25 (area1) — sub-10cm invisible ledges, imperceptible today.
- ~~**Latent landmine:** GLSL `mapStatic` early-returns `levelSDF` for authored
  levels, skipping the `uEdits`/`uGlobs` loops JS still applies.~~ **RESOLVED**
  (scope wave): `mapStatic` now branches `levelSDF`/`mapProc` for the base and
  BOTH feed the shared uEdits→uGlobs pass, matching
  `mapWorldJS = applyGlobsJS(applySdfEditsJS(base))` exactly — PICK carves
  render indoors. The authored-primitive height drifts above stay open.

### 1. Chrome load crash — "cannot access gameState before initialization"
- **Where:** initial top-level script exec (TDZ). Not the earlier `Menus`
  forward-ref, which is already fixed — this is a separate one.
- **Repro:** load in Chrome, often after a GPU overload. **Zen/Firefox run
  clean.** Chrome is the odd one out.
- **Status:** guard landed in v22a — early handlers (`anyKeyGate`, `uiOpen`)
  no-op until `engineReady` flips at BOOT, so the TDZ access path is closed by
  inspection. Kept open until a Chrome load confirms the banner stays quiet;
  the *why-does-init-abort* question is still unpinned.

## Backlog (future — low priority)

- **ENEMY AI REWORK** (owner-requested, v22i): the zealot brain is still
  direct-steer patrol/suspicious/alarm. Wants a real pass — behaviors worthy
  of the new body/telegraphs (flanking, cover use, coordination) now that
  perception, gear, and the vision-cone contract are in place. Design with
  the owner before building; no pathfinding/navmesh (waypoint discipline
  stands).
- **Loot tables** should offer the full suite of available items — audit
  `lootTable` / `lootRegions` for coverage.
- **Enemy-sourced burn WATCHED** (mod wave v1, deferred knob): the zealot spark
  roll doses the player 15 burn ≈ up to 15 hp over 7.5s — real early-game bump.
  If playtests say too hot, the one-liner is halving enemy-sourced doses in
  `applyStatus`. See docs/MODS.md balance ledger.
- **ORB tracer-slot hoarding** — a live orb holds a tracer slot for its full 6s
  aloft (cap 4 orbs). Capped, flagged "monitor" in MODS.md; revisit if tracers
  starve during orb play.
- **v22m WATCHED knobs** (consciously deferred, all single-line fallbacks —
  details in docs/MODS.md ledgers): enemy CHAOS heavy hand (+45%/stack rides
  `MOD_KEYS` rolls; knob: halve `chaosMul` for non-player) · bolt-thread TTK
  under the A5 burn=hurt law (knob: dose ×0.5) · boom-drum burn saturating the
  30-pool in ~3s (⚡ deficit prices it) · enemy TIMER airbursts + DRILL
  pass-through + relay extras = collective difficulty bump (knobs: drop
  `slowFuse`/`pierce` from `MOD_KEYS`) · RETURN-arc TTK vs crowds behind low
  cover (knob: W 2.5→2.0 or reach 14→11) · WAVE contact-drum tracer load
  (knob: cadence 0.5→0.7s) · REFUND at 1⚡ collapses the lead economy where it
  drops (knob: surcharge 2–3, never a mechanic change).
- **Manual browser pass owed** (no browser in the loop this wave): burstCam+
  POWDER+BEAM mint gate (mag −3/pull), hitscan pulse feel/converge, drum beats
  landing at the dot, thread-degradation draw order (damage-follows-draw — no
  invisible damage), grapple swing feel, whip takedown reads.
- **Dead-code orphan** (pre-existing, harmless): `showTooltip_DEAD` calls
  undefined `positionTooltip()` — the function is never invoked. Delete both
  on the next tooltip pass.

## Recently improved

- **beta 1.0.1** — Long first load froze the title card. One ~860-line fragment
  shader was compiling synchronously on the boot thread, so the intro art sat
  locked until the driver finished. Now the compile is non-blocking (parallel
  via `KHR_parallel_shader_compile`, polled off the boot thread) behind an
  honest, animated loading screen, and a cheap PREVIEW shader renders first so
  the player can move immediately while the full shader finishes and hot-swaps
  in behind it. Perceived first-load stall is gone.

## Recently fixed

- **v22r** — Fog translucency fixed at the blend target (diagnosed v22p). The
  one fog blend composited `skyColor(rayDir)` — the FULL directional sky, sun
  disk (`pow(sd,180)·2`), halo, clouds, stars — through geometry, so any nonzero
  `uFog` read *translucent* (sun through ridges, stars through walls). Now
  `col = mix(col, uSkyLo, fogAmt)`: `uSkyLo` is the celestial-free horizon
  endpoint (`skyColor`'s own low floor), so distance haze is opaque horizon tint
  with no disk/halo/cloud/star bleed. GLSL-only, no twin risk (fog is
  render-only); the sky-*miss* path still renders full `skyColor(rayDir)`.
  Subsumes the old authored-`atmos.fog` disk-bleed residual — the disk is no
  longer in the blend target for any fog source.
- **v22o** — Death mid-frame faulted the engine: `ENGINE FAULT: pr is undefined`
  at `pr.stuck` in `updateProjectiles`. A killing hit processed INSIDE a
  per-frame loop (enemy bolt → `damagePlayer` → `onDeath` → `deployTo` →
  `teardownWorld`) cleared the very array the loop was walking; the next index
  read undefined. Same mid-frame-teardown family as v22d/v22k — now fixed as a
  CLASS: `_worldGen` bumps in `teardownWorld`/`resetLevel`, and every loop that
  can kill the world through its own body (projectiles, world items, beam
  lanes, burst volleys, the enemy roster, the bag-grenade walk, the burst-fire
  pull) captures it at entry and bails when it moves. Post-death statement
  leaks closed with it: the killing shot no longer re-doses burn/bleed onto the
  stanched respawn, self-blast no longer kicks the fresh camera, and no housed
  payload fires into the world you respawned to. The orb 4-cap also stopped
  splicing mid-iteration (a relay casting the 5th pearl from inside
  `updateProjectiles` shifted the live index — silently deleting the fresh orb
  and double-triggering the spent carrier); pops are MARKED and reaped at a
  safe index instead. Node-probed both ways: the exact TypeError reproduced on
  v22n, clean after.
- **v22m** — BOOM proximity self-damage was a paper tiger: the player-splash
  path always existed (since BOOM landed), but the shooter was missing the
  ×1.6 flesh multiplier enemies ate, and armor (plates 0.6–0.75 × helmet
  0.88–0.94) buried the rest — point-blank blasts read as free. Fixed by
  symmetry: `explode()` now runs the same `dmg·falloff·1.6` math both sides
  (armor shaves AFTER), plus a view kick + pitch dip so the hit READS, and the
  blast sears both sides (`burn = own + round(0.35·dmg-at-range)`). Bare-muzzle
  BOOM: unarmored ~30 + ~9 burn — survivable, stupid, felt.
- **v22l** — Endless-valley crates were **decorative**: only the 6 authored
  origin crates were registered as lootable — the hash-scattered cell crates
  existed solely in the GLSL/JS SDF twins, with no registry entry to loot. A
  third fround-faithful copy of the cell-crate hash math now feeds the crate
  registry (gate-scaled window, token-identical `fractJS(fr(h*(13+7*k)))`
  draws), verified bit-exact against both twins across 240 rings.
- **v22l** — Barracks and walled-yard doorways in the valley shoved you back
  out: the door carves left sub-player-radius clearance troughs (0.34 m radius
  vs ~0.20/0.35 m clearance). Deepened both carves in lockstep (GLSL cellSDF +
  JS mapWorldBase, 5.4→5.8 and 7.2→7.4) → 0.40/0.45 m troughs; every drawn
  opening is now enterable (0 failures across 2,451 structures swept).
- **v22a** — Crate opened and instantly grabbed the top item: the
  `pressed('interact')` edge that opened the crate survived into the same
  frame's `menuNav()` and ran the take. Menu transitions now SPEND the edge —
  `menuSet` (and terminal open/close) clear `INPUT.frameDown`/raw maps, so no
  handler downstream inherits the press that changed rooms.
- **v22a** — Invisible enemies shooting from off-render: the shader still draws
  only the nearest 10 (`RLIM.ENEMIES`), but unrendered zealots are now DORMANT —
  they hold fire and bullets pass through them, so nothing invisible can hurt
  you or eat your shots. The cap itself stands; dormancy makes it fair.
- **v22a** — Save feedback (note, not a fix of a listed bug): saves are now
  verified by reading the slot BACK after the write — success toast reports
  slot + item count, and every failure path (no localStorage, quota full, bad
  read-back) gets its own distinct SAVE FAILED toast instead of a false
  "MEMORY WRITTEN".
- **v21q** — Building doorways (armory, NW lab) wouldn't let you in: the door
  carve was only 0.6 m half-depth, so after the SDF subtraction the threshold
  kept a ~0.15 m clearance ridge — below the 0.34 m player radius, so collision
  shoved you back out even though the doorway *rendered* open (NOT a twin desync;
  both sides agreed). Deepened the carve to 1.1 m (→ ~0.40 m clearance) on the
  standalone `BUILDING` door and the `WALLS` interior-building door, GLSL + JS.
  No visual change — the carve only extends into empty space either side of the
  thin wall.
- **v21q** — Forest tree/stump collision desync: `_hash2i` used
  `(...)*1274126177|0`, whose product exceeds 2^53 and rounds in f64, diverging
  from the shader's exact int hash in ~92% of cells — so collision trees sat
  away from the drawn ones. Switched to `Math.imul` (mirrors GLSL `lvHash`; same
  idiom `cellHash` already used).
- **v21q** — Jump/landing feel: camera Y-smoother now tracks the body 1:1 while
  airborne (was lagging then hard-snapping = the "pixel skip"); the grounded
  lerp eases the touchdown instead of an instant rigid snap.
- **v21p** — Facade/fence doorway twin desync: `FACADE`/`FENCE` door offsets are
  authored absolute, but GLSL's `lvHole` added the region center, flinging the
  carve ~96 m off the wall (facade/fence rendered solid while collision had the
  opening). Fixed in `bakeLevel` by pre-subtracting the center.
