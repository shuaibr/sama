# Pitfalls Research

**Domain:** Personal music intelligence / ear-training / quantified-self system
**Researched:** 2026-06-10
**Confidence:** HIGH (Spotify API facts from official dev blog, Feb 2026; audio analysis limits from MIREX benchmarks; generation licenses from model cards; habit-loop patterns from peer-reviewed QS literature)

---

## Critical Pitfalls

### Pitfall 1: Spotify API Erosion — The Platform Is Still Moving Under You

**What goes wrong:**
You scope Phase 0 around the stable "history + library + playlist write-back" subset of the Spotify API, ship it, and then Spotify removes or restricts more endpoints mid-project. Between Nov 2024 and March 2026 the platform made four separate policy changes: (1) Nov 2024 — audio-features, audio-analysis, recommendations, related-artists deprecated for all new apps; (2) May 2025 — extended quota restricted to organizations with 250k MAU, individuals locked out of production-tier access; (3) Feb 2026 — dev-mode user cap dropped from 25 to 5, Premium account now required for the developer; (4) March 2026 — the Feb 2026 rules retroactively applied to all existing dev-mode client IDs. The trajectory is unmistakably toward further restriction.

**Why it happens:**
Spotify is protecting its recommendation IP and combating AI-aided scraping. Each tightening is announced with short notice (weeks, not months) and retroactively affects existing apps. Developers who built on what was "safe" endpoints found them gone or rate-capped with no migration path.

**How to avoid:**
- In Phase 0 enumerate only the five scopes you actually need: `user-read-recently-played`, `user-top-read`, `user-library-read`, `playlist-modify-private`, `user-read-email` (for token identity only). Request nothing else — scope creep cannot be undone without user re-auth.
- Wrap every Spotify call behind an `SpotifyHistoryAdapter` with a contract tested against recorded fixtures. When the API changes, you change one file, not scattered calls.
- Write a smoke-test that runs monthly (cron job on the CI heartbeat) and sends an alert if any used endpoint returns 403. This is the canary before a phase breaks.
- Keep a ListenBrainz mirror of every play event as it arrives. If Spotify revokes history access, ListenBrainz becomes the fallback corpus without data loss.
- Never use `user-read-playback-state` or `user-modify-playback-state` — these are the next candidates for restriction and are not needed.

**Warning signs:**
- Any Spotify developer blog post in the feed.
- Adapter integration tests begin returning `403` or `401 insufficient_scope`.
- Community posts about new endpoint deprecations (subscribe to r/spotify_dev or the developer changelog RSS).

**Phase to address:** Phase 0 (ADR must name the five permitted scopes and document the ListenBrainz mirror strategy before writing any code).

---

### Pitfall 2: PKCE Refresh Token Rotation — Silent Auth Breakage on a Weekly System

**What goes wrong:**
You implement Authorization Code with PKCE (the correct flow for personal apps). Every time you refresh the access token, Spotify issues a *new* refresh token and invalidates the old one. If you fail to persist the new refresh token — or if two processes race to refresh simultaneously — the old token is consumed and the new one is lost. The app silently loses Spotify auth. For a weekly-cycle system, the earliest you detect this is when the Monday job fails, seven days after the break.

**Why it happens:**
PKCE refresh token rotation is a security feature, not a bug. It is easy to miss when adapting tutorial code that shows basic Authorization Code flow (no rotation) and then deploying a cron-style personal app where no human is present to re-auth.

**How to avoid:**
- Store the token bundle (access token + refresh token + expiry) in a single atomic write to a local encrypted file (e.g., `keyring` or `.env`-encrypted store). Never write access and refresh tokens separately.
- Use a named mutex / file lock around token refresh so concurrent cron calls cannot race.
- After each token refresh, immediately verify the new token with a lightweight endpoint (`/me`) before proceeding with the actual job.
- Add a heartbeat alert: if the weekly Monday job cannot acquire a valid Spotify token, send an email/notification immediately rather than silently failing.

**Warning signs:**
- Spotify adapter calls start returning `401` on Monday jobs.
- Refresh token file was last modified earlier than the previous week.
- The `/me` smoke-test in the heartbeat job fails.

**Phase to address:** Phase 0 / Phase 1 (implement before the first automated weekly cycle runs).

---

### Pitfall 3: Copyright Trap — Fetching or Storing Full Lyric Texts

**What goes wrong:**
You query a lyrics API (Genius, Musixmatch, LyricsOVH) for a multilingual song, store the full text in SQLite for downstream processing, and unknowingly accumulate a database of copyrighted lyric text. Even for personal use, storing redistributable lyric text from an API that prohibits commercial/derivative use in its ToS creates legal exposure. The specific risk is that lyrics are *separately copyrighted works* — distinct from the recording — held by music publishers (Sony Music Publishing, Universal Music Publishing) who actively enforce. The second trap: even well-intentioned "derived features" can be challenged if the extraction pipeline stores intermediate full-text.

**Why it happens:**
The pipeline logic is: fetch text → extract feature → store feature. Developers forget to delete the intermediate text. The full text sits in `raw_lyrics` column "temporarily."

**How to avoid:**
- Never call a lyrics API that returns full text if the ToS prohibits storage. Use only the Musixmatch Lyrics API with the `f_lyrics_language` filter and the snippet endpoint (max 30% of lyrics allowed under Musixmatch's developer tier).
- For the multilingual analysis pipeline: fetch → extract derived features (rhyme density, syllable count, structural diagram, theme keywords) → immediately discard the raw text. The pipeline must not write raw text to any durable store — enforce this with a unit test that queries the `lyrics_raw` column after any analysis run and asserts it is NULL or absent.
- For Arabic/Urdu/Hindi: use the Genius API only for fragment confirmation under fair-dealing (display to *yourself* in the UI is defensible; batch storage is not). Quote ≤ 4 lines per track in any generated report.
- The SPEC constraint "never storing full lyric texts" must be a hard schema constraint in Phase 3, not an implementation convention.

**Warning signs:**
- Any column named `lyrics_text`, `lyrics_raw`, `full_lyrics` in the schema.
- The analysis pipeline taking more than 2–3 seconds per track (suggests it is fetching more than it needs to).
- Unit tests that pass a full lyric string to analysis functions (the test itself demonstrates the wrong data contract).

**Phase to address:** Phase 3 (lyrics derived-features module). ADR in Phase 0 must document permitted data contract: *derived features only, intermediate text must not survive the function boundary*.

---

### Pitfall 4: librosa/Essentia Accuracy Overreach — Treating Estimates as Ground Truth

**What goes wrong:**
The `AudioFeatures` interface returns a tempo estimate and a key, and downstream code treats these as facts. In practice: librosa's tempo estimator produces half/double tempo errors (octave errors) for ~15–25% of tracks, especially at extremes (<70 BPM or >160 BPM). Key detection using chroma-based methods (librosa's default, Essentia's ACE) achieves 75–85% accuracy on *Western tonal music* in MIREX benchmarks — and this project explicitly targets Arabic (maqam), Bollywood, and Celtic modal traditions where the algorithms were *not trained*. On non-Western music MIREX scores have *declined* over time even as Western-music scores improved. Maqam and raga operate outside equal temperament; chroma features assume equal temperament. Key detection on `tarab` recordings can be effectively meaningless.

**Why it happens:**
Library documentation presents algorithms without accuracy caveats. The `librosa.beat.tempo()` return value looks authoritative. Developers pipe the output into a display or a model weight without confidence intervals.

**How to avoid:**
- Wrap every librosa/Essentia estimate in a `FeatureEstimate(value, confidence, method)` object. Never return a bare float for tempo or key.
- For tempo: use `librosa.beat.beat_track(units='time')` and compute standard deviation of inter-beat intervals. If IBI std-dev > 15%, flag the estimate as LOW confidence (variable-tempo track; this includes much of tarab and free-form blues).
- For key on non-Western tracks: tag the track's genre before analysis. If genre is in `{arabic, bollywood, celtic, maqam, raga}`, skip key detection entirely and surface "indeterminate (modal tradition)" rather than a wrong Western key.
- Never feed a low-confidence key estimate into the taste profile as a feature weight. Use it only for display.
- For stem-level analysis (production layering): librosa's spectral centroid and HPCP features on a mixed recording are an approximation of the mix, not per-instrument analysis. Label these as "mix-level features" in all outputs. If per-instrument analysis is needed, require source separation (Demucs v4) first — but do not promise stem-level accuracy without it.

**Warning signs:**
- Taste profile showing 80%+ of tracks in the same key (algorithm is stuck).
- Tempo estimates for known 90 BPM hip hop tracks returning 45 or 180 BPM.
- Any Arabic or Bollywood track returning a confident `C major` or `A minor` key.
- User notices the "key" displayed does not match what they hear on a familiar track.

**Phase to address:** Phase 2 (local analysis engine). The `AudioFeatures` interface ADR must include confidence propagation as a first-class concern.

---

### Pitfall 5: Stem Analysis Overreach Without Source Separation

**What goes wrong:**
The SPEC names "production/layering: stem-level frequency mapping" as an analysis dimension. Without source separation (Demucs), librosa operates on the full mix. Claims about "bass presence," "percussion density," or "string arrangement width" derived from FFT analysis of the mixed signal conflate every instrument. The system may generate training prompts based on features it cannot actually measure at stem level — teaching the operator wrong things about what they hear.

**Why it happens:**
The architectural intent (stem-level) and the technical implementation (full-mix spectral analysis) are written in the same layer without a clear separator. Developers use "layering features" as a label without qualifying that they are mix-level approximations.

**How to avoid:**
- In the `AudioFeatures` interface, distinguish `MixLevelFeatures` (available on any audio) from `StemLevelFeatures` (requires source separation). The type system enforces this distinction.
- Phase 2 ships only `MixLevelFeatures`. Mark stem-level analysis as deferred to an explicit sub-phase.
- If Demucs is added: compute on-demand only for tracks the operator has flagged for deep analysis (not batch). Demucs on a 4-minute track at 44kHz takes 20–40 seconds on CPU; batch processing 15 tracks per week would be 5–10 minutes — acceptable, but budget for it explicitly.
- Never display "bass: heavy, strings: light" if these are derived from full-mix spectral centroid splits. Display "low-frequency energy: 42%" and let the operator interpret.

**Warning signs:**
- Analysis reports saying "guitar prominent" on tracks without guitar, or "drums minimal" on heavy percussion tracks.
- The operator noting that analysis descriptions don't match what they hear.
- `AudioFeatures` functions that return per-instrument labels without any reference to Demucs or source separation in the code path.

**Phase to address:** Phase 2, with a clear TODO comment and ADR entry that stem-level analysis requires Demucs and is out of scope until explicitly unlocked.

---

### Pitfall 6: Taste Model Confirmation Bias — The System Curates What It Predicts

**What goes wrong:**
The prediction-before-listen scoring loop (system predicts like/dislike, operator rates, profile updates) is the primary feedback mechanism. If the discovery digest is generated from the same model that scores predictions, you get a closed loop: the model recommends what it already believes the operator will like, the operator confirms (or not), and the model grows more confident in its prior — never learning about the operator's edge preferences or genre-boundary reactions. The cross-genre bridges (Celtic ↔ blues, Bollywood ↔ R&B) that are the project's differentiator are systematically suppressed because they score low on prediction confidence at first.

**Why it happens:**
"Predicts well" is optimized. "Challenges the model" is not. Discovery diversity is sacrificed for prediction accuracy.

**How to avoid:**
- Hard-code a diversity constraint in the weekly digest generator: ≥3 of the 15 tracks must come from genres where the model's prediction confidence is LOW (below a configurable threshold). Call this the "probe" quota. These tracks are explicitly labeled `{hypothesis: "stretch", expected_response: "uncertain"}`.
- Track prediction accuracy separately by genre. If Arabic music tracks are always LOW confidence and always skipped, that's signal the model is not learning about that genre — increase the probe quota for it.
- The prediction score must be computed and logged before the operator sees the track, stored immutably. Do not allow the model to retroactively revise predictions. The evidence trail must show the original prediction.
- Monthly: run a "coverage audit" — what percentage of the nine expansion genres appeared in the last four digests? If any genre has <10% representation, force-inject it.

**Warning signs:**
- Prediction accuracy climbing toward 90%+ in the first 4 weeks (the model is playing it safe, not learning).
- The weekly digest looking like variations on the same 2–3 genres week over week.
- Cross-genre bridge log empty after 6 weeks.
- Operator reports feeling bored by the digest.

**Phase to address:** Phase 3 (prediction + curriculum). The diversity constraint must be spec'd before the discovery generator is built, not added as a patch.

---

### Pitfall 7: Habit-Loop Collapse — Over-Instrumentation Killing the Weekly Cycle

**What goes wrong:**
The system works technically but the operator stops using it. Quantified-self research shows the average tracker is used for 11 days before abandonment; the primary driver is when the tracking overhead exceeds the perceived return. For this project the kill criterion is 4 consecutive skipped weekly cycles. The pattern: Phase 1 ships a minimal loop (feasible, quick). Phase 2–3 add analysis, blind tests, training-focus generators, lyrics modules, prediction scoring. Each addition is reasonable individually. By Phase 3 the weekly ritual requires: fill response form for 15 tracks, take blind test, review ear report, check prediction accuracy, generate next week's focus, listen to 5 hours of music with prompts. The system becomes a second job.

**Why it happens:**
Each feature is built in isolation and tested in isolation. Nobody measures the total operator time budget across all features in combination. The SPEC specifies ≤15 tracks/week and ≤5 hours, but does not specify the *interaction overhead per week*.

**How to avoid:**
- Define and enforce a "weekly overhead budget": operator interaction with the system (not listening) must not exceed 20 minutes/week. Measure this explicitly. If a feature can only be used if the operator spends > 5 minutes, flag it for simplification or removal before shipping.
- The 30-second response form must be literally 30 seconds — benchmark it. Three fields maximum: feel score (1–5), standout layer (free text, optional), would-replay (yes/no). Any more fields require an ADR to justify.
- The blind test must auto-terminate after 10 minutes elapsed or 10 questions answered, whichever comes first. No open-ended sessions.
- Build the kill-criterion detector in Phase 1, not later: if no response data arrives for 7 days, the Monday job sends a simplification prompt ("The system noticed you haven't responded in a week — should I reduce the digest to 5 tracks?").
- When the 4-consecutive-skip criterion fires, the system should *automatically reduce scope* (not just alert): drop to 5 tracks/week, drop blind tests, drop lyrics module. Recovery mode, not shutdown mode.

**Warning signs:**
- Response form completion rate falling below 50% (operator is skipping some tracks).
- Blind test scores going stale (last test > 14 days ago).
- "Would you like to skip this week?" prompts being accepted 2 weeks running.
- Operator modifying config to disable modules they find burdensome.

**Phase to address:** Phase 1 must instrument the weekly overhead measurement before Phase 2 adds complexity. The simplification trigger must be implemented in Phase 1.

---

### Pitfall 8: Generation Model License and Hardware Traps

**What goes wrong:**
MusicGen (Meta, CC-BY-NC 4.0) is used for sound sketches. Two traps: (1) **License**: CC-BY-NC means the model weights are non-commercial, but the project notes "Phase 5+ optional monetization." If you publish analysis essays or playlists later, any workflow that touched MusicGen outputs in the production pipeline — even indirectly — may be contested as derivative. (2) **Hardware**: MusicGen Stereo requires ≥12 GB VRAM. On a machine with 8 GB VRAM (or CPU-only), generation is either impossible or takes 10–20 minutes per 30-second sketch, making Phase 4 effectively unusable.

**Why it happens:**
Developers assume CC-BY-NC personal-use sketches are clearly separated from commercial output. The separation is valid in principle but dissolves if sketches inform analysis essays that generate revenue. Hardware requirements are not benchmarked until Phase 4 is being built.

**How to avoid:**
- **License**: Establish the sandboxing contract in Phase 0 ADR: generated audio lives only in `sketches/` directory, is never committed to git, is never referenced in any report or essay, and the directory is `.gitignored`. If Phase 5 commercial use starts, the sketches workflow must switch to a model with a commercial-permissive license *before* the first monetized piece is published.
- Prefer Stable Audio Open (Stability AI Community License, allows commercial use below a revenue threshold) for any sketch that might influence commercial work. Decision: use MusicGen for study-only personal sketches, Stable Audio Open when there is any commercial ambiguity.
- **Hardware**: Benchmark the operator's hardware in Phase 0. If VRAM < 12 GB, plan for `musicgen-small` (4–6 GB VRAM) or CPU-mode with a time budget. Document this in the ADR. If the machine cannot run any local generation model, Phase 4 is blocked — surface this before Phase 4 begins, not during.
- MusicGen's `musicgen-small` produces noticeably lower quality than `musicgen-large` (8 B vs 3.3 B parameters); if study value requires high-quality output, CPU generation time must be acceptable or different hardware must be provisioned.

**Warning signs:**
- Sketches directory contains files named by track or artist (suggests they are being used as references, not study artifacts).
- VRAM OOM errors during Phase 4 development.
- ADR for Phase 4 does not mention which model variant will be used or the hardware requirements.
- Any sketch file being mentioned in an ear report or essay.

**Phase to address:** Phase 0 (license ADR + hardware benchmark). Phase 4 must not start without the hardware check passing.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Calling Spotify directly in business logic (no adapter) | Faster initial code | Every API change breaks multiple files; cannot mock for tests | Never — the adapter is a Phase 0 requirement |
| Storing raw lyric text "temporarily" | Simpler pipeline | Legal exposure; schema drift when "temporary" column persists for months | Never — enforce no-storage at the function boundary |
| Using `librosa.beat.tempo()` without confidence wrapping | One-line result | Downstream code treats estimates as facts; wrong keys/tempos enter the taste profile | Never — `FeatureEstimate` type must be used from day one |
| Skipping ListenBrainz mirror | Less infra complexity | Spotify history loss risk is permanent; 12 weeks of listens are unrecoverable | Never — mirror from Phase 1 Day 1 |
| Single monolithic taste profile update function | Simple to read | Cannot explain which listens moved which dimensions; evidence trail requirement violated | Never — evidence trail is a hard spec constraint |
| Blind tests without time limit | Flexibility for operator | Sessions drag; habit-loop overhead balloons; operator avoids them | Never — hard cap at 10 minutes or 10 questions |
| Adding Phase 5 monetization before 12-week baseline | Revenue sooner | Distorts the loop: system optimizes for newsletter engagement, not operator ear development | Never — the SPEC is explicit: skill-first, money later |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Spotify OAuth (PKCE) | Forgetting that each token refresh returns a *new* refresh token; old token is immediately invalid | Atomic store of the full token bundle; mutex on refresh calls; immediate `/me` verification after refresh |
| Spotify API rate limits | Not handling `429 Retry-After` — retrying immediately causes exponential rate-limit punishment | Parse the `Retry-After` header; implement exponential backoff with jitter; rate-limit at the adapter layer |
| Spotify dev-mode user cap (5 users, Feb 2026) | Building a second user into the system (e.g., a test account) and hitting the 5-user cap | Keep to 1 user (the operator) plus 1 test account = 2 users. Never invite more. |
| Spotify `user-read-recently-played` pagination | This endpoint returns only the last 50 plays (no full history); developers assume it covers all history | Call on a schedule (at minimum daily) and persist each batch; never assume a single call captures everything |
| Genius / Musixmatch lyrics API | Fetching and storing full lyric text to "analyze later" | Extract derived features in the same function; discard raw text before the function returns; assert in test |
| librosa on non-Western audio | Passing maqam or raga recordings to `librosa.key_estimate` without genre gating | Check genre tag before dispatching; skip key detection for modal/microtonal traditions |
| MusicGen via `audiocraft` | Installing `audiocraft` without pinning the version — the library moves fast and breaks the API | Pin `audiocraft==1.3.0` (or latest stable at Phase 4 time) in `requirements.txt`; add a smoketest |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Running Demucs source separation on all 15 weekly tracks | Weekly job takes 45–90 minutes; operator can't run it on a laptop | Demucs is opt-in per track, not default. Batch only on explicit deep-analysis requests | Any batch > 3–4 tracks on CPU hardware |
| Loading a full librosa analysis pipeline for metadata-only lookups | Simple "get track name" call loads 200 MB of audio | Separate the `MetadataAdapter` from the `AudioAnalysisAdapter`; never import librosa in metadata code paths | From first use — import cost is paid at import time |
| SQLite without WAL mode on concurrent cron jobs | Monday job and Wednesday job write simultaneously; database locked errors | Enable `PRAGMA journal_mode=WAL` at connection time; all writers use the same connection pool | When two scheduled jobs overlap (likely from Phase 2 onward) |
| LLM calls for every track in the weekly digest | 15 tracks × LLM call = cost spike; weekly budget blown on digest generation | Use cheapest adequate model (gemini-flash or equivalent) for bulk extraction; frontier model only for synthesis | From Phase 1 — budget cap must be in config before first LLM call |
| MusicGen `musicgen-large` on <12 GB VRAM machine | OOM crash; partial audio files in `sketches/` | Detect VRAM at startup; auto-select `musicgen-small` if VRAM < 12 GB; document the quality tradeoff | Phase 4 first run |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Spotify client secret in code or `.env` committed to git | Account takeover; Spotify can revoke and invalidate your app | Use `python-dotenv` with `.env` in `.gitignore`; add a pre-commit hook that blocks `.env` commits; use OS keyring for the refresh token |
| LLM API key in config.yaml | Key exfiltration; unexpected billing | All API keys via environment variables only; `config.yaml` contains only non-secret config (model names, budget limits); enforced by schema validation at startup |
| Storing taste profile in plaintext SQLite with listening history | Personal music consumption data is sensitive; backup sync services may upload it | SQLite file outside the git-tracked directory; `.gitignore` the database; separate `git export` of anonymized digest for version history |
| Generated sketches accidentally committed | Model license violation (MusicGen CC-BY-NC weights produce non-commercial output; committing to a public repo could be construed as redistribution) | `sketches/` in `.gitignore` from day one; CI check that `sketches/` contains no audio files in the tracked tree |

---

## UX Pitfalls

| Pitfall | Operator Impact | Better Approach |
|---------|-----------------|-----------------|
| Discovery digest delivered as a raw list of track links | Operator must open 15 tabs to preview; response rate drops | Each track in the digest includes: Spotify URI (one-click open), 2-sentence hypothesis, 2 listening prompts, 1 anchor comparison. Formatted for reading, not just links. |
| Blind test results reported as a raw score without context | "Score: 6/10" is meaningless; no learning | Report each result as: "You identified 4/6 instrument layers correctly. Missed: oud (confused with guitar). This week's focus: oud vs guitar timbre." |
| Ear report delivered Monday morning as a wall of text | Operator skips it; the feedback loop breaks | Ear report: 3 bullet points max at the top (one actionable focus, one blind-test insight, one cross-genre bridge noticed). Details follow but are not required reading. |
| Profile drift shown as feature weights (numbers) | Operator cannot relate 0.73 "rhythmic complexity" to their experience | Translate every profile change to a natural-language statement: "Your responses show you respond more strongly to syncopated rhythms this week than last month." |
| Weekly cycle running even when operator has not engaged | System sends digests to a disengaged operator; inbox noise; feel of obligation | If no response data in 7 days, skip the next digest and send a single "should I continue?" message instead. Respect the kill criterion actively. |

---

## "Looks Done But Isn't" Checklist

- [ ] **Spotify adapter:** Has integration tests against recorded fixtures, not just mocked responses — verify the fixture captures the actual paginated response shape of `user-read-recently-played`
- [ ] **Token store:** Verify the refresh token is updated atomically after every `/token` refresh call (test by revoking and re-authorizing, then checking the stored token is the new one)
- [ ] **Lyrics pipeline:** Assert that no `lyrics_text` or equivalent column contains non-NULL values after any analysis run — add this as a post-migration CI check
- [ ] **`AudioFeatures` interface:** Verify every returned value has a confidence field — grep for bare float returns from tempo/key functions
- [ ] **Kill criterion detector:** The weekly job must log a "skip event" when no responses arrive — verify the event is actually written and the simplification prompt is sent after 4 events
- [ ] **Prediction immutability:** Verify the `expected_response` field is written before any listen data is stored, and has no UPDATE path in the schema — check the migration for a missing `GENERATED ALWAYS` or application-level guard
- [ ] **LLM budget cap:** Verify the cap is enforced in code (not just in config) by writing a test that mocks 100 LLM calls and asserts an error after the budget threshold is crossed
- [ ] **MusicGen sandboxing:** Verify `sketches/` is in `.gitignore` and no audio file appears in `git status` after a generation run
- [ ] **Diversity probe quota:** Verify the weekly digest generator test includes a case where all candidate tracks are high-confidence, and asserts that ≥3 low-confidence "probe" tracks are still injected

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Spotify token breakage (PKCE rotation lost) | LOW | Re-run the OAuth authorization flow manually; the app prompts for re-auth; history gaps filled from ListenBrainz mirror |
| Spotify removes a used endpoint | MEDIUM | Update `SpotifyHistoryAdapter` to return empty list or stub data; switch to ListenBrainz as primary source; update ADR; no user-facing breakage if fallback was built in Phase 0 |
| Lyric text stored in database (copyright exposure) | LOW-MEDIUM | Delete the column via migration; add the no-storage assertion test; no legal action likely for personal non-distributed use, but clean up promptly |
| Wrong key/tempo in taste profile (librosa error) | MEDIUM | Add the `FeatureEstimate` confidence type; re-run analysis on flagged tracks with the corrected pipeline; manually review any taste profile weights derived from low-confidence features |
| Habit-loop collapse (4 skips fired) | HIGH | Execute the auto-reduce: drop to 5 tracks/week, disable blind tests and lyrics module; run in reduced mode for 4 weeks before re-evaluating; do not add features until the loop is stable again |
| MusicGen OOM on Phase 4 | LOW | Switch to `musicgen-small` in `config.yaml`; note quality reduction in ADR; no data loss |
| Taste model confirmation bias discovered (coverage audit shows genre starvation) | MEDIUM | Implement the probe quota retroactively; force-inject the starved genres for the next 4 weeks; re-examine if prediction accuracy was artificially inflated |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Spotify API scope creep | Phase 0 | ADR lists exactly 5 scopes; no others appear in any OAuth call |
| PKCE token rotation breakage | Phase 0 / Phase 1 | Integration test: simulate token refresh + verify new refresh token stored; heartbeat alert tested |
| Lyric text storage | Phase 0 (ADR) + Phase 3 (enforcement) | Migration has no `lyrics_text` column; post-run assertion test is green |
| librosa accuracy overreach | Phase 2 | Every `AudioFeatures` return includes confidence field; genre-gate test for Arabic/maqam tracks present |
| Stem analysis overreach | Phase 2 | `MixLevelFeatures` vs `StemLevelFeatures` type separation in code; no per-instrument claims without Demucs |
| Taste model confirmation bias | Phase 3 | Probe quota enforced; coverage audit implemented; prediction immutability test present |
| Habit-loop collapse / over-instrumentation | Phase 1 (overhead measurement) | Weekly overhead benchmark < 20 min; kill-criterion detector writes skip events; simplification trigger fires in test |
| MusicGen license / hardware traps | Phase 0 (ADR) + Phase 4 (benchmark) | `sketches/` gitignored; hardware check at Phase 4 startup; no sketch referenced in any report |
| OAuth / rate limit mishandling | Phase 1 | `429 Retry-After` handled with jitter; rate-limit test present in adapter suite |
| Echo-chamber feedback loop | Phase 3 | Coverage audit runs monthly; diversity constraint in digest generator is tested |

---

## Sources

- [Spotify: Introducing some changes to our Web API (Nov 2024)](https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api)
- [Spotify: Update on Developer Access and Platform Security (Feb 2026)](https://developer.spotify.com/blog/2026-02-06-update-on-developer-access-and-platform-security)
- [TechCrunch: Spotify changes developer mode API to require premium accounts, limits test users](https://techcrunch.com/2026/02/06/spotify-changes-developer-mode-api-to-require-premium-accounts-limits-test-users/)
- [Spotify: Updating the Criteria for Web API Extended Access (Apr 2025)](https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access)
- [Spotify Community: PKCE refresh token rotation behavior](https://community.spotify.com/t5/Spotify-for-Developers/Why-am-I-getting-a-new-refresh_token-when-I-request-a-refreshed/td-p/4989661)
- [MIREX 2021: Audio Key Detection benchmark results](https://www.music-ir.org/mirex/wiki/2021:Audio_Key_Detection)
- [Improving Essentia ACE algorithm — MIR blog, key detection accuracy analysis](https://musicinformationretrieval.wordpress.com/2017/06/21/improving-essentia-ace-algorithms/)
- [librosa: Octave error in beat_track / tempo](https://groups.google.com/g/librosa/c/A8Tvl3Xy0Os)
- [Bias and Feedback Loops in Music Recommendation (Semantic Scholar)](https://www.semanticscholar.org/paper/Bias-and-Feedback-Loops-in-Music-Recommendation:-on-Knees-Ferraro/f93eea90ecbd043c8d54249a1eced4024b97cb7e)
- [Understanding Echo Chambers in Recommender Systems (IJACSA 2025)](https://thesai.org/Downloads/Volume16No10/Paper_71-Understanding_Echo_Chambers_in_Recommender_Systems.pdf)
- [MusicGen model card — CC-BY-NC 4.0 license (HuggingFace)](https://huggingface.co/facebook/musicgen-large)
- [Stable Audio Open 1.0 model overview](https://openlaboratory.ai/models/stable-audio-open-1)
- [Digital self-tracking, habits and the myth of discontinuance (Sage Journals, 2024)](https://journals.sagepub.com/doi/abs/10.1177/14614448221083992)
- [Wikilegal: Fair use protections for posting music lyric excerpts (Wikimedia)](https://meta.wikimedia.org/wiki/Wikilegal/Fair_use_protections_for_posting_music_lyric_excerpts_and_translations)
- [FreqBlog: Spotify Audio Features Is Dead — alternatives (2026)](https://freqblog.com/blog/spotify-audio-features-replacement-2026/)

---
*Pitfalls research for: personal music intelligence / ear-training system (Sonic A&R)*
*Researched: 2026-06-10*
