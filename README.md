# Shadow of the Valley

A browser-based stealth/action game — a single, self-contained `index.html` (HTML + CSS + JavaScript canvas, no build step). Open the file in any modern browser to play.

## Play it

- **Locally:** double-click `index.html`, or run a tiny web server from this folder.
- **Online:** once GitHub Pages is enabled, it will be playable at
  `https://<your-github-username>.github.io/shadow-of-the-valley/`

## Project status

Actively developed. Version history was imported from ~40 hand-saved versions
into Git, so `git log` shows the full evolution of the game.

## How this project is built

This repo is developed with a small **team of AI agents**, coordinated by the
human owner. See [`CLAUDE.md`](CLAUDE.md) for the project brief every agent reads,
and [`docs/AGENTS.md`](docs/AGENTS.md) for who does what and how to run the pipeline.

| Role | Model | Typical work |
|------|-------|--------------|
| Lead engineer / manager | Claude | Architecture, tricky bugs, review, orchestration |
| Senior contractor | Gemini | Whole-file refactors, content generation, second opinions |
| Interns | Local models (LM Studio) | Drafts, boilerplate, experiments |

## Repo layout

- `index.html` — the game (this is the whole thing today)
- `CLAUDE.md` — project brief / house rules for AI agents
- `docs/SCOPE.md` — the scope & design plan: what we're building and what we're not
- `docs/` — design notes and the agent playbook
