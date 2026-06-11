# SPEC.md — Sama (سماع): Music Studio — Analysis & Taste Development

> GSD: read alongside PRINCIPLES.md. This project optimizes for SKILL
> DEVELOPMENT (analytical ear, A&R judgment) first; money is a later
> byproduct, and the spec is honest about that.

## Vision

A personal music intelligence system that (1) builds a deep taste profile
from real listening history, (2) runs structured discovery experiments
across target genres, (3) trains the operator's analytical ear — production
layering, instrumentation, rhythm, lyrical craft, emotional architecture —
through guided listening and comparison exercises, and (4) prototypes
sounds via local generation models to internalize how production choices
create feel. Effectively: an A&R apprenticeship with an agent as the
research department.

## Operator context (taste seed)

- **Mature genres (deep reference points):** hip hop, R&B — use as the
  "anchor" genres for comparative analysis.
- **Active genres:** rock, blues, soul.
- **Expansion genres:** Celtic, French (chanson → modern), Bollywood,
  Arabic (tarab → modern pop) — cross-cultural ear is the differentiator.
- Multilingual listener (English/Arabic/Urdu/Hindi exposure) — lyrics
  analysis must be multilingual-aware.

## Outcomes

- **Primary (skill):** measurably better analytical ear — tracked via
  blind tests (see Validation) and a growing personal "sound thesis" doc.
- **Secondary (financial, optional, Phase 5+):** A&R-style discovery
  newsletter ("what's coming out of X scene"), curated cross-cultural
  playlists with written analysis, sync/playlist pitching consultation.
- **Loop closes when:** 12 weekly analysis cycles completed with blind-test
  scores trending up.
- **Kill criterion:** if weekly cycles are skipped 4 weeks straight, the
  system is too heavy — simplify or stop.

## Hard constraints (encode before any code)

1. **Spotify API reality:** the audio-features and audio-analysis endpoints
   were deprecated for new third-party apps (Nov 2024). Do NOT design
   around them. Spotify is used ONLY for: listening history (top/recent
   tracks & artists), library, and playlist write-back. All acoustic
   analysis happens LOCALLY on audio you legitimately have access to
   (purchased files, Bandcamp, your own stems) via librosa/Essentia.
2. **Lyrics & copyright:** never scrape, store, or redistribute full lyric
   texts. Lyrics analysis operates on properties and excerpts under
   personal fair-dealing use: themes, rhyme-scheme maps, syllable/flow
   density, structural diagrams — derived features, not the text.
   Generated reports quote ≤ short fragments.
3. **Generation models:** local/open models only (e.g., MusicGen,
   Stable Audio Open) for sound sketches; outputs are study artifacts,
   not releases; respect each model's license.

## System architecture (four layers)

- **Abstraction:** adapters for sources (Spotify history, ListenBrainz,
  MusicBrainz metadata, Bandcamp purchases), analyzers (librosa/Essentia
  behind one `AudioFeatures` interface), generators (one `SoundSketch`
  interface), and delivery (weekly digest).
- **Isolation:** listening data and derived taste profile live in local
  SQLite, versioned exports in git; analysis runs are stateless; generated
  audio sandboxed in `sketches/` and never mixed into the reference corpus.
- **Validation:** every taste-profile update shows its evidence (which
  listens moved which dimension); discovery recommendations carry
  `{hypothesis, expected_response, actual_response}` — the system predicts
  whether you'll like a track BEFORE you hear it, and is scored on it;
  weekly blind tests (identify instrument layers / production era / scene)
  gate "ear progress" claims with data.
- **Infrastructure capacity:** local-first (analysis on your machine);
  LLM budget capped per weekly cycle; one discovery batch (≤15 tracks)
  per week to protect attention.

## Ecosystem map (initial)

- **Scale:** 1 user (you), 9 genres, ~15 new tracks/week. 10x = a small
  community of cross-cultural listeners sharing structured reviews.
- **Time:** weekly cycle — Mon profile update, Wed discovery batch,
  weekend deep-listen + blind test. Monthly "sound thesis" essay.
- **Causality hypothesis:** analytical ear improves through COMPARATIVE
  listening anchored in mature genres (e.g., "this Arabic strings
  arrangement does what a D'Angelo horn stack does") — not through more
  volume. The system must always pair new sounds with anchor references.
- **Fan-in:** all sources converge at the taste-profile model — its
  evidence trail is the highest-validation node. **Fan-out:** one weekly
  digest drives all listening; if it's noisy, the week is wasted.
- **Emergence:** expect unexpected cross-genre bridges (Celtic ↔ blues
  modal overlaps, Bollywood ↔ R&B vocal runs, tarab ↔ soul melisma) —
  instrument and log every bridge discovered; these become the newsletter
  material later.
- **Incentives:** operator gets a rare cross-cultural A&R ear; (later)
  community members get discovery they can't find on algorithmic feeds.
- **Capacity:** the bottleneck is YOUR listening attention (~5 hrs/week).
  The system curates down, never up.
- **Feedback loops:** (1) listen-response scoring → taste-profile weights;
  (2) blind-test results → next week's training focus; (3) generation
  sketches → "why does this feel wrong" notes → production-ear checklist;
  (4) prediction accuracy → discovery-strategy tuning.
- **Bottleneck #1:** acoustic data access post-Spotify-deprecation.
  Phase 0 resolves the legal local-audio pipeline before anything else.

## Analysis dimensions (the curriculum)

- **Production/layering:** stem-level frequency mapping, dynamics, space
  (reverb/width), arrangement density over time.
- **Rhythm:** tempo/feel (swing %, syncopation density), groove templates
  per genre (boom-bap vs trap hat grids vs dhol patterns vs reel rhythms).
- **Instrumentation:** timbre identification drills; genre-typical voicings
  (oud vs guitar phrasing, uilleann pipes vs harmonica roles).
- **Lyrics (derived features only):** rhyme-scheme density, multilingual
  imagery patterns, narrative structures, flow-to-beat alignment in hip hop.
- **Emotion architecture:** tension/release maps, key/mode usage per
  tradition (maqam vs blues scale vs Dorian folk modes).

## Communication loops

- Agent → operator: Wednesday discovery digest (≤15 tracks, each with
  hypothesis + 2 listening prompts + 1 anchor comparison).
- Operator → agent: 30-second structured response per track (feel score,
  standout layer, would-replay) via simple form.
- Agent → operator: Monday "ear report" — blind-test results, profile
  drift, this week's training focus.
- (Phase 5+) Operator → community: monthly cross-cultural discovery essay.

## Operations alignment (OPERATIONS.md governs)

- **Action output:** forced-choice only (Rule 1) — schema in framework-alignment.md.
- **Measure row = AOR** logged to `metrics/loop_closure.csv` (Rule 4).
- **Coordination:** single agent + output schema + operator veto. No multi-agent.
- Session hygiene: `reentry.md` on every stop (Rule 2); WIP cap (Rule 3) and
  meta-work quarantine (Rule 5) apply.

## Phases

- **Phase 0 — Data rights + pipeline ADR.** Spotify scopes (history only),
  local-audio acquisition plan (Bandcamp/purchases), lyrics-derived-features
  policy, model licenses. ADRs for all.
- **Phase 1 — Smallest closed loop.** Spotify history → taste profile v1
  with evidence trail → ONE weekly discovery digest (manual curation
  allowed) → response form → profile update. Run 3 weeks.
- **Phase 2 — Local analysis engine.** librosa/Essentia feature extraction
  on owned audio; rhythm + layering reports; first blind tests.
- **Phase 3 — Prediction + curriculum.** Like-prediction scoring; weekly
  training focus generator; lyrics derived-feature module (multilingual).
- **Phase 4 — Generation studio.** Local sound-sketch generation tied to
  weekly focus ("recreate this groove feel"); production-ear checklist.
- **Phase 5 — Optional community/monetization.** Discovery essays,
  playlists with analysis, small listener community.

## Out of scope

Streaming-service scraping, full-lyrics storage, releasing generated
music, building a recommender for other users before your own loop has
run 12 weeks.
