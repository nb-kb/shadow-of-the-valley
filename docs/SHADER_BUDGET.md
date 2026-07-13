# Shader Compile Budget — what can we add without breaking the engine?

The Chrome/Edge black-screen was never a *speed* problem — it was the fragment
shader growing **too large for Chromium's ANGLE→D3D11 compiler to link**. So
"how much content can we add?" is really two separate budgets:

- **Compile budget** — spent only by **geometry that lives in the shader**
  (scene SDFs, enemy bodies). Overspend it → the program fails to link → black
  screen. This is the one that broke.
- **Runtime budget** — FPS. `renderScale` and `rayQuality` live here. They are
  runtime uniforms; lowering them makes it run faster but **does not shrink the
  compiled shader**, so they buy you *nothing* against the compile limit.
  (Confirmed: `rayQuality` → `uMaxSteps` uniform on the `for(i<uMaxSteps)` loop,
  line ~1484, deliberately kept a uniform so ANGLE won't unroll it; `renderScale`
  only sizes the framebuffer in `resize()`.)

## Bottom line

Three of the four things you were afraid to add barely touch the compile budget:

| Content | Grows the shader? | Risk |
|---|---|---|
| New tool mod / core / projectile **behavior** | No — JS + a shared generic tracer/decal/light draw | **Free** |
| More enemies (count), smarter AI, perception | No — JS uniforms, render cap 10 | **Free** |
| Terminals / inventory / pickups / menus | No — DOM/JS | **Free** |
| **More** authored props (buildings, crates, tents, trees, wrecks…) | No — baked into the level **texture**, sampled by a fixed loop | **Free — preferred path** |
| New reticle / new held-item shape | A little GLSL | Low |
| **New enemy *body* silhouette** (a new type, not a new count) | Yes (~1–2 KB, hits **both** full variants) | Medium — test both |
| **New procedural *valley* structure** (`cellSDF`) | Yes, inline GLSL | Medium-High |
| **New authored region *kind*** (a novel prop archetype, not an instance) | Yes — extends the 11.4 KB `lvRegion` | **High — the tightest spot** |

So: keep adding mods, enemies, interactions, and *more of the existing* level
props freely. The only expensive moves are **new shader shapes** — a new enemy
body, a new procedural valley structure, or a brand-new authored prop *kind*.

## Why AUTHORED is the variant to watch

One GLSL source (`fragmentShaderSource`, ~lines 667–1764) is forked into three
programs by injecting one `#define` (lines ~1806–1808): **PREVIEW** (ground+sky
only, compiles instantly), **VALLEY** (procedural `mapProc`/`cellSDF`), and
**AUTHORED** (`levelSDF` + the `lvRegion` 10-kind switch). The unused scene field
is dead-stripped per variant — that split is what made it link at all
(pre-split, VALLEY+AUTHORED compiled together and *never* linked on Chrome).

- **AUTHORED carries the most unique scene source** (~14.9 KB, dominated by the
  11.4 KB `lvRegion`) and already had to drop the smooth 4×-`map()` normal for a
  cheap derivative normal "to keep it linking" (line ~1553). It rides closest to
  the ceiling.
- **VALLEY** has more slack (still affords the 4× normal).
- Authored level props are **data in `uLevelTex`**, stamped by `bakeLevel()` —
  adding props is more texture entries, **not** more GLSL. The exception is a new
  `kind==11` archetype, which *is* new GLSL in `lvRegion`.

## The one guardrail

After any change that touches a scene-SDF function (`lvRegion`, `cellSDF`,
`levelSDF`, `mapProc`, the enemy body in `map()`), **load-test on Chrome/Edge and
confirm the AUTHORED variant links** — watch for the `fatal('authored program
link…')` trap (~line 1865) and a black screen. Firefox passing means nothing;
the whole bug class is ANGLE/D3D-only. And never turn a raymarch loop bound into
a literal (keep it `uMaxSteps`-derived) — a constant bound re-inflates the shader
~150× via unrolling and silently miscompiles.

## Honest caveat

This ranks risk from shader *source* size and the code's own structure — it
can't measure exact D3D instruction/register margins (that needs Chrome's
compiler, which we can't run here). The ordering (AUTHORED tightest, texture-data
free, perf-knobs irrelevant to compile) is solid; the exact number of new
`kind`s or enemy bodies AUTHORED can absorb before re-breaking isn't knowable
without an on-Chrome link test — which is exactly why that test is the guardrail.
