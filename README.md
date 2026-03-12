# ERP Odoo Community

ERP basado en Odoo Community 18 para uso interno de la asociación.
Desplegado en servidor local (mini PC) mediante Docker Compose con acceso remoto seguro vía VPN.

---

## Arquitectura

```
┌─────────────────────────────────────────────────────┐
│                     mini PC (Debian)                │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌─────────────────┐ │
│  │  Traefik │   │   Odoo   │   │   PostgreSQL    │ │
│  │  (proxy) │──▶│  (app)   │──▶│     (db)        │ │
│  └──────────┘   └──────────┘   └─────────────────┘ │
│                                       │             │
│                 ┌─────────────────────┘             │
│                 ▼                                   │
│  ┌──────────────────────────┐                       │
│  │  docker-volume-backup    │──▶ Google Drive       │
│  └──────────────────────────┘                       │
└─────────────────────────────────────────────────────┘
         ▲
         │ Tailscale VPN
         │
    Usuario remoto
```

| Componente | Tecnología |
|---|---|
| Aplicación | Odoo Community 18 |
| Base de datos | PostgreSQL 15 |
| Proxy / TLS | Traefik |
| Backup | offen/docker-volume-backup → Google Drive |
| Acceso remoto | Tailscale VPN |
| CI/CD | GitHub Actions (self-hosted runner) |
| OS servidor | Debian |

---

## Etapas de implementación

- [x] Etapa 0 — Definición de arquitectura y contexto
- [x] Etapa 1.1 — Servicios Odoo + PostgreSQL levantados en local y acceso validado
- [x] Etapa 1.2 — Configuración de módulos vía UI y persistencia verificada
- [ ] Etapa 1 — Docker Compose en local (entorno de desarrollo)
- [ ] Etapa 2 — Pipeline CI/CD con GitHub Actions
- [ ] Etapa 3 — Despliegue en mini PC
- [ ] Etapa 4 — Acceso remoto seguro y validación funcional
- [ ] Etapa 5 — Resiliencia, backups y restauración

---

## Estructura del repositorio

```
.
├── .claude/                    # Contexto y reglas para el asistente de desarrollo
│   ├── CLAUDE.md
│   └── rules/
│       ├── commit-standards.md
│       └── development-approach.md
├── .github/
│   └── workflows/
│       └── deploy.yml          # Pipeline CI/CD (Etapa 2)
├── odoo/
│   └── config/
│       └── odoo.conf           # Configuración de Odoo
├── traefik/                    # Configuración del proxy (Etapa 1)
├── backup/                     # Configuración del servicio de backup (Etapa 5)
├── scripts/
│   └── server-setup.sh         # Script de configuración inicial del servidor (Etapa 3)
├── docker-compose.yml          # Stack principal
├── docker-compose.override.yml # Overrides para desarrollo local
└── .env.example                # Variables de entorno requeridas (sin valores reales)
```

> **Nota:** El archivo `.env` nunca se commitea. Copiar `.env.example` como `.env` y rellenar los valores antes de levantar el stack.

---

## Etapa 1 — Levantar en local

### 1.1 — Servicios Odoo + PostgreSQL ✓

`docker-compose.yml` define dos servicios:
- `db` — PostgreSQL 15 con volumen persistente `postgres_data`
- `odoo` — Odoo 18 con volumen persistente `odoo_data`, expuesto en `localhost:8069`

**Decisión clave:** `POSTGRES_DB=postgres` (no `odoo`). Si se pre-crea una base de datos vacía llamada `odoo`, Odoo la detecta, intenta usarla y falla porque no está inicializada. Con `postgres` como valor, Odoo muestra el asistente de creación de base de datos en el primer acceso y la inicializa correctamente.

Para levantar:

```bash
cp .env.example .env
# editar .env: cambiar POSTGRES_PASSWORD por un valor real
docker compose up -d
```

Acceder a `http://localhost:8069` y completar el asistente de creación de base de datos.

### 1.2 — Configuración de módulos y persistencia ✓

Los módulos se instalan y configuran desde la UI de Odoo: **Apps → buscar módulo → Instalar**.

Toda la configuración (módulos instalados, datos creados, ajustes) se almacena en PostgreSQL y persiste en el volumen `postgres_data`. Al reiniciar los contenedores con `docker compose down && docker compose up -d` el estado se recupera completamente sin ninguna intervención adicional.

**Verificado:** instalación del módulo Empleados (`hr`), creación y eliminación de registros, reinicio completo del stack — estado recuperado íntegramente.

> El volumen se pierde únicamente si se ejecuta `docker compose down -v` o se borra manualmente. De ahí la criticidad del backup planificado en la Etapa 5.

---

## Etapa 2 — Pipeline CI/CD

> *Pendiente de documentar*

---

## Etapa 3 — Configuración del servidor

> *Pendiente de documentar*

---

## Etapa 4 — Acceso remoto

> *Pendiente de documentar*

---

## Etapa 5 — Backups y restauración

> *Pendiente de documentar*

---

## Requisitos previos

> *Pendiente de documentar*

---

## Licencia

Copyright (c) Asociación. Todos los derechos reservados.
Ver [LICENSE](LICENSE) para los términos completos.
