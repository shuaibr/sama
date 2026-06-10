<!-- GSD:project-start source:PROJECT.md -->

## Project

**Sonic A&R — Music Analysis & Taste Development System**

A personal music intelligence system that builds a deep taste profile from the
operator's real listening history, runs structured weekly discovery experiments
across nine genres, trains the operator's analytical ear (production layering,
rhythm, instrumentation, lyrics-as-derived-features, emotional architecture)
through guided comparative listening and blind tests, and prototypes sounds via
local generation models. Effectively an A&R apprenticeship with an agent as the
research department, for a single operator.

**Core Value:** A measurably better analytical ear — tracked by weekly blind-test scores
trending up across 12 completed weekly cycles. Everything else (newsletter,
playlists, monetization) is a later byproduct.

### Constraints

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
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Critical Context: Spotify API Landscape (June 2026)

| Scope | Endpoint | Data |
|-------|----------|------|
| `user-top-read` | `GET /me/top/tracks` | Long-term, medium-term, short-term top tracks (up to 50) |
| `user-top-read` | `GET /me/top/artists` | Top artists by affinity |
| `user-read-recently-played` | `GET /me/player/recently-played` | Last 50 played tracks with timestamps |
| `user-library-read` | `GET /me/tracks` (paginated) | Saved tracks library |
| `user-library-read` | `GET /me/albums` | Saved albums |
| `playlist-read-private` | `GET /me/playlists` | User's playlists |
| `playlist-modify-private` | `POST /users/{id}/playlists` | Write discovery digest playlists back |

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Python | 3.11 | Primary language | 3.11 has the best balance of ecosystem support and performance; 3.12 still has some rough edges with audio/ML deps; 3.10 is the minimum for audiocraft |
| uv | latest (≥0.4) | Project/env management | Replaces pip + virtualenv + pip-tools in one binary; 10-100x faster than pip; handles pyproject.toml natively; Astral/Rye successor; one-command setup |
| spotipy | 2.26.0 | Spotify Web API client | Official-adjacent Python library; supports all needed scopes; SpotifyPKCE auth manager handles Authorization Code + PKCE flow with token caching; 2.26.0 tightened cache file permissions to 600 (fixes CVE-2025-27154) |
| SQLAlchemy | 2.0.x (≥2.0.50) | ORM + query layer | 2.0 unified async and sync APIs; declarative models map cleanly to the Isolation layer; swap to Postgres later without touching application code; do NOT use 2.1.x beta |
| Alembic | 1.13.x (≥1.13) | Schema migrations | Pairs with SQLAlchemy 2.0; batch migrations handle SQLite's limited ALTER TABLE; autogenerate from models; migration history in git |
| Pydantic | 2.x (≥2.7) | Schema validation + typed models | Rust-backed V2 is 5-20x faster than V1; validates all agent JSON outputs at the Validation layer boundary; use `model_validator` for cross-field rules; required for the "nothing unvalidated reaches the operator" principle |
| librosa | 0.11.x (≥0.11.0, released Mar 2025) | Primary acoustic feature extraction | Pure Python/NumPy/SciPy; easiest install (no C++ build tools); beat tracking, spectral centroid, MFCCs, chroma, onset strength, tempo estimation; covers 90% of rhythm + layering needs; stable API |

### Audio Analysis Layer

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| librosa | 0.11.0 | Primary: tempo, beat grid, spectral features, MFCCs, chroma, onset detection, harmonic/percussive separation | Always — default implementation behind `AudioFeatures` interface |
| essentia | 2.1b6.dev1438 (May 2026) | Secondary: key/mode detection (Essentia's key extractor is more accurate than librosa's), tonal analysis, genre-specific rhythm descriptors (tatum detection, groove templates) | Only when librosa's equivalent is insufficient; install only on tracks that need it; keep behind a feature flag |
| soundfile | 0.12.x | Audio I/O (WAV, FLAC, AIFF) | Load lossless formats; faster than librosa.load() for pure I/O |
| pydub | 0.25.x | MP3/AAC/OGG decode, segment extraction | Convert lossy formats to float array before passing to librosa; wraps ffmpeg |
| numpy | 1.26.x | Array operations | Locked to librosa's requirement; do not upgrade independently |

### Metadata Enrichment

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| musicbrainzngs | 0.7.1 | MusicBrainz REST API client | Artist/recording/release metadata, ISRC lookups, genre tags (folksonomy), cross-cultural instrument metadata; last release 2020 but the MB API is stable — the library works fine |
| liblistenbrainz | latest (≥0.5) | ListenBrainz client | Submit listens from Spotify history for cross-referencing; retrieve MB-enriched listen data; pylistenbrainz and liblistenbrainz are the same project, import as `pylistenbrainz` |
| requests | 2.31.x | HTTP for MusicBrainz rate-limited calls | musicbrainzngs uses requests internally; also needed for direct API calls with proper User-Agent headers (MB requires a valid User-Agent string) |

### Local Data Pipeline

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| SQLAlchemy | 2.0.x | ORM + connection pooling | All persistent state: listening history, taste profile, analysis results, prediction scores |
| Alembic | 1.13.x | Schema versioning | Every schema change; use `--autogenerate`; keep migrations in `db/migrations/` |
| Pydantic | 2.x | Inbound/outbound validation | Validate Spotify API responses before writing to DB; validate analysis results before writing to profile |
| APScheduler | 3.x | In-process weekly job scheduler | Triggers Monday profile update, Wednesday digest generation; no Redis/message broker needed for single-operator use; use `BackgroundScheduler` with SQLite job store for persistence across restarts |
| python-dotenv | 1.0.x | Environment/secrets loading | Loads `.env` file; keeps secrets out of code per PRINCIPLES.md; use `load_dotenv()` at entry point only |

### LLM Orchestration

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| anthropic | 0.26.x (official SDK) | Primary LLM client (Claude Sonnet for orchestration/synthesis) | Weekly digest generation, ear report synthesis, prediction scoring explanations — all "frontier model" tasks per PRINCIPLES.md §5 |
| litellm | 1.40.x | Provider-agnostic LLM interface + token/cost tracking | Wraps anthropic (and any cheaper provider for bulk extraction tasks); exposes `completion_cost()` and `token_counter()` per call; enforce per-run budget in config.yaml |
| tiktoken | 0.7.x | Token counting (pre-call budget check) | Count tokens before dispatching to LLM; compare against `config.max_tokens_per_run`; abort early if budget would be exceeded |

# config.yaml drives this — never inline

### Generation Layer (Phase 4)

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| audiocraft (MusicGen) | latest from GitHub | Text-to-music generation for sound sketches | `facebook/musicgen-small` (300M params, ~4GB VRAM) runs on consumer GPUs; `facebook/musicgen-medium` needs 16GB VRAM; install via `pip install -U audiocraft[inference]` |
| stable-audio-tools | latest from GitHub | Stable Audio Open 1.0 inference | 1.21B params; requires 16GB VRAM; access via `diffusers.StableAudioPipeline` (simpler) or native `stable-audio-tools` |
| torch | 2.3.x (CUDA 12.1) | PyTorch backend for both models | Pin to version compatible with audiocraft requirements; CPU-only inference is possible but 10-50x slower |
| diffusers | 0.27.x | Stable Audio Open pipeline abstraction | Hugging Face diffusers integrates Stable Audio Open natively; simpler than the native stable-audio-tools repo |
| transformers | 4.40.x | MusicGen model loading | audiocraft uses transformers under the hood; pin version |

- RTX 3060 12GB: `musicgen-small` runs comfortably; `musicgen-medium` is marginal (may OOM on long clips); Stable Audio Open will OOM
- RTX 3080/3090/4090 16GB+: both models run well
- CPU-only (no GPU): `musicgen-small` generates 5s of audio in ~3 minutes — usable for occasional study sketches
- Apple Silicon M2/M3: both models run via `device="mps"` at ~2-3x slower than CUDA

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| uv | Package management, venv creation | `uv init`, `uv add`, `uv run` — replaces pip + virtualenv; generates `uv.lock` |
| ruff | Linting + formatting | Replaces flake8 + black + isort in one tool; 100x faster than pylint; configure in `pyproject.toml` |
| pytest | Test runner | One test runner, no exceptions; `pytest-cov` for coverage; fixtures for SQLite in-memory DB |
| pytest-cov | Coverage | Required; set minimum thresholds in `pyproject.toml` |
| python-dotenv | Secrets management | `.env` file, never committed; `.env.example` committed with placeholder values |
| pre-commit | Git hooks | Run ruff + pytest --fast-only on every commit; enforces PRINCIPLES.md §2 "every merge green" |

## Installation

# Bootstrap (one-command setup per PRINCIPLES.md)

# Core data pipeline

# Audio analysis (primary)

# Audio analysis (secondary — behind feature flag)

# Metadata enrichment

# LLM orchestration

# Scheduling

# Dev dependencies

# Generation layer (Phase 4 only — heavy dependencies, install separately)

# uv add torch --index-url https://download.pytorch.org/whl/cu121

# uv add audiocraft transformers diffusers accelerate

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Spotify client | spotipy 2.26.0 | tekore, custom requests | spotipy has PKCE support, active maintenance, handles token refresh; tekore is more type-safe but smaller community |
| Audio extraction | librosa (primary) | Essentia only | Essentia has no macOS ARM64 wheels; requires compilation; harder `one-command setup`; use as secondary behind feature flag |
| Audio extraction | librosa (primary) | audioFlux | audioFlux is faster but immature Python API; limited documentation; less community support |
| DB layer | SQLAlchemy 2.0 | SQLModel | SQLModel is convenient but adds abstraction over SQLAlchemy 2.0; when SQLModel hits a limitation you debug two layers; direct SQLAlchemy 2.0 is clearer for complex queries the taste profile will need |
| Schema validation | Pydantic 2.x | attrs, dataclasses | Pydantic 2.x Rust backend is fast enough; JSON schema export for documentation; model_json_schema() for ADR artifacts |
| Scheduling | APScheduler | Celery Beat, cron | Celery requires Redis broker — unnecessary operational overhead for single-operator; cron lacks heartbeat alerting and job state persistence; APScheduler does both in-process |
| LLM client | litellm wrapping anthropic | LangChain | LangChain's abstraction cost (token overhead, hidden prompt manipulation, hard-to-debug chains) violates "boring tech" and "schema-check all agent JSON" principles; litellm is a thin wrapper with no prompt magic |
| LLM client | litellm wrapping anthropic | Direct anthropic SDK only | anthropic SDK has no cost tracking; litellm adds it with minimal overhead |
| Package manager | uv | Poetry, pip + venv | Poetry is slower; pip + venv requires pip-tools for lock files; uv is now the community standard for new Python projects |
| Music generation | MusicGen (audiocraft) | Riffusion, Suno API | Riffusion quality is lower; Suno API is cloud-only (violates local-first + license constraints); audiocraft is local, open-weight, well-documented |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `GET /audio-features` | Deprecated Nov 2024; returns 403 for new apps | librosa/Essentia local analysis |
| `GET /audio-analysis` | Deprecated Nov 2024; returns 403 for new apps | librosa beat_track(), onset_detect() |
| `GET /recommendations` | Deprecated Nov 2024; returns 403 for new apps | LLM-generated discovery from taste profile |
| `GET /artists/{id}/related-artists` | Deprecated Nov 2024; returns 403 for new apps | MusicBrainz relationships, ListenBrainz similar artists |
| `GET /browse/categories` | Removed Feb 2026 for Dev Mode apps | MusicBrainz genre tags |
| LangChain | Hidden prompt manipulation; large dependency tree; hard to schema-validate outputs; violates boring-tech principle | Direct anthropic SDK + litellm |
| SQLModel | Adds abstraction layer over SQLAlchemy 2.0; when it breaks you debug two layers; unclear maintainership | Direct SQLAlchemy 2.0 + Pydantic |
| Celery / Redis | Overkill for single-operator local scheduler; introduces Redis dependency and process management | APScheduler with SQLite job store |
| `essentia` as primary analyzer | No macOS ARM64 wheels (must compile); install complexity breaks `one-command setup` | librosa as primary; essentia as optional secondary |
| Spotipy < 2.26.0 | CVE-2025-27154: cache file has 644 permissions exposing auth tokens to other local users | `spotipy>=2.26.0` |
| `musicbrainzngs` direct HTTP calls | Rate-limit management, retries, and User-Agent handling are already in the library | `musicbrainzngs` with `set_useragent()` |
| Full lyric text storage | Copyright violation; SPEC.md explicit constraint | Derived features only: rhyme density, syllable count, flow-beat alignment ratios |
| Generated audio in reference corpus | Corrupts taste profile with synthetic data; SPEC.md hard constraint | `is_generated=True` flag; `sketches/` directory excluded from analysis queries |

## Stack Patterns by Variant

- Install Essentia from PyPI (`pip install essentia`) — wheels available
- Install audiocraft with CUDA 12.1 torch
- Enable `essentia_enabled: true` in `config.yaml`
- Do NOT install essentia from PyPI — no ARM64 wheels; either build from source or keep `essentia_enabled: false`
- audiocraft runs via `device="mps"` — 2-3x slower than CUDA but functional for study sketches
- librosa works natively on all platforms — no change needed
- librosa works fine for all acoustic analysis (no GPU dependency)
- Generation is impractical for anything beyond quick 5s clips: `musicgen-small` on CPU takes ~3 min/5s of audio
- Consider deferring Phase 4 (generation) until GPU access is available
- Development Mode requires Premium as of Feb 11, 2026 — this is a hard prerequisite
- There is no documented workaround; document as a blocking ADR decision
- Fallback: export Spotify history via `https://www.spotify.com/account/privacy/` (GDPR data export) and import JSON manually — covers historical data but not ongoing sync

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| librosa 0.11.x | numpy ≥1.20, scipy ≥1.2, Python ≥3.8 | Pin numpy to 1.26.x to stay compatible with audiocraft and essentia simultaneously |
| audiocraft (MusicGen) | torch 2.1.0+, Python ≥3.9, CUDA 12.1 | audiocraft pins torch; install torch first with correct CUDA index URL, then audiocraft |
| SQLAlchemy 2.0.x | alembic ≥1.10 | alembic 1.13.x supports SQLAlchemy 2.0 fully; do not use alembic < 1.10 with SA 2.0 |
| Pydantic 2.x | SQLAlchemy 2.0 | Use `model_config = ConfigDict(from_attributes=True)` for ORM model serialization; Pydantic V1 compat shim is available but not recommended |
| essentia 2.1b6.dev* | numpy < 2.0 | Essentia wheels are not yet built against numpy 2.x; pin `numpy<2.0` if essentia is enabled |
| spotipy 2.26.0 | requests ≥2.25.0 | spotipy uses requests for all HTTP; requests 2.31.x is compatible |
| litellm 1.40.x | anthropic ≥0.20, tiktoken ≥0.5 | litellm wraps anthropic SDK; upgrade litellm first, then check required anthropic version in litellm's deps |

## Sources

- Spotify Developer Blog, Nov 2024: [Changes to the Web API](https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api) — confirmed deprecation of audio-features/audio-analysis
- Spotify Developer Blog, Feb 2026: [Update on Developer Access and Platform Security](https://developer.spotify.com/blog/2026-02-06-update-on-developer-access-and-platform-security) — confirmed 5-user limit and Premium requirement
- Spotify Developer Docs: [Quota Modes](https://developer.spotify.com/documentation/web-api/concepts/quota-modes) — Development Mode scope and extended access criteria
- Spotify Developer Docs: [Scopes Reference](https://developer.spotify.com/documentation/web-api/concepts/scopes) — confirmed available scopes
- librosa PyPI / GitHub: [librosa 0.11.0 changelog](https://librosa.org/doc/main/changelog.html) — released March 2025; current stable
- Essentia PyPI: [essentia 2.1b6.dev1438](https://pypi.org/project/essentia/) — May 2026 build; Linux only on PyPI
- spotipy PyPI: [spotipy 2.26.0](https://pypi.org/project/spotipy/) — current; CVE-2025-27154 fix confirmed
- SQLAlchemy: [2.0.50 stable, 2.1.0b2 beta](https://www.sqlalchemy.org/blog/2026/04/16/sqlalchemy-2.1.0b2-released/) — use 2.0.x
- Alembic changelog: [1.18.4 current](https://alembic.sqlalchemy.org/en/latest/changelog.html)
- audiocraft GitHub: [MusicGen README](https://github.com/facebookresearch/audiocraft/blob/main/docs/MUSICGEN.md) — GPU requirements, model variants
- Stable Audio: [stabilityai/stable-audio-open-1.0 on HuggingFace](https://huggingface.co/stabilityai/stable-audio-open-1.0) — 1.21B params, diffusers integration
- litellm docs: [Token Usage & Cost](https://docs.litellm.ai/docs/completion/token_usage) — confirmed completion_cost() API
- musicbrainzngs PyPI: [0.7.1](https://pypi.org/project/musicbrainzngs/) — last release 2020; API stable; still the standard client
- liblistenbrainz / pylistenbrainz PyPI: [0.5.1](https://pypi.org/project/pylistenbrainz/) — Oct 2025 release; active
- uv Astral: [uv documentation](https://astral.sh/blog/uv) — Rust-based Python package manager; confirmed 2025 standard

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
