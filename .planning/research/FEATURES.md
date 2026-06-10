# Feature Research

**Domain:** Personal music intelligence — taste profiling, structured discovery, blind-test ear training, cross-cultural genre analysis
**Researched:** 2026-06-10
**Confidence:** HIGH (table stakes sourced from established products; differentiators sourced from spec + gap analysis; anti-features sourced from documented failure modes)

---

## Feature Landscape

### Table Stakes (Loop Dies Without These)

These are the minimum features the weekly operator loop needs to function. If any are absent the feedback cycle breaks.

| Feature | Why Expected / Loop Role | Complexity | Notes |
|---------|--------------------------|------------|-------|
| Listening history ingestion (Spotify top/recent + library) | Raw material for all profile updates; no input = no profile | LOW | Spotify scopes: `user-top-read`, `user-read-recently-played`, `user-library-read`. OAuth PKCE. No audio-features/audio-analysis (deprecated Nov 2024). |
| Evidence-trailed taste profile (per-dimension weights with source links) | Operator must see *why* the profile changed or they cannot trust it; without trust they ignore Monday reports | MEDIUM | SQLite rows: `{dimension, old_weight, new_weight, source_listen_id, date}`. Every update writes a row. Profile is an append-only ledger, not a mutable blob. |
| Weekly discovery digest delivery (≤15 tracks, Wed) | The weekly batch is the system's primary output; if it doesn't land, nothing else matters | MEDIUM | Delivery = markdown/HTML email or local CLI print; hard cap of 15 tracks enforced in config; tracks must carry `{hypothesis, expected_response, 2 listening_prompts, 1 anchor_comparison}`. |
| 30-second structured response form per track | Closes the feedback loop (feel score → profile update); without it the system is read-only | LOW | Three fields: feel score (1–5), standout layer (free text or enum), would-replay (bool). Must be frictionless — a CLI prompt or minimal web form, not a spreadsheet. |
| Profile update from response form | Response data that never updates the profile is wasted operator attention | MEDIUM | Batch update on Monday; delta report shows which dimensions moved and by how much. |
| Weekly blind-test delivery (weekend) | Gates all "ear progress" claims; without scored results the skill growth thesis is unfalsifiable | MEDIUM | 3–5 questions per week: instrument-layer ID, production-era guess, scene/genre attribution. Results stored with timestamp for trend analysis. |
| Blind-test score storage + trend tracking | If scores are not persisted and trended, the kill criterion (4 skipped cycles) and the 12-week goal are undetectable | LOW | SQLite: `{week_id, test_type, score, max_score, date}`. Weekly score ≥ rolling average = "trending up" signal. |
| Monday "ear report" delivery | Closes the weekly loop; operator must receive feedback before the next Wednesday batch | LOW | Combines: blind-test delta, profile drift summary, this-week training focus. Text-only is fine. |
| Heartbeat alerting on scheduled jobs | Kill criterion requires detecting 4 consecutive skipped cycles; silent failures corrupt the loop silently | LOW | Simple: if a scheduled job misses its window, write an alert. Per PRINCIPLES.md: alert at 80% budget, alert on missed runs. |
| Local audio analysis pipeline (librosa / Essentia behind `AudioFeatures` interface) | Without local analysis there is no acoustic data at all (Spotify audio-features deprecated); basis of all rhythm/layering reports | HIGH | librosa: BPM, beat grid, spectral contrast, MFCCs, chroma, onset density. Essentia adds: danceability, dynamic complexity, tonal descriptors, SVM classifiers. Input: legally owned files (Bandcamp, purchased). Interface abstracts both libraries so they are swappable. |

---

### Differentiators (The Cross-Cultural A&R Ear)

Features that exist nowhere in mainstream tools and are the entire reason this system is worth building. Each differentiates by coupling skill measurement with structured musical context.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Prediction-before-listen scoring | Forces the taste model to be falsifiable. The system predicts `{expected_feel_score, expected_standout_layer, would_replay}` before the operator hears the track; prediction accuracy is scored weekly and drives discovery-strategy tuning. No existing product does this. | HIGH | Requires a prediction module that reads the current taste profile + track metadata and outputs a prediction record before the digest is delivered. After response, prediction is scored (`exact_match / off_by_one / wrong`). Weekly prediction accuracy % is a first-class metric. |
| Hypothesis-driven track packaging | Each digest track carries explicit reasoning (`hypothesis`: "this Bollywood string arrangement mirrors D'Angelo's harmonic tension in 'Untitled'") rather than opaque algorithmic scoring. Makes the system auditable and educationally useful. | MEDIUM | LLM generates hypothesis + 2 listening prompts + 1 anchor comparison per track. Stored as structured JSON attached to each listen record. |
| Cross-genre anchor comparisons (per track in every digest) | Every new track is paired with a reference from hip hop / R&B (the operator's mature genres). This operationalises the causality hypothesis: comparative listening, not volume, builds the ear. No platform does this systematically. | MEDIUM | Anchor selection: retrieve a track from the operator's confirmed-like history in the anchor genre that shares a structural feature (BPM range, harmonic mode, similar instrumentation density) with the discovery track. |
| Cross-genre bridge logging (Celtic ↔ blues, Bollywood ↔ R&B, tarab ↔ soul) | Instruments and logs every structural overlap discovered across the nine genres. These bridges are the eventual newsletter material; they are also proof that the A&R ear is forming. No existing tool tracks this. | MEDIUM | Schema: `{bridge_id, genre_a, genre_b, track_a, track_b, shared_feature, date_discovered, operator_note}`. Bridge entries are auto-suggested by the LLM and operator-confirmed. |
| Local acoustic feature extraction for rhythm + layering reports | Post-Spotify-deprecation, no third-party app can call audio-features. This pipeline is the only legal acoustic analysis path for owned audio. Provides: BPM, beat grid, swing %, syncopation density, onset count per time window (arrangement density), spectral contrast per stem if available. | HIGH | librosa for beat tracking, onset detection, spectral analysis; Essentia for tonal and higher-level descriptors. Shared `AudioFeatures` interface hides provider. Must handle FLAC, MP3, WAV. Bandcamp FLACs preferred for quality. |
| Multilingual lyrics derived-features module | Operator listens in English, Arabic, Urdu, Hindi. Lyrics analysis across all four languages extracts rhyme-scheme density, imagery patterns, flow-to-beat alignment, narrative structure — without storing full lyric text. No mainstream ear-training or scrobbling product is multilingual-aware at the feature level. | HIGH | Pipeline: fetch lyrics via licensed API (Genius/Musixmatch). Extract features via NLP (syllable count per bar, rhyme scheme map, dominant theme clusters). Store only derived features + short fragments (personal fair-dealing). Never store full text. |
| Weekly training-focus generator (blind-test → next week's curriculum) | Blind-test results drive what the system teaches next week, creating a self-correcting curriculum. ToneGym offers adaptive difficulty within fixed exercise types; this system selects *which dimension* to focus on (rhythm vs timbre vs harmony) based on where the operator's scores are weakest. | MEDIUM | Logic: find the dimension with largest gap between target and actual blind-test score → generate a training focus brief → attach to Monday report. |
| Like-prediction accuracy as a named metric | Prediction accuracy (not just listen counts or star ratings) is surfaced as a first-class weekly metric. Accuracy trending up = taste model is well-calibrated. Accuracy flat or down = discovery strategy needs adjustment. This makes the system falsifiable end-to-end. | MEDIUM | Requires prediction scoring (above) plus a weekly accuracy rollup: `{week_id, n_predictions, n_exact, n_off_by_one, n_wrong, accuracy_pct}`. |
| Structured discovery across nine genres in rotation | The system distributes discovery across hip hop, R&B, rock, blues, soul, Celtic, French, Bollywood, Arabic — not just toward the genres Spotify thinks you like. This prevents the "filter bubble" failure mode endemic to algorithmic recommenders. | MEDIUM | Genre rotation scheduler in config: each week assigns a primary and secondary expansion genre. Discovery batch respects the assignment. |
| Sound-sketch generation tied to weekly training focus | MusicGen / Stable Audio Open generates short sketches ("recreate this groove feel") to internalize production choices. Closes the loop between hearing and making. No mainstream ear-training product includes generative sketching. | HIGH | Local GPU required (16 GB VRAM recommended for MusicGen medium). Sketches sandboxed in `sketches/` directory. Never mixed into reference corpus. Prompt constructed from weekly focus + anchor reference. |

---

### Anti-Features (Deliberately Not Building)

Features that seem natural but would break the loop, create legal risk, or dilute focus.

| Anti-Feature | Why Requested | Why Problematic | What to Do Instead |
|--------------|---------------|-----------------|-------------------|
| Streaming-service scraping (Spotify, Apple Music, YouTube Music) | More audio data, richer signals | Legal risk (ToS violation, CFAA exposure); Spotify deprecated audio-features Nov 2024 deliberately; scraping would replicate deprecated functionality illegally | Use Spotify only for history/library/playlist-write via official scopes. All acoustic analysis on legitimately owned audio. |
| Spotify audio-features / audio-analysis API calls | Easy acoustic data, no local compute | Endpoints deprecated for new third-party apps as of Nov 2024. Any design that depends on them breaks at launch. | librosa + Essentia on local files. Higher quality, no rate limits, no deprecation risk. |
| Full lyrics storage or redistribution | Rich text enables NLP | Copyright violation; lyrics are commercially licensed content; storing them creates DMCA liability even for personal use at scale | Derived features only: rhyme-scheme maps, syllable/flow density, theme clusters, structural diagrams. Quote ≤ short fragments under personal fair-dealing. Never store the full text. |
| Social / community features (shared profiles, following, feeds) | Engagement, discovery serendipity | Scope creep before the operator's own loop has run 12 weeks; community management overhead would distort the single-operator skill focus; RateYourMusic/Musicboard pattern shows moderation becomes the dominant problem | Defer to Phase 5+. The monthly "sound thesis" essay is the community seed — write it first, build audience second. |
| Recommender for other users | Monetisation path | Before 12 completed weekly cycles the taste model has no external validity; recommending to others with an unvalidated model produces noise and damages trust | Build the operator's own loop to 12-week completion first. Document cross-genre bridge findings. Then the newsletter content is already written. |
| Releasing generated audio as music | Creative output, potential income | Model licenses (MusicGen CC-BY-NC, Stable Audio Open research license) restrict commercial use of outputs; generated sketches lack the compositional intentionality of finished work; releasing them before the ear is trained would be premature | Sketches are study artifacts. Use them for "why does this feel wrong" notes. The production-ear checklist that emerges has more long-term value than the sketches. |
| Algorithmic-only discovery (no operator-curation path) | Less manual work | Spotify Discover Weekly failure mode: algorithm converges on familiar; prediction accuracy stays flat; operator loses trust when surprising tracks stop appearing | Always allow operator override / manual addition to any weekly batch. The hypothesis-driven packaging applies to manually added tracks too. |
| Unlimited weekly track volume (more than 15 tracks/week) | More exposure, faster learning | The binding bottleneck is operator listening attention (~5 hrs/week). Expanding the batch inflates cognitive load without improving outcomes; AI-curated playlist fatigue research shows this is the primary reason users disengage from discovery products | Hard cap of 15 tracks/week enforced in config. If the operator wants more, they surface it via response form ("would-replay" = yes + request next track from same artist). |
| Social listening metrics as proof-of-taste (play counts, stream counts) | Easy proxy for quality | Popularity bias: algorithms surface what's already popular, not what's cross-culturally interesting; Spotify Wrapped's "listening age" and "top artist by streams" are engagement metrics, not analytical ear metrics | Use blind-test scores + prediction accuracy as the primary proof-of-ear metrics. Play counts are logged but never used as quality signals in the taste profile. |
| LLM-generated taste labels without evidence trail | Convenient summaries | Without traceable evidence, the operator cannot contest or correct the model; taste labels become a black box that erodes trust; this is the ListenBrainz/collaborative-filtering failure mode (unexplained recommendations) | Every profile dimension update writes its evidence row. The LLM synthesises the pattern; the raw evidence is always queryable. |

---

## Feature Dependencies

```
Listening history ingestion
    └──enables──> Evidence-trailed taste profile
                      └──enables──> Prediction-before-listen scoring
                      └──enables──> Hypothesis-driven track packaging
                      └──enables──> Weekly digest delivery
                                        └──requires──> Structured response form
                                                           └──enables──> Profile update
                                                           └──enables──> Prediction accuracy scoring

Local audio analysis pipeline (AudioFeatures interface)
    └──enables──> Rhythm + layering reports
    └──enables──> Blind-test content generation (production-era questions, layering questions)
    └──enables──> Anchor comparison matching (BPM / spectral similarity)

Blind-test delivery + scoring
    └──enables──> Weekly training-focus generator
    └──enables──> Blind-test score trend (12-week goal tracking)
    └──feeds──>   Monday ear report

Weekly training-focus generator
    └──enables──> Sound-sketch generation prompt construction

Multilingual lyrics derived-features
    └──enhances──> Hypothesis-driven track packaging (can reference lyrical flow)
    └──enhances──> Cross-genre bridge logging (rhyme structures as bridge signals)

Cross-genre anchor comparisons
    └──requires──> Evidence-trailed taste profile (to know which anchor genre tracks are confirmed likes)
    └──enables──> Cross-genre bridge logging

Cross-genre bridge logging
    └──enables──> Monthly "sound thesis" essay (Phase 5+)
    └──enables──> Discovery newsletter (Phase 5+)

Sound-sketch generation
    └──requires──> Local audio analysis pipeline (to characterize the target feel)
    └──requires──> Weekly training-focus generator (to know what groove/feel to sketch)
    └──isolated──> sketches/ directory (never touches reference corpus)
```

### Dependency Notes

- **Prediction scoring requires the taste profile to exist first:** the prediction module reads current dimension weights; Phase 1 must ship a v1 profile before Phase 3 ships prediction scoring.
- **Blind tests require local audio analysis:** production-era and layer-identification questions need acoustic features to generate plausible distractors; Phase 2 must precede Phase 3 blind tests.
- **Anchor comparisons require a confirmed-like history:** the system needs at least 3 weeks of response data before anchor selection is reliable; manual anchor specification is the Phase 1 fallback.
- **Sound sketches conflict with reference corpus integrity:** `sketches/` must be excluded from any query that builds the taste profile or generates discovery tracks; a filesystem boundary (separate directory, excluded glob) enforces this.
- **Multilingual lyrics module is independent but additive:** it enhances hypothesis quality and bridge logging but does not block any other feature; it can ship in Phase 3 without breaking Phase 1 or Phase 2.

---

## MVP Definition

### Launch With — Phase 1 (Smallest Closed Loop)

The minimum that makes the feedback cycle real. Manual curation allowed.

- [ ] Spotify history ingestion (top tracks + recent plays, no audio-features calls) — without input the profile cannot exist
- [ ] Evidence-trailed taste profile v1 (5–7 dimensions: energy, rhythmic complexity, melodic density, lyrical craft proxy, emotional register, genre diversity, cultural range) — must show which listens moved which dimension
- [ ] One weekly discovery digest delivered (≤15 tracks, Wed) — even if first batch is manually curated; track record must include `{hypothesis, expected_response, 2 listening_prompts, 1 anchor_comparison}`
- [ ] 30-second structured response form (feel score, standout layer, would-replay) — must be frictionless
- [ ] Profile update from responses (Monday) — completes the cycle
- [ ] Monday ear report (text, minimal) — blind-test section placeholder until Phase 2
- [ ] Heartbeat alerting on scheduled jobs — detects missed cycles before they compound

### Add After 3 Validated Weekly Cycles — Phase 2

- [ ] Local audio analysis pipeline (librosa + Essentia behind `AudioFeatures` interface) — unlocks rhythm/layering reports and real blind tests
- [ ] Weekly blind tests (instrument layers, production era, scene) with scored results — makes skill growth falsifiable
- [ ] Blind-test score trend storage — enables 12-week goal tracking
- [ ] Training-focus generator (blind-test → next week's focus brief) — closes the curriculum loop

### Add After Phase 2 Is Stable — Phase 3

- [ ] Prediction-before-listen scoring (system predicts response before digest delivery; scored after) — requires stable taste profile from Phase 1 + acoustic features from Phase 2
- [ ] Weekly prediction accuracy metric — requires prediction scoring
- [ ] Multilingual lyrics derived-features module (English, Arabic, Urdu, Hindi — no full-text storage) — enhances hypothesis quality
- [ ] Cross-genre bridge logging (structured schema, operator-confirmed) — enables Phase 5 newsletter

### Phase 4

- [ ] Sound-sketch generation tied to weekly training focus (MusicGen / Stable Audio Open, local GPU) — requires training-focus generator from Phase 2 and audio analysis from Phase 2

### Future Consideration — Phase 5+ (Optional)

- [ ] Monthly "sound thesis" essay tooling — requires 12 completed cycles and cross-genre bridge log as source material
- [ ] Community / discovery newsletter infrastructure — requires validated operator loop; do not build before Phase 5

---

## Feature Prioritization Matrix

| Feature | Operator Value | Implementation Cost | Phase | Priority |
|---------|---------------|---------------------|-------|----------|
| Listening history ingestion | HIGH | LOW | 1 | P1 |
| Evidence-trailed taste profile | HIGH | MEDIUM | 1 | P1 |
| Weekly digest delivery (≤15 tracks) | HIGH | MEDIUM | 1 | P1 |
| 30-second response form | HIGH | LOW | 1 | P1 |
| Profile update from responses | HIGH | MEDIUM | 1 | P1 |
| Monday ear report | HIGH | LOW | 1 | P1 |
| Heartbeat alerting | HIGH | LOW | 1 | P1 |
| Local audio analysis pipeline | HIGH | HIGH | 2 | P1 |
| Weekly blind tests + scoring | HIGH | MEDIUM | 2 | P1 |
| Blind-test score trend | HIGH | LOW | 2 | P1 |
| Training-focus generator | HIGH | MEDIUM | 2 | P1 |
| Prediction-before-listen scoring | HIGH | HIGH | 3 | P2 |
| Prediction accuracy metric | HIGH | LOW | 3 | P2 |
| Multilingual lyrics derived-features | MEDIUM | HIGH | 3 | P2 |
| Cross-genre bridge logging | MEDIUM | MEDIUM | 3 | P2 |
| Hypothesis-driven track packaging (LLM) | HIGH | MEDIUM | 1–3 | P1 (starts manual, automates in Phase 3) |
| Anchor comparison per track | HIGH | MEDIUM | 1–2 | P1 (manual Phase 1, automated Phase 2) |
| Sound-sketch generation | MEDIUM | HIGH | 4 | P3 |
| Monthly sound thesis tooling | LOW | MEDIUM | 5 | P3 |
| Community / newsletter infrastructure | LOW | HIGH | 5 | P3 |

**Priority key:**
- P1: Required for the closed feedback loop
- P2: Adds falsifiability and depth once loop is running
- P3: Adds leverage / community once loop is validated

---

## Competitor Feature Analysis

| Feature | Last.fm / ListenBrainz | ToneGym / EarMaster | Spotify Wrapped / Discover Weekly | This System |
|---------|------------------------|---------------------|-----------------------------------|-------------|
| Listening history tracking | Yes — scrobbling, play counts, top artists/tracks, temporal trends | No | Yes — annual summary, top songs/artists/genres, listening age | Yes — via Spotify API, evidence-trailed |
| Taste profiling | Implicit (play count aggregation, genre tags) — no evidence trail | No | Implicit (genre clusters, listening evolution) — no evidence trail | Explicit per-dimension weights with source provenance |
| Discovery / recommendations | Collaborative filtering (similar users), weekly playlists (Troi engine) | No | Algorithmic (collaborative + content filtering), Discover Weekly | Hypothesis-driven, anchored in mature genres, prediction-scored |
| Prediction before listen | No | No | No | Yes — core differentiator |
| Ear training exercises | No | Yes — intervals, chords, scales, rhythm, melody, sight-singing; gamified | No | Yes — blind tests (instrument layers, production era, scene); purpose-built for cross-cultural genres |
| Scored progress tracking | No (play count proxies only) | Yes — ToneScore, performance index, drill-level stats | No | Yes — blind-test scores trended over 12 weeks |
| Adaptive curriculum | No | Partially — adapts difficulty within fixed exercise types | No | Yes — weekly training focus driven by weakest blind-test dimension |
| Anchor / comparative listening | No | No | No | Yes — every digest track paired with hip hop / R&B anchor |
| Cross-genre bridge logging | No | No | No | Yes — structured schema, operator-confirmed |
| Multilingual lyrics analysis | No | No | No | Yes — derived features only, 4 languages |
| Local acoustic analysis | No | No (MIDI input for exercises) | Was available (deprecated Nov 2024) | Yes — librosa + Essentia on owned files |
| Sound sketch generation | No | No | No | Yes — MusicGen / Stable Audio Open, Phase 4 |
| Evidence trail for profile updates | No | N/A | No | Yes — every update logged with source |
| Attention budget enforcement | No (passive accumulation) | No | No | Yes — hard cap of 15 tracks/week in config |

---

## Sources

Research drawing from:
- [ListenBrainz open-source platform analysis](https://medium.com/@prprevite/analyzing-listenbrainz-an-open-source-music-tracking-platform-64826f590eb1)
- [Troi recommendation engine (MetaBrainz)](https://github.com/metabrainz/troi-recommendation-playground)
- [ToneGym feature coverage (Sonofield 2026)](https://sonofield.com/blog/best-ear-training-apps-2026)
- [EarMaster feature list](https://www.earmaster.com/products/ear-training-sight-singing/earmaster-software.html)
- [Spotify Wrapped 2025 analysis (SoundGuys)](https://www.soundguys.com/spotify-wrapped-2025-149778/)
- [Recommendation fatigue analysis (GodMode Music)](https://godmodemusic.substack.com/p/recommendation-fatigue-music-discovery)
- [librosa audio analysis capabilities](https://medium.com/@noorfatimaafzalbutt/librosa-a-comprehensive-guide-to-audio-analysis-in-python-3f74fbb8f7f3)
- [Essentia audio analysis library](https://essentia.upf.edu/)
- [Musicboard / RateYourMusic journaling patterns](https://musicboard.app/)
- [MusicGen capabilities and limitations (AudioCraft)](https://audiocraft.metademolab.com/musicgen.html)
- [Stable Audio Open architecture (arXiv 2407.14358)](https://arxiv.org/html/2407.14358v1)
- [Multilingual lyrics NLP analysis (ResearchGate)](https://www.researchgate.net/publication/221573745_Natural_language_processing_of_lyrics)
- [Spotify Discover Weekly algorithm shortcomings](https://www.oriondistro.com/2026/05/19/discover-weekly/)
- [Cross-cultural music similarity (CCRMA / PLOS One)](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0189399)

---

*Feature research for: Sonic A&R — Music Analysis & Taste Development System*
*Researched: 2026-06-10*
