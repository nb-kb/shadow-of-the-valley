# Shadow of the Valley — Scope & Design Plan

Status: **draft for the creative director.** Written 2026-07-08 against build `v21r`
(`index.html`, ~5,840 lines). Agents implement; the owner decides. Anything in
**§9 Open Decisions** is a question, not a plan.

This document exists to answer one question: *what is the smallest coherent
version of this game that is actually a game, and what do we build to get there?*

---

## 1. Where the project actually stands

The honest read, from a full sweep of the source:

> **The engine is close to done. The game barely exists.**

The simulation substrate is genuinely deep and, in places, better than it needs
to be. What is missing is almost everything that turns a sandbox into a game:
an arc, an ending, stakes, and an antagonist with more than one face.

| Layer | State | Notes |
|---|---|---|
| SDF raymarched world + GLSL/JS twin | **Deep** | `mapWorldJS` ↔ `map()`; collision, LOS and physics all run off the CPU twin |
| Light-based stealth | **Deep** | `updateExposure` (2260) marches a real ray to the sun; exposure multiplies enemy detection rate (4077). This is the crown jewel. |
| Noise propagation | **Done** | movement tiers + `addNoise` pings; suppressors genuinely matter (22→4) |
| The Tool (sequencer weapon) | **Deep** | column tree, volley fire, battery wear, 20 mods, beams, glue globs |
| Inventory / rig / crates / magazines | **Done** | hand-hold → open → place; crates and corpses are real containers |
| Save system | **Done** | versioned, 3 slots, corruption-safe |
| Day/night cycle | **Done** | real sun/moon orbit, wired into stealth |
| Audio | **SFX only** | ~20 synthesized cues. **No music, no ambience.** |
| Enemies | **One archetype** | `Zealot` (3982). Steering, not pathfinding. No classes. |
| Damage model | **A single scalar** | No damage types, no status, no headshots, no durability |
| Level content | **3 levels** | `area1` (The Camp), `hideout`, `valley` (endless rings) |
| Story / objectives / ending | **Absent** | one intro line; `onBreach` increments `depth` forever |

Two loops exist and **they do not relate to each other**: an authored raid on
The Camp with a real exfil, and an infinite procedural valley with no exit. A
player who finishes one has no reason to enter the other. That disconnection —
not any bug — is the single largest problem with the game.

---

## 2. The through-line

The design doc names four references — *Noita, The Last of Us, Escape from
Tarkov, Garry's Mod* — and they pull in four directions. A scope plan's first
job is to pick the spine and let the rest serve it.

**The premise, from the design doc:** *you are a captured child escaping a
religious-extremist military group's base.*

That premise is at war with the mechanic. A powerless child does not carry a
15-slot legendary energy cannon volleying from eight barrels. Left unresolved,
every hour spent on mods makes the fiction weaker.

### The resolution: the game is about **memory**

This is not invented — it is already written into the game's own UI copy:

- Wiping a save reads **"ALL MEMORY BURNED."**
- Dying reads **"RECLAIMED."** Not *killed*. *Taken back.*
- The design doc asks for a **molar model for the cyanide pill** — a failsafe
  someone else put in your mouth.
- Save slots are reached through a screen literally called **MEMORY**.

The Covenant does not kill you. It **re-takes** you and burns what you learned.
Escape is not survival; it is *keeping your own mind.* The Tool is not your
power — it is **their** technology, stolen from the armory, and the deeper you
mod it the more you are using their language to escape them.

Everything below serves that spine. It costs nothing to adopt: the strings are
already on screen.

### Pillars

1. **You are prey, not a soldier.** Power comes from the world (loot, light,
   noise), never from a level-up. The Tool escalates; you never do.
2. **The Tool is a language, not a gun.** Ordering mods is the skill ceiling.
   Depth comes from *sequence*, never from `+X%`.
3. **Light and sound are the real weapons.** The stealth sim already earns this.
   Protect it — every feature must answer to it.
4. **The valley remembers.** The world is procedural, but the *pursuit* is
   persistent. The Covenant is coming, and it knows which way you ran.

---

## 3. The scope decision

**Recommendation: build "The Escape" — a finite, ~60–90 minute campaign with an
ending — and demote the endless valley to an unlockable mode.**

An infinite loop cannot be finished, reviewed, or felt. A finite arc can. The
rings tech already built is *not* thrown away; it is gated and given a shape.

### The arc

```
   THE CELL          THE CAMP            THE VALLEY            THE ROAD
  (new, tiny)   →   (area1, exists)  →  (valley, exists)  →   (new, small)
   unarmed          steal the Tool      3–5 rings of flight    the ending
   stealth only     escape the camp     pursuit escalates      you get out
```

- **The Cell** — you wake up with nothing. No Tool, no rig. Pure stealth. This
  teaches light and noise *before* it teaches shooting, and it earns the
  premise. Small: a handful of rooms, reuses `BUILDING`/`WALLS`.
- **The Camp** (`area1`, already built, 23 regions) — find the **armory**, take
  the Tool. Now the game opens up. The existing exfils become the camp's edge.
- **The Valley** (`valley`, already built) — the flight. **Fixed at 3–5 rings**,
  not infinite. Each ring raises pursuit, not just enemy count.
- **The Road** — a short authored final beat. The ending. Ship with *one*
  ending; ambiguity is cheap and fits the tone.

### What this buys, for very little code

- `onBreach()` (4285) already increments `depth`. Gate it: at `depth === N`,
  deploy to `road` instead of re-rolling. **That is the win condition**, and
  it's a handful of lines.
- The facility `doors` are already stubbed with `target:null` (1859) and the
  terminal comment already says doors will reuse `TERM_SCREENS` transition rows
  (4437). The plumbing for level-to-level transitions is *designed and waiting*.
- Enemy respawn is currently a teleport-in 45s after death (4051–4058) — which
  reads as a cheat. Reframe it as **pursuit**: hunters dispatched from the camp,
  arriving from the direction of the camp, more of them each ring. Same code,
  now a feature.

### Recapture, not death

Today, dying in a raid reads *"DRAGGED YOURSELF HOME,"* restores your health,
and lets you keep everything (4268). There are **no stakes.**

**Proposal:** death = **recapture**. You wake in The Cell. Your *carried* loot is
confiscated into the Camp's armory — where you can go take it back. Your
*equipped* rig stays. Hard death (`RECLAIMED`) is reserved for the cyanide pill:
the one death that is yours to choose.

This costs little (crates, armory and spawns all exist), creates a real loss,
gives a reason to re-enter The Camp, and says exactly what the game is about.

---

## 4. System plans

Ordered by leverage. `S/M/L` = relative size, not hours.

### 4.1 The Tool — depth over breadth  `M`

20 mods ship today; all are implemented and every `pend` field is consumed. The
roster is, however, **almost entirely offensive**, and the design doc's own
directive ("offensive / defensive / utility, roughly balanced"; "≥3 purely
positional") is unmet.

Build, in this order:

1. **FUSE-relay** — *"fuse should cast the next spell in the sequence from its
   point of death/collision."* This single change turns the sequencer
   **recursive** and is the largest depth-per-line win available anywhere in the
   project. It is the Noita soul of the game. Build it first.
2. **The positional trio** — `CONTACT` (fire pending charge on contact),
   `TIMER` (generalize FUSE's fixed 1.2 s into an authored delay), `LOOP/WRAP`.
   These satisfy "value is entirely positional."
3. **The defensive class** — currently a hole. Shield, smoke, decoy, reactive
   armor. Needed for pillar 1: prey needs *outs*, not more damage.
4. **`RECON` / scan-through-walls** — cheap (HUD overlay, not a raymarch), ties
   to the goggles, and pays the stealth pillar.

**Cut `TUNNEL`.** It needs many `uEdits` against a hard cap of 16
(`SDF_EDIT_CAP`, 1680) and is the one proposal with genuinely unbounded GPU
cost. Revisit only if `PICK` carving proves cheap in practice.

**Rebalance the barrels.** Code ships `common=3 … legendary=15` slots
(`BARREL_TIERS`, 2329). The design doc says **legendary = 7**. Fifteen slots
across `RACK_MAX_COLS=8` is an absurd power ceiling that no encounter design can
absorb. Pull toward the doc.

### 4.2 Damage, status & armor  `M`

The most completely-specified unbuilt system in the project (design doc §J), and
currently a single scalar. Build it as specced — it is coherent and it gives the
defensive mods something to attach to:

- Damage types **HURT / BURN / BLEED**. `BURN` reduces effective max HP; `BLEED`
  is a ~30 s DoT.
- Per-entity status containers. Note: `Zealot.hurt` (4036) has no status field
  today; `stagger` is the only transient state and is the pattern to copy.
- Healing items **gummies** (+30), **bandAid** (clears bleed), **aloe** (clears
  burn). All three are ABSENT; only `ration` (+14) and `cyanide` exist.
- **Headshots** ×2 with **helmet interception**. All enemy hit-tests currently
  use one center-of-mass sphere (`d<0.55`, 3858). The seam is already flagged in
  a comment at 3949 and `medPort` is already a dormant field (2647).
- **Helmet/plate durability** and destruction; **status resist** rolls. Neither
  field exists today.
- Health bar UI: burn grayout, bleed indicator.

Bump `SAVE_VER` when armor gains durability.

### 4.3 Enemies  `M`

One archetype, direct steering, no pathfinding. The AI's *perception* is
excellent and its *navigation* is not. In an SDF world there is no navmesh, and
building one is out of scope.

- **Do not build pathfinding.** Instead: author patrol routes as waypoint lists
  on regions, and let the existing `settle()` resolve local collisions. Zealots
  should walk paths humans would walk, because a human placed them.
- **Add 3–4 classes** to serve loot variation (design doc §I) and to make the
  pursuit escalate qualitatively, not just numerically: *Watcher* (spots,
  doesn't shoot, screams), *Hunter* (fast, close), *Warden* (armored, slow,
  drops a helmet), *Preacher* (rallies, de-escalates others' detection slower).
- Class-based loot tables. `lootTiers` (1870) already exists; key it by class.

### 4.4 World & level content  `L`

The design doc's *"the infinite box rooms is a mistake"* is the right instinct.

**A hard constraint that will bite:** `bakeLevel` (1918) asserts **≤2 regions per
16×16 broad-phase tile** and throws on overflow (1935). Any dense authored
interior — The Cell, the facility, The Road — hits this. **Raise the tile
capacity before authoring new levels.** This is a prerequisite, not a polish
item, and it is currently invisible.

Then: the facility interior (the memory beat), The Cell, The Road, and the
in-between dressing the doc asks for (obvious roads, shrubs, cars, crates).

### 4.5 Rendering  `S`–`M`

Concrete, root-caused items — all found in code, none speculative:

- **Edge bleed** *(root cause found)*: in `map()`, the bounding-sphere early-outs
  return the **previous hit's material** — `res = vec2(max(bd,0.1), res.y)` at
  1134 / 1158 / 1179. The distance is the bounding sphere but `res.y` is carried
  over, so silhouette-adjacent pixels take a neighbour's material. Fix the
  material carry, not the precision.
- **See-through sky** *(root cause found)*: `raycast` caps upward rays at the
  world slab (`skyCap`, 1260) and `uWorldTop` is raised per-frame by airborne
  projectiles/items (5648). Anything rendering above the slab shows sky through
  geometry. Make `uWorldTop` conservative.
- **Sun scale**: the doc says the size relationship is *"the opposite way
  around."* In fact there is **no size scaling at all** — the disk is a fixed
  `pow(sd,180)` (1309) with no elevation term. Nothing is inverted; the feature
  is absent. *Add* an elevation-dependent radius.
- **"Cone-cast sky for performance"**: also absent — the sky is a direct
  `skyColor(rayDir)` on ray miss (1451). Worth doing; `renderScale` defaults to
  **0.5** (quarter-res), which tells you the marcher is already expensive.
- See-through optics when aiming; default optics dial to 50%.

### 4.6 Input, menus & UI  `S`

The owner's stated #1 priority, and it is also a **bug fix**, not just a tidy:

- Three near-identical d-pad readers — `menuNav` (5460), `termNav` (4493),
  `dbgNav` (5224) — each re-implementing `rawKey['Arrow*'] || rawPad[12..15]`.
  Collapse to one.
- Four bespoke menu-open verbs dispatched at 5547–5550. Make them table-driven.
- **`pressed()` has no edge ownership.** `INPUT.frameDown` is cleared once, at
  the very end of `frame()` (5803), so *one press fires every handler that reads
  it that frame*. This is the exact cause of the "crate opens and instantly
  grabs the top item" bug: `openCrate()` runs at 3277, then `menuNav()` reads
  the *same* live edge at 5520 and takes an item. **Fix the ownership model
  (consume the edge on read) and the bug class disappears** — it is not a
  one-off.
- Only **3 of 10** settings widgets are functional (`functional:false` stubs at
  2672–2682). Either wire the audio buses or delete the dials.

### 4.7 Audio  `M`

**There is no music and no ambience.** For a stealth game whose core mechanic is
*sound*, this is the largest sensory hole in the project. A wind bed, a camp
drone, and one motif that intrudes when detection crosses a threshold would do
more for the felt experience than any rendering fix on this list.

---

## 5. Engine debt that blocks the game

These come first because they poison playtesting — you cannot get honest
feedback through them.

| # | Issue | Why it's blocking |
|---|---|---|
| 1 | **Chrome TDZ crash** — "cannot access gameState before initialization" | The game does not load for a large share of players. Nothing else matters. |
| 2 | **Invisible enemies** — `RLIM.ENEMIES:10` (625) vs **22** spawned in `area1` (1865) and up to **14** in the valley (4227) | Rendering is capped at 10; **AI and combat loops are uncapped.** Enemies you cannot see shoot you and eat your bullets. This does not read as a bug — it reads as *the stealth game being unfair*, which is the one thing it must never be. |
| 3 | **Crate double-take** | See §4.6 — a symptom of the input edge model. |
| 4 | **Edge bleed** | See §4.5 — now root-caused, so cheap. |

On (2): raising `RLIM.ENEMIES` costs uniform slots and march time. Better is a
**dormancy rule** — enemies outside the uploaded set go passive (no shooting, no
bullet collision) rather than fighting invisibly. Cap the sim to the render set,
or raise the cap *and* lower `area1`'s 22.

---

## 6. Phases

Each phase should end in a build the owner can play end-to-end.

**P0 — Stabilize.** `S`
Chrome TDZ · input edge ownership + nav/open consolidation (kills the crate bug)
· enemy legibility (dormancy) · edge bleed.
*Exit: a stranger can load it in Chrome and every enemy shooting them is visible.*

**P1 — The Spine.** `M`
This is the phase that makes it a game. Raise the `bakeLevel` tile cap · unstub
doors as terminal transitions · The Cell (unarmed opening) · gate the rings at
N → The Road · one ending · recapture-not-death · pursuit reframing of respawn.
*Exit: the game can be started, lost, and **won**.*

**P2 — Depth.** `M`–`L`
Damage types + status + healing + headshots + durability · FUSE-relay and the
positional trio · defensive mod class · barrel rebalance · enemy classes and
class-based loot.
*Exit: two runs play differently because you built the Tool differently.*

**P3 — Texture.** `M`
Music and ambience · the Tool + arms model · stylized inventory camera moves ·
sun scale, cone-cast sky, see-through optics · road/shrub/car/crate dressing ·
remove splash text, keep the loading image.
*Exit: it feels like a place.*

**P4 — Post-campaign.** `S`
Unlock the endless valley as a mode. Wire the decorative endless crates. Audit
`lootTable` for full item coverage.

Ship after P3. P4 is the reward for finishing, not a prerequisite.

---

## 7. Explicit non-goals

Saying no is most of what a scope document is for.

- **"Model everything in the game."** In an SDF raymarcher, a model is
  hand-written distance-field code — this is a bottomless pit. Model **the Tool
  and the arms** (on screen 100% of the time) and *nothing else*. Lean into the
  monochrome stencil aesthetic; it is already the game's strongest visual idea
  and it is free. Cut the backpack, helmet, plate and cyanide-molar models.
- **`TUNNEL`** — unbounded `uEdits` cost against a cap of 16.
- **Pathfinding / navmesh.** Author patrol routes instead.
- **Per-category audio gain buses** — until there is music to bus.
- **Multiple endings, dialogue, NPCs, quest log.** The doc itself says no
  elaborate narrative. One ending. The story is told in item names and UI copy.
- **Endless-valley polish before the campaign exists.**
- **Splitting `index.html`.** It stays one file (CLAUDE.md invariant #2) — but
  see §8.

---

## 8. Risks

- **The twin (GLSL ↔ JS) is the project's central hazard.** Every world-geometry
  change is two changes. The v21p/v21q bug run was *all* twin desync. Two live
  landmines: (a) any `*<big>|0` in a JS hash mirror must be `Math.imul` — the f64
  product rounds past 2^53 (this cost the forest); (b) the valley crate hash is
  hand-copied in **three** places (`cellSDF` 760, `mapWorldBase` 1630, crate
  registry 4351) and must move in lockstep.
  *Mitigation:* the node-probe technique — evaluate the JS twin SDF offline and
  measure real clearance against the 0.34 m player radius. It found the doorway
  ridge that guessing could not. Use it before every twin change, not after.
- **Single-file ceiling.** ~5,840 lines today; P1–P3 plausibly reach 8–9k. The
  constraint is an identity, not an accident — but it should be a *decision*,
  re-made at ~8k, not a default that quietly costs velocity.
- **No automated tests, ever.** Verification is manual browser playtest. The
  node-probe is the only mechanical check that exists. As the status system
  lands (a pure numbers system, no rendering), consider a handful of headless
  assertions — it is the one area where they'd be cheap.
- **GPU budget is already spent.** `renderScale` defaults to 0.5. Every new mod
  that adds world geometry (globs, edits, beams) competes with the marcher.
  Prefer mods that are "numbers and state."
- **Dead code drift.** `fireHitscan_DEAD` (3737), `runBeam_DEAD` (3795) and
  `showTooltip_DEAD` (4715) are unreferenced. Delete them before they get
  "helpfully" revived.

---

## 9. Open decisions — for the owner

Agents should not answer these.

1. **Finite campaign, or keep it endless?** This plan assumes finite (§3). It is
   the single decision everything else hangs from.
2. **How many valley rings** between The Camp and The Road? 3 is tight; 5 risks
   repetition, since rings differ only by seed and enemy count.
3. **Is "memory" the spine?** (§2) It's already in the UI copy, so it's free —
   but it's a creative call, not an engineering one.
4. **Recapture-instead-of-death** (§3): does losing carried loot to the armory
   feel like a hook, or like a chore?
5. **Does the player start unarmed?** Strongest single change for the premise,
   and it front-loads the stealth sim — but it delays the Tool, which is the
   toy people came for.
6. **Barrel ceiling**: doc says legendary = 7 slots, code says 15. Which is
   canon?
7. **Ship target**: is this a public release, a portfolio piece, or a private
   toybox? P3's cost is only justified by the first two.

---

## 10. If you only do one thing

Fix the Chrome crash. Then give the game an ending.

Everything else on this list is an improvement to a thing that currently cannot
be finished by a player, and cannot be completed by its author.
