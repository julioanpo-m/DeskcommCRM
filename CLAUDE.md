# CLAUDE.md — Consultorio Dr. Julio Porras

Contexto del proyecto para asistentes de IA (Claude Code, ECC, DeepSeek).

## Visión

CRM auto-hospedado para el consultorio del **Dr. Julio Ángel López Porras**, especialista en Inmunología Clínica y Alergia en Boca del Río, Veracruz, México. Su función es captar, pre-calificar y agendar pacientes vía WhatsApp usando agentes IA, con seguimiento humano de Miriam.

## Stack canónico

- **Frontend/Backend:** Next.js 16 (App Router)
- **Base de datos y auth:** Supabase (Postgres + Auth + Storage)
- **WhatsApp:** WAHA (WhatsApp HTTP API, no oficial)
- **IA de producción:** Anthropic Claude (agente "Calificado")
- **IA de desarrollo:** DeepSeek v4 Pro (vía cc-use, para ECC)
- **Correos:** Resend
- **Calendario:** Google Calendar API
- **Deploy:** VPS Hostinger KVM 1 (Ubuntu) + Docker Compose
- **Dominio:** crm.drjulioporrasalergia.tech

## Convenciones críticas (NO NEGOCIABLES)

- Idioma de trabajo: **español (México)**.
- Moneda: **MXN**.
- Zona horaria: **America/Mexico_City**.
- Nunca usar `any` en TypeScript.
- Nunca hardcodear secretos: van en `.env`.
- Nunca modificar producción sin backup previo.

## Pipelines (embudos)

### 1. Pacientes Nuevos
1. LEAD NUEVO
2. PRE-CALIFICACIÓN (IA)
3. CALIFICADO
4. INTENTO DE CIERRE
5. CITA CONFIRMADA
6. ASISTIÓ (Ganado)
7. NO ASISTIÓ (Perdido)

### 2. Nurturing
1. CONTENIDO EDUCATIVO (día 7)
2. CASOS DE ÉXITO (día 14)
3. PREGUNTA ABIERTA (día 21)
4. OFERTA ONLINE (día 30)
5. GUÍA DESCARGABLE (día 45)
6. ÚLTIMA OFERTA (día 60)
7. FRÍO (día 90+)

### 3. Pacientes Activos
1. PRIMERA CONSULTA
2. INICIO ITA
3. EN TRATAMIENTO
4. REVISIÓN TRIMESTRAL
5. REVISIÓN ANUAL
6. ALTA

## Reglas de negocio para el agente IA

- Máximo 3 líneas por mensaje. Una sola pregunta por mensaje.
- Llama al paciente por su nombre de pila.
- Nunca dar diagnósticos. Derivar siempre a consulta.
- Nunca prometer cura. Usar "tratar la causa".
- Nunca mencionar tratamiento específico (medicamentos, dosis).
- Nunca prometer precios ni descuentos sin confirmar con Miriam.
- Horario de atención: L-V 9:00-18:00, Sábado 9:00-13:00.
- No enviar mensajes antes de las 8:00 ni después de las 20:00.
- Máximo 1 mensaje por semana en nurturing.
- Máximo 3 mensajes en 7 días si no responde.
- Si pide no ser contactado, archivar de inmediato.
- Si menciona emergencia: "Llama al 911 o acude al hospital más cercano".
- Decir que es un asistente cuando pregunten directamente.

## Precios vigentes

- Consulta de valoración: **$1,000 MXN** (sin pruebas ni tratamiento).
- Paquete inicial: **$3,900 MXN** (consulta + Prick test + 1ª vacuna).
- Mensualidad subcutánea: **$1,700 MXN**.
- Mensualidad sublingual: **$1,000 MXN**.

Regla: **el paquete de $3,900 se ofrece primero.** La valoración de $1,000 solo si el paciente duda o pide segunda opinión.

## Etiquetas automáticas

- Fuente: `#urticaria`, `#pag-urticaria`, `#bio`, `#dm`, `#referido`, `#directo`
- Clasificación: `#lead-nuevo`, `#pre-calificado`, `#crónico`, `#agudo`, `#frío`
- Cierre: `#interesado`, `#cerrado`, `#asistió`, `#no-asistió`, `#activo`
- Remarketing: `#no-asistió-1`, `#no-asistió-2`, `#no-asistió-3`, `#no-asistió-nurturing`

## Paths importantes

- `app/` — rutas Next.js (App Router)
- `app/api/v1/` — endpoints de la API REST
- `workers/` — procesos en background (scheduler, colas)
- `supabase/` — migraciones y policies
- `tests/` — pruebas (Vitest + Playwright)
- `docs/` — documentación del proyecto
- `.env` — variables de entorno (NO commitear)
- `docker-compose.prod.yml` — orquestación de contenedores

## Deploy en producción

- VPS Hostinger KVM 1 (4 GB RAM, Ubuntu 24.04+).
- Docker Compose con contenedores: `app`, `db`, `redis`, `scheduler`, `waha`.
- SSL con Let's Encrypt (renovación automática).
- Backup diario de Postgres (retención 7 días).
- **Nunca tocar producción sin backup previo verificado.**

Comandos útiles:
```bash
# Ver logs
docker compose -f docker-compose.prod.yml logs -f app

# Reiniciar
docker compose -f docker-compose.prod.yml restart

# Actualizar a nueva versión
bash hostgator-setup-kit/update.sh

# Backup manual
bash hostgator-setup-kit/backup.sh

# Reconfigurar
bash hostgator-setup-kit/install.sh
```

## Flujo de trabajo con ECC + DeepSeek

Usamos **ECC** (Everything Claude Code) como capa de skills sobre Claude Code, con **DeepSeek v4 Pro** como motor (vía `cc-use`).

### Skills principales
- `/plan` — Planificar una funcionalidad antes de escribir código.
- `/tdd` — Escribir tests antes de la implementación.
- `/security-scan` — Revisar seguridad antes del commit.
- `/egc-grok` — Auditar todo el repositorio en una pasada.

### Regla de trabajo
1. Planificar con `/plan`. Esperar confirmación antes de escribir código.
2. Escribir tests con `/tdd`.
3. Implementar guiado por los tests.
4. Revisar seguridad con `/security-scan`.
5. Auditar con `/egc-grok` antes de desplegar.

## Reglas de commits

- Mensajes en **español**, descriptivos y en imperativo.
- Formato: `tipo(alcance): descripción corta`
- Tipos: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`.
- Ejemplo: `feat(reactivación): agregar flujo de 5 toques para no-shows`

## Definition of Done

Una tarea está terminada cuando:
- [ ] El código pasa los tests (`pnpm test`).
- [ ] No hay errores de TypeScript (`pnpm typecheck`).
- [ ] El linter está limpio (`pnpm lint`).
- [ ] La funcionalidad está documentada en `docs/`.
- [ ] Se probó manualmente en el entorno de desarrollo.
- [ ] El commit sigue las reglas de arriba.

## Anti-patterns prohibidos

- ❌ Modificar la base de datos directamente por SQL en producción.
- ❌ Commitear archivos `.env` o secretos.
- ❌ Usar `any` en TypeScript.
- ❌ Ignorar errores con `try/catch` vacíos.
- ❌ Hacer deploy sin backup previo.
- ❌ Modificar el agente IA sin actualizar los tests.
- ❌ Prometer diagnósticos o curas en respuestas del agente.
- ❌ Enviar mensajes fuera del horario permitido.

## Enlaces de referencia

- Repo del CRM: `github.com/julioanpo-m/DeskcommCRM`
- Supabase Dashboard: `supabase.com/dashboard`
- Anthropic Console: `console.anthropic.com`
- DeepSeek Platform: `platform.deepseek.com`
- Resend Dashboard: `resend.com`
- Meta Business Suite: `business.facebook.com`
- Hostinger hPanel: `hpanel.hostinger.com`

## Documentos de referencia (en docs/)

Antes de generar cualquier plan, ECC DEBE leer estos archivos en orden:

1. `docs/index.md` - Indice completo
2. `docs/00-negocio/pipeline.md` - Estructura del pipeline
3. `docs/01-contenido/blueprint-maestro-v11.md` - Sistema de contenido
4. `docs/02-anuncios/blueprint-pag-v1.md` - Sistema de anuncios
5. `docs/03-crm/arquitectura-lead-management.md` - Flujo del CRM
6. `docs/04-tecnico/estructura-proyecto.md` - Arquitectura tecnica

Estos documentos son la fuente de verdad. Si algo no esta en ellos, no existe.
