# Sonic A&R — Music Analysis & Taste Development System

A personal music intelligence system that builds a taste profile from real
listening history, runs structured discovery experiments across genres, and
trains the operator's analytical ear through guided listening, blind tests,
and local sound-sketch generation.

## Source of truth

- [`SPEC.md`](SPEC.md) — vision, hard constraints, architecture, ecosystem
  map, and phases for this project.
- [`PRINCIPLES.md`](PRINCIPLES.md) — non-negotiable engineering principles
  that apply to every phase (four-layer model, fundamentals, ecosystem
  mapping, closed-loop requirement, model tiering).

## Working on this project with GSD

GSD Core (the [open-gsd](https://github.com/open-gsd/gsd-core) distribution —
governance moved from the former upstream to open-gsd in 2026; see
https://opengsd.net) is installed project-locally in `.claude/` and committed,
so the `/gsd-*` commands are available in any Claude Code session on this
repo. To update it later:

```bash
npx @opengsd/gsd-core@latest --claude --local
```

1. In Claude Code, run `/gsd-new-project` and point it at the specs:
   > "Read SPEC.md and PRINCIPLES.md in the repo root. They are the source
   > of truth for vision, requirements, phases, and engineering principles."
2. GSD generates `.planning/` (PROJECT.md, REQUIREMENTS.md, ROADMAP.md),
   then drives phases with `/gsd-plan-phase`, `/gsd-execute-phase`, and
   `/gsd-verify-work`.

## Phase order (from SPEC.md)

0. Data rights + pipeline ADRs (Spotify scopes, local-audio plan, lyrics
   derived-features policy, model licenses)
1. Smallest closed loop: history → taste profile v1 → weekly digest →
   response form → profile update
2. Local analysis engine (librosa/Essentia) + first blind tests
3. Like-prediction scoring + training-focus curriculum
4. Generation studio (local sound sketches)
5. Optional community/monetization
