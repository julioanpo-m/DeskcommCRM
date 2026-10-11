# Arquitectura de Lead Management — Consultorio Dr. Julio Porras

**Versión:** 1.0
**Fecha:** 2026-10-11
**Fuentes:** CLAUDE.md (reglas de negocio) + `docs/plan-captacion-futura.md` (hechos verificados en código) + `docs/04-tecnico/arquitectura-real.md`
**Estado:** Vigente

Este documento describe el flujo completo del CRM: cómo entra un lead, quién lo
atiende, cómo se califica, cómo se cierra y cómo se mide. Los detalles técnicos de
ingesta (webhooks, dedup, atribución) viven en `docs/plan-captacion-futura.md`.

---

## 1. Visión operativa

- **Captar** pacientes por WhatsApp y canales de Meta.
- **Pre-calificar** con el agente IA "Calificado" (Claude).
- **Cerrar** con seguimiento humano de Miriam.
- **Agendar** en Google Calendar y **medir** cruce de anuncios × resultados.

**Actores:**

| Actor | Rol |
|---|---|
| Agente IA "Calificado" (Claude) | Conversación inicial, pre-calificación, nurturing. Nunca da precios ni diagnósticos. |
| Miriam | Cierre humano, agenda, precios, revisión de casos dudosos, no-shows. |
| Dr. Julio Porras | Consulta médica (el producto final). |
| Sistema (CRM + n8n + WAHA) | Ingesta, dedup, etiquetas, recordatorios, métricas. |

---

## 2. Flujo de vida del lead

```
                      ┌──────────────────────────────────────────────┐
  CANALES DE ENTRADA  │                                              │
  ─────────────────   │   WhatsApp CTWA ──┐                          │
  Lead Ads (n8n)  ────┼──▶ Webhook CRM ────┼──▶ INGESTA ─▶ DEDUP ─┐  │
  Comentarios (n8n) ──┤   Link en bio ─────┘        │            │  │
  Mensaje espontáneo ─┘                              ▼            │  │
                                          ┌───────────────────┐   │  │
                                          │ LEAD NUEVO        │◀──┘  │
                                          │ + etiqueta fuente │      │
                                          └─────────┬─────────┘      │
                                                    ▼                │
                                          ┌───────────────────┐      │
                                          │ PRE-CALIFICACIÓN  │      │
                                          │ Agente IA (Claude)│      │
                                          │ máx 3 líneas, 1   │      │
                                          │ pregunta por msg  │      │
                                          └────┬─────────┬────┘      │
                                  ¿califica? │         │ ¿no?        │
                                             ▼         ▼             │
                                   ┌──────────────┐ ┌─────────────┐  │
                                   │ CALIFICADO   │ │ NURTURING   │  │
                                   │ → Miriam     │ │ (Pipeline 2)│  │
                                   └──────┬───────┘ └─────────────┘  │
                                          ▼                          │
                                   ┌──────────────┐                  │
                                   │ INTENTO DE   │ Miriam ofrece   │
                                   │ CIERRE       │ paquete $3,900  │
                                   └──────┬───────┘                  │
                                          ▼                          │
                                   ┌──────────────┐                  │
                                   │ CITA         │ Google Calendar │
                                   │ CONFIRMADA   │                  │
                                   └──┬────────┬──┘                  │
                             ¿asistió? │        │ ¿no asistió?       │
                                       ▼        ▼                    │
                              ┌─────────────┐ ┌───────────────────┐  │
                              │ ASISTIÓ     │ │ NO ASISTIÓ        │  │
                              │ (Ganado)    │ │ → remarketing     │  │
                              └──────┬──────┘ │   #no-asistió-N   │  │
                                     ▼        └───────────────────┘  │
                              Pipeline 3 (Pacientes Activos)         │
                              ───────────────────────────────        │
                                                                     │
                              Conversión Purchase → Meta (ROAS)      │
                              └──────────────────────────────────────┘
```

---

## 3. Ingesta y deduplicación (resumen técnico)

Detalle completo en `docs/plan-captacion-futura.md`. Puntos clave:

- **Webhook** `POST /api/v1/webhooks/in/[token]` — token por fuente, firma HMAC-SHA256,
  `external_id` para idempotencia (retry devuelve el mismo `lead_id`).
- **WhatsApp** WAHA → `handleInbound` → la conversación puede hacer nacer el lead
  (`garantirLeadDaConversa`) con dedup por contacto (1 lead abierto por demanda).
- **Teléfonos MX:** n8n siempre envía E.164 `+52…` (el CRM normaliza como BR por
  defecto; variantes MX en plan de captación, Fase 1.4.3).
- **Atribución de anuncios:** `extrairAtribuicaoWaha` estampa `source_metadata.ad_*`
  y el tag `Meta_ads` en clics de CTWA.
- **Dedup entre fuentes:** `dedup_por_contato_dias` (opt-in por fuente) → un mismo
  teléfono no duplica lead abierto en el mismo pipeline.

---

## 4. Reglas del agente IA (resumen)

Las reglas completas y el prompt viven en `docs/03-crm/prompts-agentes.md`. Invariantes:

| Regla | Detalle |
|---|---|
| Formato | Máx. 3 líneas por mensaje. Una sola pregunta. |
| Tono | Llama al paciente por su nombre de pila. |
| Diagnóstico | **Nunca.** Derivar siempre a consulta. |
| Cura | **Nunca prometer.** Usar "tratar la causa". |
| Tratamiento | **Nunca mencionar** medicamentos ni dosis. |
| Precios | **Nunca** sin confirmar con Miriam. |
| Horarios | L-V 9:00-18:00, Sáb 9:00-13:00. Nada antes de 8:00 ni después de 20:00. |
| Frecuencia | Nurturing: máx. 1 msg/semana. Sin respuesta: máx. 3 msgs en 7 días. |
| Opt-out | "No me contactes" → archivar de inmediato. |
| Emergencia | "Llama al 911 o acude al hospital más cercano". |
| Identidad | Si preguntan directamente, decir que es un asistente. |

---

## 5. Etiquetas automáticas

| Categoría | Etiquetas |
|---|---|
| Fuente | `#urticaria`, `#pag-urticaria`, `#bio`, `#dm`, `#referido`, `#directo` |
| Clasificación | `#lead-nuevo`, `#pre-calificado`, `#crónico`, `#agudo`, `#frío` |
| Cierre | `#interesado`, `#cerrado`, `#asistió`, `#no-asistió`, `#activo` |
| Remarketing | `#no-asistió-1`, `#no-asistió-2`, `#no-asistió-3`, `#no-asistió-nurturing` |

---

## 6. Métricas y cruce con anuncios

- El CRM devuelve conversiones a Meta (`Purchase` en etapa ganada) → ROAS nativo
  en Ads Manager.
- Cruce periódico (n8n): CPL real, tasa de conversión por anuncio, ticket promedio
  por fuente, ROI por campaña. Detalle: Fases 5-7 de `docs/plan-captacion-futura.md`.
- Alertas: CPL > $50 MXN, caída de conversión > 30%, lead de alta intención sin
  atender en 30 min (avisa a Miriam).

---

## 7. Horarios y cumplimiento

- Zona horaria: America/Mexico_City.
- Nunca enviar mensajes fuera del horario permitido.
- Consentimiento obligatorio en formularios (Lead Ads) antes de contactar o de
  autorizar IA sobre captura (`authorize_ai_on_capture`).

---

## TODOs

- [ ] Diagrama de estados formal (máquina de estados) cuando el CRM lo exponga.
- [ ] Definir política de retención de historial de conversaciones (LGPD/datos personales).
- [ ] Definir escalamiento cuando el agente IA detecta caso complejo (fuera de
      urticaria/alergia ambulatoria) → criterio de derivación a otro especialista.
