# Technical Documentation — LinkedIn Engine v2.4.0

## Architecture

```
engine/
├── pipeline.py        Pipeline completo: discovery → generation (1175 loc)
├── runner.py          Orquestador: cycle, review, recommendations (880→loc)
├── variants.py        Generador: 4 arquetipos + editorial brief (858→loc)
├── llm.py             Router LLM: free-first chain, 7 providers (439→loc)
├── generate.py        CLI: 22 comandos click (739→loc)
├── orthography.py     Correccion: regex dict → pyspellchecker → Groq (792→loc)
├── quality.py         Quality gates: editorial + scoring (449→loc)
├── analytics.py       Tracking: engagement, metrics, reports (456→loc)
├── engagement.py      Scraping Apify: likes, comments, shares (425→loc)
├── hashtags.py        LLM hashtags + YAML fallback (368→loc)
├── trends.py          Patrones virales Apify (304→loc)
├── scheduler.py       Planificador: rate limits, queue (371→loc)
├── weekly_report.py   Reportes + dual-write engagement (331→loc)
├── research.py        DuckDuckGo + Reddit (323→loc)
├── feeds.py           RSS: fetch, filter, rank (508→loc)
├── images.py          HF FLUX + Pollinations (209→loc)
├── review.py          Confidencialidad + voz + formato (268→loc)
├── linkedin.py        LinkedIn API: 3-step image upload (296→loc)
├── validation.py      CVE validation + diversity penalty (276→loc)
├── brief.py           Editorial brief: hook/angle/CTA (146→loc)
├── timing.py          Clasificacion temporal (224→loc)
├── circuit.py         Circuit breaker + retry budget (180→loc)
├── timezones.py       Multi-continental scheduling (298→loc)
├── voice.py           Perfil de voz YAML + LRU cache (137→loc)
├── bitacora.py        Log persistente JSONL (194→loc)
├── hallucinations.py  Deteccion de alucinaciones (304→loc)
├── youtube.py         YouTube transcripts (deshabilitado) (286→loc)
└── __init__.py        __version__ = "2.4.0"
```

## Content Pipeline (run_cycle)

```
1. Discovery     → RSS feeds + Apify viral patterns
2. Research      → DuckDuckGo + Reddit (dual source)
3. Editorial     → build_editorial_brief (hook/angle/CTA/source)
4. Timing        → Clasificar BREAKING/ONGOING/RESOLVED/ARCHIVAL
5. Generation    → variants.py: 4 arquetipos con editorial brief
6. Quality       → editorial checks + LLM scoring (blocking gate)
7. Orthography   → regex dict → pyspellchecker → Groq proofread
8. CVE validation→ CIRCL API + cache
9. Hashtags      → LLM (Groq) → OpenAI → YAML fallback
10. Image        → HF FLUX → Pollinations fallback
11. Publish      → LinkedIn API (3-step upload) or local fallback
12. Log          → bitacora JSONL + analytics metrics.jsonl
```

## LLM Chain (free-first)

```
Provider               Model                          Cost      RPD
─────────────────────────────────────────────────────────────────────
Groq (primary)         llama-3.3-70b-versatile         Free    14,400
Groq (fast)            llama-3.1-8b-instant            Free    14,400
OpenRouter (1)         meta-llama/llama-3-70b          Paid*     1,000
OpenRouter (2)         google/gemma-2-9b-it            Paid*     1,000
OpenAI (paid)          gpt-4o-mini                     Paid    varies
OpenCodeZen (1)        deepseek-v4-pro                 Free*   varies
OpenCodeZen (2)        glm-5.2                         Free*   varies
```

*OpenRouter requires $10+ credit for 1,000 RPD. OpenCodeZen free tier available.
Pagos solo como ultimo recurso. Cada proveedor tiene circuit breaker.

## Module Details

### runner.py — Orquestador principal

```
run_cycle(region, slot, dry_run) → dict
  1. _run_research()           → research context
  2. _select_pillar()          → pillar from topic
  3. build_editorial_brief()   → hook/angle/CTA contract
  4. analyze_timing()          → temporal classification
  5. generate_post()           → post text (variant arquetipo)
  6. quality_gate()            → editorial + scoring (blocking)
  7. fix_orthography()         → spelling correction
  8. validate_generated_post() → CVE validation
  9. append_hashtags()         → LLM hashtags
  10. generate_image()         → HF/Pollinations image
  11. publish_post()           → LinkedIn API or local
  12. log_event()              → bitacora
```

### variants.py — 4 Arquetipos

| Arquetipo | Viral Lift | Structure |
|-----------|------------|-----------|
| tesis_contrariana | 4.58x | Hook → Contexto → Tesis → Evidencia → CTA |
| breach_breakdown | 3.48x | Hook → Contexto → Conflicto → Resolucion → CTA |
| war_story | 2.20x | Hook → Provocacion → Evidencia → Desafio → CTA |
| deep_dive | — | Hook → Analisis → Profundidad → CTA |

Each arquetipo has a dedicated prompt builder and validation rules.
Editorial brief is injected as binding contract via `_render_brief_contract()`.

### llm.py — Provider Chain

```python
_build_providers() → [
    # Free first
    ("groq", "llama-3.3-70b-versatile", "GENERATION"),
    ("groq", "llama-3.1-8b-instant", "GENERATION"),
    ("openrouter", "meta-llama/llama-3-70b", "GENERATION"),
    ("openrouter", "google/gemma-2-9b-it", "GENERATION"),
    # Paid last
    ("openai", "gpt-4o-mini", "GENERATION"),
    ("opencode", "deepseek-v4-pro", "GENERATION"),
    ("opencode", "glm-5.2", "GENERATION"),
]
```

401/403/422 → PipelineError(recoverable=False) → degrade to next provider.

### engagement.py — Apify Scraping

```
scrape_engagement(days, dry_run) → summary
  1. _load_published_posts()  → out/published/*.json (filtered by date)
  2. _call_apify_actor()      → async: POST /runs → poll → GET /dataset/items
  3. _match_posts()           → Level 0: URL, Level 1: text exact, Level 2: fuzzy ≥0.75
  4. _write_metrics()         → dual-write: analytics.py + weekly_report.py
```

Actor: `alizarin_refrigerator-owner~linkedin-post-scraper` (google_search mode).
Limitation: Google indexes LinkedIn posts with 1-2 week delay.

### quality.py — Blocking Gates

1. `_editorial_checks()` — deterministic (no LLM): hook presente, fuente citada,
   ángulo desarrollado, CTA de cierre. Missing any → BLOCK.
2. `_fast_checks()` — format: short text, paragraph count, disconnected ideas.
3. `_compute_fast_scores()` — LLM scoring with feedback.
4. Score < 70 OR any editorial check fails → BLOCK publication.

## Configuration Files

| File | Purpose | Referenced by |
|------|---------|---------------|
| config/voice.yaml | Tone, personality, anti-patterns | pipeline, variants, voice |
| config/weekly_rotation.yaml | Schedule by day, best_times | pipeline, scheduler |
| config/pillars.yaml | 5 pillars, 4 variant types | pipeline, variants |
| config/feeds.yaml | 5 RSS feeds, YouTube (disabled) | pipeline |
| config/hashtags.yaml | Fallback pools by pillar | hashtags |
| config/image_styles.yaml | Pollinations FLUX styles | pipeline |
| config/linkedin.yaml | API config, rate limits, scraping | pipeline, scheduler, engagement |
| config/voice_profile.md | Destilled voice profile | voice |
| config/engagement.json | Manual engagement data | weekly_report |

## Data Flow

```
RSS Feeds ─┐
            ├→ feeds.py → pipeline.py → runner.py
Apify ──────┘                              │
                         research.py ──────┤
                         brief.py ─────────┤
                         timing.py ────────┤
                         variants.py ──────┤
                         quality.py ───────┤
                         orthography.py ───┤
                         validation.py ────┤
                         hashtags.py ──────┤
                         images.py ────────┤
                         linkedin.py ──────┘
                              │
                    bitacora.py + analytics.py
                              │
                    weekly_report.py + engagement.py
```

## Testing

~279 tests across 19 test files. Run with:
```bash
python3 -m pytest tests/ -q
```

Key test modules:
- test_engagement.py — Apify matching, dual-write, URL conversion
- test_quality_gates.py — editorial blocking gates
- test_brief.py — editorial brief sanitization
- test_variants.py — 4 arquetipo prompts
- test_runner.py — full cycle integration (mocked)

## Dependencies

### Runtime (pyproject.toml)
httpx≥0.27, pyyaml≥6.0, rich≥13.0, click≥8.1, python-dotenv≥1.0, python-dateutil≥2.8

### Optional
pyspellchecker≥0.7 (install: `pip install -e ".[spell]"`)

### Dev
pytest≥8.0, pytest-asyncio≥0.23
