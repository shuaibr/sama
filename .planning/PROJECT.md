# Sama (سماع) — Music Studio: Analysis & Taste Development

> Formerly "Sonic A&R" — renamed in spec pack v2 (2026-06-11). Part of a
> six-repo portfolio governed by `OPERATIONS.md`; this repo's loop contract
> is `docs/framework-alignment.md`.

## What This Is

A personal music intelligence system that builds a deep taste profile from the
operator's real listening history, runs structured weekly discovery experiments
across nine genres, trains the operator's analytical ear (production layering,
rhythm, instrumentation, lyrics-as-derived-features, emotional architecture)
through guided comparative listening and blind tests, and prototypes sounds via
local generation models. Effectively an A&R apprenticeship with an agent as the
research department, for a single operator.

## Core Value

A measurably better analytical ear — tracked by weekly blind-test scores
trending up across 12 completed weekly cycles. Everything else (newsletter,
playlists, monetization) is a later byproduct.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Taste profile built from real Spotify listening history (top/recent
      tracks & artists, library) with an evidence trail for every profile update
- [ ] Weekly discovery digest (≤15 tracks), each track carrying
      `{hypothesis, expected_response}` plus 2 listening prompts and 1 anchor
      comparison to a mature genre (hip hop / R&B)
- [ ] 30-second structured response form per track (feel score, standout
      layer, would-replay) feeding profile updates
- [ ] Local acoustic analysis (librosa/Essentia behind one `AudioFeatures`
      interface) on legitimately owned audio — rhythm + layering reports
- [ ] Weekly blind tests (instrument layers / production era / scene) with
      scored results gating all "ear progress" claims
- [ ] Like-prediction scoring: system predicts response before listening,
      is scored on prediction accuracy
- [ ] Weekly training-focus generator driven by blind-test results
- [ ] Multilingual lyrics derived-features module (rhyme density, imagery
      patterns, flow-to-beat alignment) — never storing full lyric texts
- [ ] Local sound-sketch generation (MusicGen / Stable Audio Open) tied to
      weekly focus, sandboxed in `sketches/`
- [ ] Monday "ear report" + Wednesday discovery digest delivery loop
- [ ] Cross-genre bridge logging (Celtic ↔ blues, Bollywood ↔ R&B,
      tarab ↔ soul) instrumented from day one

### Out of Scope

- Streaming-service scraping — legal risk; Spotify used only for history,
  library, and playlist write-back
- Spotify audio-features / audio-analysis endpoints — deprecated for new
  third-party apps (Nov 2024); all acoustic analysis is local
- Full-lyrics storage or redistribution — copyright; derived features and
  short fragments only
- Releasing generated music — sketches are study artifacts; respect model
  licenses
- Recommender for other users — not before the operator's own loop has run
  12 weeks
- Community/monetization features — Phase 5+ optional byproduct, not v1

## Context

- Operator taste seed: mature genres hip hop & R&B (anchor reference points);
  active genres rock, blues, soul; expansion genres Celtic, French
  (chanson → modern), Bollywood, Arabic (tarab → modern pop).
- Multilingual listener (English/Arabic/Urdu/Hindi) — lyrics analysis must be
  multilingual-aware.
- Causality hypothesis: the ear improves through COMPARATIVE listening
  anchored in mature genres, not volume. Every new sound is paired with an
  anchor reference.
- Bottleneck #1: acoustic data access post-Spotify-deprecation — Phase 0
  resolves the legal local-audio pipeline (Bandcamp/purchases/own stems)
  before anything else.
- The binding capacity constraint is operator listening attention
  (~5 hrs/week). The system curates down, never up.
- Weekly cadence: Monday profile update + ear report, Wednesday discovery
  batch, weekend deep-listen + blind test. Monthly "sound thesis" essay.
- Kill criterion: 4 consecutive skipped weekly cycles means the system is
  too heavy — simplify or stop.
- Engineering principles in PRINCIPLES.md are non-negotiable: four-layer
  model (Abstraction/Isolation/Validation/Capacity), ADRs for significant
  decisions, one-command setup/run, no secrets in code, Python + minimal JS,
  trunk-based dev, heartbeat alerts on scheduled jobs, ecosystem map
  refreshed at every phase boundary.

## Constraints

- **Legal/API**: Spotify audio-features & audio-analysis endpoints are off
  the table (deprecated Nov 2024) — Spotify scopes limited to listening
  history, library, playlist write-back
- **Legal/Copyright**: lyrics handled as derived features only (themes,
  rhyme-scheme maps, syllable/flow density, structure); reports quote ≤ short
  fragments under personal fair-dealing use
- **Licensing**: generation via local/open models only (MusicGen, Stable
  Audio Open); outputs are study artifacts, never releases
- **Capacity**: local-first analysis; LLM budget capped per weekly cycle;
  one discovery batch ≤15 tracks/week; budgets in config, enforced in code,
  alert at 80%
- **Tech stack**: small core language set — Python + minimal JS; boring tech
  preferred; new dependencies need an ADR
- **Architecture**: every component assigned to exactly one of the four
  layers; durable state in versioned storage (SQLite + git exports); agent
  runs stateless; generated audio never mixed into the reference corpus
- **Model tiering**: orchestration/synthesis on frontier model, bulk
  extraction on cheapest adequate model; tier assignments in config.yaml
- **Operations/Output (OPERATIONS.md Rule 1)**: action outputs are
  forced-choice only — per-track reply is `REPLAY / FINE / SKIP` +
  standout-layer tag; vetoes append one line to `docs/codex.md`; no
  open-ended reports as action output
- **Operations/Measure (OPERATIONS.md Rule 4)**: AOR (% of recommendations
  acted on or vetoed within 48h) logged to `metrics/loop_closure.csv`;
  AOR < 50% over 2 weeks demotes the pipeline from push to weekly digest
- **Operations/Coordination**: single agent + output schema + operator veto
  — no multi-agent orchestration (sequential task); no heavy orchestrators
  (plain Python + SQLite + cron)
- **Budget (framework-alignment §3)**: ≤ $1.00 CAD/week LLM spend, warn at
  80%; LLM used only for digest narrative
- **Research handoff (framework-alignment §2)**: discovery candidates come
  ONLY from Gemini scene memos in `research/inbox/scenes/` + history-derived
  seeds; memos land weekly before the Wednesday digest build

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Skill-first, money later (Phase 5+) | Spec is honest that the outcome is the operator's ear; monetization pressure would distort the loop | — Pending |
| Local acoustic analysis (librosa/Essentia), no Spotify features API | Endpoints deprecated Nov 2024 for new apps; legal local pipeline is Bottleneck #1 | — Pending |
| Lyrics as derived features only | Copyright; analysis value is in structure/flow/themes, not text | — Pending |
| Comparative listening anchored in hip hop/R&B | Causality hypothesis: anchored comparison beats volume | — Pending |
| Vertical MVP structure | Phase 1 must ship the smallest closed loop (sense → decide → act → measure → improve) per PRINCIPLES.md | — Pending |
| Prediction-before-listen scoring | Forces falsifiable taste model; prediction accuracy is a named feedback loop | — Pending |
| Forced-choice response format (REPLAY/FINE/SKIP + standout layer) supersedes the original feel-score form | OPERATIONS.md Rule 1 (veto interface); spec pack v2 framework-alignment Act row | — Pending |
| AOR is the loop-closure metric, logged to metrics/loop_closure.csv | OPERATIONS.md Rule 4 — Measure row with teeth; 50%/48h thresholds revisable after 60 days of data | — Pending |
| Single-agent architecture, no swarms, no heavy orchestrators | OPERATIONS.md Rule 1 + Do-Not-Do list; sequential pipeline (arXiv 2512.08296) | — Pending |
| Portfolio status: PARKED per OPERATIONS.md Rule 3 (WIP cap = 1; flow is Active) | Planning artifacts prepared now; build work resumes when Active slot opens | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-10 after initialization*
