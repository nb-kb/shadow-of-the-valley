# Changelog

All notable changes to Shadow of the Valley. Plain-language summaries live in the
[GitHub Releases](https://github.com/nb-kb/shadow-of-the-valley/releases); this file
keeps the fuller, technical record.

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

[v1.1.0-beta]: https://github.com/nb-kb/shadow-of-the-valley/releases/tag/v1.1.0-beta
[v1.0.1-beta]: https://github.com/nb-kb/shadow-of-the-valley/releases/tag/v1.0.1-beta
