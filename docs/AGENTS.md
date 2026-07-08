# The Agent Team — Playbook

You are the **director**. You don't write code; you decide what gets built and
approve the work. Below is your team, how to spend each resource wisely, and the
pipeline a change flows through.

## Your resources (and what they cost)

| Resource | Cost to you | Best at | Spend it on |
|----------|-------------|---------|-------------|
| **Claude** (Max plan) | Your monthly Claude usage | Hardest reasoning, the shader↔JS twin, orchestration, final review | High-value, high-risk work. Your premium seat — don't burn it on boilerplate. |
| **Gemini** (Google Cloud $300 credits) | Real $ from credits (cheap on "Flash") | Reading/refactoring the whole 5,600-line file at once, generating lots of content | Big reads/refactors, dialogue/level content, second opinions |
| **Local models** (LM Studio) | Free (your PC's power) | Private, unlimited experiments | Drafts, boilerplate, brainstorming, commit messages, throwaway tries |

**Rule of thumb:** try it on **local (free)** → if not good enough, escalate to
**Gemini (cheap)** → reserve **Claude** for the parts that actually need the best
brain. This is how you avoid running out of Claude usage.

## Two kinds of "agents" — don't mix them up

1. **Claude subagents** (defined in `.claude/agents/`): specialists you summon
   *inside a Claude Code session* — `game-designer`, `gameplay-programmer`,
   `shader-twin-specialist`, `qa-playtester`, `code-reviewer`. These run on
   **Claude** (your Max plan). Use them for the skilled work.
2. **Multi-model workers via `aider`** (set up separately): the same git repo,
   driven by **Gemini or a local model** instead of Claude, to save your Claude
   usage. You pick the model with a flag.

## The pipeline (how a change flows)

```
  idea ──▶ design ──▶ code ──▶ playtest ──▶ review ──▶ commit ──▶ push
   you    designer   prog.      QA          reviewer    git       GitHub
```

You can run each stage with whichever resource fits:

- **Design** — cheap: local or Gemini can brainstorm; use the `game-designer`
  agent when you want it to fit the existing systems precisely.
- **Code** — `gameplay-programmer` (Claude) for anything touching the twin,
  saves, or the main loop; **aider + Gemini/local** for routine additions.
- **Playtest** — always **you, in a real browser**. The `qa-playtester` agent
  writes the checklist you follow. There are no automated tests.
- **Review** — `code-reviewer` (Claude) or `/code-review` before every commit,
  *especially* twin-touching changes.
- **Commit/push** — one logical change per commit; let the free/local model draft
  the commit message.

## Everyday recipes

- *"I have a rough idea"* → ask the `game-designer` agent for a spec, approve it,
  then hand the spec to the `gameplay-programmer`.
- *"Add something routine"* (a new tool mod, a HUD tweak) → try **aider with a
  local model** first; it's free.
- *"This is delicate"* (world geometry, collision, LOS) → **`shader-twin-
  specialist`** (Claude), no exceptions.
- *"Is my change safe to ship?"* → `qa-playtester` for a test checklist, then
  `code-reviewer`, then you playtest, then commit.

## Golden rules

- **You approve everything before it commits.** Agents propose; you dispose.
- **Playtest in a browser before every commit.** No exceptions.
- **Never let one twin change without the other** (see `CLAUDE.md`).
- **Never commit API keys.** They live in `.env` (git-ignored).
- **Small commits.** Easy to review, easy to undo.
