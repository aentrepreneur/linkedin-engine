# LinkedIn Engine v2.4.0

Generador de contenido LinkedIn para Applied AI Engineer / Forward Deployment Engineer.

Flujo: `tendencias → descubrir → generar → hashtags → imagen → publicar → medir`

## Comandos

| Comando | Descripcion |
|---------|-------------|
| `le run --slot morning` | Ciclo completo: discover → post → hashtags → image → publish |
| `le run --dry-run` | Simulacion sin publicar |
| `le gen --day MAR` | Generar post para un dia |
| `le gen --week` | Generar semana completa |
| `le review-daily` | Analizar posts de ayer |
| `le recommendations` | Recomendaciones basadas en analytics |
| `le schedule-info` | Ver programacion multi-timezone |
| `le report` | Reporte semanal de engagement |
| `le engagement <id>` | Update manual de metricas |
| `le engagement-scrape` | Scraping automatizado via Apify |
| `le queue --day MAR` | Generar + encolar para aprobacion |
| `le approve <id>` / `le reject <id>` | Aprobar/rechazar post en cola |
| `le plan` / `le next` | Plan semanal / proximo slot |
| `le queue:list` / `le queue:stats` | Listar cola / estadisticas |
| `le review <archivo>` | Revisar post existente |
| `le image "prompt"` | Generar imagen (HF FLUX / Pollinations) |
| `le trends` | Descubrir patrones virales (Apify) |
| `le stats` / `le history` | Bitacora / historial reciente |
| `le circuits` | Estado de circuit breakers |
| `le hashtags` | Stats de configuracion |

## Instalacion

```bash
python3 -m venv venv
./venv/bin/pip install -e ".[dev,spell]"
cp .env.example .env
# Editar .env con tus API keys (minimo: Groq gratis)
PYTHONPATH=. ./venv/bin/python -m pytest tests/ -q
```

## Variables de entorno minimas

```bash
LLM_PROVIDER=groq              # Gratis, 14,400 RPD
LLM_API_KEY=gsk_xxx             # https://console.groq.com/keys
APIFY_API_TOKEN=apify_api_xxx   # Para trends + engagement scraping
```

Ver `.env.example` para la lista completa (LinkedIn OAuth, HuggingFace, etc).

## Arquitectura

```
engine/
├── pipeline.py        Pipeline completo + discovery (1175 loc)
├── runner.py          Orquestador: cycle, review, recommendations (880→loc)
├── variants.py        Generador de posts: 4 arquetipos (858→loc)
├── llm.py             Router LLM: free-first chain (439→loc)
├── generate.py        CLI principal — 22 comandos (739→loc)
├── orthography.py     Correccion ortografica: 3 capas (792→loc)
├── quality.py         Quality gates: editorial + scoring (449→loc)
├── analytics.py       Tracking de engagement (456→loc)
├── engagement.py      Scraping Apify: likes, comments, shares (425→loc)
├── hashtags.py        LLM hashtags + YAML fallback (368→loc)
├── trends.py          Extraccion patrones virales Apify (304→loc)
├── scheduler.py       Planificador con rate limits (371→loc)
├── weekly_report.py   Reportes + engagement tracking (331→loc)
├── research.py        DuckDuckGo + Reddit research (323→loc)
├── feeds.py           RSS feed fetcher (508→loc)
├── images.py          HF FLUX + Pollinations (209→loc)
├── review.py          Confidencialidad + voz + formato (268→loc)
├── linkedin.py        Publicacion + image upload 3-step (296→loc)
├── validation.py      CVE validation + diversity penalty (276→loc)
├── brief.py           Editorial brief: hook/angle/CTA (146→loc)
├── timing.py          Clasificacion temporal del evento (224→loc)
├── circuit.py         Circuit breaker + retry budget (180→loc)
├── timezones.py       Multi-continental scheduling (298→loc)
├── voice.py           Perfil de voz YAML + cache (137→loc)
├── bitacora.py        Log persistente de eventos (194→loc)
├── hallucinations.py  Deteccion de alucinaciones (304→loc)
├── youtube.py         YouTube transcripts (deshabilitado) (286→loc)
└── __init__.py        __version__ = "2.4.0"
```

## Cadena LLM (free-first)

```
groq:llama-3.3-70b-versatile → groq:llama-3.1-8b-instant
→ openrouter:meta-llama/llama-3-70b → openrouter:google/gemma-2-9b
→ openai:gpt-4o-mini → opencode:deepseek-v4-pro → opencode:glm-5.2
```

Pagos solo como ultimo recurso. Cada proveedor tiene circuit breaker (3 fallos → 5min recovery).

## Content Sources

| Fuente | Estado | Notas |
|--------|--------|-------|
| RSS (5 feeds) | Activo | HN, BleepingComputer, Krebs, AI News, BitLife Media |
| Apify viral posts | Activo | Scraping por region (US, LATAM, EU) |
| YouTube transcripts | Deshabilitado | IP banned en cloud |
| Apify engagement | Activo | Google search mode, ~$0.04/post |

## Content Pillars

1. **Principios atemporales** — Libros clasicos aplicados a AI/automation
2. **Casos de estudio reales** — Proyectos con resultados medibles
3. **Growth hacks con tests** — Trucos probados con datos
4. **Opinion polemica** — Hot takes sobre la industria
5. **Tutoriales tecnicos** — Herramientas y trucos practicos

## Arquetipos de Post

| Arquetipo | Viral Lift | Estructura |
|-----------|------------|------------|
| tesis_contrariana | 4.58x | Hook → Contexto → Tesis → Evidencia → CTA |
| breach_breakdown | 3.48x | Hook → Contexto → Conflicto → Resolucion → CTA |
| war_story | 2.20x | Hook → Provocacion → Evidencia → Desafio → CTA |
| deep_dive | — | Hook → Analisis → Profundidad → CTA |

## Automatizacion

### Cron (produccion)

```bash
# Morning: 7AM Guatemala = 13:00 UTC
0 13 * * * /opt/linkedin-engine/scripts/cron_runner.sh run --slot morning
# Evening: 6PM Guatemala = 00:00 UTC (+1)
0 0 * * * /opt/linkedin-engine/scripts/cron_runner.sh run --slot evening
# Daily review: 8AM Guatemala = 14:00 UTC
0 14 * * * /opt/linkedin-engine/scripts/cron_runner.sh review-daily
```

### Engagement Tracking

```bash
# Manual
le engagement post_123 -l 10 -c 3 -s 1

# Automatizado (requiere posts indexados por Google, ~1-2 semanas)
le engagement-scrape --days 7
le engagement-scrape --dry-run  # sin escribir
```

## Testing

```bash
python3 -m pytest tests/ -q          # ~279 tests
python3 -m pytest tests/ -v --tb=short
```

## Documentacion

- [CHANGELOG.md](docs/CHANGELOG.md) — Historial de cambios
- [TECHNICAL.md](docs/TECHNICAL.md) — Documentacion tecnica
- [DECISION-2026-08-07-gratuitos-primero.md](docs/DECISION-2026-08-07-gratuitos-primero.md) — Decision free-first
