# Plan de Captación Multicanal — Consultorio Dr. Julio Porras

**Versión:** 1.0 (entrada canónica)
**Fecha:** 2026-10-11
**Estado:** En ejecución (Fase 2 en curso)

> **Documento detallado (fuente completa):** `docs/plan-captacion-futura.md`
> (7 fases con flujos n8n, contrato del webhook, mapeo de campos, tests y riesgos).
> Este archivo es el resumen ejecutivo canónico en la estructura `05-trabajo/`.
> En una limpieza futura, el documento detallado debe moverse aquí y conservarse
> un solo archivo (ver `docs/05-trabajo/auditoria-estructura-v1.md`).

---

## Resumen ejecutivo

Captar pacientes por **4 canales de Meta** (Lead Ads, comentarios con palabra clave,
Click-to-WhatsApp, link en bio), insertarlos en el pipeline "Pacientes Nuevos" →
LEAD NUEVO, etiquetarlos por fuente, dejarlos en manos del agente IA (Claude) y
cruzar métricas de Meta Ads con resultados reales del CRM.

**Reparto de responsabilidades:**

| Capa | Qué hace | Dónde |
|---|---|---|
| **n8n** (self-hosted en VPS) | Listeners Meta (leadgen, comentarios, DM), normalización, firma y POST al CRM, log a Google Sheets, métricas programadas, alertas | Contenedor n8n |
| **CRM** (DeskcommCRM) | Ingesta webhook con dedup, nacimiento de leads de chat, atribución CTWA, agente IA (Claude), reglas de automatización, conversiones offline a Meta | Repo del fork |
| **Config operativa** | Reglas de automatización, fuente webhook, links wa.me, credenciales | CRM UI / n8n / Meta Business |

---

## Las 7 fases

| Fase | Nombre | Estimación | Estado |
|---|---|---|---|
| F0 | Ritual de contribuidor (fork) | 0.5 d | — |
| F1 | Facebook Lead Ads (leadgen) | 4-5 d | — |
| F2 | Comentarios con palabra clave (FB/IG) | 5-6 d | **En curso** |
| F3 | Click-to-WhatsApp (casi todo existe) | 1-2 d | — |
| F4 | Link en bio (mensaje espontáneo) | 1-2 d | — |
| F5 | Métricas de Meta Ads para el agente IA | 4-6 d | — |
| F6 | Cruce de métricas con el CRM (CPL/ROAS) | 3-4 d | — |
| F7 | Alertas automáticas | 2-3 d | — |

**Total: ~20-28 días-hombre** (solapando F2/F3/F4 con revisión de F1).

---

## Contrato del webhook (resumen)

`POST /api/v1/webhooks/in/<token>` — JSON con `external_id` (idempotencia),
`full_name`, `phone_number` (E.164 `+52…`), `utm_*` y `tags`. Firma
HMAC-SHA256 con `X-Deskcomm-Signature`. Dedup por fuente opt-in
(`dedup_por_contato_dias`).

## Palabras clave de comentarios (F2)

RESCATE · EQUIPO · URTICARIA · PRECISIÓN · ONLINE · ORIGEN · NORTE · INVIERNO ·
FIESTAS · FINAL · NARIZ — un video = una palabra clave.

---

## Verificación

```bash
pnpm typecheck && pnpm lint && pnpm test:unit
pnpm checar:colisao-de-migration
pnpm release:conferir
bash .agents/skills/deskcomm-contribuir/scripts/pre-voo.sh
```

## Riesgos principales (detalle en el documento completo)

1. `wa_id` MX real por confirmar (canónica `+521…`).
2. App Review de Meta para mensajería IG (tiempos).
3. Consentimiento obligatorio en formularios.
4. Firma sobre body crudo (nunca re-serializar en n8n).
