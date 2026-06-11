# Framework Alignment: sama

**Status:** PARKED (PARKING_LOT.md occupant — revisit after flow V1) | **Archetype:** Multi-Stream Feedback
**Target Time-to-Validation:** 7d (first Wednesday digest → ≥10 structured track responses)
**Coordination archetype:** single-agent — profile update, curation, and digest are sequential; local audio analysis is library code (librosa/Essentia), not agents.

-----

## 1. The Loop

|Stage               |What it is for this project                                            |
|--------------------|-----------------------------------------------------------------------|
|**Sense** (input)   |Spotify listening history (top/recent via API) + 30-sec structured responses per track + blind-test results|
|**Decide** (process)|Taste-profile update with evidence trail; discovery batch ≤15 tracks each with like-prediction + anchor comparison|
|**Act** (output)    |Wednesday digest; per-track forced choice in reply form: `REPLAY / FINE / SKIP` + standout-layer tag|
|**Measure** (impact)|Prediction accuracy vs actual response; weekly blind-test score; logged to metrics/loop_closure.csv|
|**Improve** (update)|Profile weights + next week's training focus from misses; cross-genre bridges logged|

-----

## 2. Platform Split + Handoff Contract

**Google AI Plus — research & knowledge**
- Inputs it owns: scene/artist research (Celtic, French, Bollywood, Arabic, blues), production-history deep dives.
- Tasks it owns: weekly scene memo for the expansion genre in focus.

**Claude Code (Max 5x) — execution & production**
- Owns: GSD orchestration, Spotify history adapter, profile model, librosa/Essentia feature extraction, digest + forms.
- Compute boundary: local-first analysis on owned audio; 1 discovery batch/week; LLM only for digest narrative.

**Handoff contract:**
- Gemini output lands as: markdown scene memos.
- Delivered to: `/research/inbox/scenes/`.
- Claude Code consumes via: curation step may pull candidate artists ONLY from memos + history-derived seeds.
- Cadence: weekly, before Wednesday digest build.

-----

## 3. Architecture Gates

- **Abstraction:** `src/adapters/spotify.py`, `src/adapters/musicbrainz.py`; analyzers behind one `AudioFeatures` interface; generators behind `SoundSketch` (Phase 4).
- **Isolation:** listening data + profile in local SQLite; sketches sandboxed in `sketches/`; analysis runs stateless.
- **Validation:** every profile update shows evidence; like-predictions pre-registered before listening; no full-lyrics storage (derived features only).
- **Capacity:** ≤15 tracks/week (attention budget) · ≤$1.00 CAD/week LLM · warn at 80%.

-----

## 4. Phase 1 Checklist

- [ ] UNPARK only when Active slot opens (OPERATIONS.md Rule 3)
- [ ] Phase 0 ADRs: Spotify scopes (history only — audio-features API deprecated), local-audio plan, lyrics policy, model licenses
- [ ] Repo initialized with this doc at `docs/framework-alignment.md`; `/research/inbox/` created
- [ ] `npx get-shit-done-cc@latest`; feed this doc + SPEC.md + PRINCIPLES.md + OPERATIONS.md into `/gsd-plan-phase`
- [ ] Planner feedback noted → prune template

**Planner feedback:**
- Sections used:
- Sections ignored:
