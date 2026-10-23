# Changelog

All notable changes to Shadow of the Valley. Plain-language summaries live in the
[GitHub Releases](https://github.com/nb-kb/shadow-of-the-valley/releases); this file
keeps the fuller, technical record. Open issues live in `KNOWN_BUGS.md`.

## [Unreleased] — the creature wave (`beta 1.3.0-wip1`)

### Added
- **New enemy bodies on every faction:** a joint-table pose rig drives gaits, robe sway, the zombie slump and dragging shamble, the brute's knuckle-drag bulk, flinches, and head-look. The head hit sphere moves with the drawn head.
- **Neutral critters:** small grazing herds seeded near the busy camps that bolt from threats and drift between graze spots. Hungry undead hunt them, feed, and rise again *radioactive* (glow + dose aura).
- **Brute charge:** growl wind-up, a fast sprint at the player only, and a stagger if it hits a wall.
- **Zealot combat AI:** duck-and-peek cover cycle, a 2-second interruptible field-dressing heal, call-for-backup shouts with a rally retreat (the last soldier always runs), and blind covering fire while a squadmate moves or heals.
- **Horde choreography:** the undead escort orbits the brute in golden-angle slots with phased shambles and moan ripples.
- **Radiation rework:** rolling ground-hugging gas billows, a dose ramp, a DOSED screen post, a geiger tick, glinting rad motes, and trails behind radioactive undead.
- **Night fireflies** that scatter from running bodies (culled inside gas).
- **Gibs:** bodies burst into faction-tinted chunks on the world-item pipeline.
- **Idle life:** posted zealot pairs chat, zombies feed and twitch, gib piles become meals, herds wander.

### Changed
- Soldier respawns share the undead 5–10 minute clock.
- Loot always outranks gibs: gib bursts can't evict loot at the 64 world-item cap, and loot renders before meat in the 16 item slots.
- Critters give up their render slots to every combatant.

### Fixed
- Helm crown pulled down so it never pokes past the head hit sphere.
- Critter hazard probes fenced on the level bounds, not the valley rim.

### Known
- The new rig roughly doubles the full shader's compile time on Chrome/D3D11 (33 s → 65 s, measured on an Arc 140T). It boots cleanly on the preview program. Fix in progress: move the pose math from the shader into JS (KNOWN_BUGS #2).

## [v1.2.0-beta] — 2026-07-16 — Containment

### Added
- **Walled Area 1 with a horde gate:** a keycard-locked gate on the north wall opens onto an escalating outskirts zombie gauntlet; a PR stats page on the hideout terminal tracks runs.
- **Radiation and gas:** hazed POIs, outskirts gas and the brute's aura burn you; two gas masks, filters that wear down, and a face slot. Soldiers fear the haze and the undead crowd it.
- **Locked labs:** lab and testing-range gates take unique keycards and hold loot caches.
- **Lockboxes:** portable epic (2×2) and legendary (3×3) mini-inventories that survive death; brutes drop them.
- **Loot crates at every POI**, rotating from raid to raid.
- **ADS free-aim** on every sight: the aim leads, the camera trails, and the gun leans.
- **A diegetic inventory:** the bag UI rides an opened rucksack prop, menus duck when you move or look, sub-inventories dock beside their parent, the rig is five seats (helmet, face, harness, holster, rucksack), and magazines are real sub-inventories.
- **Inventory ergonomics:** drag and drop, quick-move, stack split, hover tooltips with input-aware verb hints.
- **Real scopes:** a render window inside a hollow scope tube; red-dot and holo sights zoom.
- **Flash-era thumbnails** for every item and per-kind world models for drops.
- **The whip:** an overhead lash with a jointed arm, a tapered cord that clips on walls and drapes on bodies, and an honest 3.4 m reach.
- **Starter kit that fights:** the seeded stash now holds energy cores, ammo, healing and basic armor.

### Changed
- Firing model matches `docs/MODS.md`: boltless ammo and powered cores fire in parallel, platform columns fire their own mags, powder is a magazine trait, and refund waives base energy only.
- Headshots are symmetric: a bare head dies to one hit, light helmets take two, heavy take three. Hits test the drawn body capsule.
- Zealots flee or stand under pressure; factions seed far apart; spawns refuse raised ground.
- Reload pulls the fullest spare into the driest mag.
- Gun collides with walls (muzzle tips up, bullets follow it) and auto-peeks around cover.

### Fixed
- Outskirts spawner probed absolute height inside the rim hills, so the gauntlet never spawned.
- Step-over climbed steep valley walls; it now refuses terrain at every angle.
- The desk terminal could deploy from inside other menus; it now sleeps under any other UI.
- Phantom rack gun on menu close (the "gun-raise replay", #7).
- Lamp strobe from light-slot eviction (#6a) and 8 Hz exposure steps (#6b).
- Enemy and world sounds are spatialized (#8).
- Brute head hitbox matched to its inflated body (#4).
- FENCE/FACADE carve heights aligned between the shader and collision (#0).
- Save hardening: rotted values clamp on load, overflow drops at your feet with a toast, and a corrupt load keeps the running session.
- Hideout crates no longer refill with free loot; five unobtainable items now drop; settings gadgets are guaranteed at home.
- Dropped items restart their despawn clock on release; the hideout floor never despawns.
- Invisible zombie bites from dormant bodies; LURE now works on the undead.
- Plain falls are swept (no tunnelling through thin decks at 30 fps).

### Performance
- Dormant enemies tick 4:1; soldier sightline marches are range-gated; hot-path allocations moved to scratch buffers; per-frame reflows at the terminal and HUD removed; auto-resolution recovers on 60 Hz panels; both full shader programs prewarm on link.

## [v1.1.1-beta] — 2026-07-14 — The Horde

### Added
- **Zombie faction** with three-way targeting (soldiers, undead, player) and a brute mini-boss; soldiers cluster at rotating POIs, the undead roam the rest.
- **Ballistics:** magazines and calibers (9 mm, 7.62 AP, flechette, slug, buckshot), the powder magazine, and the rare *hope* core (heals allies, burns the dead).
- **Optics as geometry:** red-dot, holo and iron sights; a two-slot scope adapter; the computer terminal sits in the world.
- **Nine graphics dials in the goggles** (resolution, detail, draw distance, crowd, shadows, colour and more), all uniform-driven, no recompile.

### Changed
- Enemy AI: line patrols, cover-seeking, covering fire, squad cohesion.
- Enemy render cap 10 → 20, with the material-ID band derived from `RLIM.ENEMIES`.
- Tougher HP roster (zealot 200, believer 100, zombie 100, brute 800).

### Fixed
- Dropped-loot memory leak (long-run lag), auto-resolution flicker, enemies floating off the ground, compass going dark, enemy optics drawing the player's reticle, invisible forest bushes that still collided.

## [v1.1.0-beta] — 2026-07-12

### Added
- Rename any tool or backpack from its inspect panel (per-instance `customName`, persists across save/load).
- Iron sights as a lootable tool mod (default: none), so looted optics are worth grabbing.
- Pocket watch reads the time whenever it's carried — in hand or any pouch, not just the rucksack.

### Changed
- Full mouse control in menus: left-click grabs/places and uses the terminal (the E verb, `interactInventory`); right-click inspects/activates (the X verb, `activateAtCursor`).
- Sights overhaul: removed the always-on crosshair (carry-pin kept); the aim reticle only shows with an optic or iron sights; ADS viewmodel angle realigned.
- Restored cast shadows + ambient occlusion in the authored levels (hidey hole, Area 1) — ~3× `levelSDF`, verified to still link on Chrome/ANGLE.

### Fixed
- **Cold-boot black screen:** `#introScreen` was translucent over an uncleared (transparent-black) canvas. Made it opaque and clear the canvas to a dark tone at context creation.
- **Menu clicks dropped on Chrome/Edge:** the inventory and terminal rebuilt their DOM (`innerHTML=''`) on every `mouseenter`; Chromium re-fired `mouseenter` on the recreated cell under a still pointer, looping the rebuild and destroying the tap's `mousedown` before it landed (Firefox didn't loop, so it worked there). Added same-cell dedup guards to the `renderZones` and terminal-row mouseenter handlers.
- **Special-tool names lost on save/load:** Foundling/Zealot tools set their name via `.name` (not persisted); now stored as `customName`. Save format bumped `v3 → v4` with the version gate widened so older saves aren't orphaned.

## [v1.0.1-beta] — 2026-07-11

### Fixed
- **Black screen on Chrome and Edge** (Firefox was unaffected). The single fragment program was too large for Chromium's D3D compiler and lost the GPU context. Split it into `PREVIEW` / `VALLEY` / `AUTHORED` variants via `#define`, dead-stripping the unused level SDF per variant so each links under the compiler limit; `KHR_parallel_shader_compile` for non-blocking readiness; a terminal boot-gate masks the residual first-load compile.

[Unreleased]: https://github.com/nb-kb/shadow-of-the-valley/compare/v1.2.0-beta...feat/containment
[v1.2.0-beta]: https://github.com/nb-kb/shadow-of-the-valley/releases/tag/v1.2.0-beta
[v1.1.1-beta]: https://github.com/nb-kb/shadow-of-the-valley/releases/tag/v1.1.1-beta
[v1.1.0-beta]: https://github.com/nb-kb/shadow-of-the-valley/releases/tag/v1.1.0-beta
[v1.0.1-beta]: https://github.com/nb-kb/shadow-of-the-valley/releases/tag/v1.0.1-beta
