> ⚠️ **DOCUMENTO HISTÓRICO** — Arquitectura propuesta antes de adoptar DeskcommCRM.
> La arquitectura real está descrita en CLAUDE.md y `docs/03-crm/`
> (fuente de verdad técnica: `docs/04-tecnico/arquitectura-real.md`).
> No usar este documento para decisiones técnicas nuevas.

 ESTRUCTURA DE PROYECTO — SENIOR DEVELOPER VIEW
## Sistema Alergia-Julio v13.3 + DeskcommCRM

**Documento:** DEV-STRUCTURE-v1.0
**Rol:** Arquitecto de Software Senior
**Objetivo:** Dar la estructura técnica completa para empezar a trabajar hoy.

---

## 1. STACK TECNOLÓGICO DECIDIDO

| Capa | Tecnología | Por qué |
|---|---|---|
| **CRM Core** | DeskcommCRM (Node.js + PostgreSQL) | Open source, auto-alojado, agentes IA nativos |
| **Base de datos** | PostgreSQL 16 | Robusto, gratis, soporte nativo de Deskcomm |
| **WhatsApp** | WAHA (WhatsApp HTTP API) | Integrado con Deskcomm, sin costos de Meta API |
| **Reverse Proxy** | Nginx + Certbot (Let's Encrypt) | HTTPS gratis, redirección limpia |
| **Contenedores** | Docker + Docker Compose | Portabilidad total, instalación en 1 comando |
| **IA** | OpenAI GPT-4o mini o Claude Haiku | Costos bajos, latencia aceptable |
| **Métricas** | Python 3.12 + Pandas | Análisis de CSVs, scripts ligeros |
| **Dashboard** | HTML5 + Chart.js + PapaParse | Sin dependencias, corre en navegador |
| **Automatización** | n8n (opcional) o cron jobs | Para triggers complejos |
| **Backups** | Restic + Backblaze B2 | Backups cifrados, $6/mes por 1TB |

**Costo mensual estimado:** $5 USD VPS + $6 USD backup + $5 USD IA = **~$16 USD/mes**

---

## 2. ARQUITECTURA DE ALTO NIVEL

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERNET (Usuarios)                          │
└─────────────┬───────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VPS HOSTINGER (4GB RAM)                      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              NGINX (Reverse Proxy + SSL)                 │  │
│  │              Puerto 443 (HTTPS)                          │  │
│  └────────┬──────────────────────────────┬──────────────────┘  │
│           │                              │                      │
│           ▼                              ▼                      │
│  ┌──────────────────┐           ┌──────────────────┐          │
│  │  DESKCOMMCRM     │           │  WAHA (WhatsApp) │          │
│  │  Puerto 3000     │           │  Puerto 3001     │          │
│  │                  │           │                  │          │
│  │  Node.js + IA    │           │  WhatsApp Web    │          │
│  └────────┬─────────┘           └────────┬─────────┘          │
│           │                              │                      │
│           └──────────┬───────────────────┘                      │
│                      ▼                                          │
│           ┌──────────────────┐                                  │
│           │  POSTGRESQL 16   │                                  │
│           │  Puerto 5432     │                                  │
│           │  (solo interno)  │                                  │
│           └──────────────────┘                                  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           CRON JOBS + SCRIPTS PYTHON                     │  │
│  │  - Extracción de métricas (diaria 9:00 AM)               │  │
│  │  - Auditoría SHA-256 (domingo 18:00)                     │  │
│  │  - Backups (diario 3:00 AM)                              │  │
│  │  - MVM update (lunes 11:00)                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BACKUP EXTERNO                               │
│                    Backblaze B2 o Google Drive                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. ESTRUCTURA DE CARPETAS EN LA VPS

```
/opt/alergia-julio/                        # Carpeta raíz del proyecto
│
├── docker-compose.yml                     # Orquestación de contenedores
├── .env                                   # Variables secretas (NO en git)
├── .env.example                           # Plantilla pública
├── .gitignore                             # Archivos excluidos de git
├── README.md                              # Documentación principal
│
├── 01_deskcomm/                           # DeskcommCRM
│   ├── docker-compose.yml                 # Override para Deskcomm
│   ├── data/                              # Datos persistentes
│   │   ├── uploads/                       # Archivos subidos
│   │   └── logs/                          # Logs del CRM
│   └── config/                            # Configuración custom
│       └── ai-prompts/                    # Prompts de agentes IA
│           ├── pre-calificacion.md
│           ├── cierre.md
│           └── seguimiento.md
│
├── 02_waha/                               # WhatsApp HTTP API
│   ├── sessions/                          # Sesiones de WhatsApp (QR)
│   └── config/                            # Configuración WAHA
│
├── 03_postgres/                           # Base de datos
│   ├── data/                              # Datos persistentes
│   ├── backups/                           # Backups locales
│   └── init/                              # Scripts de inicialización
│       ├── 01-schema.sql
│       ├── 02-pipelines.sql
│       └── 03-seed-data.sql
│
├── 04_nginx/                              # Reverse proxy
│   ├── nginx.conf                         # Configuración principal
│   ├── sites/                             # Configuración por dominio
│   │   └── alergia-julio.conf
│   └── ssl/                               # Certificados Let's Encrypt
│
├── 05_scripts/                            # Scripts Python de automatización
│   ├── requirements.txt                   # Dependencias Python
│   ├── extraer_metricas.py                # Extrae métricas de Deskcomm
│   ├── procesar_mvm.py                    # Calcula IRM y promueve módulos
│   ├── auditar_integridad.py              # Hash SHA-256 de CSVs
│   ├── backup_db.py                       # Backup de PostgreSQL
│   ├── generar_reporte.py                 # Reporte mensual
│   └── utils/                             # Utilidades compartidas
│       ├── db.py
│       ├── csv_helper.py
│       └── logger.py
│
├── 06_metricas/                           # CSVs maestros del sistema
│   ├── metricas_contenido.csv
│   ├── metricas_clinicas.csv
│   ├── metricas_financieras.csv
│   ├── metricas_pauta.csv
│   ├── mvm_modulos.csv
│   ├── mvm_puntos.csv
│   └── viabilidad_multiservicio.csv
│
├── 07_herramientas/                       # Herramientas del sistema
│   ├── dashboard.html                     # Dashboard de 12 indicadores
│   ├── generar-id.html                    # Constructor de IDs
│   ├── buscar-modulo.html                 # Buscador de módulos del banco
│   ├── viabilidad.html                    # Calculadora multi-servicio
│   ├── captura-metricas.html              # Captura manual de métricas
│   ├── renombrar.sh                       # Nombrador masivo (Linux)
│   ├── verificar-compliance.py            # Verificador COFEPRIS/Meta
│   └── crear-csvs.py                      # Generador de CSVs
│
├── 08_banco/                              # Banco modular (contenido)
│   ├── banco-oficial/
│   │   ├── shocks/                        # S-001.md ... S-020.md
│   │   ├── tensiones/                     # T-001.md ... T-008.md
│   │   ├── alivios/                       # A-001.md ... A-007.md
│   │   ├── firmas/                        # F-001.md, F-002.md
│   │   ├── ctas/                          # C-001.md ... C-007.md
│   │   ├── brolls/                        # B-001.md ... B-012.md
│   │   └── direccion/                     # Fichas de dirección escénica
│   └── banco-experimental/
│       ├── candidatos/
│       ├── en-prueba/
│       └── obsoletos/
│
├── 09_blueprint/                          # Documentación maestra
│   ├── BLUEPRINT-MAESTRO-v13.3.md
│   ├── VERTIENTE-A-PAUTA.md
│   ├── VERTIENTE-B-TABLERO.md
│   ├── VERTIENTE-C-ESCALAMIENTO.md
│   ├── VERTIENTE-D-VIABILIDAD.md
│   ├── VERTIENTE-E-COMPLIANCE.md
│   ├── VERTIENTE-F-PRODUCCION.md
│   └── DEV-STRUCTURE-v1.0.md              # Este documento
│
├── 10_logs/                               # Logs del sistema
│   ├── log-versiones.md
│   ├── log-politicas.md
│   ├── log-excepciones-eta.md
│   ├── log-mvm.md
│   └── log-auditoria-metricas.csv
│
└── 11_backups/                            # Backups locales temporales
    ├── postgres/
    ├── deskcomm/
    └── metricas/
```

---

## 4. ESTRUCTURA DEL REPOSITORIO GIT

```
alergia-julio/                              # Monorepo principal
│
├── .github/                                # Configuración GitHub
│   ├── workflows/
│   │   ├── deploy.yml                     # CI/CD a VPS
│   │   └── test.yml                       # Tests automáticos
│   └── ISSUE_TEMPLATE/
│       ├── bug.md
│       └── feature.md
│
├── docs/                                   # Documentación técnica
│   ├── architecture.md
│   ├── deployment.md
│   ├── api.md
│   └── runbook.md                         # Procedimientos de emergencia
│
├── infra/                                  # Infraestructura como código
│   ├── docker/
│   │   ├── Dockerfile.deskcomm
│   │   ├── Dockerfile.waha
│   │   └── Dockerfile.scripts
│   ├── nginx/
│   │   └── sites/
│   └── terraform/                          # Opcional, si migran a cloud
│       └── main.tf
│
├── apps/                                   # Aplicaciones
│   ├── deskcomm/                          # Submódulo o fork
│   ├── scripts/                           # Scripts Python
│   └── dashboard/                         # Dashboard HTML
│
├── data/                                   # Datos (NO en git)
│   ├── metricas/
│   ├── banco/
│   └── logs/
│
├── .env.example
├── .gitignore
├── docker-compose.yml
├── Makefile                                # Comandos rápidos
└── README.md
```

---

## 5. ARCHIVOS CLAVE DE CONFIGURACIÓN

### `.env.example`

```bash
# ============================================
# CONFIGURACIÓN GENERAL
# ============================================
DOMAIN=alergia-julio.com
ADMIN_EMAIL=admin@alergia-julio.com
TIMEZONE=America/Mexico_City

# ============================================
# POSTGRESQL
# ============================================
POSTGRES_USER=alergia_admin
POSTGRES_PASSWORD=CAMBIAR_ESTA_PASSWORD
POSTGRES_DB=deskcomm
POSTGRES_PORT=5432

# ============================================
# DESKCOMMCRM
# ============================================
DESKCOMM_PORT=3000
DESKCOMM_SECRET_KEY=CAMBIAR_ESTA_LLAVE
DESKCOMM_ADMIN_EMAIL=drjulio@alergia-julio.com
DESKCOMM_ADMIN_PASSWORD=CAMBIAR_ESTA_PASSWORD

# ============================================
# WHATSAPP (WAHA)
# ============================================
WAHA_PORT=3001
WAHA_API_KEY=CAMBIAR_ESTA_LLAVE
WAHA_DASHBOARD_USERNAME=admin
WAHA_DASHBOARD_PASSWORD=CAMBIAR_ESTA_PASSWORD

# ============================================
# IA (OpenAI o Claude)
# ============================================
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
# o
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-5-haiku-20241022

# ============================================
# BACKUPS
# ============================================
BACKUP_BUCKET=alergia-julio-backups
BACKUP_ACCESS_KEY=...
BACKUP_SECRET_KEY=...
BACKUP_ENCRYPTION_KEY=...

# ============================================
# NOTIFICACIONES
# ============================================
SLACK_WEBHOOK_URL=https://hooks.slack.com/...
NOTIFICATION_EMAIL=alertas@alergia-julio.com
```

### `docker-compose.yml` (raíz)

```yaml
version: '3.9'

networks:
  alergia_net:
    driver: bridge

volumes:
  postgres_data:
  deskcomm_data:
  waha_sessions:

services:
  # ============================================
  # POSTGRESQL
  # ============================================
  postgres:
    image: postgres:16-alpine
    container_name: alergia_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
      TZ: ${TIMEZONE}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./03_postgres/init:/docker-entrypoint-initdb.d
    networks:
      - alergia_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ============================================
  # DESKCOMMCRM
  # ============================================
  deskcomm:
    build:
      context: ./01_deskcomm
      dockerfile: ../../infra/docker/Dockerfile.deskcomm
    container_name: alergia_deskcomm
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      SECRET_KEY: ${DESKCOMM_SECRET_KEY}
      ADMIN_EMAIL: ${DESKCOMM_ADMIN_EMAIL}
      ADMIN_PASSWORD: ${DESKCOMM_ADMIN_PASSWORD}
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      TZ: ${TIMEZONE}
    volumes:
      - deskcomm_data:/app/data
      - ./01_deskcomm/config:/app/config
    networks:
      - alergia_net
    ports:
      - "127.0.0.1:${DESKCOMM_PORT}:3000"

  # ============================================
  # WAHA (WhatsApp HTTP API)
  # ============================================
  waha:
    image: devlikeapro/waha:latest
    container_name: alergia_waha
    restart: unless-stopped
    environment:
      WHATSAPP_API_KEY: ${WAHA_API_KEY}
      WHATSAPP_HOOK_URL: http://deskcomm:3000/api/webhooks/whatsapp
      WHATSAPP_HOOK_EVENTS: message,message.ack,session.status
      TZ: ${TIMEZONE}
    volumes:
      - waha_sessions:/app/.sessions
    networks:
      - alergia_net
    ports:
      - "127.0.0.1:${WAHA_PORT}:3000"

  # ============================================
  # NGINX (Reverse Proxy)
  # ============================================
  nginx:
    image: nginx:alpine
    container_name: alergia_nginx
    restart: unless-stopped
    depends_on:
      - deskcomm
      - waha
    volumes:
      - ./04_nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./04_nginx/sites:/etc/nginx/conf.d:ro
      - ./04_nginx/ssl:/etc/letsencrypt:ro
    networks:
      - alergia_net
    ports:
      - "80:80"
      - "443:443"

  # ============================================
  # SCRIPTS PYTHON (Cron jobs)
  # ============================================
  scripts:
    build:
      context: ./05_scripts
      dockerfile: ../../infra/docker/Dockerfile.scripts
    container_name: alergia_scripts
    restart: unless-stopped
    depends_on:
      - postgres
      - deskcomm
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      DESKCOMM_URL: http://deskcomm:3000
      DESKCOMM_API_KEY: ${DESKCOMM_SECRET_KEY}
      TZ: ${TIMEZONE}
    volumes:
      - ./06_metricas:/app/metricas
      - ./10_logs:/app/logs
      - ./11_backups:/app/backups
    networks:
      - alergia_net
```

### `Makefile` (comandos rápidos)

```makefile
.PHONY: help init up down logs restart backup restore deploy

help:
	@echo "Comandos disponibles:"
	@echo "  make init      - Inicializa el proyecto (primera vez)"
	@echo "  make up        - Levanta todos los servicios"
	@echo "  make down      - Detiene todos los servicios"
	@echo "  make logs      - Ver logs en tiempo real"
	@echo "  make restart   - Reinicia todos los servicios"
	@echo "  make backup    - Ejecuta backup manual"
	@echo "  make restore   - Restaura el último backup"
	@echo "  make deploy    - Despliega la última versión"

init:
	@echo "🚀 Inicializando sistema Alergia-Julio..."
	@cp -n .env.example .env || true
	@echo "⚠️  Edita .env con tus credenciales antes de continuar"
	@mkdir -p 06_metricas 10_logs 11_backups
	@echo "✅ Estructura inicial creada"

up:
	@echo "🔼 Levantando servicios..."
	@docker compose up -d
	@echo "✅ Servicios activos"
	@docker compose ps

down:
	@echo "🔽 Deteniendo servicios..."
	@docker compose down
	@echo "✅ Servicios detenidos"

logs:
	@docker compose logs -f --tail=100

restart:
	@docker compose restart

backup:
	@echo "💾 Ejecutando backup..."
	@docker compose exec scripts python /app/backup_db.py
	@echo "✅ Backup completado"

restore:
	@echo "♻️  Restaurando último backup..."
	@docker compose exec scripts python /app/restore_db.py --latest
	@echo "✅ Restauración completada"

deploy:
	@echo "🚀 Desplegando..."
	@git pull origin main
	@docker compose build
	@docker compose up -d
	@echo "✅ Despliegue completado"
```

---

## 6. SCRIPTS PYTHON PRINCIPALES

### `05_scripts/requirements.txt`

```txt
# Core
psycopg2-binary==2.9.9
requests==2.32.3
pandas==2.2.3
python-dotenv==1.0.1

# Utilidades
click==8.1.7
rich==13.9.4
loguru==0.7.2

# Backups
boto3==1.35.60

# Scheduler
APScheduler==3.10.4
```

### `05_scripts/extraer_metricas.py` (esqueleto)

```python
#!/usr/bin/env python3
"""
Extrae métricas de contenido desde DeskcommCRM y las guarda en CSV.
Ejecuta: lunes 9:00 AM
"""
from loguru import logger
from utils.db import get_deskcomm_connection
from utils.csv_helper import append_to_csv
from datetime import datetime, timedelta

def extraer_metricas_dia_3():
    """Extrae métricas de videos publicados hace 3 días."""
    fecha_objetivo = (datetime.now() - timedelta(days=3)).strftime("%Y-%m-%d")
    
    logger.info(f"Extrayendo métricas para el día {fecha_objetivo}")
    
    with get_deskcomm_connection() as conn:
        # Query para obtener piezas publicadas hace 3 días
        piezas = conn.execute("""
            SELECT id, id_pieza, canal, views, retencion_3s,
                   comentarios_clave, dms_emocionales, compartidos,
                   etiquetas, comentarios_defensa, comentarios_tribu, guardados
            FROM metricas_contenido
            WHERE fecha_publicacion = %s
        """, (fecha_objetivo,)).fetchall()
    
    for pieza in piezas:
        reverberancia = calcular_reverberancia(pieza)
        decision = clasificar_reverberancia(reverberancia)
        
        fila = [
            datetime.now().strftime("%Y-%m-%d"),
            pieza.id_pieza,
            pieza.canal,
            pieza.views,
            pieza.retencion_3s,
            pieza.comentarios_clave,
            pieza.dms_emocionales,
            pieza.compartidos,
            pieza.etiquetas,
            pieza.comentarios_defensa,
            pieza.comentarios_tribu,
            pieza.guardados,
            f"{reverberancia:.2f}",
            decision
        ]
        
        append_to_csv("06_metricas/metricas_contenido.csv", fila)
        logger.success(f"✓ {pieza.id_pieza}: {reverberancia:.2f}% → {decision}")

def calcular_reverberancia(pieza) -> float:
    """Calcula el índice de reverberancia según Blueprint v13.3."""
    interacciones = (
        pieza.comentarios_clave +
        pieza.dms_emocionales +
        pieza.compartidos +
        (pieza.etiquetas * 2) +
        pieza.comentarios_defensa +
        pieza.comentarios_tribu +
        (pieza.guardados * 0.5)
    )
    return (interacciones / pieza.views) * 100 if pieza.views > 0 else 0

def clasificar_reverberancia(valor: float) -> str:
    if valor < 1: return "DESCARTAR"
    if valor < 2: return "REPETIR"
    if valor < 3: return "SERIE"
    return "ESCALAR"

if __name__ == "__main__":
    logger.info("Iniciando extracción de métricas...")
    extraer_metricas_dia_3()
    logger.success("Extracción completada")
```

### `05_scripts/procesar_mvm.py` (esqueleto)

```python
#!/usr/bin/env python3
"""
Calcula IRM normalizado y aplica compuerta booleana de promoción.
Ejecuta: lunes 11:00 AM
"""
from loguru import logger
import pandas as pd

def calcular_irm(reverb_pct: float, ret_3s_pct: float, usos: int) -> float:
    """Fórmula IRM normalizada v13.3."""
    s_rev = min(5.00, reverb_pct)
    s_ret = min(5.00, ret_3s_pct / 17.00)
    s_uso = min(5.00, (usos / 20.00) * 5.00)
    return round((s_rev * 0.5) + (s_ret * 0.3) + (s_uso * 0.2), 2)

def aplicar_compuerta(puntos: int, reverb: float, usos: int) -> str:
    """Compuerta booleana v13.3."""
    if puntos < 0 or (reverb < 1 and usos >= 3):
        return "OBSOLETO"
    if puntos >= 100 and reverb >= 5 and usos >= 10:
        return "ELITE"
    if puntos >= 60 and reverb >= 3 and usos >= 5:
        return "PROMOVIDO"
    if puntos >= 30 and reverb >= 2 and usos >= 3:
        return "VALIDADO"
    if puntos >= 10 and reverb >= 2 and usos >= 2:
        return "EN_PRUEBA"
    return "CANDIDATO"

def procesar_modulos():
    logger.info("Procesando MVM...")
    df = pd.read_csv("06_metricas/mvm_modulos.csv")
    
    for idx, row in df.iterrows():
        irm = calcular_irm(row["Reverb_promedio"], row["Retencion_3s_promedio"], row["Usos"])
        nuevo_estado = aplicar_compuerta(row["Puntos"], row["Reverb_promedio"], row["Usos"])
        
        df.at[idx, "IRM"] = irm
        df.at[idx, "Estado"] = nuevo_estado
        
        logger.info(f"{row['ID_Modulo']}: IRM={irm} → {nuevo_estado}")
    
    df.to_csv("06_metricas/mvm_modulos.csv", index=False)
    logger.success("MVM procesado")

if __name__ == "__main__":
    procesar_modulos()
```

---

## 7. CRON JOBS (programación)

**Archivo:** `/etc/cron.d/alergia-julio`

```cron
# Alergia-Julio cron jobs
# Zona horaria: America/Mexico_City

# Backup diario a las 3:00 AM
0 3 * * * root cd /opt/alergia-julio && make backup >> /var/log/alergia-backup.log 2>&1

# Extracción de métricas lunes 9:00 AM
0 9 * * 1 root docker exec alergia_scripts python /app/extraer_metricas.py >> /var/log/alergia-metricas.log 2>&1

# Procesamiento MVM lunes 11:00 AM
0 11 * * 1 root docker exec alergia_scripts python /app/procesar_mvm.py >> /var/log/alergia-mvm.log 2>&1

# Auditoría SHA-256 domingo 18:00
0 18 * * 0 root docker exec alergia_scripts python /app/auditar_integridad.py >> /var/log/alergia-auditoria.log 2>&1

# Reporte mensual día 1 a las 8:00 AM
0 8 1 * * root docker exec alergia_scripts python /app/generar_reporte.py >> /var/log/alergia-reporte.log 2>&1

# Limpieza de logs >30 días
0 4 * * 0 root find /var/log -name "alergia-*.log" -mtime +30 -delete
```

---

## 8. PLAN DE IMPLEMENTACIÓN — 7 DÍAS

### DÍA 1 — Infraestructura (4 horas)

| Hora | Tarea | Comando/Acción |
|---|---|---|
| 09:00 | Contratar VPS Hostinger KVM 1 | Panel Hostinger |
| 09:30 | Configurar SSH y usuario no-root | `ssh root@vps` |
| 10:00 | Instalar Docker y Docker Compose | `curl -fsSL https://get.docker.com \| sh` |
| 11:00 | Clonar repositorio | `git clone https://github.com/tu-user/alergia-julio.git` |
| 12:00 | Configurar `.env` con credenciales | Editar archivo |
| 13:00 | Levantar servicios por primera vez | `make up` |

### DÍA 2 — DeskcommCRM + PostgreSQL (4 horas)

| Hora | Tarea |
|---|---|
| 09:00 | Verificar que Deskcomm responde en `localhost:3000` |
| 10:00 | Crear usuario admin del Dr. Julio |
| 11:00 | Configurar pipeline "Pacientes Nuevos" con 7 etapas |
| 12:00 | Crear etiquetas automáticas |
| 13:00 | Importar snippets y respuestas rápidas |

### DÍA 3 — WhatsApp + IA (4 horas)

| Hora | Tarea |
|---|---|
| 09:00 | Configurar WAHA y escanear QR |
| 10:00 | Verificar envío/recepción de mensajes |
| 11:00 | Configurar agente IA con prompts de pre-calificación |
| 12:00 | Probar 3 conversaciones de prueba |
| 13:00 | Ajustar prompts según resultados |

### DÍA 4 — Nginx + SSL (3 horas)

| Hora | Tarea |
|---|---|
| 09:00 | Configurar Nginx como reverse proxy |
| 10:00 | Emitir certificado SSL con Let's Encrypt |
| 11:00 | Verificar HTTPS en dominio |
| 12:00 | Configurar redirect HTTP → HTTPS |

### DÍA 5 — Scripts + Cron (4 horas)

| Hora | Tarea |
|---|---|
| 09:00 | Crear contenedor de scripts |
| 10:00 | Verificar script `extraer_metricas.py` |
| 11:00 | Verificar script `procesar_mvm.py` |
| 12:00 | Configurar cron jobs |
| 13:00 | Probar ejecución manual |

### DÍA 6 — Herramientas HTML (3 horas)

| Hora | Tarea |
|---|---|
| 09:00 | Desplegar `dashboard.html` |
| 10:00 | Desplegar `generar-id.html` |
| 11:00 | Desplegar `viabilidad.html` |
| 12:00 | Verificar acceso público |

### DÍA 7 — Backups + Monitoreo (3 horas)

| Hora | Tarea |
|---|---|
| 09:00 | Configurar Backblaze B2 |
| 10:00 | Verificar backup completo |
| 11:00 | Configurar alertas por email |
| 12:00 | Documentar runbook |

---

## 9. CHECKLIST DE ARRANQUE

```
[ ] VPS contratado (Hostinger KVM 1, 4GB RAM)
[ ] Docker y Docker Compose instalados
[ ] Repositorio git clonado en /opt/alergia-julio
[ ] Archivo .env configurado
[ ] PostgreSQL corriendo
[ ] DeskcommCRM corriendo
[ ] WAHA corriendo y con QR escaneado
[ ] Nginx corriendo con SSL
[ ] Dominio apuntando al VPS
[ ] HTTPS funcionando
[ ] Usuario admin del Dr. Julio creado
[ ] Usuario de Miriam creado
[ ] Pipeline de 7 etapas configurado
[ ] Agentes IA entrenados
[ ] Scripts Python funcionando
[ ] Cron jobs activos
[ ] Backups configurados
[ ] Dashboard HTML accesible
[ ] Alertas por email funcionando
[ ] Runbook documentado
```

---

## 10. COMANDOS ÚTILES PARA EL DÍA A DÍA

```bash
# Ver estado de servicios
docker compose ps

# Ver logs de un servicio
docker compose logs -f deskcomm

# Reiniciar un servicio
docker compose restart deskcomm

# Entrar al contenedor
docker exec -it alergia_deskcomm sh

# Ver uso de recursos
docker stats

# Ejecutar script manual
docker exec alergia_scripts python /app/extraer_metricas.py

# Ver tamaño de backups
du -sh /opt/alergia-julio/11_backups/*

# Ver espacio en disco
df -h

# Actualizar desde git
git pull origin main && make deploy
```

---

## 11. ESTRUCTURA DE COSTOS REALES

| Concepto | Costo mensual | Notas |
|---|---|---|
| VPS Hostinger KVM 1 | $5 USD | 4GB RAM, 50GB disco |
| Backblaze B2 (100GB) | $0.60 USD | Backups cifrados |
| IA (GPT-4o mini) | ~$5 USD | 500k tokens/mes estimados |
| Dominio (.com) | ~$1 USD | Anual dividido |
| SSL (Let's Encrypt) | $0 | Gratis |
| **Total** | **~$12 USD/mes** | **~$240 MXN/mes** |

**Por usuario adicional:** $0. Deskcomm permite usuarios ilimitados.

---

## 12. SIGUIENTE PASO

Ahora que tienes la estructura completa, elige:

**A)** Arrancamos el **Día 1** — Contratar VPS y configurar infraestructura.

**B)** Arrancamos el **Día 2** — Configurar DeskcommCRM y pipeline.

**C)** Arrancamos el **Día 3** — Configurar WhatsApp + IA.

**D)** Te armo el **runbook completo** con procedimientos de emergencia (qué hacer si se cae el servidor, si WAHA se desconecta, si la IA falla).

**E)** Te armo los **prompts exactos** de los agentes IA para pre-calificación, cierre y seguimiento.

**Mi recomendación: D + E primero.** Porque si algo falla en producción (y va a fallar), necesitas saber qué hacer. Y los prompts son lo que va a hacer que la IA funcione bien o mal. El servidor es lo más fácil de arreglar; los prompts son lo que más impacta en conversión.
