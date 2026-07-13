# beta 1.1.1 Content Patch — Implementation Plan

Master reference for the big ammo/enemy/armor/UI patch. Produced by a parallel
subsystem-mapping + synthesis + adversarial-critique pass. Line numbers are
`index.html:LINE` on branch `beta-1.1.1`; re-grep before editing.

Status legend per commit: **[ready]** safe/independent · **[gated]** waits on a
decision · **[foundation]** high-ripple, build carefully.

---

## Resolved design decisions

**Owner calls (locked):**
1. **Retire the old cores.** `slugCore`/`powderCore` removed from the fire model; Slug + Powder become **ammo types**. Migration: keep the defKeys as load-time recognizers, but a slug/powder core sitting in a tool's core-slot on an old save is **dropped cleanly** (never inject an `Ammunition` where a proj-core is expected — see critic §breakage). Grant the equivalent loose ammo to the stash if trivial.
2. **Ballistic + energy coexist; the cost is rack space.** Ammo applies **only to the `bolt` shot kind**; energy cores (spark/boom/wave/warp/orb/saw/pick/decoy) keep their hardcoded emit-switch damage and ⚡-charge model, untouched, consuming no ammo. Magazines/ammo occupy rack cells competing with batteries + cross-class mods — that spatial economy is the balance lever.
3. **ADS = aim-down-sights *time*** (bare ~0.05s → loaded ~1s) **plus** a zoom+spread ladder. Combined per tier: ADS+ `{fovAim 0.60, aimSpreadMul 0.90, adsTime ~0.30s}`, ADS++ `{0.50, 0.75, ~0.60s}` (crosses the 0.55 scoped-reticle threshold), ADS+++ `{0.38, 0.55, ~0.90s}` + reduced sway.

**Damage-composition spec (resolves the critic's C12×C18 collision):**
Ballistic (`bolt`-kind) damage = **`ammoType.dmg + p.dmg` (folded mods) × chaos × small**, replacing `base.dmg` in the emit else-branch (4436). 9mm's 24 == today's `base.dmg` default, so it's continuous. Enemies carry an ammo type on their mag, so feature 16 ("enemy damage = base core/mod values") falls out naturally as `ammo + mods` for ballistic zealots — no conflict, mod bonuses preserved on both sides.

**Sensible defaults I'm taking (say the word to change):**
- Enemy-tool roll (feat 13): `runRng()<0.5` gates barrel/platform/combo, combos/platforms depth-weighted at `legDepth()>=2`.
- Believer body (feat 15): **reuse the zealot body verbatim** — zero new geometry, zero Chrome risk; differentiate by behavior, not silhouette.
- Variable armor (feat 17): **per-hit random roll** (no new save field).
- Beams per-tick (feat 12): **dt-scale** the per-pulse dose (avoids a ~7× framerate-dependent blowup).

**PENDING owner decision — enemy burn (feat 9 side effect):** removing burn DoT leaves enemies (which have no max-HP-scar system) taking *nothing* from fire cores + incendiary ammo + beam heat. Options: (a) fire is player-only, inert vs enemies; (b) **give enemies a max-HP-scar analog** — fire shrinks their effective HP, no tick (honors "only bleed does DoT" *and* keeps fire useful — **my recommendation**); (c) small enemy burn tick (bends the no-DoT rule). Gates the enemy side of C5, C7, C12.

---

## Ordered commits

**Ready-now wave (independent, being shipped first):**
- **C1 [ready]** Held-mod HUD clear (feat 2): `updateToolReadout()` empty-hands branch `7068` never clears `#trPips` (only the wielded path `7082-7089` writes it). Add `trPips.innerHTML=''`. DOM only.
- **C2 [ready]** Uncrouch cancels slide (feat 5): `7394` replace the `slideStand` toggle with a cancel (`slideT=0; isCrouching=false; slideStand=false; sprintLatch=false`), mirroring natural-end cleanup `7427`. `slideStand` goes vestigial.
- **C3 [ready]** Zealots 100 HP (feat 14): `health=34→100` at ctor `5630` + `reinforce()` `5896`. Transient, no save bump. *Balance-couples to C4/C9/C18.*
- **C4a [ready]** Whip 35 base dmg (feat 6, partial): `4597` `hurt(18)→hurt(35)`. *(anim/cadence/reach held — see C4b.)*
- **C5a [ready]** No burn pool cap (feat 10): remove `Math.min(30,…)` at `applyStatus` `5545`.
- **C5b [ready]** Player burn no-DoT, keep max-HP debuff (feat 9, player side): tick `7627-7631` keep `burnedMax` accretion `7630`, drop the `dot+=b`; bleed `dot` `7632` stays.

**Combat/status wave:**
- **C4b [gated: whip cadence]** Whip anim/cadence/reach (feat 6): the 0.3s window is encoded in **3 coupled spots** — strike gate `4575`, reset `4576`, shader phase divisor `meleeT/0.3` at `7976` — change together (strike at ~0.45·duration). Fluid = reshape ease/snap curves `1456-1465` only (NO new SDF prims — Chrome risk). Faster cadence needs loosening the `meleeT<=0` re-trigger guard `4489` while keeping one-pull-one-strike. Farther instakill = raise takedown gate `bd<1.6` `4593` (+scan cap `2.2` `4580` if reach should exceed 2.2m).
- **C5c [gated: enemy-burn]** Enemy-side burn per the decision.
- **C6 [ready]** Bleed red particles (feat 11): push short-ttl red `beamSegs` at enemy bleed tick `5757` (`col~[3.0,0.3,0.3]`, ttl~0.25), sub-cadence-gated. Reuses the capsule tracer draw `1346-1357` — GLSL-free. **Cap per frame** (shares the 12-slot `TRACERS` pool `660`).
- **C7 [gated: enemy-burn + dt-scale]** Beams per-tick (feat 12): convert accumulator-gated pulses (spark `4868`, boom `4869`, opt pick/orb/drum) to per-frame, dt-scaled. *Also throttle `explode()` side-effects — dynLight (4-slot pool) + audio — not just damage (critic).* 
- **C9 [ready, tune w/ C18]** Variable armor + damage-based durability (feat 17): shared `absorbHit(armor,dmg)→{mult,blocked}` collapsing the player/enemy plate twin. Bands: scrap let-through `0.60-0.90`, ceramic `0.30-0.60`. Apply at `damagePlayer 5580` + `Zealot.hurt 5718-5721`. Durability −1 per 10 blocked via a **non-saved float accumulator** (int `dur` truncates on load `6359`). Per-hit roll → no save change.

**UI/mechanics wave:**
- **C8 [ready]** Armor slots static (feat 1): root cause — `#backpackUI` center-transform `81-82` + `.zoneLeft` has no CSS rule `84`, so right-column resize (rucksack width, fold `3615`) reflows the equip column. Anchor the equip column independently (CSS `81-85` + layout split `6769/6855`). CSS/DOM only.
- **C10 [ready]** Barrel swap transfers mods+suppressor (feat 7,8): rack handler `3546-3609` replace the refusals (`REMOVE MODS` `3568`, `EMPTY THE BARREL FIRST` `3603`) with transferring old barrel `slots[]`+`muzzle` into the incoming barrel; spill overflow to hands/pockets (`_retuneOrphans` idiom `6362`). **Must expose lower-barrel muzzles** `walkColumn 2974` (not optional — that's the feature-7 lock). `muzzle` is a single dedicated slot `2820` — handle the suppressor-vs-suppressor conflict.
- **C16 [ready]** Ammo boxes merge to 12 (feat 4): `Ammunition`+`Ammunition` merge branch BEFORE the swap at `interactInventory 3591` (same `ammoType` → transfer up to **loose-box cap 12**, `consumeHeld()` if absorbed). *Keep this cap SEPARATE from `mag.maxCapacity` (up to 40) — critic §breakage.*

**Ballistics foundation (high-ripple — build carefully, in order):**
- **C11 [foundation]** Typed-magazine model + one ammo slot (feat 3): `Magazine 2799` scalar `loaded` → 1×1 `.inv` holding one `Ammunition(type,count)` (mirror `Armor.inv 3142`/`buildDeviceInv 3195`); `acceptedType`→`acceptedTypes` Set; add `adsTier`, `reloadMode`. Helpers `magCount/magType/magTake/magAdd` route **ALL** `.loaded` sites — *the plan's ~21-site list is INCOMPLETE (critic); do a fresh exhaustive grep of `.loaded` + `acceptedType` before editing* (known extras: `5650, 6838, 6913, 5613-5617, 5850-5851, 5597/5599, 4008-4010, 3993, and every `acceptedType===` gate 3538/3651/3733/3992/4001-4009/5599`). Also switch the Magazine **inspect render from the text panel to the grid branch** (`3675-3684`, `heldUse 4285`, `menuInspect setFocus(2)`) or the slot never appears. **Bump SAVE_VER 4→5** `6324`; serialize `j.mag={type,count}`; migrate legacy `j.loaded` int → seed inv. renderType=2 reused → no shader.
- **C12 [foundation]** Ammo-type registry + fire-time resolution (feat 18): `AMMO` catalog + `ITEM_DEFS` entries (9mm 24 · flechette 4×5 +20% bleed · 7.62/AP 35 ignore-50%-armor · slug 3× +delay · buckshot falloff up-to-5× · powder hitscan · incendiary variants = same +burn). Wire at **fire time only** (`emitProjectile` bolt branch `4436` + `tryFirePlayer 4536-4551`) — `_res` is cache-baked, so a mag swap won't reflect unless read at fire time. Per damage-composition spec above. **Critic seams to solve:** buckshot falloff needs *hit-time* distance → make buckshot **hitscan** (only the powder path knows range); flechette 4-count must be **injected at fire time** (the TWIN loop `4425` is cache-driven); slug `+0.45s` delay must move to `fireCooldown` at fire time `4567` (resolveTool never sees ammo); incendiary variants need every compatible mag's `acceptedTypes` to include the `-inc` keys. Armor-pen flag must reach the mitigation seam (`5718/5580`, coordinate C9). Map all to existing kinds → **no new SDF**.
- **C13 [ready, data]** New magazine defs (feat 19): data-only `ITEM_DEFS` (renderType=2, shader-free) — Barrel i/ii/iii (single-load {flechette,slug} 6/10/12 ADS++), Pistol (12×9mm ADS+), SMG i/ii/iii (20/30/40 9mm ADS+), Rifle i/ii (15/30 7.62 ADS++), Sniper i (single 8×7.62 −moderate spread, uncommon, ADS+++), Sniper ii (mag 15×7.62 −much spread, rare, ADS+++), Shotgun i/ii (mag 5/10 buckshot|slug|flechette ADS+++). **Reuse existing barrel bodies — do NOT add `BARREL_TIERS.len` values** (feeds the rack SDF `wingCellCount 2917` → Chrome). *Sniper "−spread" is a per-mag spread stat distinct from ADS zoom — add a `spreadMul` field (critic gap).*
- **C14 [foundation]** ADS tier application (feat 19): aggregate the feeding mag's `adsTier` into `res.fovAim` + new `res.aimSpreadMul` + `res.adsTime` + sway. *`resolveLane` SKIPS mags `4158` — needs a NEW mag-walk pass, and must reflect mag swaps (read at fire time or ensure rackSig perturbs `6920`).* Consume at `targetFov 7580`, aim spread `4427`, look-sens `7356`; `fovAim<0.55` flips the scoped shader bool `7976` (uniform only).
- **C15 [foundation]** Reload rules (feat 20): new **mag-swap verb** — use a whole magazine from inventory to eject the tool's empty mag and insert the loaded one (*specify: which slot — columnMag vs base reserve; where the ejected mag goes; coexistence with feat-3's 1×1 pour — critic gap*). Auto-reload `tryReload 5603` gated to `reloadMode==='single'` **and** pockets/harness zones only (new zone-id filter `storageZones 3274`/`takeBagAmmo 3348`). *Belt-feed `5591` currently auto-chambers for any mag — must also honor the single-load gate.*

**Enemies + loot wave:**
- **C17 [ready, tune w/ C18]** 50% enemy tools have barrels/platforms (feat 13): `buildTool 5670` raise barrel roll to ~0.5; `5660` stop hard-skipping platforms (*populate platform sub-slots or they're useless — critic*). Tool baked into arm-pose box `1325/1329` → no shader.
- **C18 [ready, tune w/ C9]** Zealot damage = tool values (feat 16): `tryShoot 5882` `dmg:P('enemyDamage')→dmg:this.res.dmg`; melee `5875` `damagePlayer(8)`→res-derived. Per damage-composition spec. `P('enemyDamage') 391` goes vestigial. **~2× swing — tune with C9/C3.**
- **C19 [gated: die-cause]** Believer (feat 15): reuse zealot body; 60 HP, no tool; chases + triggers a lone BOOM near player (`emitProjectile('enemy',…boom)`, detonates 1.2m off muzzle `4415`, radius 4.5m `5519`). Loot: thread kill-cause through `hurt 5725 → die(by)` — drop nothing if `by==='enemy'` (self-boom), else always `boomCore 3061`. *Critic: `die()` also fires from bleed/burn with `by=undefined` → mis-branch; and a dormant Believer (render cap 10, `tryShoot`/`explode` guards `5866/5521`) does nothing — handle both.*
- **C20 [ready]** Loot registration (feat 21): add every new ammo/mag defKey to `lootTiers 2232-2246` by rarity + `MOD_KEYS 3121`/`rollPack 5695`/`rollTool 5650` where apt. **Every new defKey MUST exist in `ITEM_DEFS` or it silently loads as null.** Audit `MOD_KEYS`/loot for any leftover `slugCore`/`powderCore` refs (they're being retired).

---

## Save format
Exactly **one** `SAVE_VER` bump (4→5, `6324`), in C11. The tolerant window `6410` auto-widens so v4 saves keep loading. Only structural migration: legacy `Magazine.loaded` int → typed 1×1 inv (`itemToJSON 6331` writes `j.mag={type,count}`; `itemFromJSON 6354` rebuilds + seeds from legacy `j.loaded`). New ammo/mags are just new defKeys (`count` already persists). Retired slug/powder cores: recognize + drop cleanly on load (no core injected). Per-hit armor roll + transient enemies/Believer → no save surface. Incendiary as distinct defKeys → no per-instance flag.

## Top risks (from the adversarial review)
- **C12×C18 damage-composition** — resolved above (ammo base + mods, both sides); implement to that spec or enemy mod bonuses drop / player double-counts.
- **C11 `.loaded` audit is the #1 ripple** — a single missed `.loaded`/`acceptedType` site silently kills a feed/reload path. Fresh exhaustive grep before editing.
- **Shader/Chrome** — safe ONLY if C4b adds no whip prims, C13 adds no `BARREL_TIERS.len`, C19 reuses the zealot body. Any of those re-breaks ANGLE.
- **Twin (armor)** — C9 must edit both player + enemy mitigation via the shared helper.
- **Balance set** — C3×C4×C9×C18 move combat math together (tune as a group); C5×C7×C12 share burn semantics (gated on the enemy-burn call).
- **Buckshot/flechette/slug/incendiary fire-time seams** + **mag-swap verb** + **stack-cap separation (12 loose vs 40 mag)** — critic gaps, addressed in C12/C15/C16 above.
