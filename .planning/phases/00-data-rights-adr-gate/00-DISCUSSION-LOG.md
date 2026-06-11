# Phase 0: Data Rights + ADR Gate - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-06-11
**Phase:** 0-Data Rights + ADR Gate
**Areas discussed:** Spotify account reality, Local-audio sourcing, Hardware for generation, Phase 0 scope & ADR style

---

## Spotify Account Reality

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, Premium | Dev-mode app viable — PKCE OAuth becomes primary history source | ✓ |
| Free tier | GDPR data export becomes primary, API deferred; ADR documents blocker | |
| Not sure / will check | OAuth primary with GDPR fallback documented; verify at execution | |

**User's choice:** Yes, Premium
**Notes:** Feb 2026 Premium requirement for dev-mode apps is satisfied; GDPR export retained as contingency only.

---

## Local-Audio Sourcing

| Option | Description | Selected |
|--------|-------------|----------|
| Existing local collection | Point pipeline at existing owned files; Bandcamp later | |
| Bandcamp purchases as I go | Buy selected tracks for deep analysis; purchase workflow + budget note | |
| Both | Seed corpus from existing files now, Bandcamp for new discoveries | ✓ |

**User's choice:** Both
**Notes:** ADR must cover provenance for both paths.

---

## Hardware for Generation

| Option | Description | Selected |
|--------|-------------|----------|
| NVIDIA GPU ≥12GB VRAM | musicgen-small comfortable; medium/Stable Audio marginal | |
| Apple Silicon (M-series) | Both models via MPS at 2-3x slower | |
| CPU-only / weak GPU | musicgen-small only; Phase 4 deferred until GPU access | |
| Unknown / changes later | ADR records benchmark procedure; gate decision deferred to Phase 4 start | ✓ |

**User's choice:** Unknown / changes later
**Notes:** Phase 4 planning must begin by running the documented benchmark.

---

## Phase 0 Scope & ADR Style

| Option | Description | Selected |
|--------|-------------|----------|
| Skeleton + working code | uv project, OAuth flow, Pydantic lyrics schema (was recommended) | |
| Docs-only | ADRs as markdown only; OAuth + schema slip to Phase 1; criteria amended | ✓ |

**User's choice:** Docs-only
**Notes:** ROADMAP.md Phase 0 success criteria 1/3/4 amended accordingly; runnable proof reassigned to Phase 1.

| Option | Description | Selected |
|--------|-------------|----------|
| One ADR per decision (4) | ADR-001…004, each ≤5 lines per PRINCIPLES.md (was recommended) | |
| One combined ADR | Single data-rights ADR covering all four areas | ✓ |

**User's choice:** One combined ADR
**Notes:** One file in docs/adr/ with four compact decision sections.

## Claude's Discretion

- ADR filename/numbering convention
- Exact legal-rationale wording for personal fair-dealing use
- Benchmark procedure details within the documented VRAM tier table

## Deferred Ideas

- Project skeleton (uv, ruff, pytest, pre-commit), working PKCE OAuth flow, and Pydantic lyrics-rejection schema — moved to Phase 1 per docs-only decision.
