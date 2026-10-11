# Auditoría de la Estructura Documental — v1

**Versión:** 1.0
**Fecha:** 2026-10-11
**Ejecutada por:** Claude Code (asistente IA)
**Solicitada por:** Dr. Julio Porras
**Alcance:** `docs/` (estructura del consultorio) + referencias de CLAUDE.md

---

## Resumen ejecutivo

La documentación del consultorio vivía en 3 archivos de la estructura numerada,
un índice que referenciaba 6 archivos inexistentes, y el plan de captación
detallado suelto en la raíz de `docs/`, rodeado de ~330 archivos heredados del
proyecto upstream DeskcommCRM. Esta auditoría inventarió todo, identificó huecos
y contradicciones (incluida la de stack técnico, aclarada por el Dr. Porras), y
creó la base documental: **11 archivos nuevos y 2 ediciones**, sin borrar ni
sobrescribir nada existente.

---

## 1. Inventario inicial (lo que existía)

| Archivo | Estado |
|---|---|
| `docs/index.md` | Índice nuevo del consultorio. Referenciaba 6 archivos inexistentes. |
| `docs/01-contenido/blueprint-maestro-v11.md` | ✅ Completo. Sistema de contenido v11.0 (19 reglas, 14 ejes, banco modular, 5 formatos, métricas). |
| `docs/02-anuncios/blueprint-pag-v1.md` | ✅ Completo. Sistema PAG v1.0 (compliance por plataforma, 10 anuncios, presupuestos). |
| `docs/04-tecnico/estructura-proyecto.md` | ⚠️ Arquitectura histórica pre-DeskcommCRM; contradecía CLAUDE.md. Marcado como histórico. |
| `docs/plan-captacion-futura.md` | ✅ Alta calidad, verificado contra código. Mal ubicado (raíz). |
| `docs/original-deskcomm.md` | Índice original del upstream (portugués), rescatado antes del reemplazo. |
| ~330 archivos heredados | Documentación del upstream (`architecture/`, `design-system/`, `doctrine/`, `specs/`, etc.). |

**Carpetas de la estructura nueva que no existían:** `00-negocio/`, `03-crm/`, `05-trabajo/`.

---

## 2. Hallazgos de la auditoría

### 2.1 ¿Qué estaba bien?

- Los dos blueprints (`maestro-v11` y `pag-v1`): completos, versionados y
  autoconsistentes entre sí.
- `plan-captacion-futura.md`: el documento técnicamente más sólido (hechos
  verificados contra el código del fork).
- CLAUDE.md como fuente de reglas de negocio: consistente con el CRM real.

### 2.2 ¿Qué faltaba?

- Toda la carpeta `00-negocio/` (pipeline, precios, avatares, visión).
- Toda la carpeta `03-crm/` (arquitectura lead management, manual de Miriam,
  prompts del agente).
- `arquitectura-real.md` en `04-tecnico/`.
- `vision.md` — sigue pendiente (no estaba en el listado de archivos a crear).

### 2.3 ¿Qué estaba duplicado o mal ubicado?

1. **`docs/plan-captacion-futura.md`** en la raíz → la entrada canónica ahora es
   `docs/05-trabajo/plan-captacion.md`. *Recomendación:* en una limpieza futura,
   mover el detalle a `05-trabajo/` y conservar un solo archivo.
2. **`docs/original-deskcomm.md`** en la raíz → candidato a una carpeta
   `docs/legacy/` junto con el resto de la documentación heredada del upstream.
3. **`estructura-proyecto.md`** desactualizado → resuelto: marcado como
   DOCUMENTO HISTÓRICO (ver 2.4).
4. **~330 archivos del upstream** mezclados con la documentación del consultorio
   en la misma carpeta `docs/` → *recomendación:* separación futura
   (`docs/legacy-upstream/` o similar) para que `docs/` refleje solo el consultorio.

### 2.4 Contradicción de stack (aclarada por el Dr. Porras)

- `estructura-proyecto.md` describía una arquitectura idealizada
  (DeskcommCRM + PostgreSQL directo + OpenAI GPT-4o mini + Python + Backblaze).
- La realidad (CLAUDE.md): **DeskcommCRM como producto completo** (fork), con
  Next.js 16 + Supabase integrados, WAHA, Claude en producción, DeepSeek v4 Pro
  en desarrollo, VPS Hostinger + Docker Compose, ~$35 USD/mes.
- **Resolución:** nota de documento histórico al inicio de `estructura-proyecto.md`
  + creación de `04-tecnico/arquitectura-real.md` como fuente de verdad.
- Verificación adicional: las rutas declaradas en CLAUDE.md (`app/`, `workers/`,
  `supabase/`, `tests/`, `docker-compose.prod.yml`, `hostgator-setup-kit/`)
  **sí existen** en el repositorio.

### 2.5 Información crítica que faltaba en CLAUDE.md

- La sección "Documentos de referencia" apuntaba a 3 archivos inexistentes
  (resuelto con esta tarea; queda pendiente `docs/03-crm/` sí existe ahora).
- No declaraba el flujo de contribuidor del fork (tríada de migración,
  `pre-voo.sh`) — quedó documentado en `arquitectura-real.md` §8.
- No mencionaba el costo real (~$35 USD/mes) ni los blueprints/avatares.

---

## 3. Archivos creados (11)

| # | Archivo | Tipo de contenido |
|---|---|---|
| 1 | `docs/00-negocio/pipeline.md` | Completo — 3 pipelines con etapas, canales de ingesta, etiquetas, reglas transversales |
| 2 | `docs/00-negocio/precios.md` | Completo — 4 productos, regla de oferta ($3,900 primero), snippets, costos internos |
| 3 | `docs/00-negocio/avatares.md` | Completo — matriz K/P/A/U/_ + repositorio de frases reales |
| 4 | `docs/00-negocio/registro-herramientas-ia.md` | Esqueleto con entradas conocidas (Claude, DeepSeek, OpenAI descartada, n8n, WAHA, MCP Meta Ads) + TODOs |
| 5 | `docs/03-crm/arquitectura-lead-management.md` | Completo — flujo Lead→Pre-calificación→Cierre, ingesta/dedup, reglas del agente, métricas |
| 6 | `docs/03-crm/manual-miriam.md` | Esqueleto con hechos confirmados (rol, alertas, reglas) + TODOs para validar con Miriam |
| 7 | `docs/03-crm/prompts-agentes.md` | Borrador v1 — reglas inquebrantables, prompts por etapa, respuestas críticas, pruebas obligatorias |
| 8 | `docs/05-trabajo/plan-captacion.md` | Resumen ejecutivo canónico + enlace al detalle (`docs/plan-captacion-futura.md`) |
| 9 | `docs/05-trabajo/validacion-tesis-v1.md` | Esqueleto — tesis central, 6 sub-hipótesis, supuestos, riesgos + TODOs |
| 10 | `docs/04-tecnico/arquitectura-real.md` | Completo — stack real, topología, integraciones, costos, flujo de contribución al fork |
| 11 | `docs/05-trabajo/auditoria-estructura-v1.md` | Este documento |

## 4. Ediciones aplicadas (2, autorizadas)

1. `docs/04-tecnico/estructura-proyecto.md` — nota de DOCUMENTO HISTÓRICO al inicio.
2. `docs/index.md` — índice actualizado con los 11 archivos, la distinción
   histórico/real y los documentos auxiliares de la raíz.

**Reglas cumplidas:** sin código de producción; solo Markdown en `docs/`;
ningún archivo existente fue borrado ni sobrescrito (solo las 2 ediciones
autorizadas); nomenclatura kebab-case.

---

## 5. Pendientes y recomendaciones

**Pendientes (requieren decisión o trabajo humano):**
- [ ] Crear `docs/00-negocio/vision.md` (referenciado en el índice; fuera del
      listado de esta tarea).
- [ ] Validar `manual-miriam.md` y `prompts-agentes.md` con Miriam y el Dr. Julio.
- [ ] Llenar TODOs de `registro-herramientas-ia.md` y `validacion-tesis-v1.md`.
- [ ] Generar `docs/05-trabajo/auditoria-claude-v1.md`.

**Recomendaciones:**
1. Mover `docs/plan-captacion-futura.md` → `docs/05-trabajo/` y dejar una sola
   copia (evitar divergencia entre resumen y detalle).
2. Crear `docs/legacy/` y mover ahí `original-deskcomm.md` y la documentación
   heredada del upstream, para que `docs/` refleje solo el consultorio.
3. Agregar a CLAUDE.md: costo real (~$35 USD/mes), enlace a `arquitectura-real.md`,
   y el flujo de contribuidor del fork.
4. Mantener la convención: todo documento nuevo lleva cabecera con versión,
   fecha y estado (Vigente / Borrador / Histórico).
