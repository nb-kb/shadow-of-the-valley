# Known Bugs — Shadow of the Valley

Living list of open issues, worst-first-ish. Fixed items move to the changelog
in commit history. Line numbers drift — treat them as hints, re-grep before
editing.

Last reviewed: 2026-07-08 (build v21q).

## Open

### 1. Chrome load crash — "cannot access gameState before initialization"
- **Where:** initial top-level script exec (TDZ). Not the earlier `Menus`
  forward-ref, which is already fixed — this is a separate one.
- **Repro:** load in Chrome, often after a GPU overload. **Zen/Firefox run
  clean.** Chrome is the odd one out.
- **Status:** UNRESOLVED. Needs a browser to pin the exact line / call path.

### 2. Invisible enemies — shader slot cap
- **Cause:** shader renders at most 10 enemies (`RLIM.ENEMIES=10`, ~line 624;
  render gate ~line 1153), but the JS spawns up to 22 (facility, ~line 1863) or
  14 (raid, ~line 4216). The upload sends only the nearest-10 live enemies.
- **Symptom:** enemies ranked 11+ are invisible yet still shoot you and eat your
  bullets (JS combat loops are uncapped). Which enemies are drawn flickers with
  a per-frame distance sort.
- **Note:** enemies are *never* part of the player's movement-collision SDF by
  design — "solid" means bullet/LOS interaction only, not physical blocking.

### 3. Crate opens and instantly grabs the top item
- **Cause:** the crate opens on `pressed('interact')` in `updateInteract`, then
  the same frame `menuNav()` (crate now counts as a menu) reads the same
  `pressed('interact')` edge and runs `interactInventory()` → take.
- **Fix:** consume/clear the interact edge on open (e.g. in `openCrate`), or a
  one-frame guard.

## Backlog (future — low priority)

- Endless-valley crates are currently **decorative** — wire them into the real
  crate/loot system so they can be looted.
- **Loot tables** should offer the full suite of available items — audit
  `lootTable` / `lootRegions` for coverage.

## Recently fixed

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
