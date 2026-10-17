<!-- ================================================================= -->
<!-- Development By Angel Esquivel (Automation) [LINKEDIN-ENGINE 2026] -->
<!-- ================================================================= -->
<!-- LINKEDIN-ENGINE - Motor de generacion de contenido y audiencia para LinkedIn -->

# Changelog

All notable changes to LinkedIn Engine will be documented in this file.

Format based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [2.4.0] - 2026-08-20

### Added — Engagement Scraper (Apify Integration)

Automated LinkedIn engagement tracking via Apify actors. Replaces the
zero-data gap: 42 posts published, 0 with real engagement metrics.

- **`engine/engagement.py`** (new module, 380+ lines)
  - `scrape_engagement(days, dry_run)` — main entry point. Loads published
    posts, calls Apify, matches by text similarity (3 levels: exact→fuzzy→skip)
    or by LinkedIn URL, writes to both analytics.py and weekly_report.py.
  - `_call_apify_actor()` — async pattern: `POST /runs` → poll → `GET
    /dataset/items`. Uses `alizarin_refrigerator-owner~linkedin-post-scraper`
    actor in `google_search` mode (no cookies, ~$0.04/post).
  - `_poll_run()` — polls Apify run status with 5s interval, max 180s.
  - `_match_posts()` — 3-level matching: Level 0 (URL exact), Level 1
    (normalized text exact), Level 2 (fuzzy ≥ 0.75 via SequenceMatcher).
    Deduplication prevents double-matching.
  - `_write_metrics()` — dual-write to `Analytics.update_engagement()`
    and `weekly_report.update_engagement()`.
  - `linkedin_url_from_urn()` — converts `urn:li:share:123` to
    `https://www.linkedin.com/feed/update/urn:li:share:123`.
  - `_profile_circuit` — circuit breaker (3 failures → 5-min recovery).

- **`engine/generate.py`** — new CLI command `le engagement-scrape`
  - `--days N` (default 7), `--dry-run` flag.
  - Table output with matched posts and engagement totals.

- **`config/linkedin.yaml`** — new `scraping` section with actor_id,
  profile_username, cost_per_post_usd.

### Known Limitation

Google indexes LinkedIn posts with 1–2 week delay. Posts < 2 weeks old
return 0 results from Google-search-based Apify actors. Mitigation:
URL-based matching (Level 0) will work once posts are indexed. Manual
updates via `le engagement <post_id>` remain available as fallback.

### Tests

- New: `tests/test_engagement.py` (24 tests)
  - TestLinkedinUrlFromUrn (3), TestNormalizeText (5), TestExtractMetrics (3),
    TestMatchPosts (6, including URL match), TestLoadPublishedPosts (4),
    TestScrapeEngagement (3).

### Changed — Project Cleanup & Optimization

Comprehensive audit and cleanup: removed dead code, fixed deps, updated docs.

- **Deleted dead modules** (TEST-ONLY): `engine/analyzer.py`, `engine/auto_select.py`,
  `engine/formatter.py` — 0 production callers, only used by their own test files.
- **Deleted dead tests**: `tests/test_analyzer.py`, `tests/test_auto_select.py`,
  `tests/test_formatter.py` (66 tests total).
- **Deleted dead configs**: `config/formats.yaml`, `config/schedule.yaml`
  (0 references in production code).
- **Deleted dead scripts**: `scripts/dryrun_v230.py`, `scripts/run_with_validation.py`
  (one-shot scripts superseded by runner.py).
- **Deleted engine backup files**: 18 `*.backup.*` files in `engine/`.
- **Removed dead functions**:
  - `circuit.with_circuit_breaker()` — 0 usages
  - `runner._inject_section_emojis()` — 0 usages (anti-pattern 2026)
  - `scheduler._next_available_slot()`, `_best_time_for_day()` — 0 usages
  - `trends.load_pattern_index()` — 0 usages
  - `hashtags.get_hashtags_by_pillar()`, `get_hashtags_by_variant()` — 0 usages
- **Fixed pyproject.toml**: version `0.1.0` → `2.4.0`, added missing deps
  (`python-dotenv`, `python-dateutil`), added `[spell]` optional extra for
  `pyspellchecker`.
- **Fixed engagement.py**: import `PipelineError` from `circuit` (was `pipeline`).
- **Rewritten README.md**: removed 8 phantom commands (FASE 1/6/7), updated
  architecture diagram, test counts, and all sections to match v2.4.0 reality.

## [2.3.0] - 2026-08-06

### Added — Editorial Layer + Quality Gates (Brief-Driven Content)

Root-cause diagnosis: posts read as generic one-paragraph news summaries
("…un párrafo sin justificación, sin referencia, contexto o lógica…") with a
single generic hashtag. The generator had no editorial thesis to honor and the
quality gate never checked structure. Fixed with a pre-generation editorial
brief that dictates hook/angle/CTA/source, plus blocking quality gates that
enforce it.

- **`engine/brief.py`** (new module)
  - `build_editorial_brief()` — one LLM call (temp 0.4) converts research +
    news into a JSON contract: `hook` (dato-based, never imperative),
    `angle` (paisaje: por qué importa ahora, para defensores/LATAM),
    `cta` (engagement específico), `source` (fuente real citable).
  - Anchored exclusively in `research_context + summary` — never invents.
    On any failure returns `{}` (graceful degradation, never breaks the cycle).
  - `_brief_circuit` (3 failures → 5-min recovery); `_sanitize_brief()`
    accepts dict or single-item list, drops non-string values; `brief_to_json()`.
  - Hooks for the anti-imperative rule (ViralBrain: imperativos = 0.02x lift).

- **`engine/quality.py`** — editorial quality gates (blocking)
  - `_editorial_checks()` — deterministic (regex/word-overlap, no LLM): hook
    presente, fuente citada por nombre, ángulo/paisaje desarrollado, CTA de
    cierre. A post missing any of these **blocks publication regardless of the
    LLM score** (previously score ≥70 always passed).
  - `quality_gate()` accepts `editorial_brief` and runs the gates as Step 1b.

- **`engine/variants.py`**
  - `generate_post()` accepts `editorial_brief`; `_render_brief_contract()`
    injects the brief as a binding contract in the system prompt (estructura
    Hook → Contexto → Paisaje → CTA obligatoria; hook imperativo prohibido).
  - Reads `model` from `llm_config` and forwards it to `generate_text()`.
  - `_truncate_all_paragraphs` default `max_sentences` 4 → **3** (2.2.0 había
    destrozado la estructura al recortar párrafos enteros).

- **`engine/llm.py`** — `MODEL_GENERATION = "opencode:deepseek-v4-pro"`,
  `MODEL_FAST = "groq:llama-3.3-70b-versatile"`; `generate()` already supported
  `model=` overrides via `_resolve_provider()`.

- **`engine/hashtags.py`** — content-anchored hashtags
  - `_extract_keywords()` — deterministic domain/vendor lexicon with
    word-boundary matching (no substring false positives), OT→`otsecurity`,
    specific-first ordering.
  - `generate_hashtags()`/`append_hashtags()` accept `research_context` and
    prime the LLM with extracted candidates; `_fallback_deterministic()`
    uses content keywords before the generic YAML pool.
  - `FORMAT_MAX_HASHTAGS` back to 3 in `.env` (matching `.env.example`).

- **`engine/runner.py`** — Step 1.6 editorial brief between research and
  generation; `generate_post` now receives `editorial_brief` +
  `llm_config={"model": MODEL_GENERATION}`; quality gate call passes the brief;
  hashtag step passes `research_context`.

### Tests

- New: `tests/test_brief.py` (8), `tests/test_quality_gates.py` (7),
  `tests/test_variants.py` brief-contract + model-forwarding (6),
  `tests/test_hashtags.py` keyword extraction + deterministic fallback (8).
- Full suite: **320 passed** (baseline 290) with the 2 pre-existing
  environmental failures (pyspellchecker absent, flaky timing LLM).

### Known / Operational

- `opencode:deepseek-v4-pro` returns **401 CreditsError (insufficient balance)**
  on the OpenCodeZen account — the chain degrades to groq 70B which works. The
  brief + generation still honor the model override once balance is added.

## [2.2.0] - 2026-08-06

### Added — Research Layer + Voice Layer (Input-Grounded Content)

Root-cause diagnosis: generated posts were generic ("Es fundamental que…")
because the LLM only received an RSS headline + summary as context. The engine
was producing **input-starved** content with no external references and no
authorial voice. Fixed at the source: richer, verifiable input.

- **`engine/research.py`** (new module)
  - External research over DuckDuckGo (HTML lite) + Reddit search JSON via
    `httpx` — **zero new dependencies** (Rule 5).
  - `research_topic()` runs both sources concurrently, consolidates snippets
    into a bounded `INVESTIGACION ADICIONAL` context block the LLM must ground on.
  - 1-hour in-memory cache per query (`cached: true` on hit), circuit breaker
    `research:http` (3 failures → 10-min recovery), graceful partial-failure:
    one source down degrades to the other; both down raises `PipelineError`.
  - `_clean_url()` resolves DuckDuckGo's `uddg` redirects; `_strip_tags()`
    removes HTML; `_normalize_query()` trims stopwords and caps at 8 tokens.
  - `clear_research_cache()` for tests.

- **`engine/voice.py`** (new module)
  - `load_voice_section()` assembles a bounded voice block (~2500 chars) from
    `config/voice_profile.md` (distilled) + `config/voice.yaml` rules, with
    `lru_cache` and budget clipping. This is the first time the voice rules
    reach the live generator (`_build_system_prompt` was a legacy alias).

- **`tools/distill_voice.py`** (new CLI)
  - One-shot distillation of the author's voice into `config/voice_profile.md`
    from `profile/sections/*` + `seeds/*.yaml`. Published posts are **excluded**
    by default (`--include-published` only for diagnosis) because they are the
    pre-fix output and would contaminate the voice sample. `--dry-run` previews
    the corpus. Output forced to Spanish with 2-4 sentence paragraphs.

- **`engine/variants.py`** — `generate_post()` accepts `research_context`
  (appended to `content_ref`) and injects the voice section into the system
  prompt on every generation attempt.

- **`engine/runner.py`** — new Step 1.5 (research) between timing extension and
  context assembly; `augmented_ref` = timing + research + news; validation now
  grounds on `research_context + summary`; `_run_research` is wrapped so a
  research failure degrades gracefully and never breaks the cycle.

### Fixed

- **Hallucination validation was under-grounded**: `validate_generated_post`
  used only the RSS summary as source, so real research context could have been
  flagged as fabricated. It now uses research + summary as the source ground.

## [2.1.0] - 2026-08-05

### Added — Anti-Hallucination System (Loop-Harness Engineering)

Detected a critical content risk: the LLM invented "NEXUS" (a non-existent
security architecture) in a generated VIE post that passed review with 0 issues.
Added a defense-in-depth layer so fabricated tools/products never reach LinkedIn.

- **`engine/hallucinations.py`** (new module)
  - Whitelists: `KNOWN_PRODUCTS` (MISP, SIEM, Wazuh, Splunk, TheHive, CrowdStrike…),
    `KNOWN_ORGS` (CISA, MITRE, Microsoft…), `KNOWN_GENERIC` (SOC, XDR, ZeroTrust…)
  - Heuristic patterns: product-intro regexes ("una herramienta como X", "la plataforma Xenith"),
    verb+name ("adopté NEXUS"), CamelCase + ALL-CAPS token scans
  - Two severity levels: **blocking** (invented named product → reject publication)
    and **warning** (suspicious proper noun → manual verification)
  - Source-grounded: any term present in the RSS/transcript source is exempt
  - Whitelist normalized to uppercase at load (fixes mixed-case entries like
    CrowdStrike/Mimikatz never matching)

- **`engine/review.py`** — `full_review()` now runs hallucination checks and maps
  blocking → CRITICAL / warnings → WARNING issues (covers both `gen` and `run` flows)

- **`engine/validation.py`** — `validate_generated_post()` runs `detect_hallucinations`
  and appends blocking issues to the validation result (blocks the runner's publish gate)

- **`engine/runner.py`** — new retry-feedback branch: when validation reports an
  invented product/tool, the runner sends a CORRECCION prompt to the LLM telling it
  to use real tools from source or generic terms instead

- **`engine/variants.py` + `engine/pipeline.py`** — prompt hardening
  - New rules 18-19 in `_BASE_RULES`: PROHIBIDO inventar productos/herramientas/
    arquitecturas; NUNCA inventes nombres en mayúsculas tipo "NEXUS"
  - Anti-invention instruction added to ALL prompt builders: contrarian, breach,
    war_story, deep_dive, system prompt, generation prompt

- **`tests/test_hallucinations.py`** (new) — 16 tests: invented products blocked,
  real tools/orgs/mixed-case whitelist clean, source-grounded terms exempt, validator
  integration. Full suite: 264 passed, 1 pre-existing flaky timing test (LLM-call), 1 skipped.


## v<major> - Releases anteriores archivadas (2026-10)


<!-- End Development By Angel Esquivel (Automation) [LINKEDIN-ENGINE 2026] -->
