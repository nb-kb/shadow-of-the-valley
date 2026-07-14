# PATCH: AREA-1 CONTAINMENT — content plan (draft, 2026-07-13)

Area-1 becomes a **walled, contaminated arena**: sealed borders, a keycard
horde-gate, radioactive POIs, locked labs, faction warfare, and death-proof stash
upgrades. This is the **next patch after beta 1.1.x ships** — planning only, no
code yet. Owner-driven; leans marked, decisions flagged.

---

## Systems

### A. The Walled Arena  · world-SDF TWIN · budget · **risk: HIGH**
- Raise Area-1's perimeter (`L.bounds`) into cliff/wall geometry — impossible to
  walk off the map. GLSL world-SDF **and** JS collision in LOCKSTEP (the #1 twin
  rule — every coord change mirrored, or you clip through / hit air).
- Leave ONE incline open, sealed by the **horde gate** (System C's keycard).
- Budget: raising the *existing* boundary (taller, no new interior primitives)
  should be compile-cheap — CONFIRM at build + Chrome load-test.
- Enemy spawn y-cap (System E) keeps enemies off the new walls.

### B. Radiation / Gas  · JS + shader-tint + HUD · **risk: MED**
- Radiation zones: **ambient burn** (DoT into the burn pool), a **toxic/warning
  HUD symbol**, a screen tint while inside.
- The **non-selected POIs** (not chosen as soldier clusters this raid) get the
  radioactive haze — dynamic per raid; **zombies spawn there in higher
  concentration**.
- The **outskirts** (out of bounds) carry the gas, and **only zombies spawn near
  the player** there.
- Telegraph: haze readable from outside so avoiding/entering is a real choice.
- **Who it burns:** the player AND **soldiers** — both need masks. **Zombies are
  immune** (undead). Soldiers can spawn with a mask as random starting kit —
  masked soldiers hold the haze; **unmasked soldiers FEAR it and avoid** the gas
  (a morale/avoid behavior, ties to System E). Emergent: the hazed POIs become
  zombie turf (soldiers steer clear), reinforcing the faction map. (RESOLVED gap #5.)
- **The brute carries radiation** — a mobile haze aura trails the roving brute,
  making the mini-boss a walking hot-zone (pairs with its roving spawns).

### C. Keycards & Locked Areas  · JS · **risk: MED**
- Keycards = loot, stashable, one distinct card per lock (color-coded).
- **Horde gate** (the open incline) → its own card.
- **Weapons Testing** + **Weapon Lab** doorways → locked, a UNIQUE card each.
- Behind locked doors: contents rolled per raid — sometimes zombies, sometimes
  soldiers, sometimes ALL Believers. Loot reward inside = the reason to breach.
- Keycard-on-death tension is the point: carry it in to use it (teeth may eat it)
  vs bank it. Keep that risk/reward.
- UX: a locked-door prompt naming the card; card art matches its lock.

### D. Persistent Lockboxes  · SAVE_VER bump · **risk: MED-HIGH**
- **Epic** lockbox 2×2 · **Legendary** lockbox 3×3 — contents SURVIVE death (the
  hideout-stash rule). Safe-storage upgrades, anchored in the hideout.
- **Acquisition: killing the BRUTE is a chance to drop one** (epic or legendary)
  — makes the mini-boss the stash-upgrade source. (RESOLVED gap #6.)
- Save: one SAVE_VER bump folding in lockboxes + keycards + new raid-state
  (which POIs hazed, gate open). Old saves: corruption reads as empty, never
  crashes (invariant #4).

### E. Enemy Spawn / AI tuning  · JS · **risk: LOW**
- No spawn past a set **y-elevation** — stops off-map wandering and keeps enemies
  off the raised walls.
- Soldiers vs zombies spawn **far apart** → each coheres with its own faction
  (the cohesion already built) before finding + clashing with the other. ("Team
  up" read as own-faction grouping first.)
- **Zealot morale** — under set conditions (low HP · their faction losing ·
  outnumbered · no nearby cover) a zealot can **flee** or **stand its ground**
  instead of always pressing. Believers never flee (kamikaze); zombies never
  flee (mindless). Reads as soldiers, not drones.
- **Brute as a spawn anchor (maybe)** — the roving brute could seed roving zombie
  spawns around itself → a mobile pressure source. Optional; ties to the gauntlet.

### F. Hitbox / Headshot rework  · combat · **risk: MED**
- Tighten the JS hit radii to the drawn body FIRST — fairness is the PREREQUISITE
  (an instakill only feels fair on an honest box; folds in the open brute-hitbox
  note + System H's crosshair hit-test).
- Headshots are **SYMMETRIC — player AND enemy, a lucky shot ends either side:**
  **no helmet = 1 (instakill)** · **light / tier-1 = 2** (1 breaks it, 1 kills) ·
  **heavy / tier-2 = 3** (2 break it, 1 kills). The player wears the same rules —
  lucky enemy headshots will sometimes just kill you. (RESOLVED gap #2.)

### G. Loot-crate expansion  · JS + maybe geometry · **risk: MED**
- Every POI gets loot crates → add more crate instances.
- Positions / which are live **rotate raid to raid** (like the POI rotation).
- CONFIRM the crate render + `RLIM.ITEMS` budget as the count climbs.

### H. ADS independent aim  · viewmodel/camera · **risk: LOW-MED**
- ADS: gun + crosshair move smoothly, INDEPENDENT of the camera; the camera
  trails on a small delay. Hit-test must follow the **crosshair**, not the camera
  center (interacts with F).

### I. Outskirts stats page (PRs)  · terminal + save · **risk: LOW**
- A page on the hideout computer tracking **zombies killed** and **time in the
  outskirts** as personal records — the score-chase that makes the suicide run
  its own reward. A terminal row + a couple of persistent counters (fold into the
  Phase-1 SAVE_VER bump). Builds with Phase 2.

### J. Robust starter kit (onboarding)  · JS · **risk: LOW**
- The player's starting loot should be more robust so they can SEE what the game
  offers — a spread that showcases the systems (a couple of cores + a mod or two,
  a mag + ammo, a piece of armor, a heal, the key utilities) instead of a bare
  hand. Discovery through the kit. NOT Containment-themed — a general onboarding
  pass; strong candidate to bundle into the **1.1.x wrap-up** so testers see the
  systems now.

---

## The Gate & the Outskirts Gauntlet (ties A + B + C)
- The gate sits on a **quiet edge** — an area with little going on right now.
- A **terminal** controls it, reading **(LOCKED)** / **(OPEN)** by keycard — the
  same diegetic terminal, card-gated.
- Opening it lets the player walk OUT into the **radioactive outskirts**: a
  **suicide zombie gauntlet** — gas burn + zombies spawning near you at
  **escalating frequency the longer you linger**. The clock is the tension:
  overstay and you're swarmed. (It is NOT the way home — the interior exfil is.)
- Needs a staged **reward** out there (a loot cache / a Legendary-lockbox source /
  rare cores) so the run pays — otherwise it's chaos for its own sake. ← DECISION

## Gas masks — TWO varieties (spans B + D + F)
Two ways to breathe the haze, each a different head-protection trade:
- **Helmet mask** — a **tier-1 helmet** with a built-in respirator: light-tier
  head armor (2-shot headshot) + gas, **2 filter slots** (≈ −90% burn). Takes the
  helmet slot, so you forgo a heavy (tier-2) helmet for max gas safety.
- **Goggles mask** — a **face-slot** respirator that stacks WITH a tier-2 helmet,
  so you keep the 3-shot head protection, but only **1 filter slot** (≈ −45% burn).
  (Build detail: confirm the face slot is new vs the existing goggles-device slot.)
- Filters: each **−45% radiation burn**, container sub-inventory (like the Scope
  Adapter), common loot, and they **degrade slowly in the gas** (~1–2 casual runs,
  then spent — a consumable loop). (RESOLVED gaps #4 + #7.)
- The trade in a line: **max gas safety (helmet mask, 2 filters) costs you heavy
  head armor; max head armor (tier-2 + goggles mask) costs you a filter slot.**

---

## EXECUTION PLAN (phased: foundation → feel)

**Ground rule:** ship 1.1.x FIRST — load-test cap-20, merge `beta-1.1.1` → live —
then branch Containment off the shipped base. Never stack it on unshipped work.

**Phase 1 — Foundation: save + walls**  · manual, careful · **HIGH risk**
- ONE SAVE_VER bump scaffolding all new persistent state (lockboxes, keycards,
  raid-state: hazed POIs / gate-open). Migration: old → empty, never crash.
- Raise the border SDF twin (GLSL + JS lockstep) + the gate opening on a quiet edge.
- **Chrome + Optimus load-test HERE** — the one twin/budget checkpoint of the patch.
- ✅ Done: walls render + collide (twin synced), old saves load, can't walk
  off-map, the gate gap exists.

**Phase 2 — The gate + outskirts gauntlet**  · manual / small workflow · MED
- Gate terminal (locked/open by keycard) + the keycard item (loot + stash).
- Outskirts beyond: ambient gas burn + escalating zombie spawns (ramps with
  time-outside) + the staged reward.
- ✅ Done + **PLAYTEST**: keycard → open → walk out → radiation + escalating
  swarm + payoff. This is the core new loop.

**Phase 3 — Radiation across the map + gas mask**  · workflow-friendly · MED
- Haze on the non-selected POIs (per raid) + higher zombie concentration + the
  toxic HUD symbol + screen tint.
- Gas mask (light armor, 2 filter slots, −45%/filter) + filters as common loot.

**Phase 4 — Locked labs + lockboxes**  · workflow-friendly · MED
- Locked Testing + Lab doors (unique keycards); contents FIXED first, roll later.
- Epic/Legendary lockboxes (2×2 / 3×3, death-proof) — drop into the Phase-1 scaffold.

**Phase 5 — Combat + spawn tuning**  · workflow + adversarial verify · MED
- Hitbox tighten + headshot/helmet rules (no-helmet instakill · light 1-shot ·
  heavy 2-shot) + enemy spawn y-cap + faction separation.

**Phase 6 — Loot crates + rotation**  · workflow · MED
- More crate instances (every POI) + raid-to-raid rotation. Budget-check `RLIM.ITEMS`.

**Phase 7 — ADS independent aim**  · manual feel · LOW-MED, splittable
- ADS free-aim (gun/crosshair independent, camera trails) + crosshair hit-test.
  Its own feel pass — first to cut/defer if the patch runs heavy.

**Parallelism:** Phases 1–2 are sequential + careful (the twin + the core loop).
Once Phase 1 lands, **3 / 4 / 5 / 6 are largely independent** — build them as
parallel workflows (each adversarially verified) and merge. Test gates: after
Phase 2 (the loop) and after Phase 5 (combat feel); the Chrome/Optimus load-test
sits after Phase 1.

---

## GAPS / decisions to resolve BEFORE building ("what am I missing")

1. **EXTRACTION — RESOLVED.** The interior **exfil stays the exit**; the gate
   opens OUT to the outskirts gauntlet, not the way home. Walls just stop
   map-edge wandering. (Just confirm the exfil still sits inside the new walls.)
2. **Headshots — RESOLVED (F):** symmetric player+enemy · no helmet 1 / light 2 /
   heavy 3 · gated on honest hitboxes.
3. **Outskirts reward — RESOLVED-ish:** the PR stats page (I) is the score-chase
   reward + the brute drops a lockbox chance; a staged loot cache is optional gravy.
4. **Filters — RESOLVED (Gas mask):** degrade slowly in the gas, ~1–2 casual runs.
5. **Radiation vs whom — RESOLVED (B):** player + soldiers need masks; zombies
   immune; unmasked soldiers fear + avoid the haze; the brute carries an aura.
6. **Lockboxes — RESOLVED (D):** brute-kill drop chance, hideout-anchored.
7. **Gas mask ↔ helmet slot — RESOLVED (two varieties):** **helmet mask** = tier-1
   helmet (2-shot) + gas, 2 filter slots (head slot). **Goggles mask** = face-slot
   respirator that stacks with a tier-2 helmet (keeps 3-shot), 1 filter slot. Gas
   safety vs head armor is the build choice. *(All 7 gaps now resolved.)*

---

## Scope-creep guardrails

This is ALREADY a large patch (8 systems + the gas-mask item). If it balloons,
cut in this order:
- **tree → fence accents** (out — the walls replace the need).
- **variable-content roll** — ship the locked rooms with FIXED contents first,
  add the per-raid roll later.
- **ADS free-aim (H)** — split into its own feel patch; it's orthogonal to the
  arena theme.

Still OUT, separate patches (do not fold in): the **powder/refund/ammo combat
rework** (spec'd in MODS.md), **HOPE's interaction row**, **mod-attachment
rendering**, the **stealth map + bake-cap unlock**.
