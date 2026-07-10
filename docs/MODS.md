# Weapon-Mod System — Data Tables (for rework)

Editable reference for the tool/mod rework. Every mod below is an entry in
`ITEM_DEFS` (index.html ~line 2560+). Effects are the `effect:{}` object passed
to `new ToolMod(name,color,cat,effect,blurb)`. Edit values here, then port to
`ITEM_DEFS`; keep the GLSL/JS twins in sync where a mod adds world geometry.

Legend: **cat** = category, drives resolver semantics. **⚡** = `surcharge`
(battery cost added to that shot). Base is the core's own `base` ⚡.

---

## How resolution works (the "rack is a tree")

- A tool is **columns** (lanes); each lane resolves **bottom → top**.
- **stat** and **util** mods fold into the whole-tool frame (order-free).
- **muzzle** mods fold per-lane (one active muzzle per column).
- **mod** (sequence) mods accumulate into a *pending charge* that the **next
  proj core** consumes, then the charge resets. Multiple modifiers stack onto
  one core.
- **proj** cores emit a shot using the pending charge.
- A lane whose trailing feeders end in a lone **BEAM** becomes a tactical laser;
  anything under it "vents" (wasted).

**Base tool frame** (before mods): `fireRate 0.42 · burst 1 · spread 0.05 ·
dmg 24 · fovAim 0.62 · auto false`.

**Core base damage** (before `+dmg`): bolt uses tool `dmg` (24) and a magazine
round; spark `10`; pick `14`; grip `6`-ish glue. (`index.html:3735`.)

---

## Projectile cores (`cat: proj`) — emit a shot, consume the charge

| key | name | effect | base ⚡ | behavior |
|---|---|---|---|---|
| `boltCore` | BOLT | `{kind:'bolt', base:0}` | 0 (+1 round) | Kinetic slug from the magazine. |
| `sparkCore` | SPARK | `{kind:'spark', base:18}` | 18 | Bursting ember. Base dmg 10. |
| `pickCore` | PICK | `{kind:'pick', base:14}` | 14 | Eats stone (SDF carve), spares flesh. Base dmg 14. |
| `gripField` | GRIP | `{kind:'grip', base:6}` | 6 | Glue glob; sticks to world + kin. Behind BEAM → physgun. |
| `boomCore` | BOOM | `{kind:'boom', base:26}` | 26 | Lobbed charge: heavy arc, slow. ANY impact (world or flesh) → big blast (R 4.5, ~55 center, falloff) — hurts the shooter too. Base dmg 34. |

## Sequence modifiers (`cat: mod`) — charge the NEXT core

| key | name | effect | ⚡ | consumed at |
|---|---|---|---|---|
| `dmgPlus` | HURT+ | `{dmg:10}` | 5 | `pend.dmg` → shot dmg |
| `velPlus` | SWIFT | `{vel:1.6}` | 4 | `pend.vel` ×= (proj speed) |
| `bounce` | RICOCHET | `{bounce:2}` | 6 | `pr.bounces` (index.html:3599) |
| `pierce` | DRILL | `{pierce:2}` | 8 | `pierceLeft` (index.html:3735) |
| `heavy` | MASS | `{grav:1, dmg:6}` | 5 | `pr.vel.y-=grav*12*dt` (3838) |
| `twin` | TWIN | `{twin:2}` | 12 | shot count `round(twin)`, **cap 3** |
| `silentMod` | HUSH | `{silent:true}` | 3 | suppresses shot noise (3654) |
| `fuseMod` | FUSE | `{fuse:1.2}` | 4 | sticks, blows after 1.2s (3873) |
| `splinter` | SPLINTER | `{split:3}` | 7 | shatters into 3 on impact (3889) |
| `boomerangMod` | RETURN | `{boomerang:true}` | 6 | shot stalls ~0.55s, banks, flies home to the thrower's current position; despawns at ~0.6m ("caught"). Both passes damage; walls still kill it; bounce is overridden on the return. Enemy-fired → returns to its zealot (`pr.src`). |
| `homingMod` | SEEKER | `{homing:1}` | 7 | per-frame steer toward nearest live, non-dormant enemy in a ~40° forward cone, ≤45m, turn clamp 3.2 rad/s (stacks: ×N). Zealot-fired seeks the player. Acid-green tracer telegraphs it. RETURN's return phase overrides it. |
| `beamCore` | BEAM | `{laser:true}` | 8 | shot → continuous thread; alone → tac laser |

## Stat mods (`cat: stat`) — order-free, whole-tool

| key | name | effect | notes |
|---|---|---|---|
| `reflex` | Red Dot | `{fov:0.60, spread:0.92, unique:'fov', reticle:1}` | optic |
| `scope` | Holo Sight | `{fov:0.56, spread:0.86, unique:'fov', reticle:2}` | optic |
| `scope2x` | 2× Optic | `{fov:0.31, spread:0.74, unique:'fov', reticle:3}` | optic |
| `scope3x` | 3× Optic | `{fov:0.21, spread:0.60, unique:'fov', reticle:4}` | optic |
| `heavyBarrel` | Heavy Frame | `{dmg:8, spread:0.85}` | +8 dmg, steadier |
| `autoSear` | Auto Sear | `{auto:true, fireRate:-0.22}` | hold to fire |
| `burstCam` | Burst Cam | `{burst:3, unique:'burst'}` | 3-round burst |
| `hairTrigger` | Hair Trigger | `{fireRate:-0.14}` | faster follow-ups |

`unique:'fov'` / `unique:'burst'` = only one of that group takes effect.
`fireRate` clamps at min 0.06. `spread` multiplies; `dmg` adds; `fov` sets ADS zoom.

## Muzzle mods (`cat: muzzle`) — per-column, one active

| key | name | effect | notes |
|---|---|---|---|
| `suppressor` | Suppressor | `{silenced:true}` | silences that lane |
| `compensator` | Compensator | `{spread:0.55}` | tighter spread, that lane |

## Utility mods (`cat: util`) — whole-tool

| key | name | effect | notes |
|---|---|---|---|
| `pulse` | KINETIC | `{pulse:true}` | beam shoves what it touches |
| `lantern` | LANTERN | `{flashlight:true}` | toggleable light [F] |

---

## Pending-charge fields (the `pend` object, `index.html:3400`)

`freshPend()` = `{dmg:0, vel:1, bounce:0, pierce:0, grav:0, twin:1, silent:false,
fuse:0, split:0, contact:false, burn:0, bleed:false, boomerang:false, homing:0,
surcharge:0, laser:false}`. Each modifier adds/multiplies its field; a proj core
copies `pend`, then it resets. **All fields are consumed downstream — no dead
stats** (verified).

---

## Proposed edits (from the design doc)

Source: the design/issues bible prompt "Design a first wave of mods." Directives:
**14–18 mods**, balanced across **offensive / defensive / utility**; **≥3 purely
positional** (splitter, trigger-on-impact, delay/timer, loop/wrap); **1–2
SDF-world utility** (carve/deform, tunnel, scan-through-walls); every mod gets
name·role·flavor·precise effect·⚡ cost·preferred sequence position; design for
**ordering combos** (spec 5+, incl. one "trap" combo that looks strong but isn't);
**no flat +X% filler**.

Status vs. current code:

| proposal | cat | proposed effect | ⚡ | behavior | status |
|---|---|---|---|---|---|
| **FUSE → relay** | mod | extend `fuse` | 4+ | on impact/death, **re-cast the next core in the sequence from the impact point** (not just blow) | **extends** current FUSE (fixed delay+blow) |
| **TIMER** (tunable) | mod | `{delay:t}` | ? | generalize FUSE's fixed 1.2 s into an authored delay | extends FUSE |
| **CONTACT** | mod | `{trigger:'impact'}` | ? | fire the pending charge on first contact / proximity | **new** (positional) |
| **LOOP / WRAP** | mod | `{loop:n}` | ? | sequence/path re-emits or wraps N times | **new** (positional) |
| **SPLITTER** | mod | `{split:n}` | 7 | shatter into N on impact | **done** = SPLINTER (`split:3`) |
| **CARVE / SHAPER** | proj/util | `{kind:'carve'}` (SDF edit) | ? | deform/carve terrain | partial (PICK eats stone; BASTION build stubbed, index.html:1673) |
| **TUNNEL** | proj/util | line of SDF carves | ? | bore a passage through the world | **new** — GPU: many `uEdits`, needs a hard cap |
| **RECON / SCAN** | util | `{seeThrough:true}` | ? | mark enemies through walls (goggles-tied, stealth) | **new** (defensive/utility) — cheap (HUD overlay, not raymarch) |
| Laser-variants of cores | mod | `laser` feeds a core → continuous, less dmg | 8 | bolt/spark → held beams | **done** = BEAM (`beamCore`) |
| Grip core + physgun | proj | glue glob; +BEAM = physics-gun beam | 6 | — | **done** = GRIP (`gripField`) |
| Beam = tac-laser thread | mod | gossamer thread, alone atop a column | 8 | — | **done** = BEAM |
| _defensive role_ | — | — | — | doc wants offensive/**defensive**/utility balance; current roster is nearly all offensive | **gap to fill** (shield, smoke, decoy, reactive-armor mods) |

Verbatim design notes worth keeping:
- "fuse should cast the next spell in the sequence from its point of death/collision"
- "laser should also turn mods continuous, with a decrease to damage… the functionality of the other cores turns into a continuous beam"
- "When added in series with a core it works to create laser varieties of existing cores (bolt, spark, future cores)"
- "Grip mod should be a Grip core… glob of glue that sticks to near-by globs… works with the tactical laser as a physics gun beam"
- "Include 1–2 utility mods that exploit the SDF world (carve/deform terrain, tunnel, scan through walls)"

**Biggest unbuilt wins:** FUSE-relay (turns the sequencer recursive — huge combo depth), the positional trio (CONTACT / LOOP / TIMER), and the defensive-role gap. The SDF-utility mods (TUNNEL especially) are the ones to watch on GPU cost.

## Rework scratch space

Proposed new cores / modifiers / stats go here first (name · cat · effect ·
⚡ · behavior · does it add raymarched geometry?). Keep "numbers & state" cheap;
flag anything that adds an SDF volume or many projectiles.

| key | name | cat | effect | ⚡ | behavior | GPU cost |
|---|---|---|---|---|---|---|
| _e.g. emberCore_ | FIRE | proj | `{kind:'fire', base:16, burn:4}` | 16 | DoT on hit, no new geometry | cheap |
|  |  |  |  |  |  |  |
