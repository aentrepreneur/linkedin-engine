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

## [0.5.0] - 2026-07-15

### Added
- **FASE 8: Automation Runner**
  - `engine/runner.py`: Ciclo completo de 10 pasos (discover -> select -> variants -> format -> hashtags -> image -> publish -> log)
  - `run_cycle()`: Ejecuta un ciclo completo por slot (morning/evening)
  - `daily_review()`: Analisis diario con recomendaciones basadas en analytics
  - `get_recommendations()`: Recomendaciones de frecuencia, pilar y variante optimos
  - `check_schedule()`: Estado de slots del dia
  - `_select_pillar()`: Deteccion automatica de pilar por keywords en el topic
  - `_get_viral_lift()`: Viral lift por tipo de variante

- **FASE 9: Hashtag Selection Engine**
  - `engine/hashtags.py`: Motor de seleccion de hashtags por pilar, variante y geo
  - `config/hashtags.yaml`: Pools por pilar (broad/mid/niche), variante y geo (LATAM/Guatemala)
  - `build_hashtags()`: Combina pillar + variant + geo, deduplica, limita a max_per_post
  - `append_hashtags()`: Agrega hashtags al final del post con formato correcto
  - `get_hashtags_by_pillar()`: Seleccion por pilar (principios, casos_estudio, etc.)
  - `get_hashtags_by_variant()`: Seleccion por tipo de post
  - `get_geo_hashtag()`: Hashtags geograficos para LATAM
  - `get_trending_hashtags()`: Trending 2026 (AgenticAI, GenAI, DevSecOps, etc.)
  - `get_hashtag_stats()`: Estadisticas de pools de hashtags

- **FASE 10: LinkedIn Image Upload (3-step flow)**
  - `engine/linkedin.py`: Reescrito con soporte completo para imagenes
  - `_register_upload()`: Step 1 - Registra upload y obtiene uploadUrl + asset URN
  - `_upload_image_binary()`: Step 2 - Sube bytes binarios al uploadUrl
  - `_publish_to_linkedin()`: Step 3 - Publica con shareMediaCategory: "IMAGE"
  - `_upload_and_publish()`: Orquesta los 3 pasos con fallback a text-only
  - `publish_post()`: Acepta image_path opcional, detecta imagenes automaticamente

- **FASE 11: Crontab + Schedule**
  - `config/schedule.yaml`: Slots (7AM/6PM Guatemala), crontab entries, review a 8AM
  - `scripts/cron_runner.sh`: Wrapper para crontab con logging a logs/cron_YYYYMMDD.log
  - Comandos: `run`, `review-daily`, `recommendations`, `schedule-info`

- **CLI Updates**
  - `le run --slot morning|evening [--dry-run]`: Ciclo completo (FASE 8)
  - `le review-daily`: Analisis diario (FASE 11)
  - `le recommendations`: Recomendaciones basadas en datos (FASE 11)
  - `le schedule-info`: Estado de programacion del dia (FASE 11)
  - `le hashtags [pillar] [variant]`: Ver/configurar hashtags (FASE 9)

- **Bitacora Enhancements**
  - `engine/bitacora.py`: Campos pillar, variant, hashtag_count
  - `get_daily_summary()`: Resumen diario de publicaciones

- **Pipeline Updates**
  - `engine/pipeline.py`: Integracion de hashtags, _detect_pillar(), _detect_variant()
  - Hashtags se agregan automaticamente en generate_single()

### Fixed
- `engine/runner.py`: bug - missing `import time` at module level
- `engine/runner.py`: bug - `build_hashtags` no importado (solo se importaba append_hashtags)
- `.env`: APIFY_API_TOKEN ahora incluye prefijo `apify_api_` (verificado 200 OK)

### Tests
- `tests/test_hashtags.py`: 26 tests para hashtag engine
- `tests/test_runner.py`: 8 tests para runner (dry-run, pillar detection, viral lifts)
- `tests/test_bitacora.py`: 20 tests para bitacora con nuevos campos
- `tests/test_linkedin_upload.py`: 9 tests para LinkedIn upload flow
- Total: 272 tests (desde 196)

### Research
- Hashtags: ViralBrain Q3 2026 (30,360 posts analizados)
- Estrategia: 3-5 hashtags, mix broad + niche + geo, SEO signals
- LinkedIn API: SSE transport deprecated (April 2026), nuevo endpoint mcp.apify.com
- Apify auth: Bearer token, OAuth, x402 (USDC), Skyfire (PAY tokens)
- Content optimization: Dwell time #1 signal, carousels 2.3x, comments 15x likes

## [0.4.0] - 2026-07-13

### Added
- **FASE 7: Multi-continental Scheduling**
  - `engine/timezones.py`: Timezone-aware scheduling for Guatemala, USA East/West, Europe
  - Peak hours detection per region
  - Optimal schedule generation across regions
  - Status monitoring for all regions

### Technical Details
- 4 regions configured with UTC offsets
- Peak hours: 7-9, 12-13, 18-20 (local time)
- Schedule optimization for maximum reach

## [0.3.0] - 2026-07-13

### Added
- **FASE 6: Analytics & Engagement Tracking**
  - `engine/analytics.py`: Track publications, engagement, and metrics
  - Period metrics aggregation (7-day evaluation)
  - Variant and pillar performance analysis
  - Frequency recommendation engine

- **FASE 5: Direct Publication Mode**
  - Updated `engine/scheduler.py` for direct publication (no approval)
  - Adaptive frequency based on engagement
  - 2 posts/day default with optimization

- **FASE 4: Auto-selection**
  - `engine/auto_select.py`: AI-powered variant scoring
  - Rule-based fallback when LLM unavailable
  - Automatic best variant selection

- **FASE 3: Variant Generation**
  - `engine/variants.py`: Generate 3 variants per topic
  - Thought Leadership (4.58x viral lift)
  - Storytelling (3.48x viral lift)
  - Hot Take (2.20x viral lift)

### Technical Details
- Hybrid scoring: LLM + rule-based
- Engagement tracking: impressions, likes, comments, shares, saves
- 7-day evaluation period for frequency optimization

## [0.2.0] - 2026-07-13

### Added
- **FASE 1: Content Discovery Pipeline**
  - `engine/feeds.py`: RSS feed fetcher with keyword filtering, freshness, and relevance scoring
  - `engine/youtube.py`: YouTube transcript extraction and viral potential calculation
  - `engine/analyzer.py`: Content relevance scoring and viral angle extraction
  - `config/feeds.yaml`: 6 RSS feeds configured (Krebs, HN, BleepingComputer, Dark Reading, AI News, MIT Tech Review)
  - `config/pillars.yaml`: 5 content pillars with keywords and viral potential

- **FASE 2: Hybrid Format**
  - `engine/formatter.py`: Hybrid format (line-by-line hooks + blocks body)
  - `config/format_rules.yaml`: Format configuration
  - Line length validation (40 chars hooks, 80 chars body)

- **Tests**
  - 196 tests across 8 modules (test_engine, test_feeds, test_youtube, test_analyzer, test_formatter, test_variants, test_auto_select, test_analytics, test_timezones)

### Technical Details
- Autocontenido: solo usa dependencias ya instaladas (httpx, xml, youtube_transcript_api, dateutil)
- Freshness filter: <7 days old (configurable)
- Circuit breakers en todos los modulos externos

## [0.1.0] - 2026-07-12

### Added
- **Core Pipeline**
  - `engine/pipeline.py`: Orchestrated content generation with watchdog
  - `engine/llm.py`: LLM router (OpenCodeZen > Groq > OpenRouter > OpenAI)
  - `engine/images.py`: Pollinations FLUX image generation
  - `engine/linkedin.py`: UGC Posts API publisher
  - `engine/review.py`: Confidentiality, voice, and format checks
  - `engine/scheduler.py`: Approval queue and rate limits
  - `engine/bitacora.py`: JSONL event log
  - `engine/circuit.py`: Circuit breaker and retry budget

- **Configuration**
  - `config/voice.yaml`: Tone, writing rules, anti-patterns
  - `config/weekly_rotation.yaml`: Schedule per day with algorithm optimization
  - `config/image_styles.yaml`: Image prompt templates
  - `config/linkedin.yaml`: API config and rate limits

- **Content**
  - `patterns/`: 6 structural patterns with templates
  - `seeds/`: Stories, memes, and tools

- **Tests**
  - `tests/test_engine.py`: 18 tests for review, circuit, bitacora, and scheduler

### Technical Details
- Accent auto-correction (40+ Spanish patterns)
- Dynamic emoji injection (content-aware)
- Hybrid format (line-by-line hooks + blocks body)
- Anti-AI detection rules

## [0.6.0] - 2026-07-15

### Added
- Hugging Face FLUX.1-dev image backend (free, ~1000 imgs/mo)
- News interest scoring with bitacora dedup (7 days)
- RSS: 5 feeds optimized, YouTube disabled

### Changed
- Image prompts: cybersecurity/AI abstract only, never post text
- Groq circuit recovery: 90s > 30s
- Images: 1024x1024 square

## [0.7.0] - 2026-07-15

### Added
- LLM hashtag generation: Groq > OpenRouter > OpenCodeZen > YAML fallback
- Viral trend discovery from RSS via LLM
- `engine/trends.py`: analyze headlines, score viral potential

### Changed
- `engine/hashtags.py`: rewritten with multi-provider chain
- `engine/runner.py`: step 0 = trend detection, LLM hashtags
- `config/hashtags.yaml`: fallback-only pools

### Fixed
- `variant_type` NameError in simplified runner

### Published
- `urn:li:share:7483530400087080961` (LLM hashtags: #ArtificialIntelligenceSecurity #CybersecurityIncidents #DataProtectionRegulation)

## [2.0.0] - 2026-08-04

### BREAKING — Strategic Pivot to Verified 2026 Patterns

Based on deep research of 25 top cybersecurity LinkedIn influencers, Forbes 2026
algorithm analysis, and ViralBrain real post data from Daniel Miessler, Troy Hunt,
Rachel Tobac. Entire content strategy rewritten around what ACTUALLY works in 2026.

### Research Sources
- Forbes "5 LinkedIn Content Moves Being Punished in 2026" (Jul 2026)
- Forbes "Why Human-Written Posts Are Outperforming AI" (Jun 2026)
- ViralBrain analysis of Daniel Miessler (31K followers, 73.6% engagement)
- Public profiles: Troy Hunt (69K), Rachel Tobac (44K)
- Cherry Lane Media 25 cybersecurity influencer list
- LinkedIn 360Brew algorithm update (May 2026)

### Key Findings Implemented
- Bullet points + decorative emojis = AI signal → suppressed
- Generic openers + smooth transitions + symmetric structure → AI detection
- Outbound links: -60% reach, even in comments
- Comment baiting: actively suppressed by 360Brew
- Daily same-format posting: throttled in 2 weeks
- 3x/week with varied formats: steady reach lift
- Saves = 5x reach of likes. Dwell time is king.

### Changed — Core Content System
- **engine/variants.py** — Complete rewrite (770 lines)
  - Replaced 5 legacy variants (tres_voces, hot_take, antes_despues, historia_personal, dato_impactante)
  - 4 new 2026-verified archetypes:
    1. **tesis_contrariana** (800-2000 chars): Reframe popular belief, paragraph style, no bullets
    2. **breach_breakdown** (350-1000 chars): Specific CVE/breach, factual, fast (Troy Hunt pattern)
    3. **war_story** (500-1500 chars): Forbes recipe — 3-3-1 format, personal anecdote
    4. **deep_dive** (1200-3000 chars): Educational breakdown (Rachel Tobac pattern)
  - Anti-AI-detection rules in system prompts (17 rules + format-specific instructions)
  - Auto-paragraph detection when LLM omits \n\n
  - Format rotation to prevent same archetype consecutive days
  - First-person, opinion-driven content prioritized
  - Legacy compatibility aliases preserved

- **config/voice.yaml** — Rewritten for 2026 anti-AI
  - Removed all Make.com 3-paragraph structure (🚨🤖💬 markers)
  - No decorative emojis (max 1, contextual only)
  - No bullet points, no smooth transitions
  - 1 hashtag per post (keywords in body > hashtags for SEO)
  - Voice: "professional but human — like talking to a colleague over coffee"
  - Forbes-verified anti-patterns and forbidden phrases
  - CTA rules: never comment baiting, "worth a look" or nothing

- **config/formats.yaml** — 4 new archetype definitions with per-format constraints
- **config/format_rules.yaml** — Per-archetype min/max chars, paragraph counts, emoji limits
- **config/hashtags.yaml** — Default 1 hashtag (was 3), smaller fallback pools

### Changed — Pipeline & Engine
- **engine/pipeline.py** — `_build_system_prompt()` rewritten for 2026
  - Removed hook/body/CTA structure, removed emoji injection
  - `_suggest_emojis()` disabled (returns empty list)
  - Day config simplified (no more meme/tool review/reflection per day)
  - Hashtag default: 3 → 1

- **engine/quality.py** — Relaxed thresholds
  - Min chars: 200 → 150 (varied lengths OK in 2026)
  - Disconnected paragraphs: issue → fix (punchy standalone = good)
  - Format-specific checks for tesis, breach, deep_dive
  - Default format: tres_voces → war_story

- **engine/review.py** — Removed anti-2026 checks
  - **Removed**: "No CTA detected in last 2 lines" penalty (comment baiting is bad now)
  - **Removed**: 3-emoji threshold (max 5 now, warn only)
  - FORMAT_LIMITS: all days 350-3000 chars (was per-day with rigid 900/600/1000 requirements)

- **engine/runner.py** — Forecast updated for 4 archetypes
  - _FORECAST_SYSTEM: 5 formats → 4, new English descriptions
  - Alias map includes legacy→new mappings (tres_voces→tesis_contrariana, etc.)
  - Default format: tres_voces → war_story

- **engine/hashtags.py** — Variant names updated to new archetype keys

### Fixed — Tests
- **tests/test_quality.py**: Updated for 2026 format names + disconnected paragraph behavior
- **tests/test_variants.py**: Updated Spanish/mayuscula checks for new prompts
- **tests/test_hashtags.py**: Updated counts for 1-hashtag strategy

### Verified — End-to-End Test
```
le gen --day MAR → 2.1s via Groq, 299 chars
Post: "La integración de MISP con SIEM reduce el tiempo de detección..."
No emojis, no bullets, professional tone, real data
1 review warning: slightly short (299 of 350 min)
```

### Provider Chain (verified same session)
```
1. Groq llama-3.3-70b-versatile  ✅ free
2. Groq llama-3.1-8b-instant     ✅ free
3. OpenRouter llama-3.3-70b      ✅ créditos
4. OpenRouter deepseek-chat      ✅ créditos
5. OpenAI gpt-4o                 ✅ paid
6. OpenCode deepseek-v4-pro      ✅
7. OpenCode glm-5.2              ✅
❌ Cerebras — removed (no free tier)
```

## [0.9.0] - 2026-08-04

### Added
- **Multi-provider LLM router with Cerebras + OpenRouter** — `engine/llm.py`
  - Added **Cerebras** provider (requires payment — all models return 402)
    - Models: `gpt-oss-120b`, `gemma-4-31b`
    - `CEREBRAS_API_KEY` env var, `csk_...` key format
    - Dedicated circuit breaker: `llm:cerebras` + `llm:cerebras:scout`
  - Added **OpenRouter** provider (verified working, requires $10+ credits)
    - Models: `meta-llama/llama-3.3-70b-instruct`, `deepseek/deepseek-chat`
    - `OPENROUTER_API_KEY` env var, `sk-or-v1_...` key format
    - Auto-adds `HTTP-Referer` and `X-Title` headers for OpenRouter compatibility
    - Dedicated circuit breaker: `llm:openrouter` + `llm:openrouter:scout`
  - Updated `_resolve_provider()` to handle `cerebras:` and `openrouter:` prefixes
  - Updated `_build_providers()` priority chain:
    1. Groq llama-3.3-70b-versatile ✅ (free, verified)
    2. Groq llama-3.1-8b-instant ✅ (free, verified)
    3. Cerebras gpt-oss-120b ❌ (402 — requires payment)
    4. Cerebras gemma-4-31b ❌ (402 — requires payment)
    5. OpenRouter llama-3.3-70b ✅ (verified with credits)
    6. OpenRouter deepseek-chat ✅ (verified with credits)
    7. OpenAI gpt-4o-mini ✅ (paid, verified)
    8. OpenCodeZen big-pickle ⚠️ (free, often empty)

- **Updated `.env.example`** with all 3 providers + signup links + key format

### Provider Status (verified 2026-08-04)
| Provider | Status | Cost | Tok/Day | Speed | Model Quality (ES) |
|----------|--------|------|---------|-------|-------------------|
| Groq | ✅ | Free | 14,400 RPD | 300-800 tok/s | Good (Llama 3.3 70B) |
| Cerebras | ❌ | Paid | N/A | 2,000+ tok/s | Best (GPT-OSS 120B) |
| OpenRouter | ✅ | Credits | 1,000 RPD | Variable | Excellent (Llama 3.3 70B) |
| OpenAI | ✅ | Paid | Pay-as-go | 100-200 tok/s | Excellent (GPT-4o-mini) |
| OpenCodeZen | ⚠️ | Free | Unlimited | Slow | Unknown |

### End-to-end test (2026-08-04)
- `le gen --day MAR`: post generated in 1.7s via Groq
- Content: MISP/SIEM integration, 403 chars, pillar=herramientas
- Quality: clean Spanish, no conjugation errors, no invented stats
- Review flags: too short (403 vs 900 min), no CTA in last 2 lines

## [0.8.0] - 2026-08-03

### Fixed
- **CRITICAL: `generate()` missing `model` parameter** — `engine/llm.py:221`
  - `generate()` now accepts `model: str | None = None` for explicit model selection
  - `generate_text()` wrapper also passes `model` parameter
  - Added `_resolve_provider()` to resolve 'provider:model_name' format
  - **Impact**: orthography LLM proofread, quality gate LLM scoring, and timing analysis were ALL silently failing
  - **Before**: TypeError → fallback to fast checks → score=70 with all sub-scores=0
  - **After**: LLM runs correctly, sub-scores calculated, real quality assessment

- **Quality gate sub-scores always zero when LLM fails** — `engine/quality.py:143-145`
  - Added `_compute_fast_scores()` that calculates real sub-scores from fast checks
  - Orthography: detects conjugation errors, missing accents, double words
  - Coherence: measures paragraph connectivity, duplicate words
  - Temporalidad: checks timing alignment with event status
  - Voz: detects robotic patterns, excessive jargon
  - **Before**: score=70, breakdown={ort:0, coh:0, tmp:0, voz:0}
  - **After**: score=88, breakdown={ort:19, coh:22, tmp:25, voz:25} (real values)

- **Invented stats validation too lenient** — `engine/validation.py:172-178`
  - Changed threshold from `> 1` to `>= 1` invented stats
  - Now blocks publication with ANY invented percentage not found in source
  - **Before**: 1 invented stat → warning only → published
  - **After**: 1 invented stat → blocks publication

### Changed
- **Temperature reduced from 0.8 to 0.6** — `engine/variants.py:897`
  - Lower temperature reduces hallucination and conjugation errors
  - Better Spanish grammar from LLM

- **Anti-hallucination rules added to system prompts** — `engine/variants.py:173-212`
  - Rules 13-16: explicit instructions against inventing conjugations
  - Examples of correct/incorrect verb forms
  - Prohibits duplicating words ("Los los")
  - Prohibits using infinitives as conjugated verbs

- **LLM conjugation error patterns** — `engine/orthography.py:236-265`
  - Added 21 new regex patterns for common LLM-generated errors
  - `convertidor` → `convertido`, `investigador` → `investigadores`
  - `mies` → `miles`, `inafectado` → `infectado`
  - `revelar` → `revela`, `ofrecer` → `ofrece`
  - `evolución` → `evoluciona` (when used as verb, not noun)
  - `Los los` → `Los` (duplicate word removal)
  - Applied as Layer 1b in orthography correction pipeline

- **Image prompts no longer generic** — `engine/images.py:35-39`
  - `LIFESTYLE_STYLE_EN`: removed "empty desk with open laptop showing dashboard"
  - Now: "professional cybersecurity analyst at work, dramatic moody lighting, multiple monitors"
  - All pillar styles already had anti-generic rules, this was the last fallback
