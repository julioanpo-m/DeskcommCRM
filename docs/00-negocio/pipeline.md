# Pipeline de Pacientes — Consultorio Dr. Julio Porras

**Versión:** 1.0
**Fecha:** 2026-10-11
**Fuente:** CLAUDE.md (reglas de negocio) + Sistema DeskcommCRM
**Estado:** Vigente

El consultorio opera **tres pipelines (embudos)** en DeskcommCRM. Cada lead vive en
exactamente un pipeline a la vez; al cerrarse una etapa final puede reingresar en otro.

---

## Pipeline 1 — Pacientes Nuevos (captación)

Objetivo: convertir un contacto entrante en un paciente con cita confirmada.

| # | Etapa | Qué ocurre | Responsable |
|---|---|---|---|
| 1 | **LEAD NUEVO** | El contacto ingresa vía WhatsApp (WAHA), webhook (n8n/Meta), comentarios con palabra clave o link en bio. Se crea el lead y se etiqueta por fuente. | Sistema (automático) |
| 2 | **PRE-CALIFICACIÓN (IA)** | El agente IA "Calificado" (Claude) conversa con el paciente: nombre de pila, síntomas, tiempo de evolución. Máximo 3 líneas por mensaje, una sola pregunta. | Agente IA |
| 3 | **CALIFICADO** | Cumple criterios (p. ej. urticaria > 6 semanas sin diagnóstico). La conversación pasa a humano para el cierre. | Agente IA → Miriam |
| 4 | **INTENTO DE CIERRE** | Miriam toma la conversación y ofrece el paquete inicial ($3,900 MXN). El agente IA nunca habla de precios. | Miriam |
| 5 | **CITA CONFIRMADA** | Fecha y hora acordadas. Se registra en la agenda (Google Calendar). | Miriam |
| 6 | **ASISTIÓ (Ganado)** | El paciente llega a la consulta. El lead se gana. | — |
| 7 | **NO ASISTIÓ (Perdido)** | No llega a la cita. Entra al flujo de remarketing de no-shows. | — |

### Criterios de pre-calificación

- **Urticaria:** > 6 semanas de evolución, sin diagnóstico previo, historia + fotos.
  El workup es limitado (2-3 estudios básicos), no pruebas extensas.
- **Pediatría (avatar K):** síntomas recurrentes, tos persistente, sospecha de
  marcha atópica.
- **Regla general:** el agente IA **nunca diagnostica**. Solo califica y deriva a consulta.

### Entrada al pipeline

| Canal | Mecanismo | Etiqueta de fuente |
|---|---|---|
| WhatsApp (anuncio CTWA) | WAHA → mensaje entrante | `#dm` + atribución `Meta_ads` |
| Lead Ads (Facebook) | n8n → webhook CRM | `#pag-urticaria` / `#fb-lead-ads` |
| Comentarios con palabra clave | n8n → Graph API → webhook CRM | `#pag-urticaria` / `#cta-*` |
| Link en bio | WhatsApp con texto pre-escrito | `#bio` |
| Referido | Mención del paciente | `#referido` |
| Directo | Llega sin fuente identificable | `#directo` |

---

## Pipeline 2 — Nurturing (seguimiento de no cerrados)

Objetivo: mantener contacto educativo con leads que no cerraron cita, sin presionar.

| # | Etapa | Día | Acción |
|---|---|---|---|
| 1 | **CONTENIDO EDUCATIVO** | 7 | Pieza educativa del banco (blueprint v11) |
| 2 | **CASOS DE ÉXITO** | 14 | Testimonio o caso (anónimo) |
| 3 | **PREGUNTA ABIERTA** | 21 | Mensaje que invite a responder |
| 4 | **OFERTA ONLINE** | 30 | Invitación a consulta online (avatar U) |
| 5 | **GUÍA DESCARGABLE** | 45 | PDF de control (p. ej. guía de urticaria) |
| 6 | **ÚLTIMA OFERTA** | 60 | Último intento de cierre |
| 7 | **FRÍO** | 90+ | Sin respuesta sostenida. Se archiva el flujo activo. |

### Reglas de nurturing

- Máximo **1 mensaje por semana**.
- Máximo **3 mensajes en 7 días** si el paciente no responde.
- Si pide no ser contactado → **archivar de inmediato**.
- Nunca enviar mensajes antes de las 8:00 ni después de las 20:00.

---

## Pipeline 3 — Pacientes Activos (tratamiento)

Objetivo: seguimiento clínico-comercial del paciente que ya está en tratamiento.

| # | Etapa | Qué ocurre |
|---|---|---|
| 1 | **PRIMERA CONSULTA** | Valoración inicial. Se decide plan. |
| 2 | **INICIO ITA** | Comienza inmunoterapia (subcutánea o sublingual). |
| 3 | **EN TRATAMIENTO** | Mensualidades activas. Seguimiento por mensaje según plan. |
| 4 | **REVISIÓN TRIMESTRAL** | Cita de control cada 3 meses. |
| 5 | **REVISIÓN ANUAL** | Evaluación completa del año de tratamiento. |
| 6 | **ALTA** | Fin del tratamiento. El paciente se archiva como histórico. |

---

## Etiquetas automáticas (vocabulario oficial)

| Categoría | Etiquetas |
|---|---|
| Fuente | `#urticaria`, `#pag-urticaria`, `#bio`, `#dm`, `#referido`, `#directo` |
| Clasificación | `#lead-nuevo`, `#pre-calificado`, `#crónico`, `#agudo`, `#frío` |
| Cierre | `#interesado`, `#cerrado`, `#asistió`, `#no-asistió`, `#activo` |
| Remarketing | `#no-asistió-1`, `#no-asistió-2`, `#no-asistió-3`, `#no-asistió-nurturing` |

---

## Remarketing de no-shows

Cuando un lead llega a **NO ASISTIÓ**, entra al flujo de reactivación por toques:

1. `#no-asistió-1` — primer intento de reprogramación.
2. `#no-asistió-2` — segundo intento.
3. `#no-asistió-3` — tercer intento.
4. `#no-asistió-nurturing` — pasa al Pipeline 2 (Nurturing).

> **TODO:** definir tiempos y mensajes exactos de cada toque de reactivación
> (guion de Miriam). Ver también `docs/03-crm/manual-miriam.md`.

---

## Reglas transversales (aplican a los 3 pipelines)

- Idioma: español (México). Moneda: MXN. Zona horaria: America/Mexico_City.
- Horario de atención: L-V 9:00-18:00, Sábado 9:00-13:00.
- El agente IA opera la pre-calificación; el cierre y los precios son de Miriam.
- Si el paciente menciona emergencia: "Llama al 911 o acude al hospital más cercano".
- Si pregunta si es un humano: el agente debe decir que es un asistente.
