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
  hideout-stash rule). Safe-storage upgrades.
- Save: one SAVE_VER bump folding in lockboxes + keycards + new raid-state
  (which POIs hazed, gate open). Old saves: corruption reads as empty, never
  crashes (invariant #4).

### E. Enemy Spawn / AI tuning  · JS · **risk: LOW**
- No spawn past a set **y-elevation** — stops off-map wandering and keeps enemies
  off the raised walls.
- Soldiers vs zombies spawn **far apart** → each coheres with its own faction
  (the cohesion already built) before finding + clashing with the other. ("Team
  up" read as own-faction grouping first.)

### F. Hitbox / Headshot rework  · combat · **risk: MED**
- Tighten the JS hit radii to the drawn body (fairness; folds in the open
  brute-hitbox note).
- Headshots: **no helmet = instakill** · **light helmet = 1 shot destroys it** ·
  **heavy helmet = 2 shots** (then a bare-head follow-up kills).

### G. Loot-crate expansion  · JS + maybe geometry · **risk: MED**
- Every POI gets loot crates → add more crate instances.
- Positions / which are live **rotate raid to raid** (like the POI rotation).
- CONFIRM the crate render + `RLIM.ITEMS` budget as the count climbs.

### H. ADS independent aim  · viewmodel/camera · **risk: LOW-MED**
- ADS: gun + crosshair move smoothly, INDEPENDENT of the camera; the camera
  trails on a small delay. Hit-test must follow the **crosshair**, not the camera
  center (interacts with F).

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

## Gas mask (spans B + D + F)
- **Light armor** occupying the head slot, with a 2-slot sub-inventory for
  **filters** (container-armor, like the Scope Adapter). Common loot.
- Each filter **−45% radiation burn** (two ≈ −90%; near-immunity is the reward
  for the slots).
- **It is a mask, not a combat helmet** → wearing it gives gas protection but
  (per F) **no headshot protection**. Gas-safe vs headshot-safe is the trade.

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
2. **Headshot-instakill symmetry** — does an enemy headshot **instakill the
   player** too? Player-only, or does the player's own helmet gate it? A brutal
   call either way — make it deliberately.
3. **Outskirts reward** — the gauntlet is opt-in content; it still needs a payoff
   staged out there (loot cache / Legendary-lockbox source / rare cores) so the
   suicide run is worth the risk, not just chaos.
4. **Filter economy** — do filters **deplete** in the gas (a consumable resource
   loop, like ammo), or are they permanent −45% each? Depletion gives the gas
   teeth over time.
5. **Radiation affects whom** — player: yes (needs a mask). Zombies: immune
   (undead). Soldiers: immune, take burn, or avoid the haze? Decide — it changes
   how the hazed POIs play.
6. **Lockbox home + acquisition** — loot tier, and do they live in the hideout
   (stash upgrade) or travel with you? "Survives death" implies hideout-anchored.
7. **Gas-mask-vs-helmet slot conflict** — both want the head. Confirm masked =
   no headshot protection (the trade in the Gas mask section), so the player
   chooses gas-safety or head-safety per raid.

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
