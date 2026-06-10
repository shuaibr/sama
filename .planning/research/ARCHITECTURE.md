# Architecture Research

**Domain:** Local-first personal music intelligence and ear-training system
**Researched:** 2026-06-10
**Confidence:** HIGH

## Standard Architecture

### System Overview

The system maps directly onto the four mandatory layers from PRINCIPLES.md. Every named component lives in exactly one layer; data flows top-to-bottom for ingestion and bottom-to-top for reporting.

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — ABSTRACTION (adapters + interfaces)                      │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │SpotifyAdapter│  │ListenBrainz  │  │MusicBrainz   │              │
│  │(history +    │  │Adapter       │  │Adapter       │              │
│  │ library +    │  │(listens,     │  │(metadata,    │              │
│  │ playlist wb) │  │ stats)       │  │ tags, genres)│              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                       │
│  ┌──────┴─────────────────┴─────────────────┴───────┐              │
│  │         SourceRecord  (common schema)             │              │
│  └──────────────────────┬────────────────────────────┘              │
│                         │                                           │
│  ┌──────────────┐  ┌────┴─────────┐  ┌──────────────┐              │
│  │BandcampAdapter│  │AudioFeatures │  │SoundSketch   │              │
│  │(CSV/export   │  │Interface     │  │Interface     │              │
│  │ import only) │  │(librosa or   │  │(MusicGen or  │              │
│  └──────────────┘  │ Essentia     │  │ Stable Audio │              │
│                    │ behind one   │  │ behind one   │              │
│                    │ Python ABC)  │  │ Python ABC)  │              │
│                    └──────┬───────┘  └──────┬───────┘              │
│                           │                 │                       │
│  ┌──────────────┐  ┌──────┴───────┐         │                       │
│  │DigestDelivery│  │LyricsFeatures│         │                       │
│  │Interface     │  │Interface     │         │                       │
│  │(Markdown/    │  │(derived      │         │                       │
│  │ email/file)  │  │ features,    │         │                       │
│  └──────┬───────┘  │ no raw text) │         │                       │
│         │          └──────────────┘         │                       │
├─────────┼──────────────────────────────────-┼─────────────────────-┤
│  LAYER 2 — ISOLATION (stateless runs + durable versioned state)     │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Weekly Batch Runner  (stateless Python process)            │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │    │
│  │  │IngestionJob │  │AnalysisJob  │  │DigestGeneratorJob│    │    │
│  │  │(Mon/Wed)    │  │(async local)│  │(Wed batch)       │    │    │
│  │  └─────────────┘  └─────────────┘  └──────────────────┘    │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │    │
│  │  │ResponseIngest│  │BlindTestJob │  │PredictionScorer  │    │    │
│  │  │(operator    │  │(weekend)    │  │Job               │    │    │
│  │  │ form intake)│  └─────────────┘  └──────────────────┘    │    │
│  │  └─────────────┘                                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                         │                                           │
│  ┌──────────────────────▼──────────────────────────────────────┐    │
│  │  Durable State  (SQLite + git-versioned JSON exports)        │    │
│  │  ┌───────────┐  ┌──────────────┐  ┌──────────────────────┐  │    │
│  │  │listens.db │  │taste_profile │  │digest_history /      │  │    │
│  │  │(raw source│  │.db           │  │response_log.db       │  │    │
│  │  │ records)  │  │(profile +    │  │(weekly cycle         │  │    │
│  │  └───────────┘  │ evidence     │  │ records)             │  │    │
│  │                 │ trail)       │  └──────────────────────┘  │    │
│  │                 └──────────────┘                             │    │
│  │  ┌───────────────────────────────────────────────────────┐  │    │
│  │  │  sketches/  (generated audio — never in corpus)       │  │    │
│  │  └───────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 3 — VALIDATION (evidence, scoring, schema checks)            │
│                                                                     │
│  ┌───────────────────────┐  ┌─────────────────────────────────┐    │
│  │ ProfileUpdateValidator │  │ PredictionScorer                │    │
│  │ - every update carries │  │ - records {hypothesis,          │    │
│  │   which listens moved  │  │   expected, actual} triples     │    │
│  │   which dimension      │  │ - scores prediction accuracy    │    │
│  │ - JSON schema check    │  │   per week per genre            │    │
│  │   before commit        │  └─────────────────────────────────┘    │
│  └───────────────────────┘                                          │
│  ┌───────────────────────┐  ┌─────────────────────────────────┐    │
│  │ BlindTestScorer        │  │ DigestValidator                 │    │
│  │ - weekly blind test    │  │ - schema-checks agent JSON      │    │
│  │   scores gating "ear   │  │   output before delivery        │    │
│  │   progress" claims     │  │ - asserts ≤15 tracks,          │    │
│  └───────────────────────┘  │   each with hypothesis +        │    │
│                              │   prompts + anchor              │    │
│                              └─────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 4 — INFRASTRUCTURE CAPACITY (budgets, heartbeat, secrets)    │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  config.yaml  (all budgets live here, enforced in code)      │   │
│  │  - llm_tokens_per_week_cycle: <N>                            │   │
│  │  - max_api_calls_per_provider_per_day: { spotify: N, ... }   │   │
│  │  - digest_track_limit: 15                                    │   │
│  │  - model_tier: { orchestration: "frontier", bulk: "cheap" }  │   │
│  │  - alert_threshold_pct: 80                                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Heartbeat monitor  (one alert per scheduled job)            │   │
│  │  Secrets via env only (never in code or git)                 │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Layer | Responsibility | Communicates With |
|-----------|-------|----------------|-------------------|
| SpotifyAdapter | Abstraction | Fetch top/recent tracks, library, write playlists back via spotipy + PKCE OAuth | SourceRecord schema → IngestionJob |
| ListenBrainzAdapter | Abstraction | Fetch listen history via liblistenbrainz; optional cross-source enrichment | SourceRecord schema → IngestionJob |
| MusicBrainzAdapter | Abstraction | Metadata/genre/tag lookup via musicbrainzngs (≤1 req/sec rate limit respected) | SourceRecord → enrichment pipeline |
| BandcampAdapter | Abstraction | Import from CSV export only (no public fan-purchases API); maps to SourceRecord | listens.db |
| AudioFeatures (ABC) | Abstraction | Single interface over librosa/Essentia; returns typed FeatureSet; swapping backend touches one file | AnalysisJob |
| SoundSketch (ABC) | Abstraction | Single interface over MusicGen/Stable Audio Open; returns path in sketches/ | SoundSketchJob |
| LyricsFeatures | Abstraction | Derived-features only (rhyme density, flow alignment, imagery) — no raw lyric storage | DigestGeneratorJob |
| DigestDelivery | Abstraction | Renders digest to Markdown/email/file; swappable delivery channel | DigestGeneratorJob |
| IngestionJob | Isolation | Stateless: pulls from adapters → writes SourceRecords to listens.db | All source adapters, listens.db |
| AnalysisJob | Isolation | Stateless: reads owned-audio paths → calls AudioFeatures interface → writes FeatureSet records | AudioFeatures ABC, listens.db |
| DigestGeneratorJob | Isolation | Stateless: reads profile → generates ≤15 tracks with hypothesis/prompts/anchor → validates → delivers | TasteProfile store, DigestValidator, DigestDelivery |
| ResponseIngestionJob | Isolation | Reads operator form responses → passes to ProfileUpdateValidator → commits updates | taste_profile.db, ProfileUpdateValidator |
| BlindTestJob | Isolation | Presents blind test (instrument layers / era / scene) → collects response → writes scored result | taste_profile.db, BlindTestScorer |
| PredictionScorerJob | Isolation | Before-listen: reads profile → writes prediction; after-listen: reads actual response → scores | digest_history.db, response_log.db |
| TasteProfile store | Isolation | SQLite table for profile dimensions + evidence trail; git-versioned JSON export on every update | ProfileUpdateValidator |
| ProfileUpdateValidator | Validation | Asserts every update record contains source_listen_ids; JSON schema checks before write | taste_profile.db |
| PredictionScorer | Validation | Records {hypothesis, expected_response, actual_response}; computes weekly accuracy by genre | digest_history.db |
| BlindTestScorer | Validation | Scores blind tests; gates "ear progress" claims with numeric data | response_log.db |
| DigestValidator | Validation | Schema-validates agent JSON output; enforces track count ≤ 15; checks required fields | DigestGeneratorJob |
| config.yaml | Capacity | All budget constants — token limits, API call caps, digest size, model tiers, alert threshold | All jobs read at startup |
| Heartbeat monitor | Capacity | Per-job heartbeat; alerts if scheduled job does not complete on time | All scheduled jobs |

## Recommended Project Structure

```
sonic-anr/
├── adapters/                  # Layer 1: one module per external dependency
│   ├── __init__.py
│   ├── spotify.py             # SpotifyAdapter (spotipy + PKCE)
│   ├── listenbrainz.py        # ListenBrainzAdapter (liblistenbrainz)
│   ├── musicbrainz.py         # MusicBrainzAdapter (musicbrainzngs)
│   ├── bandcamp.py            # BandcampAdapter (CSV import only)
│   ├── audio_features.py      # AudioFeatures ABC + LibrosaBackend/EssentiaBackend
│   ├── sound_sketch.py        # SoundSketch ABC + MusicGenBackend
│   ├── lyrics_features.py     # LyricsFeatures (derived only, no raw text)
│   └── delivery.py            # DigestDelivery ABC + MarkdownFileBackend
│
├── jobs/                      # Layer 2: stateless batch jobs
│   ├── __init__.py
│   ├── ingest.py              # IngestionJob
│   ├── analyze.py             # AnalysisJob (owned audio only)
│   ├── digest.py              # DigestGeneratorJob (Wed batch)
│   ├── respond.py             # ResponseIngestionJob
│   ├── blind_test.py          # BlindTestJob
│   ├── predict.py             # PredictionScorerJob
│   └── sketch.py              # SoundSketchJob
│
├── store/                     # Layer 2: durable state + schema migrations
│   ├── __init__.py
│   ├── schema.py              # All CREATE TABLE statements + migrations
│   ├── listens.py             # SourceRecord read/write
│   ├── profile.py             # TasteProfile read/write + git export trigger
│   └── digest_log.py          # Digest history + response log
│
├── validation/                # Layer 3
│   ├── __init__.py
│   ├── profile.py             # ProfileUpdateValidator (evidence trail check)
│   ├── digest.py              # DigestValidator (schema + track count)
│   ├── prediction.py          # PredictionScorer
│   └── blind_test.py          # BlindTestScorer
│
├── config.yaml                # Layer 4: ALL budgets and model tiers
├── run.py                     # One-command entry point (one-command run)
├── setup.py / pyproject.toml  # One-command setup
│
├── data/                      # Local durable state (gitignored binary db)
│   ├── listens.db
│   ├── taste_profile.db
│   └── digest_log.db
│
├── exports/                   # Git-versioned JSON exports (committed)
│   └── profile_YYYY-MM-DD.json
│
├── sketches/                  # Generated audio — sandboxed, never in corpus
│
├── docs/
│   ├── adr/                   # ADRs for significant decisions
│   └── ecosystem-map.md       # Refreshed at every phase boundary
│
└── tests/
    └── evals/                 # At least one eval case per named skill
```

### Structure Rationale

- **adapters/:** Every external dependency is isolated behind a module boundary. Swapping librosa for Essentia, or MusicGen for Stable Audio, touches exactly one file. The ABC pattern enforces the interface contract at import time.
- **jobs/:** Stateless batch processes. Each job reads from stores via the store layer, calls adapters via interfaces, writes back through validators, never shares mutable state between runs. A failed run cannot corrupt prior data because jobs never write to the SQLite directly — they call store modules that wrap writes in transactions.
- **store/:** Owns all SQLite schema and migration logic. The `profile.py` module fires the git export trigger after every committed profile update, producing a versioned JSON snapshot in `exports/`. This gives both queryable SQLite and human-readable git history.
- **validation/:** All trust boundary crossings are checked here before any write or delivery. Nothing reaches the operator unvalidated.
- **config.yaml:** Capacity constants live only here. Jobs assert budget headroom at startup and emit an alert at 80% consumption.
- **exports/:** JSON snapshots are the only files committed to git for versioning; binary `.db` files are gitignored. `git diff exports/` gives a human-readable audit trail of taste profile evolution.
- **sketches/:** Hard boundary — generation outputs never enter `data/` or `exports/`. A lint rule or test enforces this.

## Architectural Patterns

### Pattern 1: Adapter ABC with Single-File Swap

**What:** Define a Python abstract base class for every external dependency. Concrete backends implement the ABC. Callers import only the ABC type; the concrete class is injected via config.

**When to use:** Any external dependency that might change or needs to be testable in isolation — Spotify, librosa/Essentia, MusicGen, delivery channel.

**Trade-offs:** Small overhead of defining ABCs; pays back immediately when backends need swapping (Spotify deprecated audio analysis endpoints — the adapter layer isolated that blast radius) and when running tests against a fake adapter.

**Example:**
```python
# adapters/audio_features.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class FeatureSet:
    track_id: str
    tempo_bpm: float
    beat_grid: list[float]
    spectral_centroid_mean: float
    chroma_vector: list[float]
    # ... other fields

class AudioFeaturesAdapter(ABC):
    @abstractmethod
    def extract(self, audio_path: str) -> FeatureSet: ...

class LibrosaBackend(AudioFeaturesAdapter):
    def extract(self, audio_path: str) -> FeatureSet:
        # librosa implementation
        ...

class EssentiaBackend(AudioFeaturesAdapter):
    def extract(self, audio_path: str) -> FeatureSet:
        # Essentia implementation — same return type
        ...
```

### Pattern 2: Stateless Job with Transaction-Wrapped Store Writes

**What:** Each batch job is a pure function: `(config, db_connection) → side_effect`. It reads state, calls adapters, passes through validators, then calls the store layer. All DB writes are wrapped in explicit transactions. A job crash mid-run leaves the DB in its pre-run state.

**When to use:** Every scheduled job in the weekly cycle. This is the isolation layer guarantee from PRINCIPLES.md.

**Trade-offs:** Requires explicit transaction boundaries in the store layer (not ORM magic). Worth it because a failed Monday run must never corrupt the previous week's taste profile.

**Example:**
```python
# jobs/ingest.py
def run_ingestion(config: Config, db: Connection) -> IngestResult:
    adapter = build_adapter(config)            # from config, not hardcoded
    records = adapter.fetch_recent(limit=200)   # stateless fetch
    validated = [validate_source_record(r) for r in records]
    with db:                                    # transaction boundary
        store.listens.upsert_batch(db, validated)
    return IngestResult(count=len(validated))
```

### Pattern 3: Evidence-Trail Profile Update

**What:** The TasteProfile is never mutated directly. Every update is an `ProfileUpdate` event carrying: the dimension changed, the delta value, and the list of `source_listen_ids` that justify the change. The store appends the event to a `profile_updates` audit table, then updates the current profile view. The validator rejects any update missing `source_listen_ids`.

**When to use:** Every taste profile write — this is the core Validation layer guarantee from the spec.

**Trade-offs:** More complex write path than a simple `UPDATE` query. Required: the spec explicitly mandates an evidence trail for every profile update, and it powers prediction scoring.

**Example schema:**
```sql
CREATE TABLE profile_updates (
    id          INTEGER PRIMARY KEY,
    updated_at  TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%SZ','now')),
    dimension   TEXT NOT NULL,       -- e.g. 'genre_affinity.blues'
    delta       REAL NOT NULL,
    old_value   REAL NOT NULL,
    new_value   REAL NOT NULL,
    source_listen_ids TEXT NOT NULL,  -- JSON array, required
    cycle_week  TEXT NOT NULL        -- ISO week e.g. '2026-W24'
);
```

### Pattern 4: Prediction-Before-Listen Triple

**What:** For every track in a digest, the system writes a `Prediction` record before the operator listens. After response ingestion, the `PredictionScorer` joins prediction to actual response and computes accuracy. This forces the taste model to be falsifiable.

**When to use:** Every digest generation. Prediction record is committed as part of digest generation (before delivery), not after.

**Trade-offs:** Requires that predictions are locked in before any listening data is written — ordering matters. The DigestGeneratorJob must write predictions atomically with digest delivery.

**Example:**
```sql
CREATE TABLE predictions (
    id               INTEGER PRIMARY KEY,
    digest_cycle     TEXT NOT NULL,  -- ISO week
    track_id         TEXT NOT NULL,
    hypothesis       TEXT NOT NULL,  -- why recommended
    expected_feel    INTEGER,        -- 1-5
    expected_replay  INTEGER,        -- 0 or 1
    actual_feel      INTEGER,        -- filled in by ResponseIngestionJob
    actual_replay    INTEGER,
    accuracy_score   REAL            -- computed by PredictionScorer
);
```

### Pattern 5: Git-Versioned JSON Export for Profile Audit

**What:** After every committed profile update, `store/profile.py` exports the current profile state to `exports/profile_YYYY-MM-DD_Www.json` and calls `git add + git commit` on that file. The binary `.db` is gitignored.

**When to use:** Every profile update commit.

**Trade-offs:** Requires git to be available in the runtime environment (it is — local-first system). The JSON export is human-readable and diffable, giving a free audit log via `git log exports/` and `git diff`.

## Data Flow

### Weekly Cycle — Full Data Flow

```
MONDAY (profile update + ear report)
─────────────────────────────────────────────────────────────────────
[SpotifyAdapter]──┐
[ListenBrainzAdapter]─┤ ──→ SourceRecord stream
[BandcampAdapter]──┘         │
                             ▼
                       [IngestionJob]
                             │ writes (transactional)
                             ▼
                       [listens.db]
                             │ reads
                             ▼
                  [DigestGeneratorJob: profile phase]
                             │
                             ├──→ [ProfileUpdateValidator]
                             │         │ rejects missing evidence
                             │         ▼
                             │    [taste_profile.db]
                             │         │ on commit
                             │         ▼
                             │    [exports/profile_*.json] ←─ git commit
                             │
                             └──→ [BlindTestScorer]
                                       │
                                       ▼
                                 [Monday ear report]
                                 via [DigestDelivery]

WEDNESDAY (discovery digest)
─────────────────────────────────────────────────────────────────────
[taste_profile.db]
        │ reads current profile
        ▼
[DigestGeneratorJob]
        │
        ├──→ [MusicBrainzAdapter] (metadata enrichment)
        │
        ├──→ [LyricsFeatures] (derived features for candidate tracks)
        │
        ├──→ [PredictionScorerJob: write predictions BEFORE delivery]
        │         │
        │         ▼
        │    [predictions table in digest_log.db]
        │
        ├──→ [DigestValidator] (schema check, ≤15 tracks, required fields)
        │         │ on failure: abort and alert
        │         ▼
        └──→ [DigestDelivery] → operator

WEEKEND (deep-listen + blind test)
─────────────────────────────────────────────────────────────────────
[operator completes response form]
        │
        ▼
[ResponseIngestionJob]
        │ reads form responses
        ├──→ [ProfileUpdateValidator] (evidence trail check)
        │         │
        │         ▼
        │    [taste_profile.db] (update + git export)
        │
        └──→ [PredictionScorerJob: score predictions]
                  │
                  ▼
            [predictions table: accuracy_score filled in]

[BlindTestJob]
        │ presents test (instrument layers / era / scene)
        ├──→ operator response
        ├──→ [BlindTestScorer] → scores written to response_log.db
        └──→ feeds next Monday ear report

PHASE 2+ (owned audio analysis — async, triggered per new file)
─────────────────────────────────────────────────────────────────────
[owned audio file added to local library]
        │
        ▼
[AnalysisJob]
        │ calls [AudioFeatures ABC] (LibrosaBackend or EssentiaBackend)
        │
        ▼
[feature_sets table in listens.db]
        │
        ▼
[feeds rhythm + layering reports, blind test content]

PHASE 4+ (sound sketch generation — sandboxed)
─────────────────────────────────────────────────────────────────────
[weekly training focus from BlindTestScorer]
        │
        ▼
[SoundSketchJob]
        │ calls [SoundSketch ABC] (MusicGenBackend)
        │
        ▼
[sketches/YYYY-WNN-focus-name.wav]  ← NEVER enters data/ or exports/
        │
        ▼
[operator notes on "why does this feel wrong"]
        → feeds production-ear checklist (manual doc)
```

### Feedback Loops (named per PRINCIPLES.md requirement)

| Loop | Source | Measurement | Acts On |
|------|--------|-------------|---------|
| Listen-response → profile | ResponseIngestionJob | Profile dimension delta per week | TasteProfile weights |
| Blind-test → training focus | BlindTestScorer | Score per skill area per week | Next digest's training focus |
| Prediction accuracy → discovery strategy | PredictionScorer | Accuracy % per genre per week | DigestGeneratorJob hypothesis strategy |
| Sketch → production ear | Operator notes on generated sketches | Qualitative checklist items | Audio analysis dimensions |

## Scaling Considerations

This is a single-operator system by design. Scaling considerations apply only at the 10x scenario (small listener community) which the spec marks as Phase 5+.

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1 operator (now) | Monolith is correct. SQLite, local jobs, no API server needed. All complexity is internal loop logic, not infra. |
| 10 operators (Phase 5+ community) | Add a thin API layer over the store modules. SQLite → Postgres. Job runner → simple task queue (rq or Celery). Each operator keeps isolated profile. |
| 100+ operators | Not in scope. The spec explicitly gates community features until the solo loop has run 12 weeks. |

### Scaling Priorities (if/when Phase 5 arrives)

1. **First bottleneck:** Multiple operators sharing one SQLite file. Fix: one DB file per operator, or Postgres with per-user schema.
2. **Second bottleneck:** Analysis jobs blocking on local audio when multiple users submit. Fix: async job queue with worker pool.

## Anti-Patterns

### Anti-Pattern 1: Calling Spotify/External APIs Inside Business Logic

**What people do:** Import `spotipy` directly inside `digest.py` or `profile.py`.
**Why it's wrong:** Violates the Abstraction layer rule. When Spotify deprecated the audio-features endpoint in Nov 2024, any direct callers required rewrites across multiple modules. With the adapter, changing backends touches one file.
**Do this instead:** All API calls live in `adapters/`. Business logic imports adapter ABCs only, never concrete vendor SDKs.

### Anti-Pattern 2: Storing Generated Audio in the Reference Corpus

**What people do:** Save MusicGen output to the same directory as owned audio, letting `AnalysisJob` pick it up.
**Why it's wrong:** Contaminates the reference corpus with synthetic data, corrupting the taste model. A generated sketch that happens to "feel right" would shift profile weights based on the model's biases, not the operator's real listening response.
**Do this instead:** `sketches/` is a hard sandbox. `AnalysisJob` explicitly excludes that path. A test asserts no file in `sketches/` has a record in `listens.db`.

### Anti-Pattern 3: Mutable State Between Batch Runs

**What people do:** Use a long-lived in-memory object (e.g. a class instance) to accumulate state across multiple job invocations.
**Why it's wrong:** A crashed process or mid-run exception leaves the system in an inconsistent state. The next scheduled run starts with corrupted in-memory state.
**Do this instead:** Each job is a stateless function. State is always read from SQLite at the start of a run and written back at the end via transactions. The job runner can be killed and restarted safely at any point.

### Anti-Pattern 4: Hardcoding Budget Constants

**What people do:** Write `if token_count > 50000: raise BudgetExceeded` inline in a job.
**Why it's wrong:** Budget changes require code edits and test runs. The alert threshold (80%) becomes scattered. Auditing capacity usage requires reading code, not config.
**Do this instead:** All budgets in `config.yaml`. Jobs read budgets at startup, track consumption, and call a shared `budget.check(provider, amount)` function that handles both enforcement and alerting.

### Anti-Pattern 5: Skipping the Evidence Trail for "Small" Updates

**What people do:** For minor profile tweaks ("just a small feel-score update"), directly `UPDATE profile SET dimension_value = X` without logging which listens drove the change.
**Why it's wrong:** Breaks the evidence trail invariant. Prediction scoring becomes unreliable because the model's state is no longer fully explained by logged events. The operator loses the ability to audit why the system recommended something.
**Do this instead:** Every profile write goes through `ProfileUpdateValidator`. The validator is strict — it rejects any update without `source_listen_ids`. No exceptions.

### Anti-Pattern 6: Delivering Digest Before Locking Predictions

**What people do:** Generate predictions after the operator has already seen track recommendations (or worse, after responses are logged).
**Why it's wrong:** Post-hoc "predictions" are trivially accurate and meaningless. The loop "prediction accuracy → discovery strategy" only works if predictions are locked in before listening.
**Do this instead:** `DigestGeneratorJob` writes predictions to `digest_log.db` in the same transaction as digest finalization, before any delivery call. The delivery step is the last step.

## Integration Points

### External Services

| Service | Integration Pattern | Scopes / Constraints | Notes |
|---------|---------------------|---------------------|-------|
| Spotify Web API | SpotifyAdapter via spotipy + PKCE OAuth | `user-read-recently-played`, `user-library-read`, `user-top-read`, `playlist-modify-private` only | audio-features and audio-analysis endpoints deprecated Nov 2024 — never call them |
| ListenBrainz | ListenBrainzAdapter via liblistenbrainz | API token in env | Optional enrichment source; good cross-check on Spotify history |
| MusicBrainz | MusicBrainzAdapter via musicbrainzngs | No auth required; rate limit ≤1 req/sec (enforced in adapter) | Set user-agent per API requirements |
| Bandcamp | BandcampAdapter (CSV import only) | No public fan-purchases API exists | Operator exports purchase history CSV manually; adapter parses and maps to SourceRecord |
| librosa | AudioFeaturesAdapter LibrosaBackend | Local Python library; no network | Owned audio files only; path-based input |
| Essentia | AudioFeaturesAdapter EssentiaBackend | Local C++ library with Python bindings | Alternative or supplement to librosa; GPU not required |
| MusicGen / AudioCraft | SoundSketch MusicGenBackend | Local model; ≥16GB VRAM for large model | Outputs to sketches/ only; never enters analysis pipeline |
| Stable Audio Open | SoundSketch StableAudioBackend | Local model | Alternative to MusicGen; same interface |

### Internal Boundaries

| Boundary | Communication | Contract |
|----------|---------------|----------|
| adapters/ → jobs/ | Function calls via ABC interfaces; typed dataclass return values | Job never imports concrete adapter class |
| jobs/ → store/ | Function calls via store module API; jobs never write SQL directly | All SQL lives in store/; jobs call store functions |
| store/ → validation/ | Validation called before every write; validator raises on failure | Store never writes without calling validator first |
| jobs/ → config.yaml | Jobs read config at startup via shared `config.load()` | Budget constants never hardcoded in jobs |
| DigestGeneratorJob → PredictionScorerJob | Sequential, same transaction block | Predictions written before digest delivery |
| sketches/ → data/ | Hard barrier — no path from sketches to analysis input | Enforced by test assertion and AnalysisJob path exclusion |

## Sources

- [Spotipy documentation (spotipy.readthedocs.io)](https://spotipy.readthedocs.io/en/latest/) — OAuth PKCE flow, scopes
- [Spotify Scopes reference](https://developer.spotify.com/documentation/web-api/concepts/scopes) — confirmed available scopes post-deprecation
- [liblistenbrainz (PyPI)](https://pypi.org/project/liblistenbrainz/) — official Python client for ListenBrainz
- [MusicBrainz API documentation](https://musicbrainz.org/doc/MusicBrainz_API) — rate limits, includes, tag/genre schema
- [musicbrainzngs documentation](https://python-musicbrainzngs.readthedocs.io/_/downloads/en/v0.7.1/pdf/) — Python binding patterns
- [Bandcamp Help: Does Bandcamp have an API?](https://get.bandcamp.help/hc/en-us/articles/23020649597335-Does-Bandcamp-have-an-API) — confirmed no public fan-purchases API
- [GitHub: bandcamp-purchase-history Chrome extension](https://github.com/rxdazn/bandcamp-purchase-history) — manual CSV export is the only fan-side option
- [GitHub: simon-weber/Instant-SQLite-Audit-Trail](https://github.com/simon-weber/Instant-SQLite-Audit-Trail) — trigger-based audit table pattern
- [simonwillison.net: sqlite-history](https://simonwillison.net/2023/Apr/15/sqlite-history/) — SQLite change tracking with triggers
- [Tracking SQLite Database Changes in Git (Hacker News)](https://news.ycombinator.com/item?id=38110286) — sqlite-diffable + git versioning strategy
- [MusicGen on Hugging Face Transformers](https://huggingface.co/docs/transformers/en/model_doc/musicgen) — local integration, VRAM requirements
- [APScheduler documentation](https://apscheduler.readthedocs.io/en/3.x/userguide.html) — weekly job scheduling patterns
- [Python Adapter Pattern (Refactoring Guru)](https://refactoring.guru/design-patterns/adapter) — ABC-based adapter pattern rationale
- [Spoilfy: Mapping music to MusicBrainz from Spotify (GitHub)](https://github.com/solomonxie/Spoilfy) — real-world multi-source music adapter design reference

---
*Architecture research for: local-first personal music intelligence system*
*Researched: 2026-06-10*
