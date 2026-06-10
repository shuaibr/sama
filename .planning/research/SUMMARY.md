# Project Research Summary

**Project:** Sonic A&R — Music Analysis & Taste Development System
**Domain:** Local-first personal music intelligence, ear training, and taste profiling
**Researched:** 2026-06-10
**Confidence:** HIGH

## Executive Summary

Sonic A&R is a single-operator, local-first system for building a cross-cultural A&R ear through a disciplined weekly loop: ingest Spotify listening history, generate a hypothesis-driven discovery digest (≤15 tracks), collect structured operator responses, update an evidence-trailed taste profile, and close the cycle with blind-test ear training. The recommended approach is a Python monolith organized around four strict layers — Abstraction (adapters/ABCs), Isolation (stateless batch jobs), Validation (evidence-trail enforcement), and Capacity (config-driven budget caps). All acoustic analysis happens locally via librosa (and optionally Essentia behind a feature flag), because Spotify deprecated its audio-features and audio-analysis endpoints for new apps in November 2024. LLM orchestration uses litellm wrapping Claude Sonnet with per-run token budgets enforced before dispatch.

The key differentiation over existing tools (Last.fm, ToneGym, Spotify Discover Weekly) is the combination of falsifiable prediction-before-listen scoring, hypothesis-driven track packaging with cross-genre anchor comparisons, and a structured curriculum driven by blind-test scores rather than passive listen counts. These features require a stable taste profile before they can function, which creates a clear phase dependency: the minimal closed feedback loop must ship and validate before any analytical enhancements are added.

The primary risks are: (1) Spotify API erosion — the platform has made four policy changes since Nov 2024 and the trajectory is toward further restriction, requiring an adapter-isolated architecture with a ListenBrainz mirror from day one; (2) habit-loop collapse — over-instrumentation kills operator engagement, and every feature addition must be tested against a 20-minute weekly overhead budget; (3) librosa accuracy overreach on non-Western audio — key detection algorithms trained on Western tonal music produce meaningless results for maqam and raga traditions, requiring genre-gated confidence wrapping on every acoustic estimate.

## Key Findings

### Recommended Stack

The core stack is Python 3.11 managed with uv, spotipy 2.26.0 for Spotify OAuth (PKCE), SQLAlchemy 2.0.x with Alembic 1.13.x for persistence, Pydantic 2.x for schema validation at every trust boundary, librosa 0.11.x as the primary acoustic analyzer, and litellm wrapping the Anthropic SDK for LLM orchestration with cost tracking. APScheduler (no Redis broker) runs the weekly cron jobs in-process. Generation (Phase 4 only) uses MusicGen via audiocraft or Stable Audio Open via diffusers, both requiring ≥12 GB VRAM.

**Core technologies:**
- Python 3.11 + uv: primary language and package manager — best ecosystem support; one-command setup
- spotipy 2.26.0: Spotify API client with PKCE OAuth — only version with CVE-2025-27154 fix
- SQLAlchemy 2.0.x + Alembic 1.13.x: ORM and migrations — swap SQLite to Postgres without touching app code
- Pydantic 2.x: validation at every layer boundary — Rust-backed V2; JSON schema export for ADR artifacts
- librosa 0.11.x: primary acoustic feature extraction — installs on any platform; Essentia is optional secondary behind feature flag
- litellm 1.40.x wrapping anthropic 0.26.x: LLM client with cost tracking — completion_cost() per call; budget enforced before dispatch
- APScheduler 3.x: in-process weekly scheduler — no Redis/broker needed for single-operator use

### Expected Features

**Must have (table stakes — Phase 1):**
- Listening history ingestion (Spotify top tracks, recent plays, library) — raw material for all profile updates
- Evidence-trailed taste profile with per-dimension weights and source provenance — operator must see why the profile changed
- Weekly discovery digest (≤15 tracks, Wednesday) with hypothesis, listening prompts, anchor comparison per track
- 30-second structured response form (feel score 1–5, standout layer, would-replay boolean) — must be literally 30 seconds
- Profile update from responses (Monday) and Monday ear report — closes the weekly cycle
- Heartbeat alerting on scheduled jobs — detects 4-consecutive-skip kill criterion before it compounds

**Should have (differentiators — Phase 2–3):**
- Local audio analysis pipeline (librosa + Essentia behind AudioFeatures ABC) — unlocks rhythm/layering reports and real blind tests
- Weekly blind tests (instrument layers, production era, scene) with scored results and trend tracking
- Training-focus generator driven by weakest blind-test dimension — self-correcting curriculum
- Prediction-before-listen scoring — system predicts operator response before digest delivery; scored after
- Weekly prediction accuracy metric as a first-class named metric
- Cross-genre bridge logging (Celtic ↔ blues, Bollywood ↔ R&B, tarab ↔ soul)
- Multilingual lyrics derived-features (English, Arabic, Urdu, Hindi — no full-text storage)

**Defer (Phase 4+):**
- Sound-sketch generation (MusicGen / Stable Audio Open, local GPU) — requires training-focus generator and hardware ≥12 GB VRAM
- Monthly sound-thesis essay tooling — requires 12 completed cycles
- Community / discovery newsletter infrastructure — Phase 5+

### Architecture Approach

The system is a four-layer Python monolith: every external dependency is isolated behind an ABC in `adapters/`; all scheduled work is stateless functions in `jobs/` that write only through `store/` modules with explicit transaction boundaries; all trust-boundary crossings pass through `validation/` before any write or delivery; all budget constants live exclusively in `config.yaml`. The `sketches/` directory is a hard sandbox — generated audio never enters `data/` or `exports/`. The taste profile is an append-only ledger: every update writes an evidence row with `source_listen_ids`, and the current profile state is git-exported as human-readable JSON after every committed update.

**Major components:**
1. `adapters/` (Layer 1) — SpotifyAdapter, ListenBrainzAdapter, MusicBrainzAdapter, BandcampAdapter, AudioFeatures ABC, SoundSketch ABC, LyricsFeatures, DigestDelivery
2. `jobs/` + `store/` (Layer 2) — stateless batch jobs (ingest, analyze, digest, respond, blind_test, predict, sketch) with transactional SQLite writes
3. `validation/` (Layer 3) — ProfileUpdateValidator, DigestValidator, PredictionScorer, BlindTestScorer
4. `config.yaml` + heartbeat monitor (Layer 4) — all budget caps, model tiers, alert thresholds; never hardcoded in jobs

### Critical Pitfalls

1. **Spotify API erosion** — Wrap every Spotify call behind SpotifyAdapter ABC tested against recorded fixtures; mirror every play event to ListenBrainz from day one; monthly smoke-test alerting on 403s; request only 5 required scopes at OAuth time.
2. **PKCE refresh token rotation** — Store the full token bundle atomically with a mutex on refresh calls; immediately verify new token with /me before proceeding; heartbeat alert if Monday job cannot acquire a valid token.
3. **Habit-loop collapse** — Enforce a 20-minute weekly operator interaction budget from Phase 1; the 30-second response form must be benchmarked; blind tests hard-cap at 10 minutes/10 questions; kill-criterion detector must auto-reduce scope on 4 consecutive skips.
4. **librosa accuracy overreach on non-Western audio** — Wrap every estimate in FeatureEstimate(value, confidence, method); gate key detection by genre (skip for Arabic/maqam/raga/Bollywood modal traditions); never feed low-confidence estimates into taste profile weights.
5. **Prediction post-hoc contamination** — Predictions must be written in the same transaction as digest finalization, before delivery; no UPDATE path for expected_response; enforced with immutability test.

## Implications for Roadmap

### Phase 0: Foundation and ADR Gate
**Rationale:** Spotify API scope constraints, ListenBrainz mirror strategy, lyrics data contract, MusicGen license sandboxing, and hardware benchmark for Phase 4 are all decisions that are expensive to reverse. The adapter pattern must be established before any business logic is written.
**Delivers:** Project skeleton (uv, ruff, pytest, pre-commit), SpotifyAdapter stub with PKCE OAuth and fixture-based tests, ListenBrainz mirror setup, config.yaml schema, ADRs for Spotify scope limits, lyrics no-storage contract, and sketches sandboxing.
**Avoids:** Spotify API scope creep; PKCE token rotation breakage; lyric text storage liability; MusicGen license confusion.

### Phase 1: Minimal Closed Loop
**Rationale:** The feedback cycle must be real before any analytical depth is added. A manually curated first digest is acceptable. The kill-criterion detector and heartbeat alerting must ship here — not later — because over-instrumented Phase 2–3 features will collapse an unmeasured loop.
**Delivers:** Spotify history ingestion → evidence-trailed taste profile v1 (5–7 dimensions) → weekly digest delivery (≤15 tracks) → 30-second response form → Monday profile update + ear report. Weekly overhead benchmark measured. Kill-criterion detector live.
**Addresses:** All Phase 1 table-stakes features from FEATURES.md.
**Avoids:** Habit-loop collapse; missing evidence trail; silent job failures.

### Phase 2: Local Audio Analysis and Blind Tests
**Rationale:** Blind tests require local acoustic features to generate plausible distractors. The AudioFeatures ABC must ship before blind test content generation. This phase also unlocks anchor comparison matching by BPM and spectral similarity.
**Delivers:** librosa-backed AudioFeatures ABC; MixLevelFeatures vs StemLevelFeatures type distinction enforced; weekly blind tests with scored results; blind-test trend storage; training-focus generator.
**Uses:** librosa 0.11.x, soundfile, pydub, optional Essentia behind feature flag.
**Implements:** adapters/audio_features.py, jobs/analyze.py, jobs/blind_test.py, validation/blind_test.py.
**Avoids:** librosa accuracy overreach (confidence wrapping required); stem analysis overreach (Demucs deferred).

### Phase 3: Prediction Scoring and Cross-Genre Intelligence
**Rationale:** Prediction scoring requires a stable taste profile (Phase 1) and acoustic features (Phase 2) to be meaningful. Cross-genre bridge logging and multilingual lyrics derived-features are independent but belong in this phase by complexity and dependency.
**Delivers:** Prediction-before-listen scoring (locked before delivery, immutable); weekly prediction accuracy metric; diversity probe quota (≥3 low-confidence tracks per digest); cross-genre bridge logging schema; multilingual lyrics derived-features module.
**Avoids:** Taste model confirmation bias (probe quota required); echo-chamber feedback loop; lyric text copyright trap.

### Phase 4: Sound Sketch Generation
**Rationale:** Requires training-focus generator (Phase 2) and a hardware benchmark gate. MusicGen/Stable Audio Open must be strictly sandboxed. This phase is explicitly optional and hardware-dependent.
**Delivers:** SoundSketch ABC (MusicGenBackend and/or StableAudioBackend); weekly sketch generation tied to training focus; sketches/ sandbox enforced with test assertions; VRAM auto-detection at startup.
**Avoids:** MusicGen OOM on underpowered hardware; CC-BY-NC license contamination of commercial work.

### Phase 5+: Community and Newsletter (Future)
**Rationale:** Community features must not be built before 12 completed weekly cycles. Cross-genre bridge logs from Phase 3 become the newsletter content source.
**Delivers:** Monthly sound-thesis essay tooling; community/newsletter infrastructure.

### Phase Ordering Rationale

- Phase 0 before Phase 1: Adapter pattern and ADRs must exist before any business logic calls Spotify or writes to the profile — constraints are expensive to retrofit.
- Phase 1 before Phase 2: The loop must be validated as habit-sustainable before complexity is added; blind tests require response data.
- Phase 2 before Phase 3: Blind tests require acoustic features for distractors; prediction scoring benefits from acoustic features for hypothesis quality.
- Phase 4 gate: Hardware benchmark at Phase 0 determines feasibility on the operator's machine.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3:** Multilingual NLP for Arabic/Urdu syllable counting and rhyme detection is niche; Musixmatch/Genius ToS for derived-feature extraction needs a targeted spike.
- **Phase 4:** audiocraft API changes frequently; stable-audio-tools vs diffusers integration pattern needs validation at build time; Stable Audio Open commercial license threshold needs re-reading at Phase 4 start.

Phases with standard patterns (skip research):
- **Phase 0:** Spotify PKCE OAuth, uv setup, SQLAlchemy + Alembic scaffold — all well-documented.
- **Phase 1:** Pydantic validation, APScheduler cron, SQLite evidence trail — ARCHITECTURE.md has the exact schemas.
- **Phase 2:** librosa feature extraction — STACK.md and PITFALLS.md cover the accuracy caveats fully.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All library versions sourced from official changelogs, PyPI, and Spotify developer blog |
| Features | HIGH | Table-stakes sourced from established products; differentiators from spec gap analysis |
| Architecture | HIGH | Four-layer pattern from PRINCIPLES.md; all component responsibilities explicitly mapped |
| Pitfalls | HIGH | Spotify API facts from official developer blog posts with dates; MIREX benchmark data for audio accuracy |

**Overall confidence:** HIGH

### Gaps to Address

- **Multilingual NLP tooling for Arabic/Urdu:** No specific library recommendation in research. Candidates: camel-tools (Arabic NLP) and urduhack. Needs a spike during Phase 3 planning.
- **Essentia on macOS ARM64:** No pre-built PyPI wheels as of June 2026. Document in Phase 2 ADR; does not block Phase 2 (librosa is primary).
- **ListenBrainz mirror completeness:** The pylistenbrainz submit-listen API needs a spike test to confirm listened_at timestamp round-trip is lossless.
- **Digest delivery channel:** Operator's preferred delivery mechanism is unspecified. Phase 1 ADR should pick a default (Markdown file to a digest/ directory) and document alternatives.

## Sources

### Primary (HIGH confidence)
- Spotify Developer Blog (Nov 2024) — audio-features/audio-analysis deprecation
- Spotify Developer Blog (Feb 2026) — dev-mode 5-user cap, Premium requirement
- Spotify Developer Docs: Scopes Reference, Quota Modes
- librosa 0.11.0 changelog (librosa.org) — March 2025 stable release
- spotipy 2.26.0 (PyPI) — CVE-2025-27154 fix confirmed
- SQLAlchemy 2.0.50 / Alembic 1.13.x changelogs
- MIREX 2021 Audio Key Detection benchmark — accuracy baselines
- MusicGen model card (HuggingFace) — CC-BY-NC 4.0 license, VRAM requirements

### Secondary (MEDIUM confidence)
- audiocraft GitHub MusicGen README — GPU requirements, model variants
- stabilityai/stable-audio-open-1.0 (HuggingFace) — diffusers integration
- litellm docs: Token Usage and Cost — completion_cost() API confirmed
- Sage Journals, 2024: Digital self-tracking habits and discontinuance
- Semantic Scholar: Bias and Feedback Loops in Music Recommendation

### Tertiary (LOW confidence / needs validation)
- Musixmatch developer tier ToS for derived-feature extraction — needs re-reading at Phase 3 planning
- Stable Audio Open community license commercial threshold — check model card at Phase 4 start
- camel-tools / urduhack for Arabic/Urdu NLP — candidates only, not yet evaluated

---
*Research completed: 2026-06-10*
*Ready for roadmap: yes*
