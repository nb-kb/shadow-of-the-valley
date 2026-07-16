# Known Bugs — Shadow of the Valley

Living list of open issues, worst-first-ish. Fixed items move to the changelog
in commit history. Line numbers drift — treat them as hints, re-grep before
editing.

Last reviewed: 2026-07-15 (the Containment wave on `feat/containment` — the P2
fix-up, the systems-audit sweep, and P3–P7 landed; 61 commits). #0/#4/#6a/#6b/
#7/#8 closed this wave — see Recently fixed. **One consolidated browser pass
gates all of it: `docs/PLAYTEST.md`.**

## Open

### 1. Chrome load crash — "cannot access gameState before initialization"
- **Where:** initial top-level script exec (TDZ). Not the earlier `Menus`
  forward-ref, which is already fixed — this is a separate one.
- **Repro:** load in Chrome, often after a GPU overload. **Zen/Firefox run
  clean.** Chrome is the odd one out.
- **Status:** guard landed in v22a — early handlers (`anyKeyGate`, `uiOpen`)
  no-op until `engineReady` flips at BOOT. Pre-boot handler sweep completed
  2026-07-15 — guard coverage is total by inspection. Kept open until one
  clean Chrome load post-GPU-overload confirms the banner stays quiet; the
  *why-does-init-abort* question is still unpinned.

### 2. Enemy render cap 20 — Chrome/ANGLE compile budget UNVERIFIED  ⚠ potential showstopper
- **The one to watch.** `RLIM.ENEMIES` went 10 → 16 → **20** for the horde. Both
  per-fragment enemy loops (body SDF + shading) unroll to 20 — on the branch
  family that exists precisely because the fragment shader was already near
  Chromium's ANGLE→D3D compile ceiling (black screen).
- **Tested clean at 16 on the owner's machine; 20 is UNCONFIRMED,** and the
  low-end/Optimus tester hasn't loaded any of it.
- **OWNER OBSERVED (2026-07-14):** the build is functional but throws a BRIEF
  black screen *sometimes*. NOT the full showstopper.
- **Best code-grounded hypothesis (2026-07-15):** ANGLE defers the real D3D
  compile to a program's FIRST DRAW, and `ensureFullProgram` rebinds +
  `resize()` (clearing the canvas to black) when a full variant links — once
  at boot (preview→full) and once at the first hideout↔valley swap.
  **Mitigation landed** (654881b): both full programs get a 2×2-viewport
  prewarm draw the moment they link, behind the loading screen. Needs the
  browser: correlate any REMAINING brief-black with boot vs first-transition
  via `__bootPhase` (bootDiag) — PLAYTEST.md §1.
- **Dial:** `RLIM.ENEMIES` (one line, ~L737) — the material band + every enemy
  array DERIVE from it, so lowering is safe + self-consistent. Drop to 12–14
  if Chrome/Edge black-screens. NOTE: the goggles CROWD dial does NOT reduce
  compile cost — it caps live bodies, not the compile-time unroll; only
  `RLIM.ENEMIES` does.
- The wave's GLSL deltas (gas fog d73840b, menu-prop fix 9f5ada6) ride the
  same pass — the SHADER-BUDGET audit returned null, so PLAYTEST.md §1 doubles
  as the budget checkpoint.

### 3. Powder/refund rework — browser verification owed  (rewritten 2026-07-15)
- The **firing-resolver rework LANDED** just before the wave (3c2430b…bc4445e):
  two-pool model (boltless lead fires PARALLEL to the energy cores), powder is
  a MAG trait (`anyCaliber`+`powder:true`; its lane hitscans by design), the
  coreless powder synth is RETIRED, and REFUND = base ⚡ only (`roundsFor`'s
  lead-waiver removed). MODS.md's ⚠ CORRECTED MODEL is the spec and the code
  now matches it.
- Of the old three gaps: **(a)** the HUD-invisible synth round and **(b)**
  beam-fed powder hitscanning are MOOT — the synth no longer exists and the
  hitscan lane is the design. **(c) blast ≠ literal parity survives,
  generalized:** a relayed CONTACT/TIMER delivery routes `explode()` (R2.4,
  `dmg×1.6`+falloff), so a point-blank delivery deals its direct hit PLUS
  splash — reads as "the round's damage" but isn't exact. Document as flavor
  or flatten the ×1.6 for player-owned relays — owner call, low.
- **Nobody has fired the new model in a browser yet.** The owed checks are
  folded into PLAYTEST.md §14 (mint gate, pulse feel/converge, drum beats,
  thread draw order, grapple, whip).

### 5. POI soldier-rotation needs ≥4 POIs
- The disjoint opposite-quadrant pair `[c, c+half]` only rotates with **≥4 (even)**
  POIs. A 2–3 POI level would field the same `[0,1]` pair every raid (no crash,
  no rotation). Fine for area-1's 6 POIs; a landmine only if a future level
  ships fewer. Cheap insurance when touched anyway: one `console.warn` in the
  authoring path at `pois.length<4`.

### 6. Raymarch structure-edge shimmer on the FAST preset  (#6c — the survivor)
- #6a (dyn-light eviction strobe) and #6b (8Hz exposure steps) are FIXED — see
  Recently fixed. What remains: thin structure edges shimmer on the FAST
  preset. Preset tradeoff vs a jitter experiment — judgment call in the
  browser (PLAYTEST.md §7), nothing committed.

### 9. Hideout supply crates are an infinite free-loot faucet  (audit — NOT landed this wave)
- The C13 empty-crate floor in `seedLevelSpawns` (~L7412-22) reseeds all 7
  non-stash hideout crates on EVERY `deployTo('hideout')` — every death,
  exfil, and boot — because hideout `lootTiers` falls back to area1's and
  `lootN:0` only skips the scatter. Die-loot-bank-repeat is zero-cost
  sustained income — the accounting-error class the BALANCE LAW forbids.
- Fix per the law (price/gate, never delete): guard the floor loop with the
  scatter dial — `if(L.spawn.lootN)` — leaving the guaranteed
  photometer/compass loop untouched. Owner alternative if a home trickle is
  wanted: reseed only on exfil ARRIVAL.

### 10. Save-restore integrity bundle  (audit — NOT landed this wave)
Six guards in one diff area (`itemFromJSON`/`loadGame`, ~L7650-7830), plus one
approved owner call:
- The `Tool` branch (~L7667) lacks the overlong-`j.slots` clamp its Barrel /
  Platform siblings got — an oversized blob silently GROWS `base.slots` past 4
  into phantom rack cells. Mirror the Barrel idiom; salvage tails into
  `_retuneOrphans`.
- Clamp garbage scalars on read: `Ammunition.count` → [0,stack]; `Battery`
  cap→capMax, charge→[0,cap]; `magType` validated vs acceptedTypes/anyCaliber;
  `opticSel>=0`.
- Equip restore iterates blob keys (`for(const k in b.equip)`, ~L7767) —
  iterate `EQUIP_SLOTS` instead (saveGame's own whitelist).
- Ignored `autoPlace` fallbacks (stash/lockboxes/pockets/armor-inv) → route
  into `_retuneOrphans` so overflow reports 'N LOST' instead of vanishing.
- Reset transient combat state on load success AND catch (the onDeath idiom:
  `reloadTimer`/`holsterCur`/`holsterDir`/`_lastReloadT`/`_lastSwap`).
- Exclude `maxEnemies` from the params blob walk — deploy code owns it
  (seedLevelSpawns sets 20 per raid).
- **Owner call 17 (approved, unbuilt):** ~10-line pre-load snapshot ROLLBACK in
  `loadGame`'s catch — a corrupt LOAD restores the running session instead of
  wiping it; a corrupt SLOT still reads as an empty slot (invariant #4 intact).

### 11. Loot coverage gaps  (audit — partially landed)
- `regionLoot` is LIVE now (P4 wired it: lab/testing caches + the P6 POI
  clusters). Still dead config: `lootRegions` (~L2519 — its own comment says
  "still inert"), the hideout's empty `lootTable`, and `MOD_KEYS` (~L3481 —
  defined, ZERO consumers). Wire or delete per the remainder of owner call 2;
  if deleted, also rewrite the two comments citing MOD_KEYS for safety
  properties nothing enforces.
- **Unobtainable finished items** — defined, fully wired, reachable through NO
  loot tier or enemy roll: `contactMod`, `slowFuse`, `boomerangMod`
  (~L3452-55), `refundMod` (~L3457), `barrelMagII` (~L3360). Registry
  one-liners into `lootTiers` per the data-driven invariant ('cores'/'rare'
  per the audit's split).
- **Permadeath-losable settings gadgets:** hideout `guaranteed` is still
  `['photometer','compass']` (~L2543) — extend with
  `'opticsDial','volumeKnob','manual'`. Those three seed exactly once via
  seedStashKit; a death while carrying one permanently removes
  settings/rebind/volume access from that save. The promise-kept dedupe
  already suppresses re-gifts.
- **Enemy-only five** (`serrated`/`splinter`/`homingMod`/`chaoticMod`/
  `sniperMagI` — enemy rolls only, loot them off corpses): owner call 15 says
  COMMENT them in ITEM_DEFS as hunt-rewards so the intent is recorded. Pending
  — rides the comment-debt commit (Backlog).

### 12. Horde gate has no diegetic terminal body  (spec-gap — NOT landed this wave)
- The spec locks "a terminal controls it — the same diegetic terminal", but
  the gate control is still an invisible ~6m interact disc on blank concrete
  (~L4374). The prompt/card-match/dead-zone fixes landed (d4a1989); the
  PHYSICAL slab did not — a player without the card in hand may never learn
  the gate exists.
- Fix: a small terminal-body prim near the inner face as a data-driven PROPS
  entry riding the LEVEL-PRIM twin path in BOTH regionSdf implementations — no
  special-case code; a distinct material/decal panel is the cheaper fallback.
  Chrome-budget check on merge (KNOWN_BUGS #2).

### 13. worldItem lifetime guards  (audit — NOT landed this wave)
- `w.age` accumulates across the item's whole life and only PAUSES while
  held/frozen/armed (~L9090): gear that sat 110s then was CARRIED ten minutes
  despawns seconds after being set down. Add `else w.age=0` to the exemption
  branch so release restarts the 2-minute clock.
- The cap-64 eviction (~L4275) omits the reap's `w.frozen` exemption and tests
  `_armT>0` vs the reap's `!=null` — a deliberately physgun-pinned (old) item
  silently vanishes when the 65th item spawns. Match the reap's predicate
  exactly.
- **Owner call 5 (approved, unbuilt):** EXEMPT the hideout from the 120s/50m
  reap — the safe-zone floor is de-facto storage and vanishing loot there
  reads as a bug. The 64-cap backstop stays.

### 14. Starter kit still can't showcase the game  (System J — owner said BUILD NOW, unbuilt)
- `seedStashKit` (~L7072-84) seeds two bare frames, three mags, and the seven
  settings utilities — zero cores, ammo boxes, heals, or armor vs the spec's
  "a couple of cores + a mod or two, a mag + ammo, a piece of armor, a heal".
  (Post-rework a racked mag DOES throw boltless plain lead — but nothing in
  the kit teaches the rack, and the energy path is absent entirely.)
- The constraining comment cites a 6×4 stash; `STASH_COLS/ROWS` is **6×8 = 48**
  (~L7213) — half the stash sits free. Audit's concrete seed (~20 cells):
  boltCore ×2, sparkCore + battS, dmgPlus, barrelC, ammo9 ×2, bandAid ×2 +
  ration, helmetCap + platesL, holster1. Fix the stale 6×4 comment in the same
  diff; playtest that autoPlace packs it.

### 15. Uniform upload batching  (perf — backlog-grade, NOT landed)
- Worst case ~235 scalar `gl.uniform*` calls/frame: element-wise loops for
  items/enemies/tracers/decals/dyn plus static gate/seed data re-sent every
  frame — on Chrome/ANGLE roughly 0.1-0.3ms/frame of pure driver overhead.
- The batched pattern is proven in-file (`_colsBuf` → one uniform4fv): flat
  Float32Array per uniform array, one `gl.uniform{3,4}fv` each, dirty-flag
  decals/vcrates/seed/gate. Mechanical but wide — convert ONE array per commit
  with a visual check between.

## Backlog (future — low priority)

- **Comment-debt — one sanctioned comment-only commit** (exactly the
  lockstep-grep trap BALANCE.md warns about; no code change): "brute keeps its
  300" ~L6860 (the ctor + reinforce set **800**); "RLIM.ENEMIES=16" ~L7334 and
  "revives to its full 16" ~L7361 (it is **20**; the horde math beside them
  already assumes 20); seedStashKit's 6×4-stash claim (6×8 now — rides #14);
  the ITEM_DEFS hunt-reward comments (owner call 15, see #11).
- **ENEMY AI REWORK** (owner-requested, v22i) — **LARGELY DONE (beta 1.1.x).**
  Soldiers line-patrol, seek cover, lay covering fire, hold squad discipline,
  walk their POI together; a whole second faction (zombies + brute) with swarm
  AI + 3-way targeting landed on top; this wave added zealot morale
  (flee/stand), faction-separated seeding, gas fear, and step-over parity.
  Still direct-steer (no navmesh — by design). Remaining polish if wanted:
  flanking, smarter cover rotation.
- **Crouch is camera-only** — no crawl/prone vocabulary exists anywhere in the
  movement code. Note it before designing anything that assumes one.
- **area1 literals in the generic POI seeding path** — harmless today; fix
  only when a second POI level is authored.
- **Enemy-sourced burn WATCHED** (mod wave v1, deferred knob): the zealot spark
  roll doses the player 15 burn ≈ up to 15 hp over 7.5s — real early-game bump.
  If playtests say too hot, the one-liner is halving enemy-sourced doses in
  `applyStatus`. See docs/MODS.md balance ledger.
- **ORB tracer-slot hoarding** — a live orb holds a tracer slot for its full 6s
  aloft (cap 4 orbs). Capped, flagged "monitor" in MODS.md; NOTE the pool now
  fills projectiles-first (6d9e498), so orbs starve pin needles before they
  starve bullets. Revisit if tracers starve during orb play.
- **v22m WATCHED knobs** (consciously deferred, all single-line fallbacks —
  details in docs/MODS.md ledgers): enemy CHAOS heavy hand (+45%/stack; knob:
  halve `chaosMul` for non-player) · bolt-thread TTK under the A5 burn=hurt law
  (knob: dose ×0.5) · boom-drum burn saturating the 30-pool in ~3s (⚡ deficit
  prices it) · enemy TIMER airbursts + DRILL pass-through + relay extras =
  collective difficulty bump (knobs: drop `slowFuse`/`pierce` from enemy
  rolls) · RETURN-arc TTK vs crowds behind low cover (knob: W 2.5→2.0 or reach
  14→11) · WAVE contact-drum tracer load (knob: cadence 0.5→0.7s). The old
  "REFUND collapses the lead economy" watch is CLOSED — the corrected model
  never waives lead (1b57cf9) and `surchargeHard` guards the rest (bc4445e).
- **Manual browser pass owed** — repointed: every owed check now lives in
  `docs/PLAYTEST.md` (§14 carries the firing-model list: mint gate, pulse
  feel/converge, drum beats at the dot, thread-degradation draw order, grapple
  swing, whip takedown reads).
- **slideStand** — deleted as dead (82d367a). If a hold-crouch-to-slide /
  release-to-stand verb is ever wanted, that commit marks the seam.

## Recently improved

- **beta 1.0.1** — Long first load froze the title card. One ~860-line fragment
  shader was compiling synchronously on the boot thread, so the intro art sat
  locked until the driver finished. Now the compile is non-blocking (parallel
  via `KHR_parallel_shader_compile`, polled off the boot thread) behind an
  honest, animated loading screen, and a cheap PREVIEW shader renders first so
  the player can move immediately while the full shader finishes and hot-swaps
  in behind it. Perceived first-load stall is gone.

## Recently fixed

- **feat/containment wave (2026-07-15)** — the P2 fix-up + audit sweep + P3–P7
  (61 commits). Every check the builders flagged is consolidated in
  `docs/PLAYTEST.md`. Highlights:
  1. **#0 carve-height twin drift CLOSED** — JS aligned to the GLSL (FENCE
     gate carve y 1.2→1.3, FACADE recess 1.2→1.25; was JS ~L2555-56/L2565-66
     vs GLSL ~L1075/L1081): collision now matches what renders, zero visual
     change (90aa1fd).
  2. **#4 brute head hitbox CLOSED** — `enemyHeadR` (brute 0.30, rank 0.24)
     covers the GLSL `d-=0.12` inflate at all five head-test sites (1c06127);
     P5 finished the job: body hits test ONE capsule hugging the drawn flesh,
     r 0.32 / brute 0.44 (2da355e).
  3. **#7 "gun-raise replay" CLOSED — the proposed fix chased a trigger that
     never existed.** Real mechanism: `menuSet('closed')` collapsed `uMenuMode`
     to 0 while `menuAnim` eased 1→0 over ~0.77s, and the GLSL routed mode-0 to
     the RACK-POSE gun prop settling down-screen with the worn gun gated
     behind the same ease — the "raise" was a phantom rack prop, and the same
     root drew a phantom rack gun behind the CRATE view. Fixed display-only in
     GLSL: explicit prop branches per mode, worn gate on `uMenuMode==0`
     (9f5ada6).
  4. **#8 spatial audio CLOSED** — `spatialOut` (distance gain + StereoPanner)
     shipped with `playSound(type,pos)`; the wave finished the coverage sweep:
     enemy fire + reload (the owner's literal complaint), die() thud, grenade
     tick, carve spark, LURE ping, ricochet/fuse clicks, item-drop bounce
     (4842abb). Player hit-confirmation + crackHelm stay CENTERED by design
     (hitmarker UX, owner call 14). One browser listen owed: pan SIGN + gain
     curve (PLAYTEST.md §6).
  5. **#6a lamp strobe CLOSED** — the 4-slot eviction now ranks by the same
     faded term the upload sends, so a dying muzzle flash stops evicting a
     steady lamp (947d3f2). **#6b exposure steps CLOSED** — a render-only
     smoothed copy feeds `uExposure`; AI/photometer keep the stepped field
     (0c40d2d).
  6. **P2 had shipped broken three independent ways — all fixed:** the
     outskirts spawner probed absolute y=1 inside 6-38m rim hills, so the
     gauntlet was SILENT → probes local ground now, spawns ramp 3s→0.45s,
     OUTSK_MAX=16 (c80d099); `stepOver` climbed 46-78° valley walls at full
     speed (containment breach past the sealed gate) + standing sawtooth → a
     vertical-gradient test refuses terrain at EVERY angle, ledges still
     mount, enemies share the fix (16318d6); the desk terminal never slept
     under other UIs — two E presses mid-bag could DEPLOY → sleeps under
     bag/manual/debug, auto re-wakes (8f3fc88) + six-seam lifecycle hardening:
     blind-confirm gate, snooze latch, world-swap close, cursor/prompt/facing
     (49d7ffb).
  7. **Containment seams:** horde gate re-locks every camp raid (4f00146);
     guaranteed-card placement randomized + ownership dedupe walks two-deep —
     no more duplicate compass/photometer (24facc5); gate matches its OWN card
     by defKey, prompt names it, open-gate dead zone gone (d4a1989); the
     gauntlet no longer drains the arena — wake bait is out-of-bounds corpses
     only (owner call 9), `_home` restore, 30s stand-down reap (867217d); the
     valley live far-yank was ALWAYS cap-blocked → cap gates dead respawns
     only (72cbbb0).
  8. **AI seams:** dormant zombie lunge gated — the invisible bite is gone
     (77541a6); the undead read noisePings — LURE works on them, radius capped
     at their 22m sense (7821c67, owner call 12); PERSONNEL RECORD counts only
     player kills via `_lastBy` (0c676bd, owner call 11); zero-distance
     retreat NaN guard (7364a41); `explode()` passes the source entity so
     blast victims retaliate cross-faction (00bf683); Believers excluded from
     the guaranteed-suppressor slot (ec43ecd); own-spot respawn defers while
     the player camps it (98314f8); patrol blockage tested along the final
     movement intent (e3c5fa9); soldier step-over equalized to the player's
     ~0.46m + face-pogo latched out; undead keep the 0.9m clamber as flavor
     (1de141f, owner call 6).
  9. **Movement:** plain falls now swept — the anti-tunnel start pose is
     captured before gravity, thin decks/roofs hold at 30fps (5640e5a); idle
     stance planted on 28-45° slopes, clamp 0.06→0.12 (58b66f2, owner call 8);
     a free-rope landing cashes into the armed slide — the momentum chain
     closes (0ec21fd, owner call 7).
  10. **Perf/render:** dormant enemies slow-tick 4:1 with banked dt (e55b2e9);
     soldier sightline march range-gated (6696a47); dormancy ranking cached +
     25m² hysteresis at the cap boundary (4704ea5); top hot-path allocations
     onto the scratch idiom (ac45862); desk-terminal forced reflow killed
     (2a989cf); HUD pips/scopeGlass DOM dedup (c22a4ac); world-item cull rides
     the VIEW dial (77e42a5); auto-res recovers on 60Hz panels — grow at
     fpsEma>58, wall-time cadence (b4f82c3); projectile tracers claim the
     12-slot pool before pin needles (6d9e498); full-program prewarm draws
     (654881b).
  11. **UI truth:** terminal meta text to the 14px floor + diegetic scale
     floor 0.55→0.8 (95126b5); enum dials show off-preset values raw instead
     of masquerading as the first label (5afd480).
  12. **Dead code out:** ground-scatter remnants (f9fdb0f), showTooltip_DEAD +
     the unreachable locked-widget UP-step (33e4956), the never-true
     slideStand flag (82d367a), the dead j.wielded/_wield save pair (ad235fd).
  13. **Features landed (not fixes):** P3 gas — masks/filters/face slot
     (47fdb31), radiation field + outskirts gas + brute aura + HUD (b65bb23),
     soldier burn/fear + undead crowd the haze (2229606), green haze fog GLSL
     (d73840b) · P4 — unique lab/testing keycards (fdedc19), regionLoot caches
     (73a7499), death-proof lockboxes + brute drop (b7c6a1d) · P5 — capsule
     hit tests (2da355e), symmetric tiered headshots (0c0dc37), spawn y-cap
     (2a9d5b5), faction-separated seeding (054e5ed), zealot morale (c398faa) ·
     P6 — five new POI crate clusters (833460b), per-raid crate rotation
     (8ea4d3c) · P7 — free-aim lead/trail (b9ccf28), gun lean + DOM crosshair
     + iron-glyph handoff (8318dfb), FREEAIM goggles row (b0af3a8, f4cb1bf).

- **beta 1.1.x combat pass (2026-07-14, 3c2430b…bc4445e — pre-wave)** — the
  powder/refund/ammo rework MODS.md spec'd: two-pool model (boltless mags fire
  plain lead PARALLEL to energy cores), platform columns fire their own mags,
  powder/refund corrected to the model (coreless synth + refund round-waiver
  DROPPED), wing own-mag billing + powder `surchargeHard`. Browser pass owed —
  PLAYTEST.md §14.

- **beta 1.1.x (the horde arc)** — a run of fixes landed with the ballistics +
  zombie work: (1) **material-ID band overflow** — raising the enemy cap made
  render-slots past 10 shade as tracer blobs; the band is now templated on
  `RLIM.ENEMIES`, so a cap change can't overflow again. (2) **Enemy-optic leak** —
  sights/reticles were drawn on EVERY gun from the *player's* uniforms, so dropped
  + enemy guns wore your optic; gated behind `cfgd`. (3) **Powder bare-fallback
  discard** — a core-less Powder-Mag tool fired a fallback 9mm and threw away its
  own round; the fallback now honors a loaded powder mag. (4) **DRILL+HOPE
  infinite self-heal** — a piercing heal mote re-healed you every substep; it now
  spends itself on the body it mends. (5) **Forest understory desync** — JS
  collided with bushes the GLSL never drew (a clobbered TREES shrub flag); the
  bushes render now, so the cover is honest. (6) **Enemy night-vision over-nerf**
  — fog was blinding enemies after dark; fog no longer factors into enemy sight.
  (7) Powder **C3 refund waiver** was vacuous (never reached a real carrier) —
  rewired onto the powder lane, then RETIRED with the corrected model (see the
  combat pass above).

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
