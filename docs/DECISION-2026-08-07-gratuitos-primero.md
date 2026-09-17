# DECISION 2026-08-07: Gratuitos primero, pagos al final

## Contexto
`MODEL_GENERATION` apuntaba a `opencode:deepseek-v4-pro` (pago) y forzaba el
modelo pago al frente. La cuenta OpenCodeZen devolvía `CreditsError:
Insufficient balance` (HTTP 401). URL y key son correctos; el problema es
saldo insuficiente. Además, el 401 se relanzaba como `httpx.HTTPStatusError`
crudo y rompía el primer intento de generación en vez de degradar.

## Decisión
1. `MODEL_GENERATION` → `groq:llama-3.3-70b-versatile` (gratuito).
   El pago (OpenCode/OpenAI) queda SOLO como último recurso en la cadena.
2. Cadena `_build_providers()`: groq 70B → groq 8B → openrouter →
   openrouter:scout → openai → opencode deepseek-v4-pro → opencode glm-5.2.
3. Fix robustez: `_try_provider` convierte 401/403/422 en
   `PipelineError(recoverable=False)` en vez de relanzar el HTTPStatusError
   crudo. El path de modelo explícito degrada a la cadena gratuita en vez de
   romper el primer intento.

## Validación end-to-end (publicación real)
- LinkedIn: `urn:li:share:7491663589653643264`
- Cycle: `cycle_1786151434`
- Variante: `tesis_contrariana` · Pillar: `casos_estudio` · chars=750
- valid=True · quality=70 (pasó en intento 4 tras 85/80/78) · timing=RESOLVED
- Post: `out/published/post_1786151786.json`
- Suite tests: 320 passed. Coste de generación: $0 (groq).

## Anexo: dry-run previo
- `cycle_1786151141` (dry): quality pasó en attempt 2 (score 75), 3 hashtags
  específicos (incidentresponse, phishing, otsecurity).
