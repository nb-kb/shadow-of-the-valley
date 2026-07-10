# Known Bugs — Shadow of the Valley

Living list of open issues, worst-first-ish. Fixed items move to the changelog
in commit history. Line numbers drift — treat them as hints, re-grep before
editing.

Last reviewed: 2026-07-09 (build v22e).

## Open

### 0. Carve-height twin drift family + authored-level edit skip (latent)
- **Found by the v22e hideout probe.** Same pattern as the fixed gate lintel:
  FENCE gate carve JS y-center 1.2 vs GLSL 1.3, FACADE door recess JS 1.2 vs
  GLSL 1.25 (area1) — sub-10cm invisible ledges, imperceptible today.
- **Latent landmine:** GLSL `mapStatic` early-returns `levelSDF` for authored
  levels, skipping the `uEdits`/`uGlobs` loops JS still applies — a PICK carve
  fired inside an authored level would change collision invisibly. Both are
  cleared on deploy, so it cannot bite until someone carves indoors.

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

- Endless-valley crates are currently **decorative** — wire them into the real
  crate/loot system so they can be looted.
- **Loot tables** should offer the full suite of available items — audit
  `lootTable` / `lootRegions` for coverage.

## Recently fixed

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
