# Manual Operativo de Miriam — Consultorio Dr. Julio Porras

**Versión:** 1.0 (borrador)
**Fecha:** 2026-10-11
**Estado:** Esqueleto con hechos confirmados — completar TODOs con Miriam

Miriam es la persona encargada del **seguimiento humano** del CRM: cierre de
pacientes calificados, agenda, precios y manejo de no-shows. El agente IA
"Calificado" (Claude) hace la pre-calificación; **nada de dinero se cierra sin Miriam**.

---

## 1. Rol y responsabilidades (confirmadas)

- Tomar la conversación cuando el lead pasa a **CALIFICADO** (handoff IA → humano).
- Ofrecer el **paquete inicial ($3,900 MXN)** primero. La valoración ($1,000 MXN)
  solo si el paciente duda o pide segunda opinión. Ver `docs/00-negocio/precios.md`.
- **Única autorizada** para comunicar precios y descuentos.
- Agendar citas (martes o jueves) en Google Calendar.
- Atender alertas del sistema (ver sección 4).
- Revisar comentarios de redes que no matchean palabra clave (`revision_manual`).

## 2. Horario

- Atención: L-V 9:00-18:00, Sábado 9:00-13:00 (horario del consultorio).
- Nunca enviar mensajes antes de las 8:00 ni después de las 20:00.

## 3. Herramientas

- **CRM (DeskcommCRM):** kanban del pipeline "Pacientes Nuevos", etiquetas, historial.
- **WhatsApp (WAHA):** mensajes y alertas entrantes.
- **Resend:** correo de respaldo para alertas.
- **n8n:** logs de captación (Google Sheets) y workflows de Meta.
- **Google Calendar:** agenda de citas.

## 4. Alertas que recibe (confirmadas en plan de captación)

| Alerta | Condición | Canal |
|---|---|---|
| Lead de alta intención sin atender | Etapas PRE-CALIFICACIÓN/CALIFICADO con `#interesado`/`#calificado` sin actividad en 30 min | WhatsApp (WAHA) |
| Fallo terminal de captación | Webhook 4xx no recuperable (datos malformados) | WhatsApp + Resend |
| Comentario "EQUIPO" | Usuario pide contacto humano en comentarios | WhatsApp (DM manual) |
| Alerta de métricas | CPL > $50 MXN, conversión baja > 30% (Fase 7) | WhatsApp + Resend |

## 5. Flujo de trabajo diario (ESQUELETO — llenar con Miriam)

> **TODO:** describir la rutina real de Miriam: hora de revisión de la bandeja,
> orden de prioridades, cuántas veces al día revisa alertas.

- [ ] Revisar leads CALIFICADOS pendientes de cierre.
- [ ] Revisar alertas de alta intención (> 30 min sin atender).
- [ ] Confirmar citas del día (recordatorios).
- [ ] Revisar no-shows y aplicar toque de remarketing.
- [ ] (por definir) cierre del día: registrar pendientes.

## 6. Guion de cierre humano (ESQUELETO)

> **TODO:** guion exacto de Miriam para INTENTO DE CIERRE: cómo ofrecer el paquete,
> cómo responder objeciones frecuentes (precio, miedo a medicación, "es estrés").

Objecciones frecuentes conocidas (del blueprint, EJE 6): precio (PRE), desconfianza
(DES), "ya probé todo" (YA), "es crónico" (CRO), tiempo (TIE), medicación (MED).

## 7. Manejo de no-shows (ESQUELETO)

> **TODO:** mensajes exactos de los toques `#no-asistió-1/2/3` y cuándo pasar a
> `#no-asistió-nurturing`. Ver `docs/00-negocio/pipeline.md`.

## 8. Reglas inquebrantables (aplican también a Miriam)

- Nunca prometer diagnósticos ni curas. Usar "tratar la causa".
- Nunca mencionar tratamiento específico (medicamentos, dosis) por mensaje.
- Si el paciente pide no ser contactado → archivar de inmediato.
- Si menciona emergencia → "Llama al 911 o acude al hospital más cercano".

## 9. TODOs generales

- [ ] Validar este manual con Miriam y completar secciones 5-7.
- [ ] Definir respaldo: ¿quién cubre a Miriam en vacaciones?
- [ ] Definir guion para derivación a otro especialista (casos fuera de alergia).
- [ ] Documentar plantillas de WhatsApp de Miriam (respuestas rápidas del CRM).
