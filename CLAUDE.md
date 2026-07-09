# Shadow of the Valley — Project Brief for AI Agents

Read this before touching any code. It is the house rulebook for every agent
(Claude, Gemini, or a local model) working on this game.

## What this project is

A **browser stealth/action game** rendered in real time with **WebGL2 + GLSL
raymarching** over a **signed-distance-field (SDF) world**. The entire game is
one self-contained file: **`index.html`** (~5,600 lines of HTML + CSS + JS +
GLSL). **There is no build step and no dependencies.** You play it by opening
`index.html` in a modern browser.

## How to run and test it

- **Run:** open `index.html` in Chrome/Edge/Firefox (WebGL2 required).
- **Watch the console:** there is a visible error trap (`fatal()`, ~line 366)
  and an on-screen debug console. A black screen usually means a shader compile
  error or a thrown exception — check the browser devtools console first.
- **Controls / debug:** in-game "field manual" overlay shows key remaps; a debug
  console lets you live-edit tunable parameters (see the `PARAMS` block ~line 372).
- There are **no automated tests.** Verification is *manual playtesting in the
  browser.* Never claim a change works without loading it in a browser.

## Architecture map (section headers live in the file as comment banners)

| Area | Roughly where | Notes |
|------|---------------|-------|
| Tunable params / RNG / vector kit | ~360–430 | `PARAMS` are live-editable via debug console |
| GLSL shader source | ~620–1470 | `precision highp float`; the raymarcher |
| WebGL setup | ~1474–1545 | `webgl2`, `makeShader`, `resize` |
| **JS twin of the world SDF** | ~1550–2300 | `mapWorldJS`, `regionSdfJS`, `levelSdfJS`, `worldNormalJS`, collision, LOS |
| Authored levels | ~823–1120, ~1900–2300 | level primitives baked to a texture |
| Tool / inventory / crafting | ~2430–3200 | the "rack is a TREE"; sequencer, cores, mods, armor |
| Settings devices (goggles/headphones) | ~2646–2820 | open registry: "new setting = one entry" |
| Menu / inventory state machine | ~2820–3180 | single source of truth, everything derived |
| Enemy AI | ~3964–4190 | perception + state machine |
| Raid loop / terminal / **save system** | ~4277–4600 | versioned JSON in localStorage, 3 slots |
| Render + main loop | ~4600–5641 | `frame()` + `requestAnimationFrame` |

## Critical invariants — break these and the game breaks silently

1. **The GLSL shader and the JS world math are TWINS.** The same SDF geometry is
   computed twice: once in GLSL (what you *see*) and once in JS (`regionSdfJS`,
   `levelSdfJS`, `mapWorldJS`, and `cellHash`, which the comments say **MUST
   mirror the shader's hash**) for collision, AI line-of-sight, and physics.
   **If you change world geometry in one, you MUST make the identical change in
   the other**, or the player will collide with things that aren't drawn (or walk
   through walls that are). This is the #1 source of subtle bugs — call it out
   explicitly in any change that touches world shape.
2. **Single-file constraint.** Everything stays in `index.html` unless the human
   owner explicitly decides to split it. Do not add a bundler, npm packages, or
   external files without asking.
3. **Data-driven registries are the extension points.** Settings devices, terminal
   rows, and tool mods are designed so a new entry is *one line/object* and the
   nav/render/interact code is untouched. Add features that way; don't special-case.
4. **Save format is versioned.** If you change the shape of saved data, bump the
   version and handle old saves (corruption must read as "empty slot", never crash).
5. **Preserve the code's voice.** Terse, lower-level JS; ALL-CAPS comment banners
   as section dividers; monospace/military stencil aesthetic. Match it.

## House rules for every agent

- **Small, surgical diffs.** This is a large single file — change the minimum.
  Never reformat or "tidy" unrelated code. Never reorder sections.
- **Show your reasoning for risky changes**, especially anything touching the
  shader↔JS twin, the save format, or the main loop timing.
- **Verify in a browser before saying "done."** If you can't, say so plainly.
- **One logical change per commit.** Commit messages: imperative mood, e.g.
  `Add smoke-grenade tool core` / `Fix LOS desync on ramps`.
- **Never commit secrets** (API keys, `.env`). They belong in `.gitignore`.
- **When unsure about game design intent, ask the human owner** — they are the
  creative director. Agents implement; the owner decides.
- **Know the plan before you change the game.** [`docs/SCOPE.md`](docs/SCOPE.md)
  is what we're building, in what order, and what we've explicitly cut. Its
  "Open Decisions" are the owner's, not yours.

## The team (who does what)

This repo is built by a human owner directing a team of AI agents. See
[`docs/AGENTS.md`](docs/AGENTS.md) for the full playbook and how to route work
across models to control cost.

- **Claude** (lead engineer) — architecture, the tricky shader/twin work, review.
- **Gemini** (contractor) — whole-file reads/refactors, content generation.
- **Local models via LM Studio** (interns) — drafts, boilerplate, experiments.
