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

| key | name | effect | base ⚡ | behavior | GPU |
|---|---|---|---|---|---|
| `boltCore` | BOLT | `{kind:'bolt', base:0}` | 0 (+1 round) | Kinetic slug from the magazine. | cheap |
| `sparkCore` | SPARK | `{kind:'spark', base:18, burn:15}` | 18 | Bursting ember — the fire rides the blast now. Base dmg 10 + 15 burn dose. Niche: blast-first with a real side of fire. | cheap |
| `emberCore` | EMBER | `{kind:'spark', base:22, burn:28}` | 22 | Fire that keeps eating — commit-to-burn. 28 of the 30-cap pool ≈ 14s of 2 hp/s. Niche: +4⚡ over SPARK buys the near-full pool. | cheap |
| `pickCore` | PICK | `{kind:'pick', base:14}` | 14 | Eats stone (SDF carve), spares flesh. Base dmg 14. | uEdits (capped) |
| `gripField` | GRIP | `{kind:'grip', base:6}` | 6 | Glue glob; sticks to world + kin. Behind BEAM → physgun. | cheap |
| `decoyCore` | LURE | `{kind:'decoy', base:10}` | 10 | Chirping puck — sticks where it lands, sings to the camp for 12s. | cheap |
| `boomCore` | BOOM | `{kind:'boom', base:26}` | 26 | THE BLAST PRIMITIVE — no flight, no tracer slot: it detonates the frame it is cast (R 4.5, dmg 34+pend, ×1.6 vs flesh w/ falloff — hurts the shooter too). Bare muzzle fire goes off ~1.2m out: survivable, stupid. Relay/contact-cast (`atPoint`) it goes off AT the cast point — [CONTACT,BOLT,BOOM] = explosive rounds, [FUSE,X,BOOM] = X's death detonates. FUSE on the boom ITSELF restores the lob (flies heavy, sticks, blows after the fuse — the timed charge). TWIN pools, never fans: ONE blast ×1.4 radius. BEAM-fed: a drum of small charges (see the beam notes). | cheap |
| `waveCore` | WAVE | `{kind:'wave', base:10}` | 10 | Wide slow wall of pressure: 10 m/s, dmg 16, washes a ~1.8m swath, each foe pays ONCE (`pr.hit`/`pr.hitPl` sets), shove 7 + stagger. Never impact-dies on flesh (pierce inert on it BY DESIGN); the world still kills it. Enemy-fired = flat 16, dodgeable. | 1 tracer slot (perp-bar) |
| `sawCore` | SAW | `{kind:'saw', base:1}` | 1 | Teeth, not reach: unlocks the WHOLE tool's cycle to the 0.06 floor; its own bite dmg 8, life 0.16s ≈ 4m at speed 26 (SWIFT is THE reach knob). 16.7⚡/s vs regen 16 + wearRoll per tooth: self-governing attrition. | cheap (~3 live teeth) |
| `warpCore` | WARP | `{kind:'warp', base:15}` | 15 | You arrive where it dies — impact, fuse, or the end of its arc. Speed 40, dmg 6. Arrival is LOUD (noise 18 + muzzle flare) unless HUSH-fed (noise 4, both ends). Sheds threads at resolve. Player-only: zealots never roll it AND `warpArrive` owner-guards. Rare loot tier. | cheap |
| `powderCore` | POWDER | `{kind:'bolt', base:0, powder:true}` | 0 (+1 round) | A bolt that pays in lead: its feeders' ⚡ surcharges are waived — except REFUND's own (`surchargeHard`), and never under a thread (BEAM keeps full price — the named trap). | cheap |
| `slugCore` | SLUG | `{kind:'slug', base:0}` | 0 (+3 rounds) | Three rounds, one fat blow: ×3 TOOL hurt (`base.dmg*3 + pend.dmg` — HURT+ additive AFTER the ×3), speed 48, fat radii (0.9 vs flesh, 0.30 vs world). Whole tool +0.45s cycle unless relay-housed (some shot's `relayNext`). Sheds threads at resolve. | cheap |
| `orbCore` | ORB | `{kind:'orb', base:8}` | 8 | Just an orb: 6 m/s, dmg 6, 6s aloft then its payload pops. The blank canvas — it becomes what you feed it. Cap 4 player orbs live; the 5th pops the oldest. BEAM noses it (puppeteer). | hoards tracer slots 6s — capped, monitor |

## Sequence modifiers (`cat: mod`) — charge the NEXT core

| key | name | effect | ⚡ | consumed at | GPU |
|---|---|---|---|---|---|
| `dmgPlus` | HURT+ | `{dmg:10}` | 5 | `pend.dmg` → shot dmg | cheap |
| `velPlus` | SWIFT | `{vel:1.6}` | 4 | `pend.vel` ×= (proj speed) | cheap |
| `bounce` | RICOCHET | `{bounce:2}` | 6 | `pr.bounces` (index.html:3599) | cheap |
| `pierce` | DRILL | `{pierce:2}` | 8 | `pierceLeft` (index.html:3735) | cheap |
| `heavy` | MASS | `{grav:1, dmg:6}` | 5 | `pr.vel.y-=grav*12*dt` (3838) | cheap |
| `twin` | TWIN | `{twin:2}` | 12 | shot count `round(twin)`, **cap 3** | cheap |
| `silentMod` | HUSH | `{silent:true}` | 3 | suppresses shot noise (3654) | cheap |
| `fuseMod` | FUSE | `{fuse:1.2}` | 4 | sticks, blows after 1.2s — then RELAYS: casts the next core in the lane from the blast point (`relayNext`; TWIN copies don't chain — only the first carries it). On the same core FUSE outranks CONTACT | cheap |
| `contactMod` | CONTACT | `{contact:true}` | 5 | FUSE-relay's instant sibling: on the charged core's FIRST TOUCH (world or flesh) it casts the NEXT core in the lane from the hit point (`relayNext`) — no timer, no stick. The carrier's own impact stays normal (damage, decal). No next core in the lane → plain impact, the mod vents (like FUSE with nothing above it, the cast is empty). CONTACT+FUSE on one core: FUSE governs (stick+timer), CONTACT vents — its 5⚡ still paid. TWIN first-copy rule applies (copies don't chain) | cheap |
| `slowFuse` | TIMER | `{fuseAdd:1.5}` | 3 | `pend.fuse += 1.5` — more patience on the same fuse; stacks, and extends FUSE's 1.2s | cheap |
| `splinter` | SPLINTER | `{split:3}` | 7 | shatters into 3 on impact (3889) | cheap |
| `boomerangMod` | RETURN | `{boomerang:true}` | 6 | shot stalls ~0.55s, banks, flies home to the thrower's current position; despawns at ~0.6m ("caught"). Both passes damage; walls still kill it; bounce is overridden on the return. Enemy-fired → returns to its zealot (`pr.src`). | cheap |
| `homingMod` | SEEKER | `{homing:1}` | 7 | per-frame steer toward nearest live, non-dormant enemy in a ~40° forward cone, ≤45m, turn clamp 3.2 rad/s (stacks: ×N). Zealot-fired seeks the player. Acid-green tracer telegraphs it. RETURN's return phase overrides it. BEAM-fed (v22i): the damage thread BENDS fully onto the nearest mark within a ~12° cone of the aim (`seekBias`, 45m, no LOS — the cone is the clamp); twin strands share the one mark; grip/pick never bend; its 7⚡ prices through the thread's per-second rate. | cheap |
| `beamCore` | BEAM | `{laser:true}` | 8 | shot → continuous thread; alone → tac laser | cheap |
| `refundMod` | REFUND | `{refund:true, surcharge:10}` | 10 (never waived) | `pend.refund` → resolveLane cost formula (waives next core's base ⚡) + `roundsFor()` (waives lead). Its own 10⚡ rides `pend.surchargeHard` — the one no core forgives, POWDER included. Idempotent flag: REFUND+REFUND = 20⚡ of nothing. | cheap |
| `chaoticMod` | CHAOS | `{chaos:1, surcharge:6}` | 6 | `pend.chaos` → `pr.chaos` steer wobble (2.4·chaos rad/s, return leg immune); threads arc like lightning (endpoint wander ×(1+0.6·chaos) + 3-joint strand render); pulses jitter. With SEEKER on threads: arcs bite the nearest warm thing (chain lightning). Stacks: wider wander. GRIP/PICK threads stay straight BY RULE. Zealots CAN roll it — white-blue tracer telegraphs the wobble. | 3 one-frame flash segs per strand, gated on `flashHeadroom()>=3` (degrades to 1 straight) |

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
fuse:0, split:0, contact:false, burn:0, bleed:false, boomerang:false, homing:0,
surcharge:0, laser:false, refund:false, surchargeHard:0, chaos:0}`. Each modifier
adds/multiplies its field; a proj core copies `pend`, then it resets.
**All fields are consumed downstream — no dead stats** (re-verified, mod wave v1):

- `contact` — consumed at (a) resolveLane's relay wiring (`relayNext`, the
  instant sibling of FUSE's), (b) the flight first-touch branches
  (`updateProjectiles` world hit + `impactOrPierce` flesh hit), both guarded
  `&&!(pr.fuse>0)` — FUSE governs on a shared core.
- `refund` — consumed at (a) the resolveLane cost formula (waives the core's
  base ⚡), (b) `roundsFor()` (waives lead).
- `surchargeHard` — consumed at the resolveLane cost formula: the un-waivable
  slice of the surcharge pool. Only REFUND writes it; POWDER cannot touch it.
- `chaos` — consumed at `emitProjectile` (→`pr.chaos` steer wobble),
  `runContinuousBeams` (arc render + endpoint wander + homing bias), and
  `runBurstLasers`/`fireHitscan` (pulse jitter + jagged flash).

Effect-only flags (never land on `pend`): `powder` (read in the resolveLane proj
branch cost formula), `beltFeed` (foldFrame → `out.beltFeed` → `tickBeltFeed`).
New `pr` fields: `chaos` (steer block), `hit`/`hitPl` (wave once-per-foe) — all
consumed in `updateProjectiles`.

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
| **TIMER** (tunable) | mod | `{delay:t}` | ? | generalize FUSE's fixed 1.2 s into an authored delay | **done** = TIMER (`slowFuse`, `{fuseAdd:1.5}`) |
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
  C-P3 wave train · burstCam F (3 stacked walls) · RETURN F (the tide comes
  back) · SEEKER F (homing wall) · DRILL I (wave never impact-dies on flesh —
  pierce unread on it) · RICOCHET F (sweeps back off the wall) · SPLINTER F
  (dies on wall → 3 bolts) · HUSH F · MASS F · SWIFT F · HURT+ F ·
  GRIP/PICK/LURE – · BOOM via FUSE-chain F.
- **SAW**: FUSE-relay F (point-blank tooth from the impact) · CONTACT F ·
  TIMER F · TWIN F · BEAM C-P2 4m chainsaw · burstCam: plain burst F,
  laser-burst S · RETURN **T "YO-YO NO"** (life 0.16 < stall 0.55 — the tooth
  dies before it turns; 6⚡ for nothing) · SEEKER F (barely turns, honest) ·
  DRILL F · RICOCHET F · SPLINTER F (dust storm) · HUSH F (4⚡ silent teeth) ·
  MASS F · SWIFT F (THE reach knob) · HURT+ F · frame-level: SAW+SLUG
  **T "THE WOODCHIPPER"** (order law: floor 0.06 then +0.45 → 0.51s cycle —
  looks like a fat machine gun, is a slow drum-mugging) · SAW+autoSear F (the
  true saw) · SAW+hairTrigger I.
- **WARP**: FUSE-relay F, both seats ([FUSE,WARP]=delayed door;
  [FUSE,X,WARP]=cast onward from X's death; [FUSE,WARP,X]=you arrive as X
  casts) · CONTACT F ([CONTACT,WARP,X]: you blink to the impact AND X casts
  from it, instantly) · TIMER F · TWIN T-soft
  (double-blink, 12⚡ to be yanked twice) · BEAM S (laser stripped; BEAM vents
  its 8 into the chip) · burstCam F (triple-blink, self-punishing, allowed) ·
  RETURN **T "THE LASSO"** — the named headline trap: looks like a recall
  rope; the catch splices with NO payload → 21⚡ to teleport nowhere unless it
  clips a wall mid-return. Decided: kept, deliberately · SEEKER F
  "SWAP-STRIKE" (blink onto the mark) · DRILL I · RICOCHET F "BANK SHOT"
  (bounce a blink around the corner) · SPLINTER F (arrive in a shower) ·
  HUSH F "GHOST STEP" (whisper both ends) · MASS F (grenade-arc jump) ·
  SWIFT F (longer blink) · HURT+ F · BOOM via FUSE F "BREACH-AND-ENTER"
  ([FUSE,BOOM,WARP]: the blast opens the wall, the warp carries you through).
- **POWDER**: all sequence feeders F with surcharges waived (the point) ·
  REFUND C-P1 (lead waived instead of base; 10⚡/0rd sidearm — pockets-empty
  tech) · BEAM **T** (waiver refuses threads — you built a budget laser and
  got a full-price one) · FUSE-relay F (cycle gate keeps it honest) · TWIN F
  (0⚡ doubling; rack cells + cap 3 + lead are the price).
- **SLUG**: FUSE-relay F + relay-housed exemption (the "in trigger" rule) ·
  CONTACT F · TIMER F · TWIN F (3 fat slugs, still 3 rounds total — copies
  don't eat lead, priced by the cycle penalty) · BEAM S · burstCam
  **T "DRUM-EATER"** (9 rounds a pull; partial-burst break mirrors bolt) ·
  RETURN F (fat boomerang, both passes ×3) · SEEKER F · DRILL F · RICOCHET F ·
  SPLINTER F (32-dmg frags — watched) · HUSH F · MASS F (mortar slug) ·
  SWIFT F · HURT+ F (additive AFTER ×3, declared).
- **REFUND**: energy cores F (base→0) · ballistic C-P1 (lead→0) · BEAM-fed I
  (base already discounted by rate math; its 10 inflates the rate —
  anti-synergy, documented) · REFUND+REFUND T (idempotent flag, 20⚡ of
  nothing) · TWIN F (12 stands — guardrail).
- **BELT-FEEDER**: orthogonal util — F with everything · SLUG note: 1.1rd/s
  trickle vs 3rd/shot, honest · suppressor/silenced gates its tick noise ·
  zealots excluded.
- **CHAOS**: projectiles F (wobble) · SEEKER F (drunken missile — both steers
  run) · BEAM C-P5 lightning · BEAM+SEEKER C-P5 **CHAIN LIGHTNING** (the
  owner's ask, verbatim; on a shared thread CHAOS's 40° arc bias replaces
  SEEKER's 12° bend) · burstCam pulses
  C-P5 · GRIP/PICK threads S (stay straight) · RETURN F (return leg immune —
  the way home flies true) · WARP F (drunken blink, self-priced) · everything
  else F.
- **BOOM** (instant since v22i — the blast primitive): bare fire F-but-dumb
  (blast 1.2m off your own muzzle, self-damage stands) · CONTACT-carrier F
  **"EXPLOSIVE ROUNDS"** ([CONTACT,BOLT,BOOM]: every hit detonates AT the
  wound/wall) · FUSE on the boom itself F (restores the lob: flies, sticks,
  timed charge; TIMER does too — ANY fuse time buys the flight back;
  [FUSE,BOOM,WARP] BREACH-AND-ENTER intact) · FUSE/CONTACT on a
  CARRIER F (the death/touch detonates at the point) · TWIN: ONE blast ×1.4
  radius, never N blasts (12⚡ buys area, not count) · SEEKER / RICOCHET /
  DRILL / RETURN / SWIFT / MASS on a bare boom I (no flight to steer, bounce,
  pierce, return or hasten — surcharge still real; FUSE-fed they read again,
  it's a projectile then) · CONTACT on the boom itself I (nothing to touch —
  it vents; put CONTACT on the carrier) · SPLINTER F (blast + 3 frags from the
  fire) · burstCam T-soft (three stacked blasts in your own face for one ⚡
  price — three doses of your own medicine) · BEAM C (the drum, see ledger) ·
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
one mark, grip/pick stay true, wave/orb trains keep seeking as projectiles).
With CHAOS the wider 40° chain-lightning bias (`chaosBias`) takes over instead
— both consume homing on threads; the 7⚡ prices through the beam's
per-second formula either way (e.g. SEEKER+BEAM+BOLT: 2+(0+7+8)·0.55 =
10.25⚡/s).

## The named traps (deliberate, all kept)

| trap | build | what it looks like | what it is |
|---|---|---|---|
| THE LASSO | RETURN+WARP | a recall rope | the catch splices with NO payload — 21⚡ to teleport nowhere unless it clips a wall mid-return |
| THE WOODCHIPPER | SAW+SLUG | a fat machine gun | order law: floor 0.06 then +0.45 → 0.51s cycle, a slow drum-mugging |
| YO-YO NO | RETURN+SAW | a boomerang blade | tooth life 0.16 < stall 0.55 — it dies before it turns; 6⚡ for nothing |
| DRUM-EATER | burstCam+SLUG | triple fat blows | 9 rounds a pull; partial-burst break mirrors bolt |
| POWDER+BEAM | POWDER under a thread | a budget laser | the waiver refuses threads — full price under BEAM |

## Free-lunch audit (all closed)

1. POWDER+REFUND → refund's 10 rides in `surchargeHard`, powder can't waive it
   → floor 10⚡/0rd. CLOSED.
2. REFUND+TWIN → refund waives BASE only; TWIN's 12 stands → twin BOOM = 22⚡
   not 12. CLOSED.
3. POWDER feeding a relay chain → relay casts stay free (pre-existing design)
   but the CYCLE GATE holds: the relay'd core sits in the lane cycle and must
   be paid at its own turn or the column starves. Powder saves only its
   feeders' sur. CLOSED, documented.
4. BELT-FEEDER+SLUG → trickle 1.1 rd/s < 3 rd/shot; the belt moves lead,
   never mints it. CLOSED.
5. POWDER+BEAM → waiver disabled under `pend.laser` (else 2⚡/s lead-free
   thread). CLOSED. (Also the named POWDER trap.)
6. SLUG/WARP + BEAM/burstCam → laser stripped at resolve + burstLaser excludes
   new kinds → no round-free fat threads, no 60m hitscan blinks, no ×3-dmg
   free pulses. CLOSED.
7. SAW spam → 16.7⚡/s vs regen 16 + wearRoll per tooth: self-governing
   attrition. CLOSED.

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
  over lethal. Grip/pick threads never bend; wave/orb trains already seek as
  projectiles.
