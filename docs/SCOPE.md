# Shadow of the Valley — Scope & Design Plan

Status: **owner-ratified.** Rewritten 2026-07-08 after a vision session with the
creative director; this version supersedes the earlier "finite campaign" draft.
Written against build `v21r` (`index.html`, ~5,840 lines). Agents implement; the
owner decides. Anything in **§8 Open Decisions** is a question, not a plan.

---

## 1. What this game is

> **A simulation of a game.** — the owner

It does not need a campaign, an ending, a story, or a progression economy to
justify itself. It needs a complete, honest loop that feels like a whole game
lives inside it — the way a snow globe holds a whole world. It is an
**extraction game with real persistence**, not a roguelike: what you bank is
safe forever; what you carry is always at risk.

### The loop

```
  HIDEY HOLE  →  THE VALLEY  →  AREA 1  →  THE VALLEY  →  HIDEY HOLE
   (bank it)     (hoof it)      (raid)      (haul it)      (bank it)
```

- The **hidey hole** is home: terminal, save, and a **persistent stash**.
- The **valley** is not a separate endless mode — it is **the land between
  places**. Every trip out and every trip home is a walk through it. The
  existing rings tech becomes travel legs, re-seeded per trip.
- **Area 1 (The Camp)** is the destination: the shoot-and-loot raid.
- **Teeth:** go down in the valley and your carried loot is **just gone**. No
  corpse run (owner-rejected: "too robust for the core loop"), no recapture.
  Banked loot is never at risk. That split — banked versus carried — is the
  entire tension of the game.

### The three pillars

What a player should walk away saying:

1. **"The shadows I could hide in were so well telegraphed."**
2. **"I've barely scratched the surface of the weapon customization."**
3. **"The UI is so clean and smooth."**

Note what is *not* a pillar: story, ending, enemy archetypes, music. Those are
either cut or strictly supporting cast.

### The telegraphing rule (universal)

**Information lives on the thing itself.** This is the design contract that
unifies all three pillars:

- **Enemy armor renders on their bodies.** One glance tells you what they
  resist.
- **Weapon mods render on guns — theirs AND yours.** Read an enemy's offense
  before it reads you; your own tool is as legible as theirs.
- **Shadows read in the world**: deep shadows are visibly deep, and the
  player's hands darken as they slip into cover. Backed by a **light-meter
  gadget** — an inventory item that states the literal distance you'd be
  spotted from. The sim already computes this honestly (exposure ×
  `enemyVision`), so the meter tells the truth, not a vibe.
- Nothing important is hidden in menus or dice rolls.

---

## 2. Where the code stands

From a full source sweep (unchanged from the survey; still accurate):

| Layer | State | Notes |
|---|---|---|
| SDF world + GLSL/JS twin | **Deep** | collision, LOS, physics all run off the CPU twin |
| Light-based stealth sim | **Deep** | `updateExposure` (2260) marches a real ray to the sun; exposure multiplies detection (4077). The crown jewel — pillar 1 is mostly *feedback* work, not sim work. |
| Noise propagation | **Done** | suppressors genuinely matter (22→4) |
| The Tool (sequencer) | **Deep** | column tree, volley fire, battery wear, 20 mods |
| Inventory / crates / magazines | **Done** | hand-hold → open → place; real containers |
| Save system | **Built** | versioned, 3 slots, corruption-safe — but see §4 Patch 0: owner reports it failed in practice; needs repro. Saves only worn/carried gear; **no stash**. |
| Day/night | **Done** | wired into stealth |
| Audio | **SFX only** | no music (not a pillar; stays out of scope) |
| Enemies | **Placeholder** | one archetype, direct steering, no gear rendering |
| Damage model | **A single scalar** | no types, statuses, headshots, durability |
| Levels | **3** | `area1`, `hideout`, `valley` |

**The gap between §1 and this table is the plan.** Everything below closes it.

---

## 3. The work — patches + four updates

The owner's framing and ordering: *"Tool mods to get statuses designed first,
armor and healing for implementation of status defense, and enemy to apply all
of it at random or in classes, and a way to endlessly play."* So: the mod wave
**designs the statuses on the offense side**, armor/healing builds the defense
against them, and the enemy update applies the whole stack. The endless play is
the loop itself (§1) — round trips forever, no ending required.

### Patch 0 — Stability & clean UI  `S`
The pillar-3 work, and the bugs that poison playtesting.

- **Chrome TDZ crash** ("cannot access gameState before initialization").
  The game must load everywhere. Nothing else matters until it does.
- **Input edge ownership.** `INPUT.frameDown` is cleared once per frame (5803),
  so one press fires every handler that reads it — this *is* the crate
  double-take bug (open at 3277, taken again at 5520). Fix the model (consume
  on read); the bug class disappears.
- **Consolidate the three d-pad readers** (`menuNav` 5460, `termNav` 4493,
  `dbgNav` 5224) and the four bespoke menu-open verbs (5547–5550) into one
  table-driven system.
- **Save repro.** The owner tried to save and it didn't work. The code path
  reads sound (`saveGame` 4550), so this is either a real bug or a UX failure —
  and if the author can't tell whether a save happened, pillar 3 is failing
  either way. Reproduce, fix, and make save feedback unmistakable.
- **Enemy legibility**: `area1` spawns 22 zealots against `RLIM.ENEMIES:10`
  (625) with uncapped AI/combat loops — invisible enemies shoot you. Dormancy
  rule (unrendered = passive) or cap alignment.
- **Edge bleed** (root-caused): `map()`'s bounding-sphere early-outs carry the
  previous hit's material (`res = vec2(max(bd,0.1), res.y)`, 1134/1158/1179).
  Fix the material carry.
- Settings stubs: wire or delete the 7 non-functional dials (2672–2682).

*Exit: loads in Chrome, one input model, saves visibly work, every enemy
shooting you is visible.*

### Update 1 — The Loop  `M`
Completing the game. The valley becomes travel; the stash makes it matter.

- **Round trip**: deploying from the hideout routes *through* a valley leg to
  Area 1; exfil from Area 1 starts the **walk home** through a fresh leg
  (re-seed the existing rings tech per leg) instead of teleporting.
- **The hidey-hole stash**: a persistent home container whose contents save.
  Extend the save schema (bump `SAVE_VER`, keep corruption-safe behavior).
  The loop becomes: raid → haul → bank → sortie with only what you'll risk.
- **Teeth**: down in the valley = carried loot gone, wake at the hidey hole,
  equipped rig keeps (owner may tune what "carried" covers — see §8).
- Retire "DRAGGED YOURSELF HOME" free-heal-keep-everything (4268).
- Balance note: with persistence, loot tables must respect banked wealth —
  an extraction economy, not roguelike per-run scaling.

*Exit: the full loop — out, raid, home, bank — is playable and losing hurts.*

### Update 2 — Tool Mod content wave (statuses designed here)  `M`
Pillar 2's payload, and the birthplace of the status system: damage types enter
the game on the **offense** side, as mods that inflict them.

- **Damage types HURT / BURN / BLEED** land here — the type tag on shots, the
  per-entity status containers (`Zealot.hurt` 4036 has none today; `stagger`
  is the pattern to copy), and the mods that inflict them: fire core (burn),
  serrated (bleed).
- **FUSE-relay** — *"cast the next core in the sequence from its point of
  death/collision."* Makes the sequencer recursive; the single largest
  depth-per-line win in the project.
- The **positional trio**: CONTACT, TIMER, LOOP/WRAP.
- The **defensive class** (shield, smoke, decoy, reactive) — prey needs outs.
- **RECON/scan** (HUD overlay, cheap, feeds the goggles + stealth).
- **Barrel retune (owner-ratified): 3 / 5 / 7 / 9 / 11** slots across rarities
  (start at 3, +2 per tier). Code ships 3/6/9/12/15 (`BARREL_TIERS` 2329) —
  straight table change.
- **Mods render on the player's own tool** (the universal telegraphing rule).

*Exit: statuses exist and hurt, and two players' tools look different, act
different, and both feel barely scratched.*

### Update 3 — Armor & Healing (status defense)  `M`
The defense against everything Update 2 introduced. Fully specced in the
design doc (§J).

- **Status-resist rolls** on armor — the counterpart to burn/bleed mods.
- **Helmet/plate durability** and destruction.
- **Headshots** ×2 with **helmet interception** — the seam is already flagged
  (comment 3949, dormant `medPort` 2647); hit-tests are currently one sphere.
- Healing items: **gummies** (+30), **bandAid** (clears bleed), **aloe**
  (clears burn). Only `ration` and `cyanide` exist today.
- Health bar UI: burn grayout, bleed indicator.
- Save: durability and statuses persist (`SAVE_VER` bump).

*Exit: every offense from Update 2 has a defense, and armor is a real choice.*

### Update 4 — Enemies (apply all of it)  `M`–`L`
The rework: enemies get the full Update 2+3 stack, **applied at random or in
classes** (owner: either is acceptable — see §8):

- **Randomized armor** → randomized resistances. **Renders on the body** —
  reading an enemy tells you how to fight it.
- **Randomized weapon loadouts** → randomized offense. **Mods render on their
  guns.** (Half-built already: enemies roll tools from the player's mod pool,
  `buildTool` 4004; deeper rings roll better parts.)
- Enemy body models replace placeholders (SDF work — the one place "model it"
  is in scope, because telegraphing demands it).
- Loot follows gear: they drop what they visibly wore.
- Keep: no pathfinding. Author patrol routes as waypoints; `settle()` handles
  local collision.

*Exit: no two patrols read the same, and reading them is how you beat them.*

### Shadowcraft (small, independent — slot in any time after Patch 0)  `S`–`M`
Pillar 1's feedback layer. The sim is done; the *seeing* isn't.

- Deepen rendered shadow contrast so cover is visibly cover.
- **Darkening hands**: the player's hands/tool dim with their exposure value.
- **The light-meter gadget**: inventory item stating the true spot-distance,
  computed from the live exposure sim. Fits the compass/watch gadget family.

---

## 4. Explicit non-goals

Most are owner decisions from the vision session; the rest are engineering.

- **No campaign, no ending, no story arc, no quests, no NPCs.** Owner's call:
  the loop is the game. (The earlier draft's "The Escape" campaign is dead;
  the "memory" theming survives only as free flavor in existing UI copy.)
- **No corpse runs, no recapture.** Death in the valley = carried loot gone.
- **No roguelike balancing.** Persistence is real; balance around the stash.
- **No music system** — not a pillar. Revisit only after all four updates.
- **No pathfinding/navmesh.** Waypoint patrols.
- **No `TUNNEL` mod** — unbounded `uEdits` cost vs. a hard cap of 16 (1680).
- **No "model everything."** Model what telegraphing requires: the Tool + arms,
  enemy bodies + gear, mods on guns. Nothing else (no backpack model, no molar).
- **No splitting `index.html`** (CLAUDE.md invariant) — revisit at ~8k lines.

---

## 5. Engine debt & hazards

- **The GLSL↔JS twin is the central hazard.** Every world-geometry change is
  two changes. Live landmines: (a) any `*<big>|0` in a JS hash mirror must be
  `Math.imul` (f64 rounds past 2^53 — this cost the forest); (b) the valley
  crate hash exists in **three** hand-copies (`cellSDF` 760, `mapWorldBase`
  1630, registry 4351). *Mitigation:* the node-probe technique — evaluate the
  JS twin offline, measure clearance vs. the 0.34 m player radius — before
  every twin change, not after.
- **`bakeLevel` throws at >2 regions per broad-phase tile** (1935). Any denser
  authored space (e.g. a real facility interior) hits this. Raise it *when*
  needed; it's a known wall, not a current blocker.
- **GPU budget is already spent** (`renderScale` default 0.5). Update 3's
  enemy gear rendering and Shadowcraft both add marcher cost — budget them
  together, and prefer "numbers and state" mods in Update 4.
- **No automated tests.** Manual browser playtest only. Update 2 (pure numbers,
  no rendering) is the one place cheap headless assertions would pay — consider
  them there.
- **Dead code**: `fireHitscan_DEAD` (3737), `runBeam_DEAD` (3795),
  `showTooltip_DEAD` (4715). Delete before they get revived.

---

## 6. Rendering notes (root-caused, for whoever picks them up)

- **Edge bleed** = material carry in `map()`'s early-outs (see Patch 0).
- **See-through sky** = geometry above the `uWorldTop` slab cap (1260, 5648).
  Keep `uWorldTop` conservative.
- **Sun scale**: nothing is inverted — there is *no* elevation scaling at all
  (fixed `pow(sd,180)`, 1309). *Add* an elevation-dependent radius.
- **Cone-cast sky**: absent; sky is a direct `skyColor(rayDir)` on miss (1451).
  A cheap win if Shadowcraft needs to buy back GPU budget.

---

## 7. Decisions already made (the record)

All owner-ratified in the 2026-07-08 vision session:

1. Simulation of a game — no ending. The loop is the game.
2. Round trip: hidey hole → valley → Area 1 → valley → hidey hole. The valley
   is travel, not a mode.
3. Teeth: down in the valley, carried loot is just gone.
4. Three pillars: telegraphed shadows · deep customization · clean UI.
5. Universal telegraphing: armor renders on bodies, mods render on all guns
   (player's included).
6. Shadow feedback: world-readable (deep shadows, darkening hands) **plus** a
   light-meter gadget stating true spot-distance.
7. Enemy variety = randomized armor (resistances) + randomized loadouts,
   applied **at random or in classes** (either acceptable). Built after the
   offense (mods/statuses) and defense (armor/healing) both exist.
8. Barrels: 3/5/7/9/11 slots by rarity.
9. Extraction with persistence, not a roguelike; the hidey hole gets a stash.
10. Work packaging & order (owner's): patches → Loop → **Tool Mods (statuses
    designed)** → **Armor & Healing (status defense)** → **Enemies (apply all
    of it)** — and the loop itself is the way to endlessly play.

## 8. Open decisions — for the owner

1. **What exactly is "carried" vs. safe on death?** Bag contents only, or also
   worn rig/tool? (Plan assumes: bag gone, worn rig + wielded tool keep.)
2. **Valley leg length** — how long should one travel leg feel? One ring's
   walk? Scale with how much you're hauling?
3. **Stash size** — bounded (forces choices) or effectively unlimited?
4. **Do valley enemies escalate** the more trips you make, or stay flat?
   (Escalation reuses `depth`; flat is simpler to balance.)
5. **Light-meter gadget acquisition** — starting kit, or found loot?
6. **Enemy gear: pure random, or grouped into classes?** Owner has said either
   works. Pure random is leaner; classes make patrols readable as *kinds*
   ("that's a burner squad") at the cost of authoring the groupings.

---

## 9. If you only do one thing

Fix the Chrome crash. Then build the stash and the walk home — the two pieces
that turn what exists into the game described in §1.
