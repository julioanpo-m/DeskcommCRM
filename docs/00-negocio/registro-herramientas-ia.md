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
| **n8n** | n8n (self-hosted) | Orquestador de captación multicanal (webhooks Meta, normalización, alertas) — no es IA generativa, es automatización | ✅ **Instalada** | 0 (contenedor en VPS) | Instalada 2026-10-12 en `/root/n8n/` de la VPS (proyecto Compose aparte, red interna del CRM, puerta basic auth en Caddy). Ver `docs/04-tecnico/arquitectura-real.md`. |
| **Meta Ads MCP Server** | Comunidad open source | Dar métricas de anuncios al agente IA en turnos sin contacto | 🟡 **Planeada** | 0 (desplegado en VPS) | Fase 5 del plan de captación. |
| **WAHA** | DevLikePro | API de WhatsApp (canal, no IA) | ✅ **Adoptada** | 0 (self-hosted) | Listada aquí por ser el canal que alimenta al agente IA. |
| **CapCut Pro** | ByteDance | Edición de video del banco modular (5 avatares) | ✅ **Adoptada** | ~$40 USD/año | Decisión 2026-10-11: comprar Pro **individual** (NO Teams). Velocidad de edición para contenido semanal. |
| **Canva Pro** | Canva | Cintillos, disclaimers, portadas y creativos PAG | ✅ **Adoptada** | ~$40 USD/año | Decisión 2026-10-11: comprar Pro **individual** (NO Teams). Equipo de 2 no justifica Teams. |
| **DaVinci Resolve Free** | Blackmagic | Respaldo de edición: cátedras largas, acabados, IA local | ✅ **Adoptada** | $0 (Free) | Instalar gratis. Studio ($295 único) solo si el video se vuelve canal principal. |
| **n8n Starter (nube)** | n8n | (evaluada) orquestador en la nube | ❌ **Descartada** | €20/mes | No comprar: se corre self-hosted en la VPS, sin límite de ejecuciones y con datos médicos en servidor propio. |
| **PostHog Scale** | PostHog | (evaluada) analytics de producto SaaS | ❌ **Descartada** | — | Oferta de $10 vs $750/mes oficial = cuenta robada/compartida; además es analytics de producto, no de consultorio. |
| **Canva Teams** | Canva | (evaluada) plan para equipos | ❌ **Descartada** | — | 500 miembros para 2 personas no tiene sentido; cuentas Teams compartidas son de las más baneadas. |
| **Figma Professional EDU** | Figma | (evaluada) diseño de UI | ❌ **Descartada** | — | No hay educador verificado ni diseño de UI propio (DeskcommCRM ya existe); Figma Free alcanza. |
| **ElevenLabs Creator** | ElevenLabs | (evaluada) voz sintética/clonada | ❌ **Descartada** | $22/mes | Voz clonada daña la confianza médica; la voz real del Dr. Julio es gratuita y auténtica. |

---

## Decisión final de herramientas (2026-10-11)

- **COMPRAR:** CapCut Pro individual (~$40/año) + Canva Pro individual (~$40/año).
- **INSTALAR GRATIS:** DaVinci Resolve Free + n8n self-hosted en la VPS.
- **NO COMPRAR:** n8n Starter, PostHog, Canva Teams, Figma EDU, ElevenLabs.

**Razones:**
- CapCut Pro: velocidad de edición para contenido semanal de 5 avatares.
- Canva Pro: diseño de cintillos, disclaimers y portadas (equipo de 2 no justifica Teams).
- n8n self-hosted: ya hay VPS con Docker; sin límite de ejecuciones y datos médicos en servidor propio.
- PostHog: precio 75x sospechoso (cuenta robada/compartida).
- ElevenLabs: voz clonada daña la confianza médica.

**Regla de seguridad (LFPDPPP):** nunca conectar estas herramientas al flujo de
datos de pacientes. Solo para contenido y anuncios.

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
