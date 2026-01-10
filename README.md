# AfterBot
AFTER isn’t “another bot”—it’s Strategic Orientation with evidence: it turns ideas into executable plans with KPI thresholds, owned dependencies, and risk guardrails (kill-switch/rollback). No capacity? We say NO. No metrics? No plan. Can your digitalization operate at that bar?

## Files
- `Agent After Bot Refinamiento X.json`: current n8n workflow definition.
- `Agent After Bot Refinamiento X - improved.json`: updated workflow variant with refinements.

## Documentación (ES) — AfterBot Tool
AfterBot es un flujo de n8n que recibe mensajes de Telegram, normaliza intención, enruta entre Alignment Engine y AFTER, y construye respuestas listas para enviar por Telegram. La lógica principal vive dentro de los nodos de tipo `code` que operan sobre el `output` del workflow y el estado persistido por chat.

### Flujo general
1. **Entrada Telegram**: el trigger captura mensajes y callbacks desde Telegram.
2. **Normalización e intención**: se limpia el texto, se detectan comandos (`/continue`, `/after`, `/clarify`, `/briefejecutivo`, `/reset`) y se asignan `intent` y `after_action`.
3. **Enrutamiento**: un `Switch` manda el payload a la ruta de Alignment, AFTER o greeting.
4. **Procesamiento LLM**: se combinan las respuestas LLM en `Merge LLMs`.
5. **Post-merge**: se rehidrata el contexto, se calcula `switch_key`, se cachean resultados AFTER, y se decide el siguiente paso (alignment, after, brief, greeting o fallback).
6. **Salida Telegram**: se construye un mensaje final según el `switch_key` y se envía vía Telegram.

### Comandos soportados
Todos los comandos deben escribirse explícitamente en el chat para avanzar:
- `/continue`: continúa el flujo de Alignment usando respuestas del usuario.
- `/clarify`: reevalúa Alignment con nueva idea o ajustes.
- `/after`: inicia refinamiento AFTER con la idea vigente.
- `/refinebrief`: refinamiento AFTER y preparación de brief.
- `/briefejecutivo` o `/brief`: solicita el brief ejecutivo cuando esté listo.
- `/reset`: reinicia el hilo, vuelve a greeting.

### Componentes principales
#### 1) TG Pre-route Normalizer + Edits2
Normaliza texto, detecta comandos y evita contaminar la idea con respuestas tipo A1/A2. Mantiene `idea_text` viva y enruta respuestas al flujo correcto.
- **Detectores**: greeting, reset, comandos por línea o inline.
- **Rutas**: `intent=alignment` para `/continue` y `/clarify`; `intent=after` para `/after` y `/brief`.
- **Respuestas**: identifica bloques de respuestas (A1/A2/Q1) para tratarlas como aclaraciones.
- **Estado**: mantiene `thread` por `routing_key` y limpia hilos antiguos.

#### 2) Post-Merge Organizer + Switch Enricher3
Organiza la salida de LLMs, rehidrata contexto y decide el `switch_key` final.
- **Rehidratación**: si llega `/briefejecutivo` sin payload AFTER, usa el último `after_result` cacheado.
- **Latch de brief**: guarda solicitudes pendientes de brief hasta que `brief_ready=true`.
- **Score Alignment**: calcula y asigna `gate` según el score (discard/clarify/after_refine).
- **Estado persistido**: cachea resultados AFTER y actualiza `thread.mode`.

#### 3) Build Telegram Response (Greeting / Alignment / After)
Construye mensajes de salida específicos según el `switch_key`.
- **Greeting**: explica AFTER, Alignment y formato de idea.
- **Alignment (clarify)**: solicita respuestas A1/A2 y pide `/continue`.
- **After (after_brief)**: compone resumen ejecutivo, roadmap, riesgos, dependencias, objetivos y siguientes pasos.
- **Fallback**: recuerda al usuario que debe enviar un comando explícito.

### Datos clave en `output`
- `intent`: `alignment`, `after` o `greeting`.
- `after_action`: `continue`, `clarify`, `refine` o `brief`.
- `idea_text`: idea viva que alimenta refinamiento.
- `clarify_answers_text`: respuestas a preguntas (Alignment o AFTER).
- `after_result`: resultado estructurado de AFTER (resumen, KPIs, roadmap, historias, etc.).
- `switch_key`: determina la ruta final de respuesta (`alignment.clarify`, `after.after_brief`, `brief.ejecutivo`, `greeting`, etc.).
