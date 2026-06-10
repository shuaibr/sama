# Roadmap: Sonic A&R — Music Analysis & Taste Development System

## Overview

Five phases transform a raw Spotify listening history into a working A&R apprenticeship loop: first locking down the legal and architectural constraints (Phase 0), then shipping the smallest verifiable weekly feedback cycle (Phase 1), adding local acoustic analysis and blind-test scoring (Phase 2), layering prediction scoring and cross-genre intelligence (Phase 3), and finally unlocking local sound-sketch generation tied to the weekly training focus (Phase 4). Community and monetization are v2 and intentionally out of scope.

## Phases

- [ ] **Phase 0: Data Rights + ADR Gate** - Lock Spotify scopes, local-audio pipeline, lyrics policy, and generation licenses as ADRs before any code touches data
- [ ] **Phase 1: Smallest Closed Loop** - Ingest history → taste profile → weekly digest → structured response → Monday update; heartbeat and kill-criterion live from day one
- [ ] **Phase 2: Local Analysis Engine** - librosa/Essentia behind AudioFeatures ABC; rhythm and layering reports; weekly blind tests with scored results and training-focus generator
- [ ] **Phase 3: Prediction + Cross-Genre Intelligence** - Immutable like-prediction scoring; weekly accuracy metric; probe-track quota; cross-genre bridge logging; multilingual lyrics derived-features
- [ ] **Phase 4: Generation Studio** - Local sound-sketch generation via SoundSketch ABC, sandboxed in sketches/, tied to weekly training focus

## Phase Details

### Phase 0: Data Rights + ADR Gate
**Goal**: Every irreversible constraint is documented as an ADR and enforced by schema before any operational code is written
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: RIGHTS-01, RIGHTS-02, RIGHTS-03, RIGHTS-04
**Success Criteria** (what must be TRUE):
  1. Operator can complete Spotify PKCE OAuth flow using only history/library/playlist scopes and the scope choice is recorded in a committed ADR
  2. Local-audio acquisition pipeline (Bandcamp/purchases/own stems) is documented in an ADR with a legal rationale visible to the operator
  3. Lyrics derived-features-only policy is encoded in an ADR and enforced by a Pydantic schema that rejects full lyric text at the store boundary
  4. Generation-model licenses (MusicGen CC-BY-NC, Stable Audio Open) and a local hardware benchmark gate are documented in ADRs before any generation dependency is installed
**Plans**: TBD

### Phase 1: Smallest Closed Loop
**Goal**: Operator can complete one full weekly cycle — ingest, digest, respond, profile update, ear report — with silent-failure alerting and kill-criterion detection live from the first run
**Mode:** mvp
**Depends on**: Phase 0
**Requirements**: INGEST-01, INGEST-02, INGEST-03, PROF-01, PROF-02, DISC-01, DISC-02, RESP-01, RESP-02, OPS-01, OPS-02, OPS-03, OPS-04
**Success Criteria** (what must be TRUE):
  1. Operator can run one command to ingest Spotify listening history (top/recent tracks, artists, library) into local SQLite, with every play event simultaneously mirrored to ListenBrainz and enriched with MusicBrainz metadata
  2. Wednesday digest of ≤15 tracks arrives with a hypothesis, 2 listening prompts, and 1 anchor comparison per track; operator can log a ~30-second structured response (feel score, standout layer, would-replay) per track
  3. Monday ear report reflects the prior week's responses and shows which listens moved which taste-profile dimension (evidence trail visible)
  4. Taste profile state is exported as human-readable versioned JSON in git after every committed update
  5. Every scheduled job emits a heartbeat; silent failures alert the operator; four consecutive skipped cycles trigger the kill-criterion flag; weekly operator overhead is measured and stays ≤20 minutes
**Plans**: TBD

### Phase 2: Local Analysis Engine
**Goal**: Operator can extract acoustic features from owned audio, receive rhythm and layering reports, take weekly scored blind tests, and receive a training focus derived from their weakest blind-test dimension
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: AUDIO-01, AUDIO-02, AUDIO-03, TRAIN-01, TRAIN-02, TRAIN-03
**Success Criteria** (what must be TRUE):
  1. Operator can run one command to extract acoustic features from legitimately owned audio via the AudioFeatures interface (librosa primary, Essentia optional behind a feature flag)
  2. Every analyzed track produces a rhythm report and a production-layering report; every estimate carries a confidence value; key/mode detection is skipped (not applied) for Arabic, maqam, and raga traditions
  3. Operator can take a weekly blind test (instrument layers, production era, scene) hard-capped at 10 minutes, with scores stored and trend-tracked
  4. All "ear progress" claims are gated on blind-test data; a weekly training focus is generated from the dimension with the weakest blind-test trend
**Plans**: TBD
**UI hint**: yes

### Phase 3: Prediction + Cross-Genre Intelligence
**Goal**: The system makes falsifiable predictions before the operator listens, anti-echo-chamber probe tracks are enforced, cross-genre bridges are logged, and multilingual lyrics are analyzed as derived features
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: PRED-01, PRED-02, PRED-03, XGEN-01, LYR-01
**Success Criteria** (what must be TRUE):
  1. System records a like-prediction for every digest track in the same transaction as digest finalization — before delivery — with no UPDATE path available after that point
  2. Weekly prediction-accuracy metric is computed and visible in the Monday ear report, enabling the operator to tune discovery strategy
  3. Every digest includes ≥3 low-confidence probe tracks outside the operator's established taste bands
  4. Cross-genre bridges discovered during listening (e.g., Celtic ↔ blues modal overlap, tarab ↔ soul melisma) are logged in a structured schema available as newsletter raw material
  5. Operator can analyze lyrics as derived features (rhyme-scheme density, imagery patterns, flow-to-beat alignment) across English, Arabic, Urdu, and Hindi without any full lyric text being stored
**Plans**: TBD

### Phase 4: Generation Studio
**Goal**: Operator can generate local sound sketches tied to the weekly training focus, sandboxed so generated audio never enters the reference corpus
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: GEN-01, GEN-02
**Success Criteria** (what must be TRUE):
  1. Operator can generate a sound sketch via the SoundSketch interface (MusicGen or Stable Audio Open backend); the sketch is written to sketches/ and blocked from entering data/ or exports/ by an enforced test assertion
  2. Weekly sketch prompts are driven by the current training focus, and each sketch feeds a production-ear checklist that the operator can review and annotate
**Plans**: TBD

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 0. Data Rights + ADR Gate | 0/TBD | Not started | - |
| 1. Smallest Closed Loop | 0/TBD | Not started | - |
| 2. Local Analysis Engine | 0/TBD | Not started | - |
| 3. Prediction + Cross-Genre Intelligence | 0/TBD | Not started | - |
| 4. Generation Studio | 0/TBD | Not started | - |
