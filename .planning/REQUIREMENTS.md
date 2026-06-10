# Requirements: Sonic A&R

**Defined:** 2026-06-10
**Core Value:** A measurably better analytical ear — tracked by weekly blind-test scores trending up across 12 completed weekly cycles.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Data Rights & Foundations

- [ ] **RIGHTS-01**: Operator can authenticate with Spotify via PKCE OAuth using only history/library/playlist scopes (no audio-features), with the scope decision recorded in an ADR
- [ ] **RIGHTS-02**: Legal local-audio acquisition pipeline (Bandcamp/purchases/own stems) is documented in an ADR before any acoustic analysis ships
- [ ] **RIGHTS-03**: Lyrics derived-features-only data contract (no full-text storage, ≤ short fragments in reports) is encoded as an ADR and enforced by schema
- [ ] **RIGHTS-04**: Generation-model licenses and a local hardware benchmark gate are documented in ADRs before any generation work

### Ingestion

- [ ] **INGEST-01**: Operator can ingest Spotify listening history (top/recent tracks, artists, library) into local SQLite on a schedule
- [ ] **INGEST-02**: Every play event is mirrored to ListenBrainz from day one (Spotify-erosion insurance)
- [ ] **INGEST-03**: Tracks are enriched with MusicBrainz metadata behind a source adapter

### Taste Profile

- [ ] **PROF-01**: Taste profile updates carry an evidence trail showing which listens moved which dimension
- [ ] **PROF-02**: Profile state is exported as human-readable versioned JSON in git after every committed update

### Discovery Loop

- [ ] **DISC-01**: Operator receives a Wednesday discovery digest of ≤15 tracks, each with `{hypothesis, expected_response}`, 2 listening prompts, and 1 anchor comparison to a mature genre
- [ ] **DISC-02**: Operator receives a Monday ear report (blind-test results, profile drift, week's training focus)
- [ ] **RESP-01**: Operator can log a ~30-second structured response per track (feel score, standout layer, would-replay)
- [ ] **RESP-02**: Responses update taste-profile weights in the next Monday cycle (listen-response feedback loop)

### Local Audio Analysis

- [ ] **AUDIO-01**: Operator can extract acoustic features from legitimately owned audio via one `AudioFeatures` interface (librosa primary, Essentia optional)
- [ ] **AUDIO-02**: Operator receives rhythm and production-layering reports per analyzed track
- [ ] **AUDIO-03**: Every acoustic estimate carries a confidence value; key/mode detection is genre-gated for non-Western traditions (maqam, raga)

### Ear Training

- [ ] **TRAIN-01**: Operator can take a weekly blind test (instrument layers / production era / scene), hard-capped at 10 minutes
- [ ] **TRAIN-02**: Blind-test scores are stored with trend tracking and gate all "ear progress" claims
- [ ] **TRAIN-03**: A weekly training focus is generated from the weakest blind-test dimension

### Prediction & Cross-Genre Intelligence

- [ ] **PRED-01**: System records a like-prediction for every digest track immutably before delivery (no post-hoc edits)
- [ ] **PRED-02**: Weekly prediction-accuracy metric is computed and reported (discovery-strategy tuning loop)
- [ ] **PRED-03**: Every digest includes ≥3 low-confidence probe tracks (anti-echo-chamber quota)
- [ ] **XGEN-01**: Cross-genre bridges discovered during listening are logged in a structured schema (newsletter raw material)

### Lyrics (Derived Features)

- [ ] **LYR-01**: Operator can analyze lyrics as derived features only (rhyme-scheme density, multilingual imagery patterns, flow-to-beat alignment) across English/Arabic/Urdu/Hindi

### Generation Studio

- [ ] **GEN-01**: Operator can generate local sound sketches via one `SoundSketch` interface, sandboxed in `sketches/` and never mixed into the reference corpus
- [ ] **GEN-02**: Weekly sketches are tied to the training focus and feed a production-ear checklist

### Operations & Reliability

- [ ] **OPS-01**: Every scheduled job emits a heartbeat; silent failures alert the operator
- [ ] **OPS-02**: Token/API/runtime budgets live in config, are enforced in code, and alert at 80%
- [ ] **OPS-03**: Kill-criterion detector flags 4 consecutive skipped weekly cycles and recommends simplification
- [ ] **OPS-04**: One-command setup and one-command run; weekly operator overhead measured and kept ≤20 minutes

## v2 Requirements

Deferred to future release (Phase 5+ per SPEC.md). Tracked but not in current roadmap.

### Community & Monetization

- **COMM-01**: Monthly cross-cultural discovery essay tooling (sound thesis)
- **COMM-02**: Curated playlists with written analysis for an external audience
- **COMM-03**: Discovery newsletter infrastructure
- **COMM-04**: Small listener community with structured reviews

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Spotify audio-features/audio-analysis endpoints | Deprecated for new apps Nov 2024 — designing around them is a dead end |
| Streaming-service scraping | Legal risk; Spotify used only for history/library/playlist write-back |
| Full-lyrics storage or redistribution | Copyright; derived features and short fragments only |
| Releasing generated music | Sketches are study artifacts; model licenses (CC-BY-NC) forbid it |
| Recommender for other users | Not before the operator's own loop has run 12 weeks |
| Real-time anything | Weekly batch cadence is the spec; protects the attention bottleneck |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| RIGHTS-01 | Phase 0 | Pending |
| RIGHTS-02 | Phase 0 | Pending |
| RIGHTS-03 | Phase 0 | Pending |
| RIGHTS-04 | Phase 0 | Pending |
| INGEST-01 | Phase 1 | Pending |
| INGEST-02 | Phase 1 | Pending |
| INGEST-03 | Phase 1 | Pending |
| PROF-01 | Phase 1 | Pending |
| PROF-02 | Phase 1 | Pending |
| DISC-01 | Phase 1 | Pending |
| DISC-02 | Phase 1 | Pending |
| RESP-01 | Phase 1 | Pending |
| RESP-02 | Phase 1 | Pending |
| OPS-01 | Phase 1 | Pending |
| OPS-02 | Phase 1 | Pending |
| OPS-03 | Phase 1 | Pending |
| OPS-04 | Phase 1 | Pending |
| AUDIO-01 | Phase 2 | Pending |
| AUDIO-02 | Phase 2 | Pending |
| AUDIO-03 | Phase 2 | Pending |
| TRAIN-01 | Phase 2 | Pending |
| TRAIN-02 | Phase 2 | Pending |
| TRAIN-03 | Phase 2 | Pending |
| PRED-01 | Phase 3 | Pending |
| PRED-02 | Phase 3 | Pending |
| PRED-03 | Phase 3 | Pending |
| XGEN-01 | Phase 3 | Pending |
| LYR-01 | Phase 3 | Pending |
| GEN-01 | Phase 4 | Pending |
| GEN-02 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 30 total (note: REQUIREMENTS.md initialization counted 26; actual count is 30 — OPS section adds 4)
- Mapped to phases: 30
- Unmapped: 0 ✓

---
*Requirements defined: 2026-06-10*
*Last updated: 2026-06-10 after roadmap creation — traceability table complete*
