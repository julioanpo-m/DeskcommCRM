# Registro de Herramientas IA — Consultorio Dr. Julio Porras

**Versión:** 1.0 (borrador)
**Fecha:** 2026-10-11
**Estado:** En construcción — llenar TODOs conforme se evalúen nuevas herramientas

Registro único de herramientas de IA (y automatización adyacente) evaluadas,
adoptadas o descartadas para el sistema del consultorio. Objetivo: no repetir
evaluaciones y dejar constancia del porqué de cada decisión.

---

## Cómo usar este registro

Antes de adoptar una herramienta nueva:
1. Buscar aquí si ya fue evaluada.
2. Si no, evaluarla contra: costo, necesidad real, riesgo de datos de pacientes,
   facilidad de operación por una sola persona.
3. Registrar el resultado con fecha y decisión.

---

## Registro

| Herramienta | Proveedor | Rol en el sistema | Estado | Costo aprox. | Decisión / Notas |
|---|---|---|---|---|---|
| **Claude** (agente "Calificado") | Anthropic | Agente IA de producción que atiende pacientes por WhatsApp (pre-calificación) | ✅ **Adoptada** | Por consumo de tokens | Única IA que habla con pacientes. Ver `docs/03-crm/prompts-agentes.md`. |
| **DeepSeek v4 Pro** | DeepSeek | IA de desarrollo (vía cc-use / ECC, para programar) | ✅ **Adoptada** | Por consumo de tokens | Solo para código y documentación; nunca toca datos de pacientes. |
| **OpenAI GPT-4o mini** | OpenAI | (histórico) agente de conversación | ❌ **Descartada** | ~$5 USD/mes estimado | Evaluada en la arquitectura histórica (`estructura-proyecto.md`). Se descartó al adoptar Claude para producción. |
| **Claude Haiku** | Anthropic | (histórico) alternativa de bajo costo | ❌ **Descartada** | Bajo | Misma razón: la producción quedó en Claude (modelo completo) y el desarrollo en DeepSeek. |
| **n8n** | n8n (self-hosted) | Orquestador de captación multicanal (webhooks Meta, normalización, alertas) — no es IA generativa, es automatización | 🟡 **Planeada** | 0 (contenedor en VPS) | Ver `docs/05-trabajo/plan-captacion.md` y `docs/plan-captacion-futura.md`. |
| **Meta Ads MCP Server** | Comunidad open source | Dar métricas de anuncios al agente IA en turnos sin contacto | 🟡 **Planeada** | 0 (desplegado en VPS) | Fase 5 del plan de captación. |
| **WAHA** | DevLikePro | API de WhatsApp (canal, no IA) | ✅ **Adoptada** | 0 (self-hosted) | Listada aquí por ser el canal que alimenta al agente IA. |

---

## Criterios de evaluación (estándar del consultorio)

1. **Datos de pacientes:** la herramienta de producción jamás debe enviar datos
   clínicos a un tercero sin consentimiento (ver reglas de negocio en CLAUDE.md).
2. **Operación por una sola persona:** Dr. Julio o Miriam deben poder operarla
   sin un equipo técnico permanente.
3. **Costo marginal:** se prefiere self-hosted cuando el volumen lo justifica.
4. **Separación de roles:** producción (habla con pacientes) ≠ desarrollo
   (escribe código). Ninguna herramienta cruza ambos roles sin decisión explícita.

---

## TODOs

- [ ] Evaluar herramienta de transcripción de consultas (si el Dr. Julio quiere
      registrar notas clínicas por voz). Candidatos por definir.
- [ ] Evaluar generación de imágenes/video con IA para el blueprint (Regla 18:
      solo fondos y B-roll; declaración obligatoria).
- [ ] Registrar costos mensuales reales de cada herramienta adoptada
      (facturación de Anthropic y DeepSeek).
- [ ] Documentar credenciales de lectura de Meta Ads (`ad_insights_connections`).
