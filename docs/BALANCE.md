# Balance Number Reference — Shadow of the Valley

Tuning ledger for the beta 1.1.x pass. Line numbers are against `index.html` on
branch `beta-1.1.1` and have drifted further on `feat/containment` (the
containment wave re-baselined the headshot/armor sections below, 2026-07-15) —
re-grep before editing.

**PARAM** = live-editable in the debug console (in the `PARAMS = {…}` block,
~line 380). Everything else is a hardcoded literal in the JS.

---

## HP

### Player
- `index.html:2116` — **100 / 100** — starting `health` / `maxHealth`.
- `index.html:5541` — `maxHealth − burnedMax` — effective max HP (`effMaxHP()`); burn scars the ceiling until treated.
- `index.html:7630` — cap `maxHealth−10` — burn can't scar below 10 HP of ceiling.
- `index.html:5958` — **+25** — heal each time you advance a depth/leg.
- `index.html:5979` — `= maxHealth` — full heal on run start / boot.
- `index.html:6009` — `max(1, round(maxHealth·0.3))` — respawn at 30% max HP.

Consumables (`useItem`):
- `index.html:4302` — **+14** — Field Ration.
- `index.html:4307` — **+30** — Gummies.
- `index.html:4312` — clears bleed — Band-Aid.
- `index.html:4317` — clears burn + `burnedMax` — Aloe Vera.

### Enemies (Zealot ctor — faction/type ternary; grep `this.health=this._brute`)
Current roster (beta 1.1.x, owner-tuned "tougher"):
- **Zealot** (soldier) — **200 HP**.
- **Believer** (soldier, kamikaze bomber) — **100 HP**.
- **Zombie** (undead — no armor, no loot) — **100 HP**.
- **Brute** (undead mini-boss) — **800 HP**.
- Set in the ctor + BOTH `reinforce()` paths + the HOPE heal cap — keep in
  lockstep (grep `_brute?800` / `_believer?100:200`).
- Respawn: soldiers `respawnT=45 s`; rank undead **300–600 s (5–10 min)**; the
  lone brute on a **15-min** game-time clock (`bruteRespawnT`).
- Render cap `RLIM.ENEMIES=20` (was 10); horde spawn 10 soldiers / 9 zombies /
  1 brute at two rotating POIs. See KNOWN_BUGS #2 (compile-budget load-test).

No other entity has HP (crates/props aren't damageable).

---

## DAMAGE

### Enemy → player
- `index.html:391` — **11** — `enemyDamage` **PARAM** (zealot bolt dmg, min 0 / max 40).
- **8** — soldier point-blank swipe when dry on ammo (`dist<1.4`).
- **Undead lunge** (grep `this._brute?24:12`): zombie **12**, brute **24** per
  hit on a 1 s cadence — melee only; the undead never shoot.

> Note (beta 1.1.x, updated 2026-07-15): the enemy HP/damage + undead numbers
> above are current. The ballistic AMMO system LANDED (the two-pool combat
> pass, 3c2430b…bc4445e) — docs/MODS.md's ⚠ CORRECTED MODEL is authoritative.
> Caliber catalog (`const AMMO`, grep it): 9mm 24 (== chamber default, inert) ·
> 7.62 35 + armorPen 0.5 · flechette 4×5 + 20% bleed · slug ×3 + 0.45s delay ·
> buckshot 5×24 wide fan. POWDER + SLUG **cores** are retired — powder is a
> Powder-Mag trait (anyCaliber + free relay mods when powered), slug a caliber.

### Player projectile "core" damage
Note: in the core defs, `base:` is the **⚡ energy cost, not damage**. Actual
impact damage per core kind is hardcoded in the emit function:
- `index.html:4436` — per-kind damage table: `boom 34 · saw 8 · wave 16 ·
  warp 6 · orb 6 · spark/pick/decoy 12 · else(bolt) ballistic base`, then
  `+ p.dmg` (mods) `× chaosMul × small`. Post-rework the bolt's base is the
  LOADED CALIBER via `_ballBase` (grep it — `mult` calibers scale the chamber
  24, flat calibers replace it); the old `slug base.dmg×3` CORE row is retired.
- `index.html:4419` — **34** — BOOM instant-detonation path (`34 + p.dmg`).
- `index.html:4230` — **24** — `base.dmg` default chamber damage (BOLT with a
  9mm or bare fallback; the slug CALIBER ×3 → 72).

Damage mods (added to `p.dmg`):
- `index.html:3049` — **+8** — Heavy Frame barrel.
- `index.html:3069` — **+10** — HURT+.
- `index.html:3073` — **+15** — MASS.

### Melee
- `index.html:4597` — **18** — whip-snap melee.
- `index.html:4594` — **400** — stealth takedown (from behind, `bd<1.6`).
- `index.html:4580` / `:4586` — reach `2.2`, arc dot `>0.7`.

### Explosion / blast
- `index.html:5519` — radius `R = (boom?4.5:2.4)·rMul` (TWIN boom ×1.4, thread pulse ×0.75).
- `index.html:5523` — enemy dmg = `pr.dmg·(1−d/R)·1.6` (linear falloff, ×1.6 at center).
- `index.html:5526` — player self-blast = `pr.dmg·(1−pd/R)·1.6`, then armor.
- `index.html:5471` — SPLINTER shard = `max(2, round(pr.dmg·0.45))`.
- `index.html:4868/4869` — SPARK/BOOM thread pulses.
- `index.html:4871` — PICK carve dmg 8.
- `index.html:7676/7696` — **34** — worn/dropped BOOM self-detonation.

### Status (burn / bleed)
- `index.html:5545` — burn pool cap **30**.
- `index.html:5547-5548` — bleed cap **5** stacks, **15 s** timer.
- `index.html:5753 / 7628` — burn tick **2 HP/s**.
- `index.html:5757 / 7632` — bleed tick **1 HP/s per stack**.
- `index.html:3056` SPARK burn **8**; `index.html:3057` EMBER burn **15**.
- Blast burn = `round(pr.dmg·fall·0.35)`.
- `index.html:3076` — SERRATED mod → +1 bleed stack.

### Headshots — P5 SYMMETRIC TIERS (containment wave, spec F — supersedes the old ×4 model)
- `headshotDmg(en,dmg,head)` (~L5729): no head → `dmg`; **bare rank head →
  9999 (one shot ENDS a zombie/zealot/believer)**; the **800hp BRUTE alone
  keeps the old ×4** (a one-tap mini-boss would gut the P4 lockbox hunt —
  owner-flagged); helmeted head → `crackHelm`, returns `dmg` (the helm eats
  the kill).
- `crackHelm(en)` (~L5721) — spends **`hs`, not `dur`** (dur stays the
  body-wear channel; a dropped helm carries its cracks): **light tier hs 1
  (2 shots end you), heavy tier hs 2 (3 end you)**; helm vanishes at 0.
- **The player wears the same law** — `damagePlayer(dmg,head)` (~L6154) grew
  the head channel: enemy rounds test a head sphere riding the eye line; bare
  head = the shot kills; a worn helm spends `hs` (HELMET CRACKED toast),
  applies `×reduce`, and SHATTERS through the armorWear break path when spent
  (seated filters spill). **Plates never read heads.**
- Hit geometry (P5 honest-body): enemy body = ONE vertical capsule, axis
  y+0.20..y+1.40, r **0.32** (brute **0.44**, riding the GLSL `d-=0.12`
  inflate) — `enemyBodyR`/`enemyBodyDist` ~L5716; head sphere `enemyHeadR`
  = **0.24** (brute **0.30**) ~L5711. Slugs keep a +0.30 fat-ordnance grace.
- Continuous THREADS stay contact ticks, not shots: bare-head ×2 and the 0.4s
  crack cadence stand (a beam held on a head strips a helm in 1-2 cadences —
  priced, aimed, watch it in the playtest).

---

## ARMOR

### Armor item defs (`ITEM_DEFS`, literals)
Plates (body — multiply incoming):
- `index.html:3091` — **Scrap Plates** `damageMult 0.75`, `dur 20`.
- `index.html:3092` — **Ceramic Plates** `damageMult 0.6`, `dur 30`.

Helmets (head — shave incoming; `hs` = headshots eaten before shattering, P5):
- **Padded Cap** `reduce 0.94`, `dur 10`, **hs 1** (light).
- **Steel Pot** `reduce 0.88`, `dur 18`, **hs 2** (heavy).
- **Rig Helmet** `reduce 0.90`, `medPort`, `dur 14`, **hs 2** (heavy — the
  heavy read is owner-tunable, flagged in 0c0dc37).
- **Respirator Helm** (gas kit, P3) `reduce 0.92`, `dur 12`, **hs 1** (light),
  `mask` with **2 filter seats** — max gas safety forgoes the heavy helmet.

Gas kit (P3 — protection lives in LIVE filters, not the mask shell):
- **Filter Mask** — `face` slot (new EQUIP slot, rides UNDER any helmet),
  `mask`, **1 filter seat** — max head armor costs a filter seat.
- **Gas Filter** — cartridge, `life 600` (breathes down ONLY in the gas,
  ~1-2 casual runs); each live filter **−45% gas burn, −90% cap**
  (`radProtection`).

Per-piece burn/bleed resist (rolled per instance, any `dur` armor):
- `index.html:3141` — `resist.burn` & `resist.bleed` each = `round(rng()·60)/100` → **0–0.60** random.
- `index.html:6360` — save-load clamp 0–0.6.

### Where armor reduces damage
- **Player — `damagePlayer(dmg,head)`** (~L6154): BODY hits — `if(plates)
  dmg *= plates.damageMult;` then `if(helmet) dmg *= helmet.reduce;`, each
  wearing 1 durability (`armorWear`). HEAD hits (P5) ride the head channel —
  see Headshots above: bare = death, helm spends `hs` + `×reduce`, plates
  never read heads. (The old "no separate head channel on received fire yet"
  note is retired.)
- **Enemy — `Zealot.hurt`** `index.html:5718-5720`: plates `×damageMult` for non-head hits only (head bypasses plates; helm cut already applied in `headshotDmg`).
- **Status resist (both)** — `playerResist` `:5560`, `Zealot.resist` `:5710`: `m *= 1 − resist[kind]` across worn pieces, in `applyStatus`.

### Helmet crack
- `index.html:5686` — zealot helm roll by depth.
- `index.html:4826/4863/5020` — beam ticks crack helm every 0.4 s.
- headshot on a helmeted enemy spends `hs` (consumes the kill — see Headshots).
- `index.html:5734` — helm drops as world item on death (carries its cracks:
  `hs` rides itemToJSON/FromJSON as an additive v6 field, clamped to the def
  tier on read).

---

## FORMULAS — where damage-after-armor is computed

- **`damagePlayer(dmg,head)`** (~L6154) — the ONLY player-damage entry. Head
  channel (bare = death / helm `hs`), plates ×, helmet ×, wear, subtract,
  death. **← rework player armor here.**
- **`Zealot.hurt(dmg,by,part)`** (~L6400) — enemy-damage entry; plates × for
  body hits (head bypasses plates). **← enemy body armor here.**
- **`headshotDmg(en,dmg,head)`** (~L5729) — bare rank head 9999 / brute ×4 /
  helmeted ×1 + `hs` crack. **← head channel / helmet here.**
- **`crackHelm(en)`** (~L5721) — helmet `hs` spend (dur stays body-wear).
- **`explode(pr)`** `index.html:5513-5529` — blast falloff `×(1−d/R)×1.6` both sides + burn.
- **`applyStatus(t,pr)`** `index.html:5542-5549` — burn/bleed application + resist.
- **`armorWear(slot)`** `index.html:5551-5557` — durability tick + break-off.
