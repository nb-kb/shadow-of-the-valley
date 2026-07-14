# Balance Number Reference — Shadow of the Valley

Tuning ledger for the beta 1.1.1 pass. Line numbers are against `index.html` on
branch `beta-1.1.1`; they drift — re-grep before editing.

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

> Note (beta 1.1.x): the enemy HP/damage + undead numbers above are current; the
> ballistic AMMO system (calibers, magazines, powder detonation) is not yet
> ledgered here — those live in the code + docs/MODS.md.

### Player projectile "core" damage
Note: in the core defs, `base:` is the **⚡ energy cost, not damage**. Actual
impact damage per core kind is hardcoded in the emit function:
- `index.html:4436` — per-kind damage table: `boom 34 · saw 8 · slug base.dmg×3 · wave 16 · warp 6 · orb 6 · spark/pick/decoy 12 · else(bolt) base.dmg`, then `+ p.dmg` (mods) `× chaosMul × small`.
- `index.html:4419` — **34** — BOOM instant-detonation path (`34 + p.dmg`).
- `index.html:4230` — **24** — `base.dmg` default chamber damage (BOLT; SLUG ×3 → 72).

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

### Headshots
- `index.html:5207-5212` — `headshotDmg(en,dmg,head)`: no head → `dmg`; bare head → **×4**; helmeted head → cracks helm, returns `dmg` (×1, helm eats the bonus).
- `index.html:5200-5206` — `crackHelm(en)` — decrement `helm.dur`; destroyed at 0.

---

## ARMOR

### Armor item defs (`ITEM_DEFS`, literals)
Plates (body — multiply incoming):
- `index.html:3091` — **Scrap Plates** `damageMult 0.75`, `dur 20`.
- `index.html:3092` — **Ceramic Plates** `damageMult 0.6`, `dur 30`.

Helmets (head — shave incoming):
- `index.html:3093` — **Padded Cap** `reduce 0.94`, `dur 10`.
- `index.html:3094` — **Steel Pot** `reduce 0.88`, `dur 18`.
- `index.html:3095` — **Rig Helmet** `reduce 0.90`, `medPort`, `dur 14`.

Per-piece burn/bleed resist (rolled per instance, any `dur` armor):
- `index.html:3141` — `resist.burn` & `resist.bleed` each = `round(rng()·60)/100` → **0–0.60** random.
- `index.html:6360` — save-load clamp 0–0.6.

### Where armor reduces damage
- **Player — `damagePlayer`** `index.html:5580-5581`: `if(plates) dmg *= plates.damageMult;` then `if(helmet) dmg *= helmet.reduce;`. Applied to ALL incoming hits (no separate head channel on received fire yet); each wears 1 durability (`armorWear` `:5551`).
- **Enemy — `Zealot.hurt`** `index.html:5718-5720`: plates `×damageMult` for non-head hits only (head bypasses plates; helm cut already applied in `headshotDmg`).
- **Status resist (both)** — `playerResist` `:5560`, `Zealot.resist` `:5710`: `m *= 1 − resist[kind]` across worn pieces, in `applyStatus`.

### Helmet crack
- `index.html:5686` — zealot helm roll by depth.
- `index.html:4826/4863/5020` — beam ticks crack helm every 0.4 s.
- `index.html:5210` — headshot cracks a helmeted enemy (consumes the ×4).
- `index.html:5734` — helm drops as world item on death.

---

## FORMULAS — where damage-after-armor is computed

- **`damagePlayer(dmg)`** `index.html:5575-5589` — the ONLY player-damage entry. Plates ×, helmet ×, wear, subtract, death. **← rework player armor here.**
- **`Zealot.hurt(dmg,by,part)`** `index.html:5715-5726` — enemy-damage entry; plates × for body hits. **← enemy body armor here.**
- **`headshotDmg(en,dmg,head)`** `index.html:5207-5212` — bare ×4 / helmeted ×1 + crack. **← head channel / helmet here.**
- **`crackHelm(en)`** `index.html:5200-5206` — helmet durability.
- **`explode(pr)`** `index.html:5513-5529` — blast falloff `×(1−d/R)×1.6` both sides + burn.
- **`applyStatus(t,pr)`** `index.html:5542-5549` — burn/bleed application + resist.
- **`armorWear(slot)`** `index.html:5551-5557` — durability tick + break-off.
