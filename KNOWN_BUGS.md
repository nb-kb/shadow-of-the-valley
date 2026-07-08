# Known Bugs — Shadow of the Valley

Living list of open issues, worst-first-ish. Fixed items move to the changelog
in commit history. Line numbers drift — treat them as hints, re-grep before
editing.

Last reviewed: 2026-07-08 (build v21p).

## Open

### 1. Chrome load crash — "cannot access gameState before initialization"
- **Where:** initial top-level script exec (TDZ). Not the earlier `Menus`
  forward-ref, which is already fixed — this is a separate one.
- **Repro:** load in Chrome, often after a GPU overload. **Zen/Firefox run
  clean.** Chrome is the odd one out.
- **Status:** UNRESOLVED. Needs a browser to pin the exact line / call path.

### 2. Area 1 — armory / forest ("lumber yard") collision off
- **Where:** the armory building (`BUILDING`, box `[2,102,20,116]`, door `S`,
  ~line 1832) and the adjacent forest / logged-stumps region (~line 1834).
- **Symptom:** collision inaccurate in that corner of Area 1 — the one spot
  left after the v21p facade/fence doorway fix.
- **Note:** the armory door offset is *relative* (0), which should already be a
  correct GLSL↔JS twin — so this is a **different cause** than the facade fix.
  Unclear yet whether it's the armory walls/door, the forest `TREES`/stumps
  collision, or the timber props. **Uninvestigated** — needs in-browser pinning.

### 3. Jump / landing feel (regression from the fort-jitter work)
- **Symptom:** landing is rigid; the descent "skips a few pixels."
- **Cause:** camera Y-smoother (~line 5364) hard-snaps when the body moves
  >0.6 m in a frame, and landing does an exact snap to the ground (~line 5358).
- **Fix (small, global — helps both levels):** track the body 1:1 while
  airborne (gait-smoothing is only for footfalls); ease the landing snap.
- **Priority:** low / not urgent.

### 4. Invisible enemies — shader slot cap
- **Cause:** shader renders at most 10 enemies (`RLIM.ENEMIES=10`, ~line 624;
  render gate ~line 1153), but the JS spawns up to 22 (facility, ~line 1863) or
  14 (raid, ~line 4216). The upload sends only the nearest-10 live enemies.
- **Symptom:** enemies ranked 11+ are invisible yet still shoot you and eat your
  bullets (JS combat loops are uncapped). Which enemies are drawn flickers with
  a per-frame distance sort.
- **Note:** enemies are *never* part of the player's movement-collision SDF by
  design — "solid" means bullet/LOS interaction only, not physical blocking.

### 5. Crate opens and instantly grabs the top item
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

- **v21p** — Facade/fence doorway twin desync: `FACADE`/`FENCE` door offsets are
  authored absolute, but GLSL's `lvHole` added the region center, flinging the
  carve ~96 m off the wall (facade/fence rendered solid while collision had the
  opening). Fixed in `bakeLevel` by pre-subtracting the center.
