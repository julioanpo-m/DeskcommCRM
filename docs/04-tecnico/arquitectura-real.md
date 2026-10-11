# Arquitectura Real — Consultorio Dr. Julio Porras

**Versión:** 1.0
**Fecha:** 2026-10-11
**Estado:** Vigente — **este documento es la fuente de verdad técnica.**
**Aclaración:** `docs/04-tecnico/estructura-proyecto.md` es un DOCUMENTO HISTÓRICO
(arquitectura propuesta antes de adoptar DeskcommCRM). Si hay contradicción,
manda este documento y CLAUDE.md.

---

## 1. Decisión central

El consultorio **no construye el CRM desde cero**. Usa **DeskcommCRM como producto
completo** (fork del repositorio open source), configurado y extendido para el
caso de uso del consultorio. DeskcommCRM ya trae integrados Next.js 16 + Supabase.

---

## 2. Stack canónico

| Capa | Tecnología | Rol |
|---|---|---|
| **CRM** | DeskcommCRM (fork, auto-hospedado) | Producto completo: pipeline, agentes IA nativos, webhooks, automatización |
| **Frontend/Backend** | Next.js 16 (App Router) | Viene integrado en DeskcommCRM |
| **Base de datos y auth** | Supabase (Postgres gestionado + Auth + Storage) | Datos, autenticación, storage |
| **WhatsApp** | WAHA (WhatsApp HTTP API, no oficial) | Canal de mensajería |
| **IA de producción** | Anthropic Claude | Agente "Calificado" que atiende pacientes |
| **IA de desarrollo** | DeepSeek v4 Pro (vía cc-use, ECC) | Programación y documentación |
| **Correos** | Resend | Correo transaccional (alertas, respaldo) |
| **Calendario** | Google Calendar API | Agenda de citas |
| **Automatización externa** | n8n (self-hosted, planeado) | Orquestación de captación Meta (plan de captación) |
| **Deploy** | VPS Hostinger KVM 1 (Ubuntu 24.04+) + Docker Compose | Producción |
| **Dominio** | crm.drjulioporrasalergia.tech | Acceso al CRM |

---

## 3. Topología de producción

```
┌───────────────────────────────────────────────────────────┐
│              VPS HOSTINGER KVM 1 (4 GB RAM)               │
│                 Ubuntu 24.04+ · Docker Compose            │
│                                                           │
│   ┌──────────┐   ┌──────┐   ┌──────────┐   ┌──────────┐  │
│   │   app    │   │  db  │   │  redis   │   │ scheduler│  │
│   │ Deskcomm │   │Postgres│  │  cache/ │   │ workers/ │  │
│   │ Next.js  │   │(Supabase)│ │ colas   │   │ cron    │  │
│   └────┬─────┘   └──────┘   └──────────┘   └────┬─────┘  │
│        │                                        │        │
│   ┌────┴─────┐                                  │        │
│   │   waha   │── WhatsApp (QR sesión)           │        │
│   └────┬─────┘                                  │        │
│        │                                        │        │
│        └──────────────┬─────────────────────────┘        │
│                       │                                    │
│   Nginx + Let's Encrypt (SSL, renovación automática)      │
│                       │                                    │
└───────────────────────┼────────────────────────────────────┘
                        ▼
              crm.drjulioporrasalergia.tech
```

Servicios definidos en `docker-compose.prod.yml`: `app`, `db`, `redis`, `scheduler`, `waha`.

---

## 4. Integraciones externas

| Servicio | Uso |
|---|---|
| **Anthropic API** | Agente IA de conversación con pacientes |
| **DeepSeek API** | Asistente de desarrollo (cc-use) — sin datos de pacientes |
| **WAHA** | Sesión de WhatsApp del consultorio |
| **Resend** | Correos de alerta |
| **Google Calendar** | Agenda del Dr. Julio |
| **Meta Graph API** (planeado) | Lead Ads, comentarios, métricas vía n8n |

---

## 5. Estructura del repositorio (fork DeskcommCRM)

| Ruta | Contenido |
|---|---|
| `app/` | Rutas Next.js (App Router) |
| `app/api/v1/` | Endpoints de la API REST (webhooks, CRM, MCP) |
| `workers/` | Procesos en background (scheduler, colas) |
| `supabase/` | Migraciones y policies (incluye `baseline.sql`) |
| `tests/` | Pruebas (Vitest) |
| `docs/` | Documentación del proyecto |
| `hostgator-setup-kit/` | Scripts de instalación/backup/update de la VPS |
| `docker-compose.prod.yml` | Orquestación de contenedores |

---

## 6. Operación

- **Backups:** diario de Postgres (retención 7 días) + backups manuales
  (`bash hostgator-setup-kit/backup.sh`). **Nunca tocar producción sin backup previo verificado.**
- **Actualización:** `bash hostgator-setup-kit/update.sh`
- **Reconfiguración:** `bash hostgator-setup-kit/install.sh`
- **Logs:** `docker compose -f docker-compose.prod.yml logs -f app`

---

## 7. Costos reales

| Concepto | Costo aprox. |
|---|---|
| VPS Hostinger KVM 1 | ~$5 USD/mes |
| Supabase + servicios | por consumo |
| IA (Claude + DeepSeek) | por consumo de tokens |
| **Total estimado** | **~$35 USD/mes** |

> **TODO:** registrar facturación real mensual en `docs/00-negocio/registro-herramientas-ia.md`
> cuando haya 2-3 meses de histórico.

---

## 8. Flujo de contribución al fork

Este repo es un **fork** del producto DeskcommCRM. Todo cambio de código sigue el
ritual del contribuidor (skill `deskcomm-contribuir`):

- Ramificar desde el `origin/main` del producto (no del fork — el `main` del fork
  trae el CLAUDE.md local del consultorio).
- Tríada de migración: header `-- manifest:` + espejo exacto en
  `supabase/baseline.sql` + número NNNN único (`pnpm checar:colisao-de-migration`).
- Fragmento de release en `.changes/` y `pnpm release:conferir`.
- Test de sabotaje + `bash .agents/skills/deskcomm-contribuir/scripts/pre-voo.sh`.
- **Nunca** incluir secretos, tokens ni datos del consultorio en el diff del PR.

---

## 9. Verificación

```bash
pnpm typecheck && pnpm lint && pnpm test:unit
pnpm checar:colisao-de-migration
pnpm release:conferir
```
