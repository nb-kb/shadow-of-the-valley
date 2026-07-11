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
- **The relay convention (v22j, extended v22 P-grammar):** FUSE/CONTACT/**TIMER**
  wire the next proj core as their relay payload (`relayNext`). Under a
  **non-laser carrier** that payload is
  **housed** (`shot.housed`, recursive down the chain): the fire cycle passes
  over it entirely (carrier→carrier — a housed BOOM can never bare-fire in your
  face), enemy `tryShoot` and the continuous-beam head pick skip it too, and the
  CARRIER's pull pre-pays the whole chain — each housed shot's assembled cost
  (base + its own feeders' ⚡) plus its rounds (`roundsFor` recurses). REFUND on
  the carrier waives the carrier's own base/lead only — the housed addition
  stands (feed REFUND to the housed core itself to waive that link). POWDER
  waives mod surcharges only, never housed core bases. **CONTACT-thread
  carriers house and DRUM (v22 P-beamC C2)**: the thread fires half-size casts
  of the housed payload at its contact point every 0.5s — priced through the
  rate (root.cost folds the chain in), ballistic beats bill their lead per
  beat. **FUSE-only BEAM carriers keep the old exemption**: a thread can't
  stick, so the payload keeps its cycle seat and pays at its own turn (FUSE's
  wiring under a thread stays a priced vent). **The family reads:** CONTACT = last touch · TIMER = bell or
  touch · FUSE = stick, then the bell. Every cast leaves along the struck FACE
  (world) or RADIALLY out of the wound (flesh) — stored flight dir only for
  mid-air expiries (TIMER bell, orb/warp life-end). **EXTRA CASTS COST A THIRD
  (v22 P-grammar, `castRelay`):** the pull pre-pays ONE delivery of the housed
  chain; every cast past the first (in-body DRILL ticks, SPLINTER shard
  deliveries) draws `ceil(chain ⚡/3)` — the burst-pulse precedent — and a
  ballistic chain's extras eat their rounds from the carrier's own mag (the
  POWDER law: lead is a full price). Can't pay either bill: the cast simply
  doesn't happen. Zealots pay no ⚡ — their extras cap flat at 3 per projectile.
- A lane whose trailing feeders end in a lone **BEAM** becomes a tactical laser;
  anything under it "vents" (wasted).

**Base tool frame** (before mods): `fireRate 0.42 · burst 1 · spread 0.05 ·
dmg 24 · fovAim 0.62 · auto false`.

**Core base damage** (before `+dmg`): bolt uses tool `dmg` (24) and a magazine
round; spark `10`; pick `14`; grip `6`-ish glue. (`index.html:3735`.)

---

## Projectile cores (`cat: proj`) — emit a shot, consume the charge

| key | name | effect | base ⚡ | behavior | GPU |
|---|---|---|---|---|---|
| `boltCore` | BOLT | `{kind:'bolt', base:0}` | 0 (+1 round) | Kinetic slug from the magazine. | cheap |
| `sparkCore` | SPARK | `{kind:'spark', base:18, burn:15}` | 18 | Bursting ember — the fire rides the blast now. Base dmg 10 + 15 burn dose; v22 P-balance: the blast itself sears another `0.35×dmg-at-range` (~+4) on top. Niche: blast-first with a real side of fire. | cheap |
| `emberCore` | EMBER | `{kind:'spark', base:22, burn:28}` | 22 | Fire that keeps eating — commit-to-burn. 28 of the 30-cap pool ≈ 14s of 2 hp/s. Niche: +4⚡ over SPARK buys the near-full pool. | cheap |
| `pickCore` | PICK | `{kind:'pick', base:14}` | 14 | Eats stone (SDF carve), spares flesh. Base dmg 14. | uEdits (capped) |
| `gripField` | GRIP | `{kind:'grip', base:6}` | 6 | Glue glob; sticks to world + kin. Behind BEAM → physgun. | cheap |
| `decoyCore` | LURE | `{kind:'decoy', base:10}` | 10 | Chirping puck — sticks where it lands, sings to the camp for 12s. | cheap |
| `boomCore` | BOOM | `{kind:'boom', base:26}` | 26 | THE BLAST PRIMITIVE — no flight, no tracer slot: it detonates the frame it is cast (R 4.5, dmg 34+pend, ×1.6 vs flesh w/ falloff — BOTH sides since v22 P-balance: the shooter eats the same ×1.6, armor shaves after, plus a view kick so the hit READS). Every explosion also SEARS: burn dose = own `burn` + `round(0.35×dmg-at-range)` into the 30-cap pool, both sides (bare BOOM: 12 at the epicentre, ~7 at 2m). Bare muzzle fire goes off ~1.2m out: survivable, stupid — and now felt (unarmored ~30, ceramic+steel ~16, plus ~9 burn). Relay/contact-cast (`atPoint`) it goes off AT the cast point — [CONTACT,BOLT,BOOM] = explosive rounds, [FUSE,X,BOOM] = X's death detonates. FUSE on the boom ITSELF restores the lob (flies heavy, sticks, blows after the fuse — the timed charge). TWIN pools, never fans: ONE blast ×1.4 radius. BEAM-fed: a drum of small charges (see the beam notes). | cheap |
| `waveCore` | WAVE | `{kind:'wave', base:10}` | 10 | Wide slow wall of pressure: 10 m/s, dmg 16, washes a ~1.8m swath, each foe pays ONCE (`pr.hit`/`pr.hitPl` sets), shove 7 + stagger. Never impact-dies on flesh (pierce inert on it BY DESIGN); the world still kills it. Enemy-fired = flat 16, dodgeable. BEAM-fed (v22 P-beamA): no train — a pressure corridor thread (10m, ±1.5m, per-tick wash — LOS-gated from the ray foot, never sideways through world — plus a 2.2 m/s press landed in the zealot's own move step, post-pathing so chasers feel it; 2.2 < the 2.6 walk ceiling = held, never locked; TWIN pools width to ±2.1, 11.9⚡/s). | 1 tracer slot (perp-bar); thread: axis+bar |
| `sawCore` | SAW | `{kind:'saw', base:1}` | 1 | Teeth, not reach: unlocks the WHOLE tool's cycle to the 0.06 floor; its own bite dmg 8, life 0.16s ≈ 4m at speed 26 (SWIFT is THE reach knob). 16.7⚡/s vs regen 16 + wearRoll per tooth: self-governing attrition. | cheap (~3 live teeth) |
| `warpCore` | WARP | `{kind:'warp', base:15}` | 15 | You arrive where it dies — impact, fuse, or the end of its arc. Speed 40, dmg 6. Arrival is LOUD (noise 18 + muzzle flare) unless HUSH-fed (noise 4, both ends). Sheds threads at resolve. Player-only: zealots never roll it AND `warpArrive` owner-guards. Rare loot tier. | cheap |
| `powderCore` | POWDER | `{kind:'bolt', base:0, powder:true}` | 0 (+1 round) | A bolt that pays in lead: its feeders' ⚡ surcharges are waived — except REFUND's own (`surchargeHard`). Under BEAM (v22 P-beamC C1 — the old named trap REDEEMED by owner order): a rounds-powered HITSCAN laser — the lane never threads, it pulses through `fireHitscan` on the pull path, 1 round + 0⚡ per pulse, fire-rate governed, every lane mod live (DRILL pierce, SEEKER 12° lock, CHAOS jag, TWIN fan headroom-gated at 1rd total, HUSH; a CONTACT/FUSE-housed payload casts at the pulse's contact point — aim ray only, pre-paid by the pull like any first delivery, no touch no cast). | cheap |
| `slugCore` | SLUG | `{kind:'slug', base:0}` | 0 (+3 rounds) | Three rounds, one fat blow: ×3 TOOL hurt (`base.dmg*3 + pend.dmg` — HURT+ additive AFTER the ×3), speed 48, fat radii (0.9 vs flesh, 0.30 vs world). Whole tool +0.45s cycle unless relay-housed (some shot's `relayNext`). Sheds threads at resolve. | cheap |
| `orbCore` | ORB | `{kind:'orb', base:8}` | 8 | Just an orb: 6 m/s, dmg 6, 6s aloft then its payload pops. The blank canvas — it becomes what you feed it. Cap 4 player orbs live; the 5th pops the oldest. BEAM noses it (puppeteer). | hoards tracer slots 6s — capped, monitor |

## Sequence modifiers (`cat: mod`) — charge the NEXT core

| key | name | effect | ⚡ | consumed at | GPU |
|---|---|---|---|---|---|
| `dmgPlus` | HURT+ | `{dmg:10}` | 5 | `pend.dmg` → shot dmg | cheap |
| `velPlus` | SWIFT | `{vel:1.6}` | 4 | `pend.vel` ×= (proj speed) | cheap |
| `bounce` | RICOCHET | `{bounce:2}` | 6 | `pr.bounces` (index.html:3599) | cheap |
| `pierce` | DRILL | `{pierce:1}` | 8 | v22 P-grammar: **pierce-ALL** — flesh never stops the shot (`impactOrPierce` returns false first), only the world or the lifetime ends it. Call sites bill **one hit per body per 0.1s** (`pr._dHit` map keyed on the body, `pr.life` as the clock) — a fast bolt = exactly 1 bill per body ("pierces all enemies", paid by geometry); a slow core dwelling in a body ticks ~10/s, capped. Hitscan pulses take every swept body before the wall (`pierceAll`). Blast kinds under DRILL pass through flesh and resolve on the environment ([DRILL,SPARK] blasts at the backstop; WARP blinks THROUGH the crowd) — declared. Stacking DRILL = idempotent 8⚡ of nothing — declared. Price held at 8⚡: per-body single-bill is FAIRER than the old accidental in-body triple-tap. | cheap |
| `heavy` | MASS | `{grav:1, dmg:6}` | 5 | Projectiles: `pr.vel.y-=grav*12*dt`. BEAM-fed (v22 P-mass M1): a damage thread (bolt/spark/boom/saw) SAGS — the beam is a stream at its core's projectile pace (bolt 82 · spark 34 · boom/saw 26, ×SWIFT) under the same 12·grav pull: drop = 6·grav/V²·s² along the path (`droopPath`), marched as 4 chord segs in the B1 arc budget class (TWIN clamps to 2 strands). Shorter effective reach: a level grav-1 bolt thread grounds ~34m out (grav 2 ~24m; boom ~10m — the mortar thread); SWIFT ×1.6 keeps the full 50m (0.87m drop). The dirt hit IS the contact point: pulses, the C2 drum and B2 shards anchor there. GRIP/PICK/WAVE/ORB threads never droop (straight-line verbs). | cheap |
| `twin` | TWIN | `{twin:2}` | 12 | shot count `round(twin)`, **cap 3** | cheap |
| `silentMod` | HUSH | `{silent:true}` | 3 | suppresses shot noise (3654) | cheap |
| `fuseMod` | FUSE | `{fuse:1.2}` | 4 | sticks, blows after 1.2s — then RELAYS: casts the next core in the lane **OFF the face it stuck to** (v22 P-grammar: `rdir` set to the surface normal at stick time — a stuck FUSE is a directional mine; `relayNext`; TWIN copies don't chain — only the first carries it). On the same core FUSE outranks CONTACT. v22j: the relay target is **HOUSED** — out of the fire cycle, its ⚡/lead billed to the carrier's pull (see ledger) | cheap |
| `contactMod` | CONTACT | `{contact:true}` | 5 | FUSE-relay's instant sibling: on the charged core's **LAST TOUCH** (v22 P-grammar: RICOCHET spends its banks first on the world; flesh triggers any time) it casts the NEXT core in the lane from the hit point (`relayNext`) — no timer, no stick; the cast leaves along the struck face / radially out of the wound. The carrier's own impact stays normal (damage, decal). No next core in the lane → plain impact, the mod vents (like FUSE with nothing above it, the cast is empty). CONTACT+FUSE on one core: FUSE governs (stick+timer), CONTACT vents — its 5⚡ still paid. Under DRILL: the payload delivers EVERY in-body tick (see THE AUGER, extras billed). TWIN first-copy rule applies (copies don't chain). v22j: the relay target is **HOUSED** — out of the fire cycle, its ⚡/lead billed to the carrier's pull (see ledger). BEAM-fed (v22 P-beamC C2): a CONTACT thread HOUSES its payload and DRUMS half-size casts of it at the contact point every 0.5s — per lane, aim strand only (TWIN never multiplies the drum); ballistic beats bill their lead per beat at the mag, a dry mag HOLDS the beat until lead arrives; the ⚡ is already in the thread rate. | cheap |
| `slowFuse` | TIMER | `{timer:1.5}` | 3 | v22 P-grammar rework: **a clock, not glue** — `pend.timer` (stacks). The next core triggers at the bell (`pr.timerT`, 1.5s in flight) OR on first touch, whichever first, casting its payload either way — no stick. TIMER is the third relay trigger (wires `relayNext`, housed + carrier-pays identically). Feeding a core that also has FUSE: merged at resolve (`pend.fuse += timer`, timer zeroed) — patience, not a second clock. [TIMER,BOOM] = the flak shell (flies, blows at the bell or first touch). defKey `slowFuse` unchanged (saves serialize by defKey). | cheap |
| `splinter` | SPLINTER | `{split:3}` | 7 | shatters into 3 on impact (`splinterBurst`). v22 P-grammar: split now routes through `triggerPayload` on flesh too — a TIMER/FUSE relay casts even when the shot shatters (the old split-branch silently dropped the relay: declared fix). Under CONTACT the shards CARRY the payload to where THEY land (frags inherit `relay`/`rdir`/`rbase`, `_relayPaid:true`) — the parent's terminal was the free cast, every shard delivery draws `ceil(chain ⚡/3)` + lead (see CLUSTER WRIT). BEAM-fed (v22 P-beamB B2, I→C): a damage thread's contact point sprays up to 3 shard rays EVERY frame it touches something — world hits reflect off the face, flesh shatters THROUGH (forward shrapnel, distinct from DRILL's one full line); shards are bolt-shards regardless of parent (0.45× the bolt line, 6m, saw parent 2.7m, own chest/head intercept, A5 cook); every TWIN strand shatters at ITS OWN contact; RETURN arcs shed only at their first world clip; flash-budgeted, shards starve first. grip/pick/wave/orb threads stay inert (7⚡ priced). | cheap |
| `serrated` | SERRATED | `{bleed:true, surcharge:5}` | 5 | `pend.bleed` → `pr.bleed` → `applyStatus`: opens a wound — wounds STACK (`status.bleedN`, cap 5), 1 hp/s per stack, `bleedT` runs 15s from the freshest cut (resists shave the clock). Both sides, same rate. Blast kinds pass the flag through the explosion dose — once. Continuous-thread ticks (bolt/saw line, arc bites, wave corridor, shards) bank cuts on a cadence (`threadCut`): first touch cuts at once, then one stack per 0.5s of contact per target — a cut is a WOUND, not a frame. Projectile hits keep 1 cut per landed shot; DRILL's 0.1s in-body ticks keep their declared ledger. | cheap |
| `boomerangMod` | RETURN | `{boomerang:true}` | 6 | shot stalls ~0.55s, banks, flies home to the thrower's current position; despawns at ~0.6m ("caught"). Both passes damage; walls still kill it; bounce is overridden on the return. Enemy-fired → returns to its zealot (`pr.src`). BEAM-fed (v22 P-beamB B1, implicit-inert→C): a damage thread (bolt/spark/boom/saw) bends into a flat D-loop — out along the (seek/chaos-bent) aim, bulging ±2.5m to one side, home to the muzzle; ONE bite per enemy per tick (damage BESIDE and part-way behind cover, never more single-target dps); a world hit truncates the loop there; spark/boom pulse at the APEX; TWIN mirrors it (right+left, flash-budget cap 2 loops); saw R=4 hand-width halo; grip/pick/wave/orb threads stay inert (6⚡ priced). | cheap |
| `homingMod` | SEEKER | `{homing:1}` | 7 | per-frame steer toward nearest live, non-dormant enemy in a ~40° forward cone, ≤45m, turn clamp 3.2 rad/s (stacks: ×N). Zealot-fired seeks the player. Acid-green tracer telegraphs it. RETURN's return phase overrides it. BEAM-fed (v22i): the damage thread BENDS fully onto the nearest mark within a ~12° cone of the aim (`seekBias`, 45m, no LOS — the cone is the clamp); twin strands share the one mark; grip/pick never bend; its 7⚡ prices through the thread's per-second rate. | cheap |
| `beamCore` | BEAM | `{laser:true}` | 8 | shot → continuous thread; alone → tac laser. THREAD HEAT LAW (v22 P-beamA A5): every per-tick LINE hurt (bolt/saw threads, wave corridor) doses EQUAL burn via `applyStatus` (pool cap 30, resists apply) — threads COOK; spark/boom tip pulses stay discrete blasts dosing `pend.burn` via `explode`, unchanged | cheap |
| `refundMod` | REFUND | `{refund:true, surcharge:1}` | 1 (never waived) | `pend.refund` → resolveLane cost formula (waives next core's base ⚡) + `roundsFor()` (waives lead). Its own 1⚡ rides `pend.surchargeHard` — the one no core forgives, POWDER included. Idempotent flag: REFUND+REFUND = 2⚡ of nothing. v22 P-balance: 10→1 by owner decree — the real price is the rack cell + loot rarity; watch knob 2–3⚡, never a mechanic change. | cheap |
| `chaoticMod` | CHAOS | `{chaos:1, surcharge:6}` | 6 | `pend.chaos` → `pr.chaos` steer wobble (2.4·chaos rad/s, return leg immune); threads arc like lightning (endpoint wander ×(1+0.6·chaos) + 3-joint strand render); pulses jitter. With SEEKER on threads: arcs bite the nearest warm thing (chain lightning). Stacks: wider wander. v22 P-balance: hits **+45% per stack, cap ×1.9** (`chaosMul` at all five damage assemblies — instant boom, projectile literal, hitscan, beam thread, both beam drums); frags inherit the boosted parent dmg and never re-multiply. GRIP/PICK threads stay straight BY RULE. Zealots CAN roll it — white-blue tracer telegraphs the wobble AND the heavier hand (WATCHED). | 3 one-frame flash segs per strand, gated on `flashHeadroom()>=3` (degrades to 1 straight) |

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

| key | name | effect | notes | GPU |
|---|---|---|---|---|
| `pulse` | KINETIC | `{pulse:true}` | beam shoves what it touches | cheap |
| `lantern` | LANTERN | `{flashlight:true}` | toggleable light [F] | cheap |
| `beltFeeder` | BELT-FEEDER | `{beltFeed:true}` | chambers 1 round / 0.9s from your pockets (`takeBagAmmo`, hungriest racked mag drinks first); ticks a faint r=3 noise ping per round unless the tool is suppressed. Trickle 1.1 rd/s — moves lead, never mints it. Zealots excluded (reserve model). | cheap |

---

## Pending-charge fields (the `pend` object, `index.html:3400`)

`freshPend()` = `{dmg:0, vel:1, bounce:0, pierce:0, grav:0, twin:1, silent:false,
fuse:0, timer:0, split:0, contact:false, burn:0, bleed:false, boomerang:false,
homing:0, surcharge:0, laser:false, refund:false, surchargeHard:0, chaos:0}`.
Each modifier adds/multiplies its field; a proj core copies `pend`, then it resets.
**All fields are consumed downstream — no dead stats** (re-verified, mod wave v1):

- `contact` — consumed at (a) resolveLane's relay wiring (`relayNext`, the
  instant sibling of FUSE's), (b) the flight touch branches
  (`updateProjectiles` world hit — AFTER the bounce branch since v22 P-grammar —
  + `impactOrPierce` flesh hit + the DRILL in-body tick gates), all guarded
  `&&!(pr.fuse>0)` — FUSE governs on a shared core.
- `timer` (v22 P-grammar) — consumed at (a) the resolve-time FUSE merge
  (`pend.fuse+=pend.timer` when both >0), (b) resolveLane's relay wiring (third
  trigger), (c) the instant-boom gate (`!(p.fuse>0||p.timer>0)` — a timed BOOM
  flies), (d) emit → `pr.timerT`, the flight bell in `updateProjectiles`.
- `refund` — consumed at (a) the resolveLane cost formula (waives the core's
  base ⚡), (b) `roundsFor()` (waives lead).
- `surchargeHard` — consumed at the resolveLane cost formula: the un-waivable
  slice of the surcharge pool. Only REFUND writes it; POWDER cannot touch it.
- `chaos` — consumed at `emitProjectile` (→`pr.chaos` steer wobble),
  `runContinuousBeams` (arc render + endpoint wander + homing bias),
  `runBurstLasers`/`fireHitscan` (pulse jitter + jagged flash), and — v22
  P-balance — `chaosMul` (+45% dmg per stack, cap ×1.9) at the five damage
  assemblies (instant boom, projectile literal, hitscan, beam thread, both
  beam drums).
- `bleed` — consumed at `applyStatus` (v22 P-balance rework): each dose stacks
  `status.bleedN` (cap 5) and refreshes `bleedT` to 15s·resist; both DoT ticks
  (player status block, zealot update) drain 1 hp/s per stack and zero `bleedN`
  when the clock runs out. Every `bleedT=0` site (bandAid, hideout stanch, run
  resets, enemy reset, player death) zeroes `bleedN` with it — no orphan stacks.

Effect-only flags (never land on `pend`): `powder` (read in the resolveLane proj
branch cost formula; since v22 P-beamC also stamped onto the shot as
`shot.powder` — consumed by the beam skip, `burstLaser`, the pull routing/pulse
and the seq chip), `beltFeed` (foldFrame → `out.beltFeed` → `tickBeltFeed`).
New `pr` fields: `chaos` (steer block), `hit`/`hitPl` (wave once-per-foe),
and — v22 P-grammar — `timerT` (the flight bell), `_dHit` (DRILL per-body 0.1s
bill map), `_relayPaid` (castRelay: first delivery pre-paid, extras billed),
`_exCasts` (zealot extras flat cap 3) — all consumed in `updateProjectiles` /
`castRelay`.

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
| **FUSE → relay** | mod | extend `fuse` | 4+ | on impact/death, **re-cast the next core in the sequence from the impact point** (not just blow) | **done** = FUSE relay (`relayNext`) |
| **TIMER** (tunable) | mod | `{delay:t}` | ? | generalize FUSE's fixed 1.2 s into an authored delay | **done, reworked v22 P-grammar** = TIMER (`slowFuse`, `{timer:1.5}` — a clock in flight, bell or touch) |
| **CONTACT** | mod | `{trigger:'impact'}` | ? | fire the pending charge on first contact / proximity | **done, reworked v22i** = CONTACT (`contactMod`): now the instant relay — first touch casts the next core from the hit point |
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

**Biggest unbuilt wins** (FUSE-relay, CONTACT and TIMER have since shipped): LOOP/WRAP (the last positional), the defensive-role gap (shield, smoke, reactive-armor mods — LURE covers decoy), and the SDF-utility mods (RECON scan; TUNNEL is the one to watch on GPU cost).

## Rework scratch space

Proposed new cores / modifiers / stats go here first (name · cat · effect ·
⚡ · behavior · does it add raymarched geometry?). Keep "numbers & state" cheap;
flag anything that adds an SDF volume or many projectiles.

| key | name | cat | effect | ⚡ | behavior | GPU cost |
|---|---|---|---|---|---|---|
| _e.g. emberCore_ | FIRE | proj | `{kind:'fire', base:16, burn:4}` | 16 | DoT on hit, no new geometry | cheap |
|  |  |  |  |  |  |  |

---

# MOD WAVE v1 — SHIPPED (branch v22/scope-implementation)

Nine items landed (P1–P6): WAVE · SAW · WARP · POWDER · SLUG · ORB · REFUND ·
CHAOS · BELT-FEEDER, plus the SPARK/EMBER burn retune. Rows merged into the cat
tables above. ZERO GLSL edits — no new world geometry, the TWIN RULE was never
triggered. No SAVE_VER bump — items serialize by defKey; new keys are free.

## Interaction matrix (F=composes-free · C=needs-code, cited package · S=nonsense-suppressed · T=TRAP, deliberate · I=inert-by-design, surcharge still real · –=can't meet)

- **WAVE**: FUSE-relay F (wall sticks, casts; or carrier chain) · CONTACT F
  (the wall casts onward where it dies on the WORLD — flesh never triggers it,
  the wave washes through by design) · TIMER F · TWIN F (2-3 walls, 1 tracer slot each) · BEAM
  C-A1 pressure corridor (10m, ±1.5m, per-tick wash LOS-gated at the ray
  foot — never sideways through world — + 2.2 m/s post-pathing press, TWIN
  pools width, 11.9⚡/s) · burstCam F (3 stacked walls) · RETURN F (the tide comes
  back) · SEEKER F (homing wall) · DRILL I (wave never impact-dies on flesh —
  pierce unread on it) · RICOCHET F (sweeps back off the wall) · SPLINTER F
  (dies on wall → 3 bolts) · HUSH F · MASS F · SWIFT F · HURT+ F ·
  GRIP/PICK/LURE – · BOOM via FUSE-chain F.
- **SAW**: FUSE-relay F (point-blank tooth from the impact) · CONTACT F ·
  TIMER F · TWIN F · BEAM C-P2 4m chainsaw · burstCam: plain burst F,
  laser-burst S · RETURN **T "YO-YO NO"** (life 0.16 < stall 0.55 — the tooth
  dies before it turns; 6⚡ for nothing — the PROJECTILE; the saw THREAD under
  BEAM+RETURN arcs fine: C-B1's 4m hand-width halo) · SEEKER F (barely turns, honest) ·
  DRILL F · RICOCHET F · SPLINTER F (dust storm) · HUSH F (4⚡ silent teeth) ·
  MASS F · SWIFT F (THE reach knob) · HURT+ F · frame-level: SAW+SLUG
  **T "THE WOODCHIPPER"** (order law: floor 0.06 then +0.45 → 0.51s cycle —
  looks like a fat machine gun, is a slow drum-mugging) · SAW+autoSear F (the
  true saw) · SAW+hairTrigger I.
- **WARP**: FUSE-relay F, both seats ([FUSE,WARP]=delayed door;
  [FUSE,X,WARP]=cast onward from X's death; [FUSE,WARP,X]=you arrive as X
  casts) · CONTACT F ([CONTACT,WARP,X]: you blink to the impact AND X casts
  from it, instantly; v22 P-grammar face-cast: [CONTACT/FUSE,BOLT,WARP]'s
  payload-warp now flies OFF the struck face — a bounce-back door — instead of
  dying 6cm into the wall as a de-facto blink-to-impact; declared, the combo
  most likely to be missed in playtest) · TIMER F · TWIN T-soft
  (double-blink, 12⚡ to be yanked twice) · BEAM S (laser stripped; BEAM vents
  its 8 into the chip) · burstCam F (triple-blink, self-punishing, allowed) ·
  RETURN **T "THE LASSO"** — the named headline trap: looks like a recall
  rope; the catch splices with NO payload → 21⚡ to teleport nowhere unless it
  clips a wall mid-return. Decided: kept, deliberately · SEEKER F
  "SWAP-STRIKE" (blink onto the mark) · DRILL F (I→F v22 P-grammar: flesh
  never stops it — **blink THROUGH the crowd**, the warp resolves on the
  environment) · RICOCHET F "BANK SHOT"
  (bounce a blink around the corner) · SPLINTER F (arrive in a shower) ·
  HUSH F "GHOST STEP" (whisper both ends) · MASS F (grenade-arc jump) ·
  SWIFT F (longer blink) · HURT+ F · BOOM via FUSE F "BREACH-AND-ENTER"
  ([FUSE,BOOM,WARP]: the blast opens the wall, the warp carries you through).
- **POWDER**: all sequence feeders F with surcharges waived (the point) ·
  REFUND C-P1 (lead waived instead of base; 1⚡/0rd sidearm — pockets-empty
  tech, priced by loot rarity since the v22 decree) · BEAM **C-C1** (was T —
  REDEEMED by owner order: a rounds-powered HITSCAN laser — the lane never
  threads, pulses ride the pull path through `fireHitscan`, 1rd + 0⚡ each,
  all lane mods live; REFUND'd: 1⚡ hard, 0rd; the burstCam 0⚡/0rd volley
  mint is closed by routing — powder never enters `_laserBursts`, burstCam
  powder fires 3 honest hitscans for 3 rounds) · FUSE-relay F (cycle gate
  keeps it honest) · TWIN F
  (0⚡ doubling; rack cells + cap 3 + lead are the price).
- **SLUG**: FUSE-relay F + relay-housed exemption (the "in trigger" rule) ·
  CONTACT F · TIMER F · TWIN F (3 fat slugs, still 3 rounds total — copies
  don't eat lead, priced by the cycle penalty) · BEAM S · burstCam
  **T "DRUM-EATER"** (9 rounds a pull; partial-burst break mirrors bolt) ·
  RETURN F (fat boomerang, both passes ×3) · SEEKER F · DRILL F · RICOCHET F ·
  SPLINTER F (32-dmg frags — watched) · HUSH F · MASS F (mortar slug) ·
  SWIFT F · HURT+ F (additive AFTER ×3, declared).
- **REFUND**: energy cores F (base→0) · ballistic C-P1 (lead→0) · BEAM-fed I
  (base already discounted by rate math; its 1 barely moves the rate —
  anti-synergy shrunk to a rounding error, documented) · REFUND+REFUND T
  (idempotent flag, 2⚡ of nothing) · TWIN F (12 stands — guardrail).
- **BELT-FEEDER**: orthogonal util — F with everything · SLUG note: 1.1rd/s
  trickle vs 3rd/shot, honest · suppressor/silenced gates its tick noise ·
  zealots excluded.
- **CHAOS**: projectiles F (wobble) · SEEKER F (drunken missile — both steers
  run) · BEAM C-P5 lightning · BEAM+SEEKER C-P5 **CHAIN LIGHTNING** (the
  owner's ask, verbatim; on a shared thread CHAOS's 40° arc bias replaces
  SEEKER's 12° bend) · burstCam pulses
  C-P5 · GRIP/PICK threads S (stay straight) · RETURN F (return leg immune —
  the way home flies true) · RETURN-arc threads F (v22 P-beamB: CHAOS wobbles
  the loop's bulge W ±40%/stack and the aim wanders — no jagged joints on arcs
  BY RULE, the loop IS the multi-seg) · chaos joints and B2 shards share the
  flash pool (declared starve order: aim thread > extra strands > joints >
  shards) · WARP F (drunken blink, self-priced) · BOOM F
  **"LOUD BARGAIN"** (accepted: an instant blast has no flight, so no wobble
  to pay — but since v22 P-balance the shooter feels all of a 49.3-dmg blast
  1.2m from their own nose; self-priced) · dmg +45%/stack (cap ×1.9)
  everywhere the charge lands — the wild path is the price · everything
  else F.
- **MASS**: BEAM **C-M1 droop** (owner: "mass + beam should affect the beam
  downward arc"): a damage thread SAGS like a hose of lead — the beam is a
  stream at its core's projectile pace (bolt 82 · spark 34 · boom/saw 26,
  ×SWIFT) under the 12·grav projectile pull: drop = 6·grav/V²·s² along the
  path (`droopPath`), 4 chord segs marched like the B1 arc (per-seg bites +
  dedupe, A5 cook, world truncation; SAME flash reservation class — TWIN
  clamps to 2 strands, `(pts?4:1)` shard reserve). Grounds at range from a
  level muzzle: bolt ~34m (grav 2 ~24m), boom ~10m (the mortar thread) — the
  dirt hit IS the contact point (spark/boom pulses, the C2 drum and B2 shards
  all anchor there) · SWIFT F (vel² divides the droop — ×1.6 keeps the full
  50m at 0.87m drop; the flat-shooter's knob) · RETURN F **drooping arc loop**
  (falls out of `droopPath` on the arc's pts, no special case: comes home LOW
  — ~0.29m off the dirt at grav 1 — a heavy loop truncates at the ground; the
  home knee sags on a CLONE of the muzzle anchor; truncated drooping arcs keep
  B1's apex-pulse quirk, declared) · TWIN F (each jittered strand droops its
  own path) · SPLINTER F (shards anchor at the ground clip) · GRIP/PICK
  threads S (straight BY RULE — a drooping physgun is nonsense; wave/orb keep
  their own verbs; 5⚡ priced) · projectile side unchanged. Verified by node
  probe (real extracted `droopPath`/`runContinuousBeams`, flat-world stub):
  droop table grav 1/2, SWIFT flattening, ground termination, mortar drum at
  the dirt, drooping loop knees, clip-anchored shards, 10-flash worst case
  (headroom 0, floor holds), grip/pick straight, curve bites + A5. 32/32.
- **BOOM** (instant since v22i — the blast primitive): bare fire F-but-dumb
  (blast 1.2m off your own muzzle; self-damage truly stands since v22
  P-balance — ×1.6 flesh both sides, view kick, burn dose) · CONTACT-carrier F
  **"EXPLOSIVE ROUNDS"** ([CONTACT,BOLT,BOOM]: every hit detonates AT the
  wound/wall) · FUSE on the boom itself F (restores the lob: flies, sticks,
  timed charge; TIMER too, its own way — v22 P-grammar [TIMER,BOOM] F
  **"FLAK"**: flies, blows at the bell or first touch, whichever first;
  [FUSE,BOOM,WARP] BREACH-AND-ENTER intact) · FUSE/CONTACT on a
  CARRIER F (the death/touch detonates at the point) · TWIN: ONE blast ×1.4
  radius, never N blasts (12⚡ buys area, not count) · SEEKER / RICOCHET /
  DRILL / RETURN / SWIFT / MASS on a bare boom I (no flight to steer, bounce,
  pierce, return or hasten — surcharge still real; FUSE-fed they read again,
  it's a projectile then) · CONTACT on the boom itself I (nothing to touch —
  it vents; put CONTACT on the carrier) · SPLINTER F (blast + 3 frags from the
  fire) · burstCam T-soft (three stacked blasts in your own face for one ⚡
  price — three doses of your own medicine) · BEAM C (the drum, see ledger) ·
  CONTACT-carrier BEAM C-C2 (the GENERALIZED drum: any CONTACT thread drums a
  half-size housed BOOM at the dot every 0.5s, ~17+0.5·dmg, R×0.75 — the
  direct BEAM+BOOM 0.45s branch is untouched) ·
  hitscan/burst-laser BOOM F (unchanged: full blast at the ray's end per
  pulse, ceil(34/3)=12⚡ each) · zealots: never rolled, see availability. 1. BEAM noses it C-P3
  (puppeteer) · 2. life-expiry pops its payload C-P3 (one shared line with
  warp) → SPLINTER = 6s drifting nova F · 3. FUSE+ORB+X wall-caster F ·
  4. CHAOS tesla-wander F · 5. SEEKER balloon-hunter (6 m/s stalker) F ·
  6. RICOCHET room-roamer F · 7. DRILL plows a file of men F · plus TWIN 2-3
  pearls (cap 4 live) F · RETURN lazy loop F · MASS T-lite (grav sinks a
  6 m/s pearl fast — paying to worsen it) · burstCam laser-burst S, plain
  burst F.

**SEEKER×BEAM resolution (superseded v22i):** bare SEEKER×BEAM is no longer
inert — the owner asked for it: "beam should home with seeker." `pend.homing`
on a damage thread (bolt/spark/boom/saw) now bends the aim onto the nearest
live, non-dormant mark within a **12° cone** (cos 0.978, 45m, no LOS —
`seekBias`, full lock, the cone gate IS the bend clamp; twin strands share the
one mark, grip/pick stay true, orb trains keep seeking as projectiles; the
wave corridor never seeks — it is aim, v22 P-beamA).
With CHAOS the wider 40° chain-lightning bias (`chaosBias`) takes over instead
— both consume homing on threads; the 7⚡ prices through the beam's
per-second formula either way (e.g. SEEKER+BEAM+BOLT: 2+(0+7+8)·0.55 =
10.25⚡/s).

**RETURN×BEAM / SPLINTER×BEAM (v22 P-beamB):** RETURN on a damage thread was
implicit-inert → **C-B1 return arc** (owner: "Return + beam, projectile beam
should arc back to the player"): the thread bends into a flat D-loop.
BOLT+BEAM+RETURN = 2+(0+8+6)·0.55 = **9.7⚡/s**; +TWIN **16.3⚡/s** — buys
damage BESIDE and part-way behind cover, a positional sweep, not more dps.
×CHAOS F wobbling loop (W jitter + wandering aim, free) · ×SEEKER F apex-lock
(the `fb` bend upstream, untouched) · ×TWIN F two mirrored loops,
flash-budget-capped at 2 (the third strand never draws NOR damages) ·
"beam + twin + reload" read sanely: RELOAD/BELT-FEEDER is frame-level util,
orthogonal to threads (threads drink ⚡, not lead; the belt keeps chambering
during holds — already true, declared unaffected). SPLINTER×BEAM was I →
**C-B2 shatter** (owner: "Beam + splinter should shatter into smaller beams
on contact… each ray getting a splinter"): BOLT+BEAM+SPLINTER = **10.25⚡/s**;
full stack TWIN+SPLINTER+CHAOS+BEAM+BOLT = 2+(0+12+7+6+8)·0.55 =
**20.15⚡/s** — corridor saturation at a real battery deficit. Single-target
honesty: shards fan wide and mostly miss one foe (3 converged shards ≈ 17.8
dps needs a corner pocket) — the crowd verb costs the wall to use.

## Balance ledger (v22j housed payloads)

- **HOUSED-SKIP** (owner ask, verbatim: "Contact should skip the next core in
  the chain when it's the payload, to prevent mis-firing payloads outside the
  trigger") — every FUSE/CONTACT relay target under a **non-laser carrier** is
  `housed` (`shot.housed`, recursive down the chain — a housed shot's own relay
  is housed too, same rationale for FUSE as for CONTACT). The volley walk, the
  continuous-beam head pick, the rack ▶/pips and zealot `tryShoot` all pass
  over housed shots — a housed BOOM can never bare-fire in your face; it fires
  ONLY from its carrier's impact/death. A lane with nothing live simply holds
  (dry click, no hang); a lane's FIRST shot can never be housed, so all-housed
  lanes are structurally unauthorable anyway.
- **CARRIER-PAYS pricing** (powerful-but-paid): the old cycle gate — "the
  relay'd core must be paid at its own turn or the column starves" — is gone
  for housed shots, so the CARRIER's pull now pre-pays the whole chain: each
  housed shot's assembled cost (base + its own feeders' ⚡, so a mid-chain
  CONTACT's 5 is not a free mod) rides the root carrier's `cost`, and housed
  ballistic payloads bill their rounds through `roundsFor` (recursive). Worked
  prices: **[CONTACT,BOLT,BOOM] = 31⚡ + 1rd** per pull (explosive rounds — the
  boom casts at the wound) · **[FUSE,BOLT,SPARK] = 22⚡ + 1rd** ·
  **[CONTACT,SLUG,BOOM] = 31⚡ + 3rd** (slug is the CARRIER — keeps the +0.45s
  cycle) · chained [CONTACT,BOLT,CONTACT,BOOM,SPARK] = 54⚡ + 1rd ·
  [CONTACT,BOOM] leaves BOOM the carrier (26+5 = 31⚡, relay empty → vents) ·
  [FUSE,BOLT,SLUG] = 4⚡ + 4rd, housed slug still dodges the cycle penalty.
- **Waiver edges, decided:** REFUND on the carrier waives the carrier's OWN
  base/lead only — the housed chain's addition stands (rack REFUND as the
  housed core's feeder to waive that link instead: [CONTACT,BOLT,REFUND,BOOM]
  = 6⚡ since the v22 REFUND decree — was 15⚡ at REFUND 10). POWDER as carrier still waives only MOD surcharges, never housed
  core bases ([CONTACT,POWDER,BOOM] = 26⚡ + 1rd). burstCam casts the chain per
  burst copy at one ⚡ price — the same accepted T-soft as its stacked blasts.
- **BEAM carriers (rewritten v22 P-beamC C2):** CONTACT-thread carriers now
  HOUSE their payload and the thread DRUMS half-size casts of it at the
  contact point every 0.5s — the old "26⚡ trap" objection is VOID because the
  payload actually casts, repeatedly, and the pricing is automatic:
  `root.cost += shot.cost` folds the chain into the carrier and the thread
  rate formula reads carrier cost. Worked: [CONTACT,BEAM,BOLT]+BOOM — carrier
  0+5+8 = 13, +26 housed → rate 2+39·0.55 = **23.45⚡/s** (vs the hand-tuned
  direct boom-thread's 20.7 — dearer, and it keeps the bolt thread's own
  13.2 dps: correct). FUSE-only beam carriers keep the old exemption (a
  thread can't stick; FUSE under a thread stays a priced vent) — the payload
  keeps its cycle seat and its own price. Direct BEAM+BOOM (boom IS the
  thread) keeps its dedicated 0.45s branch untouched.
- **HUD honesty:** housed chips read '⌂ housed' (their ⚡/lead already on the
  carrier chip), tipBrief marks a racked housed core, the FREE tag hides on a
  carrier billing a chain, and pips/rack ▶ never rest on a housed entry. No
  new pend fields — `housed`/`housedRoot` live on the resolved shot, consumed
  by the walk, `roundsFor`, and the readout. ZERO GLSL touched.

## The named traps (deliberate, all kept)

| trap | build | what it looks like | what it is |
|---|---|---|---|
| THE LASSO | RETURN+WARP | a recall rope | the catch splices with NO payload — 21⚡ to teleport nowhere unless it clips a wall mid-return |
| THE WOODCHIPPER | SAW+SLUG | a fat machine gun | order law: floor 0.06 then +0.45 → 0.51s cycle, a slow drum-mugging |
| YO-YO NO | RETURN+SAW | a boomerang blade | tooth life 0.16 < stall 0.55 — it dies before it turns; 6⚡ for nothing |
| DRUM-EATER | burstCam+SLUG | triple fat blows | 9 rounds a pull; partial-burst break mirrors bolt |

(The POWDER+BEAM trap is GONE — redeemed by owner order into the C-C1 hitscan
laser, see the P-beamC ledger.)

## Free-lunch audit (all closed)

1. POWDER+REFUND → refund's surcharge rides in `surchargeHard`, powder can't
   waive it — and the floor is now **1⚡/0rd BY DECREE** ("Refund mod should be
   1 energy"). REFUND's real price is the rack cell + loot rarity; watch knob
   if it sours: surcharge 2–3, never a mechanic change. CLOSED (as priced).
2. REFUND+TWIN → refund waives BASE only; TWIN's 12 stands → twin BOOM = 13⚡
   not 12 (was 22⚡ at REFUND 10). CLOSED.
3. POWDER feeding a relay chain → relay casts stay free at cast time, and
   (superseded v22j) the cycle gate is GONE for housed payloads — the carrier
   now pre-pays the chain's bases/rounds instead, and POWDER's waiver never
   touches that addition (it trims only its feeders' mod surcharges). Same
   money, collected at the pull instead of the payload's turn. CLOSED, re-shut
   under the housed model.
4. BELT-FEEDER+SLUG → trickle 1.1 rd/s < 3 rd/shot; the belt moves lead,
   never mints it. CLOSED.
5. POWDER+BEAM → SUPERSEDED (v22 P-beamC C1, owner order): the waiver under a
   laser is re-opened BECAUSE a powder lane never threads — it PULSES on the
   pull path, and every pulse pays a round (the 2⚡/s lead-free thread this
   item closed can no longer exist). The one real mint in the redeem —
   burstCam+POWDER+BEAM volleys riding `_laserBursts` at 0⚡ AND 0 lead (the
   volley walk's `rds=bl?0:` line) — is closed by routing: `burstLaser`
   excludes powder, so burstCam powder fires 3 hitscans through the plain
   burst loop, 3 rounds, honest. CLOSED (as re-priced in lead).
6. SLUG/WARP + BEAM/burstCam → laser stripped at resolve + burstLaser excludes
   new kinds → no round-free fat threads, no 60m hitscan blinks, no ×3-dmg
   free pulses. CLOSED.
7. SAW spam → 16.7⚡/s vs regen 16 + wearRoll per tooth: self-governing
   attrition. CLOSED.
8. In-body tick loop (DRILL+CONTACT dwell) → closed by the 10/s per-body gate
   (`pr._dHit`, 0.1s bill window shared by damage AND cast) + `ceil(chain ⚡/3)`
   per extra delivery + lead-billing + zealot flat cap 3. CLOSED by pricing.
9. Zero-cost housed-bolt tick chain ([DRILL,CONTACT,X,BOLT]: extras have no ⚡
   base) → closed by lead-billing: every extra cast eats its rounds from the
   carrier's own mag — a 10rd/s mag-dump through a body, governed by mag
   capacity and reload. Powerful-but-paid. CLOSED.
10. Shard-relay multiplication (SPLINTER+CONTACT: up to 4 extra deliveries) →
    frags ship `_relayPaid:true`, so every shard delivery draws its third +
    lead. [CONTACT,SPLINTER,BOLT,BOOM] ≈ 74⚡ for 5 blasts vs 155⚡ pulled
    singly — a real discount, battery-bounded. FEATURE, not free. CLOSED.

## Balance ledger (v22 P-balance: CHAOS hand · bleed stacks · REFUND decree · honest blasts)

Five items landed (B1–B5): CHAOS +45%/stack (cap ×1.9) · BLEED stacking rework ·
REFUND 10→1 · self-splash honesty · explosion sear. ZERO GLSL edits — no world
geometry touched, the TWIN RULE was never triggered. No SAVE_VER bump (no
serialized shape changed; `status.bleedN` is runtime-only).

- **B1 CHAOS damage** — `chaosMul(ch)=1+0.45·min(2,ch)` applied at all five
  damage assemblies; frags inherit boosted `pr.dmg`, no double dip. Enemy CHAOS
  rolls get it too — the white-blue wobble tracer IS the telegraph. **WATCHED**;
  one-line fallback: halve `chaosMul` for `owner!=='player'`, or drop
  `chaoticMod` from `MOD_KEYS`.
- **B2 BLEED** — one cut = 15 dmg potential (was 20 over 30s at 0.67 hp/s);
  three quick cuts = 3 hp/s; cap 5 governs every multi-hit source (TWIN,
  SPLINTER frags, burst-laser 1 stack/pulse — each dose paid for). Same rate
  both sides per owner — the machinery is symmetric (`applyStatus` + both DoT
  ticks), but `serrated` sits in no zealot roll (`MOD_KEYS` nor the barrel
  slot list), so no enemy source exists today; adding it is the one-word knob.
  HUD `bleedTick` unchanged (keys off `bleedT>0`).
- **B4 self-splash root cause** — the player-splash path always existed (since
  BOOM landed, git 4785f40); the FELT bug was the missing ×1.6 flesh multiplier
  (enemies got it, the shooter didn't) buried under plates(0.6–0.75) ×
  helmet(0.88–0.94). Fixed by symmetry, not a new mechanism; only literal-zero
  gates in the file are the dbgGod checkbox and `gameState!=='play'`. View kick
  + pitch dip added so the hit READS. Boom-drum siege now bites its user up to
  ~19 dps inside 1.8m — declared, the 20.7⚡/s deficit already prices the verb.
- **B5 explosion sear** — `explode()` doses `burn = own burn +
  round(0.35×dmg-at-range)` through the existing `applyStatus` lite-object
  path, both sides; the 30-cap pool, resists, and the single-dose rule
  (direct-hit `applyStatus` already skipped for spark/boom) all stand.
  Boom-drum ~4–5 burn/pulse saturates the pool in ~3s of sustained thread —
  **WATCHED**, the ⚡ deficit prices it. Bare-muzzle self-splash → ~9 burn ≈ +9
  scar toward the `maxHealth−10` floor: the LOUD BARGAIN is self-priced.
- **B3 REFUND decree** — worked prices updated above; the 1⚡/0rd bolt sidearm
  collapses most of the lead economy wherever the mod drops. Decreed; loot
  rarity is now REFUND's entire price. Watch knob: surcharge 2–3.

## Balance ledger (v22 P-grammar: payload grammar — G1–G6)

Six items landed: CONTACT-after-RICOCHET · face/radial casts · DRILL pierce-all ·
castRelay pricing · SPLINTER shard delivery · TIMER rework. ZERO GLSL edits — no
world geometry touched, the TWIN RULE was never triggered. No SAVE_VER bump
(TIMER's defKey `slowFuse` unchanged; all new fields are runtime-only).

- **G1 CONTACT after RICOCHET** — the world-hit chain is now bounce → contact →
  fuse → plain: CONTACT has the same precedence vs RICOCHET that FUSE always had
  (bounce already preempted fuse) — family symmetry. Flesh still triggers any
  time ("OR on flesh", owner's ruling). [RICOCHET,CONTACT,BOLT,BOOM] = the bank
  shot: banks first, blasts on the final impact.
- **G2 face/radial casts (unified, incl. FUSE)** — justification for unify: a
  stored-dir cast from a world impact fired INTO the face just hit — every
  non-BOOM relay off a wall was a latent dud; the fix is one line at each site
  where the normal is already computed; a stuck FUSE becomes a directional mine.
  Mid-air expiries keep the stored flight dir (no surface to read).
- **G3/G4 THE AUGER** — DRILL+CONTACT delivers the payload every in-body tick
  (10/s cap: damage tick and cast tick are ONE bill window). Worked prices:
  [DRILL,CONTACT,BOLT,BOOM] pull = 39⚡+1rd; one body at bolt speed = 1 tick =
  the free delivery; the wall's terminal then draws ceil(26/3)=9⚡. Slow-core
  dwell ([...ORB,BOOM], 47⚡ pull): ~2 ticks/body, extras 9⚡ each —
  battery-bounded siege. [DRILL,CONTACT,X,BOLT]: extra bolts cost 1 round each
  (+0⚡) — a 10rd/s mag-dump governed by mag and reload. Brown-out rule: can't
  pay, no cast (incl. tool swapped mid-flight — the extras brown out; accepted
  honest gutter, `sh.lane` null-guarded). Split-through-payload declared fix: a
  TIMER/FUSE relay casts even when the shot shatters.
- **G5 CLUSTER WRIT** — SPLINTER+CONTACT: the shards deliver the payload where
  THEY land; parent's terminal was the free cast, each shard delivery draws its
  third + lead. ≈74⚡ for 5 BOOMs vs 155⚡ singly — discounted, paid, bounded.
- **G6 TIMER family reads** — CONTACT = last touch · TIMER = bell or touch ·
  FUSE = stick, then the bell. TIMER+FUSE on one core merges F (patience, not a
  second clock); [TIMER,BOOM] FLAK F. LURE's hardcoded fuse:12 unaffected.
- **DRILL×blast kinds, declared (owner sign-off pending):** [DRILL,SPARK]/
  fused-BOOM/WARP pass through flesh and resolve on the environment — the
  letter of "flesh never stops it". Revert knob: one ordering line in
  `impactOrPierce`. DRILL+SERRATED: in-body ticks can hit the bleed cap 5 in
  ~0.5s of dwell — the cap governs, declared.
- **WATCHED:** enemy TIMER airbursts + DRILL pass-through + relay extras
  (flat cap 3) ride existing `MOD_KEYS` rolls — collectively a difficulty bump;
  fallback knobs are single lines (drop `slowFuse`/`pierce` from `MOD_KEYS`).

## Balance ledger (v22 P-grenade: the lit core)

One item: a held BOOM core arms in-hand — [F/△] lights a 5s clock, no
un-arming ("no fingers un-pull a pin"), it goes off wherever it is: in hand,
thrown (the existing hurl verbs — G / hands-mode fire; no new input), pinned,
or resting. ZERO GLSL edits — no world geometry, TWIN RULE not triggered.

- **Numbers (post-B4/B5):** in-hand detonation pd≈1.2–1.4 → raw ~40, armored
  ~21, +~9 burn — a real gamble, not a suicide. Thrown, enemies eat
  34·falloff·1.6 + the 0.35 burn dose. Blast noise 36 via `explode`; the beeps
  stay OFF the noise map — a lit core is not a free LURE.
- **Stow guard:** an armed core never enters a grid (one gate at the top of
  `interactInventory`'s held-item block — rack, belt, rucksack, swap and pour
  all funnel through it). `_armT` is runtime-only and never serializes; if a
  mid-raid save ever serializes worldItems, a lost `_armT` reads as a disarmed
  dud — accepted.
- **Fields consumed:** `it._armT` (heldUse arm + updateWorldItems tick + stow
  gate), `w._beepT` (beep cadence 0.55s → 0.2s under 1.6s). No SAVE_VER bump.

## Balance ledger (v22 P-beamA: pressure corridor + thread heat law)

Two items, both inside `runContinuousBeams`. ZERO GLSL edits — every visual
rides the existing tracer width tiers; no world geometry, TWIN RULE not
triggered. No new pend fields, no save impact.

- **A1 — BEAM+WAVE pressure corridor** (owner: "Beam+wave, should make a
  short wide continuous beam"): the 0.55s wave TRAIN is REPLACED by a short
  wide corridor thread — range clamped 10m, half-width CW ±1.5m, TWIN POOLS
  width to ±2.1 (boom-rMul precedent; K stays 1, the 12⚡ buys wide, never
  strands). Every live non-dormant foe in the corridor pays
  `(16+pend.dmg)·0.55·dt` per tick (8.8 dps/foe base, 14.3 with HURT+; no
  headshots — pressure has no point) and is pressed `dir·24·dt` (≈ one old
  wave-shove per 0.3s of contact). NO stagger BY RULE: 11.9⚡/s is
  regen-positive (< 16) and a stagger-locking corridor at surplus ⚡ is free
  crowd god-mode — the press is the paid verb. The corridor washes THROUGH
  (flesh never clamps t, only the world does). Price: WAVE+BEAM 11.9⚡/s;
  +TWIN 18.5⚡/s (deficit 2.5/s). Draw: standard k=0 axis thread + ONE
  one-frame perp bar at the end (col sum 1.45 → fat 0.035 tier, the WIDE
  read) — strictly cheaper than the old 4-5-wall train.
- **DECLARED INTERIM (designer risk 1):** the train's emitted walls used to
  carry `relayNext` per copy — between P-beamA and P-beamC, a
  [CONTACT/FUSE, …, WAVE-under-BEAM, X] chain casts NOTHING (X keeps its
  cycle seat, fires at its own turn — old exemption). Accepted; C2 restores
  payload casting better-defined (housed + drummed). **RESOLVED — P-beamC C2
  landed:** a CONTACT wave-thread now houses X and drums half-size casts of
  it at the corridor's end every 0.5s; FUSE-only wave-threads keep the seat.
- **A5 — THREAD HEAT LAW** (owner: "Beam should do burn damage equal to
  hurt"): every per-tick LINE hurt doses equal burn through `applyStatus`
  (resists apply) — bolt/saw thread ticks and the A1 corridor now; B1 arc /
  B2 shard ticks when P-beamB lands. NOT spark/boom tip pulses — discrete
  blasts keep dosing `pend.burn` via `explode`, unchanged. Numbers: bolt
  thread 13.2 hurt/s → 13.2 burn-dose/s → the 30-cap pool fills in ~2.3s →
  ≤30 banked afterburn at 2 hp/s (~15s cook, free hurtFlash telegraph); saw
  4.4/s → pool in ~6.8s; HURT+ adds 5.5/s; bare-head ticks double (dose =
  hurt by law); helm/plates `resist('burn')` shave the dose. Old per-tick
  full-pend dosing dies with it (bolt/saw pend.burn is 0 today — nothing
  lost). Player-side exposure: none — zealots never run threads. Price: no ⚡
  change — the 30-pool + 2hp/s drain self-limit; this is the beam identity.
  **WATCHED:** bolt-thread TTK in playtest; declared knob = dose ×0.5 in the
  synthetic pr, one line.
- **Fields consumed:** none added — the corridor and the heat law reuse
  `pend.twin/dmg/bleed` and the existing strand state; wave no longer reads
  `st.acc[k]` (branch replaced; acc stays consumed by spark/boom/pick/orb).

## Balance ledger (v22 P-beamB: return arc + splinter shatter)

Two items, both inside `runContinuousBeams` (the seg-budget geometry pack).
ZERO GLSL edits — every visual rides the existing tracer width tiers; no
world geometry, TWIN RULE not triggered. No new pend fields, no save impact —
loop and shard state is frame-local. Zealots never run threads (their
`tryShoot` filters `pend.laser` lanes), so neither verb has an enemy side.

- **B1 — RETURN ARC** (owner: "Return + beam, projectile beam should arc back
  to the player; same with beam + chaos + return, beam + twin + reload,
  etc"): RETURN on a bolt/spark/boom/saw thread bends it into a flat D-loop —
  muzzle → +side bulge (W = 2.5·(1 ± 0.4·chaos)) → apex (reach 14m, saw 4m: a
  hand-width halo, allowed comedy) → −side bulge → muzzle, side sign
  alternating per strand (+,−,+). Four segments marched in order; a segment
  hitting the world TRUNCATES the loop there (draw to the hit, drop the rest
  this frame — SPLINTER anchors at that first clip). Damage per SEGMENT, one
  bite per enemy per tick (dedupe set): bolt/saw pay the full line formula
  ((24|8+dmg)·0.55·mul)·dt with head/helm rules and the A5 cook; spark/boom
  keep their 0.45s pulse — it fires at the APEX. Draw: all four segs FLASH
  (dt-ttl) = exactly 4 slots per strand per frame (the persistent idiom would
  stack ~3 deep ≈ 12 slots — flash is the honest arc price); K pre-clamped to
  `floor(flashHeadroom()/4)` so TWIN tops out at 2 mirrored loops and a
  strand that can't draw never damages. CPU: ≤4 short marches (≤14m) REPLACE
  the strand's one 50m march — declared acceptable. Price: BOLT+BEAM+RETURN
  9.7⚡/s, +TWIN 16.3⚡/s.
- **B2 — SPLINTER SHATTER** (owner: "Beam + splinter should shatter into
  smaller beams on contact, beam + twin is also affected, each ray getting a
  splinter"): a damage thread with `split>0` sprays shard rays at its contact
  point every frame it touches something — each TWIN strand at ITS OWN
  contact, verbatim. World hit: base = reflect off `worldNormalJS(end)`,
  origin nudged +0.06 off the face; flesh hit: base = dir, THROUGH-shatter
  shrapnel (distinct from DRILL's single full line); arcs shed at their first
  world clip only — an unclipped loop sheds nothing. Per shard per frame:
  `jitterDir(base,·,5)` (0.2–0.4 rad, re-rolled — the sparkler), march 6m
  (saw parent 2.7m), own chest/head intercept (head ×2, helm `_helmT` tick),
  tick = 0.45·(24+dmg)·0.55·mul·dt — bolt-shards regardless of parent kind
  (the `splinterBurst` frags-become-bolts precedent) — plus A5 burn=hurt and
  bleed on the 0.5s cut cadence (`threadCut`). Budget FIRST: nSh = min(3,
  round(split), flashHeadroom()−(arc?4:1)·(K−1−k)) — reserve what a
  remaining strand actually DRAWS (an arc loop flashes 4 segs; the old
  1-slot reserve let strand-0 shards spend the later loops' slots and
  breach the projectile-tracer floor); declared starve order aim thread >
  extra strands > chaos joints > shards (later strands' shards starve
  first); a shard that can't draw doesn't march or hurt. Draw: one flash seg per shard, parent col ×0.8 (bolt sum 3.84 →
  hairline laser tier). Worst case aim thread + 2 flash strands + ≤9 shards,
  all flash-gated — the 2-slot projectile floor and the RLIM.TRACERS upload
  cap hold by construction. Price: BOLT+BEAM+SPLINTER 10.25⚡/s; full stack
  TWIN+SPLINTER+CHAOS+BEAM+BOLT 20.15⚡/s.
- **ADAPTATION (declared):** `chaosMul` now rides arc-line and shard ticks
  ONCE, same as the straight thread — P-balance B1 (CHAOS +45%/stack)
  landed after this spec was written; shards inherit the boost with no
  double dip, the `splinterBurst` frag precedent (`chaos:0` on frags).
- **Verified (node probe, real extracted `runContinuousBeams`/`applyStatus`):**
  4 flash segs per loop · bulge-side bite at 13.2·dt with burn=hurt · one
  bite per enemy though out+home segs both graze him · truncation at a wall
  + 3 reflected shards that never pierce the face · planted-on-ray shard
  bite at 0.45× with burn=hurt · unclipped loop sheds nothing · TWIN 3 → 2
  mirrored loops at 0.65× · spark arc pulses at the apex · straight-thread
  reflect + flesh through-shatter · saw 4m halo, 2.7m shards · wave corridor
  untouched under RETURN. 28/28.
- **WATCHED:** arc TTK vs crowds behind low cover (the loop's whole point);
  knob = W 2.5→2.0 or loop reach 14→11, one line each. Degradation ORDER
  (aim > strands > joints > shards) needs the in-browser eyeball — damage-
  follows-draw is the invariant to test (no invisible damage).
- **Fields consumed:** none added — B1 reads `pend.boomerang`, B2 reads
  `pend.split` (both long-standing pend fields, already consumed by the
  projectile paths); `loopClip`/`arcApex` are per-strand frame locals.

## Balance ledger (v22 P-beamC: powder hitscan + contact drum)

Two items — the economy/resolver pack (`resolveLane` + `tryFirePlayer` +
`burstLaser` + `emitProjectile` + the `runContinuousBeams` drum). ZERO GLSL
edits; no world geometry, TWIN RULE not triggered. No new pend fields, no
save impact — `powder`/`small` live on resolve-time shot objects, `dAcc` on
lane state.

- **C1 — POWDER HITSCAN** (owner: "Powder core + beam should work to make
  beams cost ballistics. Hitscan laser gun w/ all effects of mods in
  barrel"): the named POWDER+BEAM trap is REDEEMED. A powder lane never
  threads (`runContinuousBeams` skips it) — it PULSES on the pull path:
  fire-rate-governed `fireHitscan` pulses, each 1 round + 0⚡, POWDER's
  waiver now standing under the laser. All lane mods live, mirroring the
  burst-laser aim block: SEEKER `seekBias` 12° lock, CHAOS bias+tilt, TWIN
  fan (headroom-gated; copies don't eat lead — projectile parity), DRILL
  pierce-all, HUSH/suppressor silence, autoSear = hitscan hose at 1rd/pulse.
  Damage `res.dmg+pend.dmg` (24 base — a bolt's paper dps, buying true
  hitscan + perfect converge + wall-mark). **REFUND+POWDER+BEAM = 1⚡ hard,
  0rd** (ADAPTATION: the spec said 10⚡ hard — the P-balance REFUND decree
  10→1 landed first; the hard surcharge still can't be waived, the number is
  just smaller now). The build's real price: LEAD + core loot rarity + three
  rack cells — powerful-as-described per powder's law and the owner's
  verbatim ask. **The mint is closed by routing:** `burstLaser` excludes
  powder, so burstCam+POWDER+BEAM can never ride `_laserBursts` at 0⚡/0rd —
  it fires 3 honest hitscans, 3 rounds (VERIFY in browser: mag drops by 3
  per pull). HUD: the seq chip now shows the rd price on a powder-laser lane
  (`!pend.laser||sh.powder` gate — pulses pay lead, the chip says so). Knob
  if playtest hates it: pulse dmg −2, never a mechanic change.
- **C2 — CONTACT DRUM** (owner: "Beam + contact should trigger housed status
  for core payloads and rapidly shoot smaller payloads at contact point"):
  the v22j BEAM exemption is NARROWED — a CONTACT+BEAM carrier now HOUSES
  its relay payload and the thread DRUMS half-size casts of it at the
  contact point every 0.5s (`st.dAcc`, aim strand k===0 only — TWIN
  multiplies the thread, never the drum). The beat is
  `{...relayNext, small:0.5, pend twin:1}` emitted `atPoint` at the thread's
  end: `small` halves dmg (instant boom: ×0.5 + rMul 0.75 replacing twin
  pooling; projectile path: whole dmg sum ×0.5, burn rounded ×0.5). Lead:
  `roundsFor(relayNext)` billed per beat at the carrier's mag — a dry mag
  HOLDS the beat (full head, fires the moment lead arrives; the thread keeps
  flowing). ⚡ automatic through the rate. A chain hanging off the beat
  (relayNext rides intact) casts FULL size and is priced full at the root;
  extras bill through `castRelay` as ever. TWIN on a drummed payload:
  I-by-design (beat forces twin:1), 12⚡ priced.
- **Per-core beat table** (0.5s cadence, half-size, at the contact point):
  BOOM blast at the dot ~17+0.5·dmg R×0.75 · SPARK half ember, burn ×0.5 ·
  BOLT half-dmg slug onward, 1rd/beat (2rd/s through-shooter) · WAVE half
  wall from the dot (≤4-5 live walls = old-train tracer class — WATCHED,
  fallback cadence 0.7s) · ORB half pearl laid at the dot (player cap 4 pops
  oldest — self-budgeting) · SLUG half fat blow, 3rd/beat = 6rd/s (lead is
  the governor) · WARP relay-warp from the dot — loud rapid mobility, noise
  per beat unless HUSH (17.4⚡/s bolt-carrier build; powerful-but-paid +
  telegraphed, kept per the law) · PICK carve/beat (uEdits capped) · GRIP
  glob/beat (FIFO 10) · LURE chirper/beat (noise machine, fine).
- **Worked price:** [CONTACT,BEAM,BOLT]+BOOM = 23.45⚡/s (13 carrier + 26
  housed through the rate) — dearer than the direct boom thread's 20.7 and
  it keeps the bolt line's 13.2 dps. Free-lunch note: drum lead billed per
  beat via `roundsFor` at the mag, chain priced full at the root.
- **Zealot side:** `tryShoot` already filters `pend.laser` AND `housed`
  lanes — a zealot rolling [CONTACT,BEAM,X,Y] simply mutes that lane's
  payload (rare roll, cosmetic, declared — slightly widened by C2).
- **Verified (node probe, real extracted `resolveLane`/`emitProjectile`/
  `burstLaser`/`runContinuousBeams`/`tryFirePlayer`):** housing + 39⚡ fold +
  23.45 rate · FUSE-only exemption stands · powder cost 0 under BEAM, REFUND
  hard 1 · burst mint closed (no `_laserBursts` entry, 3 pulses = 3 rounds) ·
  powder lane never threads (0 charge, 0 segs) · pulse pays 1rd/0⚡ at
  fire-rate · TWIN fans 2 pulses on 1rd · drum beats at ~0.5s at the contact
  point, half-dmg 17 / rMul 0.75 / spark burn 8 · TWIN carrier drums once ·
  drummed TWIN payload pools nothing · bolt beat bills 1rd, dry mag holds,
  fires the frame lead arrives. 52/52.
- **Fields consumed (NO DEAD STATS):** `shot.powder` → beam skip +
  `burstLaser` gate + pull routing/pulse + seq-chip rd display ·
  `shot.small` → both `emitProjectile` dmg lines + rMul + burn ·
  `st.dAcc` → the drum cadence. Nothing else added.
- **NOT VERIFIED IN BROWSER** (no browser here): the C1-4 mint gate
  (burstCam+POWDER+BEAM mag −3/pull), the pulse feel/converge, drum beats
  landing at the dot, WAVE-beat tracer load — fold into the shared manual
  pass.

## Zealot availability

`MOD_KEYS` += `waveCore`, `chaoticMod`, `boomerangMod`, `homingMod` (enemy
WAVE: 10 m/s, flat 16 dmg, dodgeable area-denial — intended; CHAOS wobble,
SEEKER acid-green and RETURN are all telegraphed by tracer tint; enemy
boomerangs return to their zealot, enemy seekers hunt the player — cone+range
only, no LOS, cover still beats them). EXCLUDED, with reasons: `warpCore` (never teleports the zealot —
decided; double-guarded by the `pr.owner!=='player'` check in `warpArrive`,
and never rolled), `sawCore` (their fixed shootCooldown ignores fireRate —
dead stat on them), `slugCore` (their tryShoot consumes 1 round flat — would
undercount), `powderCore`/`refundMod`/`beltFeeder` (zealots pay no ⚡ and use
the reserve model — economy mods are dead on them), `orbCore` (bare orb is
filler from a random pool), `boomCore` (v22i: BOOM is an instant blast at the
muzzle — a zealot bare-firing it through `tryShoot` would immolate itself;
never in `MOD_KEYS` nor any buildTool roll, and the exclusion is now declared
in a comment at `MOD_KEYS`. Their relay lanes stay bolt/spark by roll — no
boom there either).

## Balance ledger (P6 retune + accepted debts)

- **SPARK retune**: `base:18, burn:15` — blast-first with a real side of fire.
  **EMBER retune**: `base:22, burn:28` — commit-to-burn; 28 of the 30-cap pool
  ≈ 14s of 2 hp/s, +4⚡ over SPARK. Tracer col chain makes both read orange
  when burn>0 (`p.burn` tint precedes spark pink) — accepted: fire reads as
  fire.
- **Enemy spark burn WATCHED**: the zealot spark roll (<0.40 branch) now doses
  the player 15 burn × playerResist ≈ up to 15 hp over 7.5s — a real
  early-game difficulty bump. Fallback knob (one line, deferred): halve
  enemy-sourced doses in `applyStatus`. Ship, playtest, decide.
- **hairTrigger devalued next to SAW** — SAW floors the whole tool's cycle at
  0.06 for 1⚡; hairTrigger's −0.14 looks poor beside it. Accepted — different
  slot economy (stat slot vs a lane's proj core).
- **boltCore→POWDER upgrade path** — POWDER is a strict upgrade over BOLT when
  unfed (identical) and better when fed; gated only by loot rarity (cores tier
  vs common). Accepted as upgrade path; if it feels bad, the knob is powder
  dmg −2, not a mechanic change.

## Balance ledger (v22i combat-semantics rework)

- **CONTACT redefined** (owner ask: "should fire the NEXT core from its
  collision") — now the instant sibling of FUSE-relay: wires `relayNext`, casts
  the next core from the first touch, world or flesh; the carrier's own impact
  stays normal. Same 5⚡. Precedence: FUSE governs on a shared core (chosen
  over the reverse because a fused BOOM needs its flight back — CONTACT-governs
  would deadlock the instant blast), CONTACT vents, surcharge still real.
  CONTACT now also relay-houses a SLUG ([CONTACT,X,SLUG] dodges the +0.45s
  cycle penalty, same as FUSE housing — the `relayNext` check is shared).
  Old behavior (trigger-payload-on-first-impact with no next core) collapses
  into "plain impact" — nothing lost: bare payload-kinds already trigger on
  impact.
- **BOOM instant** (owner ask: "instant explosion, not a projectile") — same
  26⚡, same blast (R 4.5, dmg 34+pend). Emission point nudged 1.2m forward on
  muzzle fire so bare-fire is survivable-but-stupid (~22 self at 1.5m), not
  suicide; relay/contact casts detonate exactly AT the cast point (`atPoint`
  arg through `triggerPayload`→`emitProjectile`). FUSE restores flight (the
  old lob, now opt-in): stick+timer+blast. TWIN = one blast ×1.4 radius via
  `explode`'s new `rMul` — a nerf vs the old 2 projectiles, deliberate (12⚡
  buys area). Flight-only feeders (SEEKER/RICOCHET/DRILL/RETURN/SWIFT/MASS)
  are inert on a bare boom, surcharge still real — the I-column pattern.
  burstCam stacks its blasts same-frame at one ⚡ price incl. self-damage ×3:
  accepted as T-soft. Zealots structurally excluded (see availability).
- **BEAM+BOOM = the drum** (owner ask: "continuous small explosions") — the
  thread's contact point pulses a small charge every 0.45s: R 1.8
  (2.4×`rMul:0.75`), dmg 14 (+0.6·pend.dmg, ×0.65/strand under TWIN-split),
  replacing the old 1.1s half-damage big blast. Priced by the standing
  per-second formula: 2+(26+8)·0.55 = **20.7⚡/s** — vs regen 16 it runs a
  −4.7⚡/s deficit, a battery-limited siege verb (SPARK-thread, for scale:
  16.3⚡/s). Self-damage applies inside 1.8m — point-blank thread work stings.
- **SEEKER bends the thread** (owner ask: "beam should home with seeker") —
  bare SEEKER×BEAM graduates from inert to a 12° aim-lock (`seekBias`; full
  lock inside the cone, the gate is the clamp — no turret sweeps). Costed by
  SEEKER's 7⚡ through the beam rate: SEEKER+BEAM+BOLT = **10.25⚡/s** (bolt
  thread alone: 6.4). CHAOS still outranks it on a shared thread (chain
  lightning, 40°). Twin strands bend to the SAME mark by design — readable
  over lethal. Grip/pick threads never bend; orb trains already seek as
  projectiles (the wave corridor never seeks — v22 P-beamA).
