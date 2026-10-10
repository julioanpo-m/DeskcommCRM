# Plan — Captación multicanal Meta → n8n → DeskcommCRM (7 fases)

## Contexto

El consultorio del Dr. Julio Porras (alergólogo, Boca del Río, Veracruz) captará pacientes por 4 canales de Meta (Lead Ads, comentarios "URTICARIA", Click-to-WhatsApp, link en bio), los insertará en el pipeline "Pacientes Nuevos" → LEAD NUEVO, los etiquetará por fuente, alimentará al agente IA (Claude) y cruzará métricas de Meta Ads con resultados reales del CRM. n8n auto-hospedado (contenedor en el VPS, subdominio propio con SSL — Meta exige callback HTTPS) orquesta lo externo; el CRM aporta ingesta, dedup, IA y motor de automatización.

### Hechos verificados en el código (exploración, con referencias)

- **Ingesta webhook existe**: `POST /api/v1/webhooks/in/[token]` (`app/api/v1/webhooks/in/[token]/route.ts`). Token del path → tenant (`webhook_sources.path_token`); rate limit 60/min; body JSON o form-urlencoded plano. Firma opcional HMAC-SHA256 hex del body crudo exacto (`x-deskcomm-signature`, `lib/webhooks/inbound.ts:117`). **La autorización de IA en captura exige fuente con secret** (route ~línea 735) → la fuente se crea con secret y n8n firma.
- **Mapeo** (`mapInboundPayload`): aliases por defecto cubren `full_name`, `phone_number`, `email`; lo demás → `custom_fields`; `utm_*` → `source_metadata`. `external_id` → idempotencia (índice único org+source; retry responde 200 con el mismo `lead_id`). **`tags: []` hardcodeado** (route:582) → se extiende la ruta.
- **Teléfonos**: `normalizePhoneBR` convierte números sueltos de 10-11 dígitos a `+55…` (¡un MX sin `+52` se volvería brasileño!); con `+` pasa directo y `phoneLookupVariants` devuelve intacto lo que no empieza con 55 (`lib/channels/phone-variants.ts:40`). → n8n SIEMPRE envía E.164 `+52`.
- **WhatsApp entrante**: WAHA → `handleInbound` (`lib/waha/ingest.ts`) → `messages` → trigger DB emite `message.received` (payload con `body_preview` = texto, máx 280 chars). `aplicarEfeitosPosEntrada` (`lib/channels/pos-entrada.ts`): opt-out → `abrirDemanda` (nace el lead vía `garantirLeadDaConversa`, `lib/leads/nascimento-do-lead.ts`) → `avaliarCampanha` → `pedirDespachoDoAgente` → **el agente IA contesta solo** (`lib/agent-engine/agent/inbound-turn.ts`, Claude, envía solo vía tool).
- **Atribución de anuncios ya existe**: `extrairAtribuicaoWaha` lee `externalAdReply` de clics CTWA → `fn_estampar_atribuicao_de_anuncio` (primer toque, `contacts.source_metadata.ad_*`); el lead de conversación hereda `source` + tag `Meta_ads`. Marcadores de sitio `[ref:…]`/`[dk1:…]` (`lib/leads/origem-do-site.ts`).
- **Dedup por chat**: `fn_nascer_lead_da_conversa` (migración 0256) — advisory lock por org+contact; 1 lead abierto por demanda; reescribir tras cerrar crea otro.
- **Motor de automatización** (`lib/automation/engine.ts`): triggers `message.received`, `lead.created`, etc.; condiciones `field op value` (`eq|neq|contains`, dot-path); contexto de `message.received` = payload + contacto; de `lead.created` = lead* + contacto*. Acciones: `add_tag`, `call_webhook` (POST JSON con `X-Webhook-Delivery`/`X-Webhook-Signature`, 3 reintentos 1s/5s — `lib/automation/actions/call-webhook.ts`, contrato en `docs/integracao/webhooks-de-saida.md`), `send_ai_message`, etc.
- **Métricas Meta ya existen (lectura)**: `lib/plataformas-de-anuncio/meta/insights.ts` (`lerContas/lerCampanhas/lerInsights` — spend, impressions, reach, cpm, ctr, cpc, results, cost_per_result), credencial de lectura `ad_insights_connections`, pantalla y rutas `app/api/v1/ads/meta/*` (solo sesión manager). **Escribir de vuelta** (conversiones offline): `lib/conversoes/` + ledger `ad_conversion_dispatches` (Purchase en `lead.won`, `MetaEtapa:<uuid>` por etapa). **MXN soportado** (`lib/money.ts`: MOEDAS_SERVIDAS incluye MXN).
- **MCP del CRM**: servidor Streamable HTTP con `api_tokens` (`app/api/mcp/route.ts`), tools `crm_list_leads`/`crm_get_lead` (exponen tags, stage, value_cents…). **MCP externo soportado por org** (`organizations.settings.mcp_externo`, #2147) pero **los turns con contacto NO cargan herramientas externas** (anti-fuga de datos) → el Meta Ads MCP sirve para análisis sin contacto, no dentro de conversaciones.
- **No existe**: código de comentarios FB/IG ni leadgen (n8n escucha Graph API directamente); cruce campaña × resultado CRM (ROAS/CPL); snapshots programados; Google Sheets.
- **Convenciones del fork (contribuidor)**: ramificar desde el `origin/main` del producto (no del fork — su `main` trae el CLAUDE.md local del consultorio); hooks `armar-hooks.sh`; tríada de migración = header `-- manifest:` + espejo exacto en `supabase/baseline.sql` + número NNNN único (`pnpm checar:colisao-de-migration`); fragmento de release en `.changes/`; test de sabotaje; `pre-voo.sh`. Tests Vitest con nombres de frase en PT; patrón de test de ruta: `tests/unit/captacao-na-etapa-exigente-cria-o-lead.test.ts`; docs en `docs/integracao/`.

### División de responsabilidades (arquitectura objetivo)

| Capa | Qué hace | Dónde |
|---|---|---|
| n8n | Listeners Meta (leadgen, comentarios, DM), normalización externa, firma y POST al CRM, log a Google Sheets, reintentos con backoff, métricas programadas, alertas | Contenedor n8n en VPS |
| CRM | Ingesta webhook con dedup, nacimiento de leads de chat con dedup por contacto, atribución CTWA, agente IA (Claude), reglas de automatización (keywords → tags → webhook), conversiones offline a Meta, MCP para lectura externa | Repo DeskcommCRM |
| Config operativa | Reglas de automatización, fuente webhook, links wa.me, credenciales | CRM UI / n8n credentials / Meta Business |

---

## Contrato del webhook CRM (aplica a Fases 1 y 2)

**Request** — `POST /api/v1/webhooks/in/<token>` · `Content-Type: application/json` · `X-Deskcomm-Signature: <hmac_sha256_hex(secret, body crudo exacto)>`

```json
{
  "external_id": "1234567890123456",
  "full_name": "María González Pérez",
  "phone_number": "+5212291234567",
  "email": "maria.gonzalez@example.com",
  "sintomas": "Ronchas y comezón en brazos por la noche",
  "tiempo_sintomas": "3 semanas",
  "leadgen_id": "1234567890123456",
  "form_id": "1234567890",
  "ad_id": "987654321",
  "adset_id": "555555",
  "campaign_id": "444444",
  "page_id": "111111",
  "created_time": "2026-10-10T09:15:00-06:00",
  "utm_source": "facebook",
  "utm_campaign": "444444",
  "utm_content": "987654321",
  "tags": ["#fb-lead-ads", "#urticaria", "#agudo"]
}
```

- `full_name`/`phone_number`/`email` → resueltos por `mapInboundPayload` (aliases por defecto). `sintomas`, `tiempo_sintomas`, ids → `custom_fields` solos. `utm_*` → `source_metadata`. `tags` → nuevo (ver Fase 1.5).
- **Fase 2 (comentarios)**: mismo endpoint con payload sin teléfono — `{"external_id":"<comment_id>","full_name":"<nombre del comentarista>","palabra_clave":"URTICARIA","video_id":"…","red_social":"instagram","comment_id":"…","perfil_url":"…","tags":["#cta-urticaria"]}` — válido porque la ruta exige nombre O teléfono O email; el contacto nace por nombre + ids sociales.
- **Respuesta**: 200 `{"data":{"lead_id":"<uuid>"}}` (+`X-Request-Id`); duplicado idempotente → 200 con el mismo id.
- **Errores × acción n8n**:

| Status | code | Acción |
|---|---|---|
| 200 | — | Log "creado"; fin (retries seguros vía `external_id`) |
| 400/401/404/422 | `invalid_json`/`invalid_request`/`invalid_signature`/`not_found`/campo numérico | **Terminal**: log + notificar a Miriam de inmediato |
| 429 | `rate_limited` (Retry-After: 60) | Reintento con retardo = max(backoff, 60 s) |
| 5xx / timeout / red | — | Reintento con backoff exponencial (máx 5) |

---

## Fase 1 — Facebook Lead Ads (leadgen) · estimación 4-5 días

### 1.1 Flujo n8n

```
[Facebook Lead Ads Trigger] (Meta webhook leadgen → n8n; obtiene lead vía Graph API)
        ▼  item: {id (leadgen_id), created_time, ad_id, adset_id, campaign_id, form_id, page_id, field_data[{name, values[]}]}
[Code: FbLeadAds.normalizar] (bundle del repo: aplanar field_data, E.164 +521…, síntomas→tags, duración→#agudo/#crónico, construir `corpo` UNA vez)
        ▼
[IF válido] ──NO──▶ [Log Sheets "rechazado_datos"] → FIN (sin retry: dato, no falla)
        ▼ sí
[Crypto: HMAC-SHA256 hex(corpo)]  (mismo string que se envía)
        ▼
[HTTP POST crm…/webhooks/in/<token>]  header X-Deskcomm-Signature · body = string crudo (nunca modo JSON: n8n no debe re-serializar)
        ▼ 200 {lead_id}
[Log Sheets "creado"] → FIN
        ▼ error
[Error Workflow]: 4xx (no 429) → [Log "fallo_terminal"] → [Notificar Miriam: WAHA sendText → WhatsApp + Resend → email] → FIN
                  429/5xx/timeout → [Log "reintento N"] → [Wait min(2^N×5min, 80min); 429 usa Retry-After] → [Execute Workflow "POST firmado"] → error vuelve aquí con N+1; N=5 → terminal + notificar
```

- Seguridad anti-loop: contador `attempts` viaja en el item (Error Trigger conserva `data`); respaldo en nodo Static Data; tope 5 → ~2.6 h máx por lead.
- El POST firmado es un sub-workflow reutilizado por los reintentos.

### 1.2 Campos de Facebook y mapeo

| Campo FB | Destino | Notas |
|---|---|---|
| `id` (leadgen_id) | `external_id` | idempotencia |
| `created_time` (unix) | `custom_fields.created_time` (ISO, America/Mexico_City) | — |
| `full_name` | nombre | requerido |
| `phone_number` | E.164 `+521XXXXXXXXXX` | ver reglas 1.4 |
| `email` | lowercase/trim | opcional |
| Preguntas personalizadas | `custom_fields.sintomas`, `custom_fields.tiempo_sintomas` | crear formulario con claves explícitas (API del form) o match por texto de pregunta (mapa configurable) |
| `form_id`/`ad_id`/`adset_id`/`campaign_id`/`page_id` | `custom_fields` + `utm_*` (`utm_source=facebook`, `utm_campaign`, `utm_content`) | `source_metadata` para atribución (Fase 6) |

### 1.3 Normalizador — módulo puro del repo

`lib/integrations/n8n/facebook-lead-ads.ts` (TS puro, **cero imports** para que el bundle sea autocontenido) + bundle IIFE `n8n/facebook-lead-ads.node.js` generado con esbuild (`pnpm n8n:bundle`, esbuild agregado como devDependency — hoy solo transitivo vía vite) pegado al Code node. Test de sincronía reconstruye el bundle y compara (patrón del test docs↔código), y ejecuta el bundle en `vm` con un fixture.

- `normalizarTelefoneMX(raw): string | null` — quitar no-dígitos; acepta `+52 1 …`, `+52…`, `52…` (12-13 dígitos) y lada de 2-3 dígitos; **forma canónica `+52` + `1` + 10 dígitos locales = `+521XXXXXXXXXX`** (13 dígitos, formato `wa_id` de WhatsApp MX — decisión del usuario); dígito local inicial 2-9; **número suelto de 10 dígitos → `null`** (el CRM lo leería como BR).
- `mapearSintomasATags(texto)` — texto sin acentos/minúsculas vs tabla configurable: urticaria/ronchas/habones/picazón/comezón/sarpullido → `#urticaria`; rinitis/estornudos/nariz/moco/congestión → `#rinitis`; asma/tos/sibilancias/respirar → `#asma`; dermatitis/eczema/piel → `#dermatitis`; alimentos/comida/alimentaria → `#alergia-alimentaria`; ojos/conjuntivitis → `#conjuntivitis`; multi-match permitido; sin match → sin tag de síntoma.
- `clasificarDuracion(texto)` — "días"÷7, semanas×1, meses×4.345, años×52.18; frases crónicas ("desde niño", "siempre", "toda la vida") → crónico; umbral clínico **< 6 semanas → `#agudo`; ≥ 6 semanas → `#crónico`**; ilegible → null.
- `normalizarLeadDoFacebook(item, opcoes)` — valida (bloqueante solo si teléfono Y email inservibles), construye payload con **orden de claves fijo** (contrato de la firma) y `corpo = JSON.stringify(payload)` (el único string que se firma y envía). Tags: `#fb-lead-ads` siempre primero + síntoma + duración.

### 1.4 Cambios en el CRM (PR al producto, genérico — sin secretos ni datos del consultorio)

1. **Aceptar `tags` en la ingesta** — helper puro `extrairTagsDoPayload` en `lib/webhooks/inbound.ts` (array de strings, `normalizarTags`, tope 20 — convención del repo); la ruta destructuring `tags` del payload antes del mapeo (como `external_id`, línea ~282) y `tags: tagsNormalizadas` en `leadInput` (línea 582). Malformado → ignorado sin rechazar la captación. Dedup por `external_id` no re-aplica tags (primer envío gana).
2. **Dedup entre fuentes (opt-in)** — migración (tríada): columna `webhook_sources.dedup_por_contato_dias int null` (null = comportamiento actual, no rompe tenants). En la ruta: si la fuente tiene N días y el contacto ya tiene lead ABIERTO en el MISMO pipeline con `created_at ≥ now() − N días` → 200 con el lead existente, outcome `duplicado`, **unión de tags** sobre el lead existente.
3. **Variantes MX de teléfono** — `lib/channels/phone-variants.ts`: para números `52…`, variantes con/sin el `1` móvil (`+52229…` ↔ `+521229…`), misma prudencia que el nono dígito BR → un solo contacto cuando webhook y WhatsApp entregan grafías distintas. Verificar `wa_id` MX real en WAHA antes de fijar la canónica.
4. Fase 0 (ritual de contribuidor): cargar skill `deskcomm-contribuir`, `quem-sou.sh`, rama desde `origin/main` del producto, `armar-hooks.sh`, tríada de migración, fragmento `.changes/`, test de sabotaje, `pre-voo.sh`.

### 1.5 Configuración Meta (Fase 1)

1. Business Manager: portafolio verificado + Página vinculada. 2. App tipo Business; permisos `leads_retrieval`, `pages_show_list`, `pages_read_engagement` (+`ads_read` para F5; +`ads_management` solo si se pausan campañas en F7). 3. System User (Admin) con la Página y la App → token de larga duración. 4. Suscribir Página al webhook `leadgen` → URL pública del trigger n8n (`POST /v22.0/{page_id}/subscribed_apps?subscribed_fields=leadgen`; **un solo webhook leadgen por app**). 5. App Live + Standard access (activos propios no requieren review; `leads_retrieval` avanzado sí → screencast). 6. Instant Form: full_name, phone_number, email + preguntas personalizadas con claves explícitas + **pregunta de consentimiento obligatoria**. 7. Probar con Lead Ads Testing Tool. Nota: retención de leads 90 días (n8n consulta a tiempo, sin pérdida).

### 1.6 Configuración n8n (Fase 1)

Contenedor en el VPS (Compose, `WEBHOOK_URL=https://n8n.<dominio>/`, volumen, `N8N_ENCRYPTION_KEY`); credenciales: Meta OAuth2, Crypto (Hmac secret = secret de la fuente), Google Sheets (Service Account), HTTP (WAHA, Resend); variables de entorno de n8n: `CRM_WEBHOOK_URL`, `WAHA_URL/API_KEY/SESSION`, `OPERADORA_CHAT_ID`, `RESEND_API_KEY`, id de la hoja de Sheets (nunca en el JSON del workflow). Workflows versionados en el repo: `n8n/workflows/facebook-lead-ads*.json` + `n8n/README.md` (orden de importación: sub → error → principal; religar IDs).

### 1.7 Configuración CRM (Fase 1)

Crear fuente webhook: `path_token` aleatorio ≥32 chars, `secret` ≥16 chars (**obligatorio** para que `authorize_ai_on_capture` funcione), `default_pipeline_id`/`default_stage_id` = "Pacientes Nuevos"/"LEAD NUEVO" (UUIDs por org), `dedup_por_contato_dias = 30` (tras 1.4.2). `field_map` por defecto alcanza.

### 1.8 Tests (Fase 1)

- Normalizador (`lib/integrations/n8n/facebook-lead-ads.test.ts` + fixtures reales en `tests/fixtures/facebook-lead-ads/`): teléfonos (`+52 1 229 123 4567`→`+5212291234567`, `5212291234567`→idem, `+522291234567`→idem, CDMX, **10 dígitos sueltos → null**, basura → null); síntomas (cada tag, acentos/mayúsculas, multi-match, sin match); duración ("3 semanas"→agudo, "6 semanas"→crónico límite, "2 meses"/"1 año"/"desde niño"→crónico, "3 días"→agudo); email (trim/lower, inválido→null); armado (orden exacto de claves, `external_id`, `utm_*`, `#fb-lead-ads` primero, validación tel+email).
- Ruta (`tests/unit/a-captacao-aceita-tags-do-payload.test.ts`, patrón de `captacao-na-etapa-exigente-cria-o-lead.test.ts`): tags normalizadas en el lead; `tags` no cae en `custom_fields`; no-array/no-string ignorado sin 400; tope 20; reenvío mismo `external_id` → 200 sin re-aplicar tags; sin clave `tags` → comportamiento actual; dedup opt-in (mismo teléfono, fuente con 30 días → 200 + unión de tags; fuente sin flag intacta). Sabotaje: revertir el cambio → exactamente los casos previstos en rojo.
- Variantes MX (`lib/channels/phone-variants.test.ts`): `+52229…` y `+521229…` → misma persona.
- Sincronía del bundle (rebuild-compare + ejecución en `vm` + vector de referencia en el doc).

---

## Fase 2 — Comentarios con palabra clave en video FB/IG · estimación 5-6 días

El CRM **no tiene** código de comentarios IG/FB → n8n escucha Meta Graph API; la lógica vive en el repo como módulo puro testeable (`lib/integrations/n8n/comentarios-meta.ts` + bundle).

### 2.1 Tabla de palabras clave (fuente única en el módulo, `ACOES_POR_PALAVRA`)

| Palabra | Acción del DM | Etiqueta |
|---|---|---|
| RESCATE | Link de pre-calificación + agendar | `#cta-rescate` |
| EQUIPO | Notificar a Miriam para contacto humano | `#cta-equipo` |
| URTICARIA | PDF guía + oferta de valoración en el mismo DM | `#cta-urticaria` |
| PRECISIÓN | Link de agenda para Prick test | `#cta-precision` |
| ONLINE | Explicación de consulta online | `#cta-online` |
| ORIGEN | Link de pre-calificación | `#cta-origen` |
| NORTE | Link de pre-calificación | `#cta-norte` |
| INVIERNO | Link de pre-calificación | `#cta-invierno` |
| FIESTAS | Link de pre-calificación | `#cta-fiestas` |
| FINAL | Link de pre-calificación | `#cta-final` |
| NARIZ | Link de pre-calificación | `#cta-nariz` |

Cada video del Dr. Julio usa **una** palabra clave distinta. Plantilla exacta del DM de URTICARIA (conversión del PDF en puerta de entrada, no destino):

> "Hola [Nombre]. Soy del consultorio del Dr. Julio.
> Te comparto la guía de urticaria crónica: [link PDF]
>
> Cuando la leas, si quieres que el Dr. Julio revise tu caso específico, responde SÍ y te agendo una valoración."

### 2.2 Flujo

```
[Meta Graph webhooks]  Página FB: campo feed · IG: campo comments   (verify_token en GET)
        ▼
[Webhook n8n] → [Code: ComentariosMeta.procesar]
  1. Extraer: texto del comentario, from_id, post/video_id (media_id en IG), nombre del usuario.
  2. Normalizar: MAYÚSCULAS, sin acentos, sin puntuación.
  3. Coincidencia por PALABRA COMPLETA (word boundary) contra las 11 claves → sin falsos positivos.
        ▼
¿Coincide? NO → [Log Sheets "revision_manual"] → FIN (no se responde; Miriam revisa)
        ▼ SÍ
[Anti-spam]  ignorar from_id == page_id (comentarios propios);
             cooldown 24h por usuario + dedup de usuarios ya procesados (Static Data + Sheets "usuarios_procesados");
             cap global: máx 195 DMs/hora (límite Meta) — contador local en Static Data, exceso → Wait
        ▼ pasa
(a) [HTTP Graph: respuesta pública ROTADA]  FB: POST /{comment_id}/replies · IG: POST /{comment_id}/replies
    ("Te enviamos la información por privado 📩" + 2-3 variantes)
(b) [HTTP Graph: DM específico de la palabra]  IG: POST /{ig_id}/messages · FB: Messenger API
(c) [POST al CRM webhook]  external_id=comment_id · full_name=nombre del comentarista · SIN teléfono
    (válido: la ruta exige nombre O teléfono O email) · custom_fields {palabra_clave, video_id, red_social,
    comment_id, perfil_url} · tags [#cta-<clave>]  → lead en "Pacientes Nuevos" / LEAD NUEVO
(d) [Log Sheets "comentarios"]  fecha, palabra clave, video de origen (id único de Meta), nombre, resultado de cada paso
```

- Sin teléfono, el contacto nace por nombre + ids sociales (identidad débil, aceptada); si después llega por WhatsApp, el dedup por teléfono no aplica (no hay qué casar) — unión manual por nombre/perfil_url (iteración 2: casar por `custom_fields.perfil_url`).
- "Link de pre-calificación" = URL del formulario de pre-calificación del consultorio (fuente webhook propia que ya ingiere al CRM); las URLs del PDF, agenda y consulta online van en variables de entorno de n8n.

### 2.3 Seguimiento post-PDF de URTICARIA (n8n, Sheets como estado)

- Webhook IG `messages` (DMs entrantes) → si el usuario responde **"SÍ"** → **MCP `crm_move_lead_stage`** → etapa CALIFICADO + notificar a Miriam (WAHA/Resend).
- Cron n8n (cada hora): leads URTICARIA sin respuesta → **48h**: DM recordatorio "¿Pudiste leer la guía? ¿Tienes alguna duda?"; **7 días**: MCP `crm_move_lead_stage` → pipeline "Nurturing", etapa CONTENIDO EDUCATIVO (DÍA 7).
- Estado por usuario en Sheets ("estado_seguimiento", "fecha_ultimo_dm") + Static Data como respaldo.

### 2.4 Meta for Developers (Fase 2)

Permisos: `pages_read_engagement`, `pages_manage_posts` (responder comentarios FB), `pages_messaging` (Messenger DM); IG: `instagram_basic`, `instagram_manage_comments`, `instagram_manage_messages`. Suscribir Página (feed) y cuenta IG (comments, messages) a la app → URLs del webhook n8n. **App Review suele exigirse para mensajería IG en producción** (screencast del flujo). Token: System User con Página + cuenta IG.

### 2.5 Tests (Fase 2)

Módulo `lib/integrations/n8n/comentarios-meta.ts` (+ bundle + test de sincronía, patrón F1):
- Normalización: "¡Urticaría!" / "URTICARIA" / "urticaria," → URTICARIA; acentos y puntuación fuera.
- Word boundary: "norte" matchea, "norteño" no; "nariz" matchea, "narices" no (fijar y testear el límite exacto); las 11 claves; sin match → null.
- DM por clave: cada clave produce su mensaje; plantilla URTICARIA exacta con nombre interpolado (fixture).
- Anti-spam (función pura `permitirEnvio(ultimoEnvio, agora, contadorHora)`): cooldown 24h, cap 195/h.
- Rotación de la respuesta pública (determinista, testea las N variantes).
- Fixtures reales de payloads de comentario FB (feed) e IG (comments). E2E manual: comentar cada palabra clave en videos de prueba → respuesta pública + DM correcto + lead con tag + fila en Sheets.

---

## Fase 3 — Click-to-WhatsApp · estimación 1-2 días (casi todo ya existe)

- **Atribución automática (ya existe)**: clic en anuncio CTWA → Meta referral (`ctwa_clid`) → `extrairAtribuicaoWaha` estampa `contacts.source_metadata.ad_*`; el lead de conversación nace con `source: meta_ads` + tag `Meta_ads`.
- **Lo que se agrega (configuración, sin código CRM)**:
  1. Regla `message.received` → condición `contact.source_metadata.ad_platform = "meta"` (o keyword del anuncio en `event.body_preview`, p. ej. "URTICARIA", si la condición sobre source_metadata no está expuesta en el RuleEditor) → `add_tag #click-whatsapp` + `call_webhook` → n8n log → Sheets.
  2. Regla espejo `lead.created` (condición `contact.tags contains #click-whatsapp`) → `add_tag` sobre el lead (si el orden del drain lo permite; si no, el tag queda en el contacto — verificar en E2E).
  3. El texto del anuncio define el mensaje pre-llenado del WhatsApp (plantilla con "URTICARIA" como referencia detectable).
- **La IA responde automáticamente** (canal `open` por defecto — verificar gate; si `allowlist`, `avaliarCampanha` autoriza por keyword de campaña).
- Tests: validación E2E (anuncio de prueba → lead con `Meta_ads` + `#click-whatsapp` + fila en Sheets). Sin cambios de código en el repo salvo docs.

---

## Fase 4 — Link en bio (mensaje espontáneo) · estimación 1-2 días

- **4 links wa.me diferenciados** con texto pre-escrito: `?text=Hola,%20vengo%20de%20Instagram` / `…de%20Facebook` / `…de%20TikTok` / `…de%20tu%20perfil` (o link corto del CRM con `[ref:…]` si se prefiere atribución formal).
- **Detección en CRM** (reglas de automatización, sin código): 4-5 reglas `message.received` → `event.body_preview contains "instagram"` → `add_tag #bio-instagram` (+ espejos facebook/tiktok/"tu perfil"→`#bio-directo`) + `call_webhook` → n8n log → Sheets. Reglas espejo en `lead.created` para propagar al lead.
- **Sin marcador** ("Hola" a secas): instrucción en el playbook del agente publicado: *"Si el primer mensaje no identifica origen, pregunta: ¿De dónde nos escribes? ¿Instagram, Facebook o TikTok?"* → la siguiente respuesta dispara la regla correspondiente.
- Alternativa con atribución formal: links rastreables del CRM (`ad_tracking_links`/`captura-de-clique`, `[dk1:…]`) — recomendable en iteración 2.
- Tests: E2E manual por red social (4 rutas) + verificación de la pregunta del agente en el caso sin texto.

---

## Fase 5 — Métricas de Meta Ads para el agente IA · estimación 4-6 días

Tres vías complementarias (la restricción de MCP externo — turns con contacto no cargan tools externos — define el reparto):

1. **Meta Ads MCP Server (vía 1, recomendada por el usuario)** — desplegar un MCP server de Meta Ads (open source) en el VPS; registrarlo en el CRM como `mcp_externo` (org-scoped, action `definirServidorMcpExterno`) → disponible para el agente en **turns sin contacto** (análisis, prospección) y para Miriam vía Claude Code. Tokens: System User `ads_read`.
2. **Herramienta MCP propia del CRM (nueva)** — `crm_meta_ads_insights` en `lib/mcp/tools/` reutilizando `insights.ts` + credencial `ad_insights_connections` (CPL, CTR, impresiones, alcance por campaña en rango de fechas). Así el agente consulta métricas **en cualquier turn** (incluso con paciente), con el permiso y el scope de la org. Incluye tests (tool en catálogo, respuesta de `lerInsights` con fixture del Graph).
3. **n8n Cron + Graph API (vía 3)** — snapshot diario de insights (HTTP Request con token `ads_read`) → Sheets (tabla "metricas_diarias"): campaña, fecha, spend, impressions, ctr, results, cost_per_result. Alimenta Fases 6-7 y sirve de histórico.
- "Business AI Breakdown": **no es una métrica estándar de Marketing API** — aclarar con el usuario (¿desglose por `results[].action_type`, ¿`conversion_values`?) antes de implementar.

---

## Fase 6 — Cruce de métricas con el CRM · estimación 3-4 días

**Lo que ya existe y se habilita primero**: reglas `meta_ads_conversion_rules` (evento Purchase en la etapa ganada) + ledger `ad_conversion_dispatches` → el CRM ya devuelve conversiones con valor a Meta cuando el lead tiene atribución → **ROAS nativo en Ads Manager** (configurar etapa ASISTIÓ/ganada → evento, valor = `value_cents`).

**El cruce nuevo (n8n, sin código CRM)**:
```
[Cron n8n diario 08:05 America/Mexico_City]
  ├─ leer métricas del día: Sheets "metricas_diarias" (F5.3) o Graph API directo
  ├─ leer leads del CRM: MCP Client n8n → crm_list_leads (filtro por tags de fuente + utm_* de source_metadata) → stage/status, value_cents, currency
  └─ calcular: CPL real (gasto ÷ leads que asistieron), tasa de conversión por anuncio (won÷leads por ad_id),
       ticket promedio por fuente (Σ value_cents won ÷ won, por moneda — nunca mezclar), ROI por campaña ((valor ganado − gasto) ÷ gasto)
        ▼
[Log Sheets "dashboard_roi"] + [resumen semanal WhatsApp a Miriam/Dr. (WAHA)] los lunes
```
- Unión por `utm_campaign`/`utm_content` (F1) y por `contacts.source_metadata.ad_*` (F3) — ambos ya quedan en el CRM.
- Iteración 2 (opcional, con código): tabla snapshot + ruta de métricas en el CRM para dashboard interno.

---

## Fase 7 — Alertas automáticas · estimación 2-3 días

| Alerta | Mecanismo | Canal |
|---|---|---|
| **CPL > $50 MXN por campaña** | n8n Cron diario post-métricas → comparar `cost_per_result` → si supera umbral | WAHA → WhatsApp a Miriam/Dr. Julio + Resend respaldo; opcional pausar campaña (`POST /v22.0/{campaign_id}` status=PAUSED — requiere scope `ads_management`) |
| **Conversión baja > 30%** | n8n compara ventana 7 días vs anterior (histórico de Sheets) → caída >30% | WAHA "revisar creativos" + Resend |
| **Lead de alta intención sin atender en 30 min** | n8n Cron cada 5 min → MCP `crm_list_leads` (etapas PRE-CALIFICACIÓN/CALIFICADO + tags `#interesado`/`#calificado` + sin actividad 30 min) → alerta | WAHA a Miriam |

- Alternativa para precisión <5 min: cron CRM al minuto (`app/api/v1/cron/facebook-sla` + entrada en `docker/scheduler/entrypoint.sh` + test guard `cron-routes-scheduled.test.ts`) + `call_webhook` → n8n notifica. Solo si 5 min no alcanza.
- Umbrales en variables de entorno de n8n (no hardcodeados). Anti-spam: máximo 1 alerta por campaña cada 24 h (Static Data).

---

## Deduplicación entre las 4 fuentes (reglas)

| Escenario | Mecanismo | Estado |
|---|---|---|
| F1 retry del mismo lead | `external_id = leadgen_id` (índice único org+source) | existe |
| F2 comentario repetido/webhook repetido | `external_id = comment_id` + dedup por `X-Webhook-Delivery` en log | n8n + CRM |
| F3/F4 mensajes repetidos/rafagas | `fn_nascer_lead_da_conversa` (lock por org+contact; 1 lead abierto por demanda) | existe |
| Webhook F1 → luego WhatsApp del mismo paciente | `garantirLeadDaConversa` ve lead abierto → `ja_existe` (no duplica; la conversación se cuelga del mismo contacto) | existe |
| WhatsApp → luego webhook F1 con el mismo teléfono | **Nuevo**: `dedup_por_contato_dias` opt-in → 200 con lead existente + unión de tags | Fase 1.4.2 |
| Dos formularios FB del mismo teléfono | idem (external_id distintos, teléfono igual) | Fase 1.4.2 |
| Grafías MX `+52229…` vs `+521229…` (webhook vs wa_id) | Variantes MX en `phoneLookupVariants` → un solo contacto | Fase 1.4.3 |

---

## Plan de implementación (orden y dependencias)

- **F0** Ritual de contribuidor (0.5 d) → habilita todo lo del repo.
- **F1** (4-5 d) — núcleo: ruta+tags, dedup opt-in, variantes MX, normalizador+bundle, tests, docs, workflows n8n, config Meta/CRM. Bloquea F2-F7.
- **F2** (5-6 d) — puede ir en paralelo con F1 (n8n/Meta puro) salvo el POST al CRM y el seguimiento vía MCP, que dependen de F1.
- **F3** (1-2 d) y **F4** (1-2 d) — configuración sobre F1 (log central, reglas) + E2E.
- **F5** (4-6 d) — MCP externo + tool CRM + snapshot n8n. Tras F1 (usa infra y convenciones).
- **F6** (3-4 d) — tras F5 (consume snapshots) y F1/F3 (datos en el CRM).
- **F7** (2-3 d) — tras F5/F6 (umbrales sobre métricas) + F1 (canales de aviso).
- **Total: ~20-28 días-hombre** (solapando F2/F3/F4 con revisión de F1).

## Verificación

```bash
pnpm typecheck && pnpm lint && pnpm test:unit     # suite completa (footer sin cortar es la autoridad)
pnpm checar:colisao-de-migration                 # tríada de migraciones
pnpm release:conferir                            # fragmento .changes
bash .agents/skills/deskcomm-contribuir/scripts/pre-voo.sh   # antes del PR
```
E2E por fase: F1 Lead Ads Testing Tool → lead en Kanban "LEAD NUEVO" con tags + fila Sheets + fallo forzado → reintentos → WhatsApp de Miriam; F2 comentario real en video de prueba; F3 anuncio CTWA de prueba; F4 cuatro links + "Hola" → pregunta de origen; F5/F6 métricas del día en Sheets + dashboard; F7 umbral forzado → alerta.

## Riesgos y preguntas abiertas

1. **`wa_id` MX real**: confirmar con un número mexicano cómo WAHA entrega el ID antes de fijar `+521…` como canónica (afecta dedup webhook↔WhatsApp). Plan B: canónica `+52` sin `1` + variantes.
2. **"Business AI Breakdown"** no es métrica estándar — aclarar qué dato se quiere (F5).
3. **Orden del drain** en reglas `message.received` → tag contacto → regla `lead.created` ve el tag: verificar en E2E; si no, el tag queda en el contacto (aceptable, documentar).
4. **`normalizarTags` y el `#`**: el CRM normaliza a minúsculas/trim/40 chars; verificar que conserva el `#` (vocabulario del usuario); si lo quita, decidir convención.
5. **App Review de Meta**: mensajería IG (F2) suele requerirla para producción; planificar screencast y tiempos de aprobación.
6. **Consentimiento**: el Instant Form debe llevar la pregunta de consentimiento (base legal para contactar y para `authorize_ai_on_capture`).
7. **Crypto node de n8n**: HMAC-SHA256 hex soportado; si el plumbing de salida es incómodo, fallback HMAC en el Code node con `crypto` del sandbox — decidir en el primer smoke test.
8. **Firma sobre body crudo**: el HTTP Request debe enviar el string exacto del Code node (nunca modo JSON); el E2E 401→200 lo detecta.
9. **PR del fork**: diff solo con archivos genéricos del producto; CLAUDE.md local del consultorio y secretos/tokens/chatIds NUNCA en el diff (viven en n8n env/credentials y BD del CRM).
10. **Pausar campañas (F7)** requiere `ads_management` y prudencia: dejarlo como acción opcional con confirmación, no automática en v1.
11. **Links de pre-calificación/PDF/agenda (F2)**: dependen de URLs del consultorio (el formulario de pre-calificación debe existir como fuente webhook propia que ya ingiere al CRM); van en variables de entorno de n8n — definirlos antes de F2.
12. **Estado del seguimiento post-PDF (F2)**: Sheets es el estado de los recordatorios 48h/7d; si falla la escritura se pierde el seguimiento → respaldo en Static Data + reintentos; iteración 2: moverlo al motor de follow-up del CRM cuando exista canal IG en el CRM.

## Archivos críticos

- `app/api/v1/webhooks/in/[token]/route.ts` — tags + dedup opt-in.
- `lib/webhooks/inbound.ts` — `extrairTagsDoPayload`.
- `lib/channels/phone-variants.ts` — variantes MX.
- `lib/integrations/n8n/facebook-lead-ads.ts`, `lib/integrations/n8n/comentarios-meta.ts` (nuevos) + `scripts/bundle-*.ts` + `n8n/*.node.js` + `n8n/workflows/*.json`.
- `supabase/migrations/NNNN_dedup_por_contato.sql` + espejo en `supabase/baseline.sql`.
- `lib/mcp/tools/` — tool `crm_meta_ads_insights` (F5).
- Tests: `tests/unit/…` (patrón `captacao-na-etapa-exigente-cria-o-lead.test.ts`), fixtures `tests/fixtures/facebook-lead-ads/`, `tests/fixtures/comentarios-meta/`.
- Docs: `docs/integracao/facebook-lead-ads.md`, `docs/integracao/comentarios-meta.md`, `docs/integracao/captacao-multicanal.md` (+ fila en `docs/index.md`), `.changes/` fragment.
