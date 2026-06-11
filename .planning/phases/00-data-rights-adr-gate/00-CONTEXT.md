# Phase 0: Data Rights + ADR Gate - Context

**Gathered:** 2026-06-11
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 0 documents every irreversible data-rights constraint as a committed ADR
before any operational code exists: Spotify scope strategy, the legal
local-audio acquisition pipeline, the lyrics derived-features-only contract,
and generation-model licenses with a hardware benchmark procedure. Per
operator decision, this phase is **documentation-only** — no project skeleton,
no runnable OAuth flow, no schema code. Runnable proof moves to Phase 1.

</domain>

<decisions>
## Implementation Decisions

### Spotify Account Reality (RIGHTS-01)
- **D-01:** Operator HAS Spotify Premium — dev-mode app is viable; PKCE OAuth
  API is the PRIMARY listening-history source. ADR documents the scope set
  (`user-top-read`, `user-read-recently-played`, `user-library-read`,
  `playlist-read-private`, `playlist-modify-private`) and notes the GDPR
  data-export fallback only as contingency against further API erosion.

### Local-Audio Sourcing (RIGHTS-02)
- **D-02:** BOTH sources: seed the analysis corpus from the operator's
  existing owned files (MP3/FLAC on disk) AND buy selected tracks on Bandcamp
  for new discoveries going forward. The ADR must cover provenance
  documentation for both paths and note a purchase-budget consideration.

### Hardware for Generation (RIGHTS-04)
- **D-03:** Hardware is UNKNOWN / may change. The Phase 0 ADR records the
  benchmark PROCEDURE (VRAM detection, model-tier table: musicgen-small needs
  ~4GB, medium 16GB, Stable Audio Open 16GB; MPS and CPU fallbacks) but the
  actual gate DECISION is deferred to Phase 4 start. Phase 4 planning must
  begin by running the benchmark.

### Phase 0 Scope & ADR Shape
- **D-04:** Phase 0 is DOCS-ONLY. No uv skeleton, no working OAuth flow, no
  Pydantic schema in this phase. Roadmap success criteria 1 and 3 must be
  amended: criterion 1 becomes "scope choice recorded in committed ADR"
  (OAuth implementation moves to Phase 1); criterion 3 becomes "lyrics
  contract documented in ADR with the enforcing-schema requirement explicitly
  assigned to Phase 1's store implementation".
- **D-05:** ONE COMBINED ADR file covering all four data-rights areas (not
  four separate files), living in `docs/adr/`. Keep each decision's section
  compact per PRINCIPLES.md's 5-line ADR spirit; one file, four decision
  sections with status/context/decision/consequences each.

### Claude's Discretion
- ADR numbering/filename convention (e.g., `docs/adr/0001-data-rights.md`)
- Exact wording of legal rationale for personal fair-dealing use
- Benchmark procedure details (commands, thresholds) within the tier table above

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project source of truth
- `SPEC.md` — Hard constraints §"Hard constraints (encode before any code)": Spotify API reality, lyrics & copyright, generation models. Phase 0 exists to encode these.
- `PRINCIPLES.md` §2 — 5-line ADR requirement in `docs/adr/`; security and release-hygiene fundamentals the ADR must reference.

### Research (informs ADR content)
- `.planning/research/PITFALLS.md` — Spotify API erosion timeline, PKCE token rotation, license traps (CC-BY-NC), habit-loop constraints.
- `.planning/research/STACK.md` — Library versions and the deprecated-endpoint NOT-to-use list the ADR should cite.
- `.planning/research/SUMMARY.md` — Phase 0 deliverable rationale and gaps to address.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — greenfield repo; no source code exists yet. Repo contains only
  SPEC.md, PRINCIPLES.md, README.md, and `.planning/`.

### Established Patterns
- None yet. Phase 0 sets the ADR pattern that all future phases follow.

### Integration Points
- `docs/adr/` directory will be created by this phase; Phase 1 cites these
  ADRs when implementing the OAuth flow and lyrics schema.

</code_context>

<specifics>
## Specific Ideas

- Operator confirmed Premium subscription — no Development-Mode blocker; the
  Feb 2026 Premium requirement is satisfied.
- GDPR export (`https://www.spotify.com/account/privacy/`) documented as
  fallback only, not primary.

</specifics>

<deferred>
## Deferred Ideas

- Project skeleton (uv, ruff, pytest, pre-commit), working PKCE OAuth flow,
  and the Pydantic lyrics-rejection schema — moved from Phase 0 to Phase 1
  per D-04 (docs-only decision).

</deferred>

---

*Phase: 0-Data Rights + ADR Gate*
*Context gathered: 2026-06-11*
