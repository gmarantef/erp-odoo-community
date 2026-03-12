# ERP Odoo Community

ERP basado en Odoo Community 18 para uso interno de la asociación.
Desplegado en servidor local (mini PC) mediante Docker Compose con acceso remoto seguro vía VPN.

---

## Arquitectura

### Enfoque actual: acceso por Tailscale

Todos los accesos al servidor pasan por Tailscale. La granularidad entre usuarios se gestiona mediante una política de acceso (ACL) en Tailscale:

- **Usuarios ERP**: acceso únicamente al puerto 8069 (Odoo)
- **Admins**: acceso completo al servidor (SSH + todos los puertos)

```
        Tailscale (ACL: solo puerto 8069)
       ┌─────────────────────────────────────────┐
       │                                         ▼
  Usuario ERP                  ┌─────────────────────────────────────────────┐
                               │                mini PC (Debian)             │
                               │                                             │
  Admin                        │  ┌──────────┐     ┌─────────────────┐       │
       │                       │  │   Odoo   │────▶│   PostgreSQL    │       │
       └──── Tailscale ────────│─▶│  :8069   │     │     (db)        │       │
             (SSH + todo)      │  └──────────┘     └─────────────────┘       │
                               │        │                                    │
                               │        ▼                                    │
                               │  ┌──────────────────────────┐               │
                               │  │  docker-volume-backup    │─▶ Google Drive│
                               │  └──────────────────────────┘               │
                               └─────────────────────────────────────────────┘
```

Política de acceso Tailscale (ACL):

```json
{
  "groups": {
    "group:admins":   ["admin@asociacion.es"],
    "group:usuarios": ["user1@asociacion.es", "user2@asociacion.es"]
  },
  "acls": [
    { "action": "accept", "src": ["group:admins"],   "dst": ["minipc:*"]    },
    { "action": "accept", "src": ["group:usuarios"], "dst": ["minipc:8069"] }
  ]
}
```

### Alternativa documentada: acceso público con proxy inverso

En esta alternativa los usuarios ERP acceden al servidor a través de internet mediante un dominio, sin necesidad de instalar Tailscale. El acceso admin sigue siendo por Tailscale vía SSH.

La granularidad es estructural: el router solo expone el puerto 443 a internet. El resto del servidor es inaccesible desde fuera de la Tailnet.

**Flujo de un usuario ERP:**

```
Navegador
    │
    │  https://erp.asociacion.es
    ▼
  DNS
    │  resuelve la IP pública del router
    ▼
Router (IP pública)
    │  port forwarding: 443 → IP local del mini PC
    ▼
Traefik :443 (mini PC)
    │  termina TLS (Let's Encrypt), enruta a Odoo
    ▼
Odoo :8069
```

**Flujo de un admin:**

```
Admin → Tailscale → SSH → mini PC
```

**Stack adicional necesario:**

| Elemento | Detalle |
|---|---|
| Dominio | Nombre de dominio propio o DNS |
| DNS | Registro apuntando a la IP pública del router |
| Router | Port forwarding 443 → mini PC |
| Traefik | Proxy inverso + TLS con Let's Encrypt |

---

### Componentes del stack

| Componente | Tecnología |
|---|---|
| Aplicación | Odoo Community 18 |
| Base de datos | PostgreSQL 15 |
| Backup | offen/docker-volume-backup → Google Drive |
| Acceso remoto | Tailscale VPN |
| CI/CD | GitHub Actions (self-hosted runner) |
| OS servidor | Debian |

---

## Etapas de implementación

- [x] Etapa 0 — Definición de arquitectura y contexto
- [x] Etapa 1.1 — Servicios Odoo + PostgreSQL levantados en local y acceso validado
- [x] Etapa 1.2 — Configuración de módulos vía UI y persistencia verificada
- [x] Etapa 1.3 — Configuración del servidor Odoo vía odoo.conf
- [x] Etapa 1.4 — Healthcheck en PostgreSQL y arranque ordenado de servicios
- [x] Etapa 1.5 — docker-compose.override.yml para separación dev/producción
- [x] **Etapa 1 — Docker Compose en local (entorno de desarrollo)**
- [ ] **Etapa 2 — Servidor simulado con Vagrant y despliegue del stack**
- [ ] **Etapa 3 — Validación de acceso remoto vía Tailscale**
- [ ] Etapa 4 — Pipeline CI/CD con GitHub Actions
- [ ] Etapa 5 — Despliegue en mini PC real
- [ ] Etapa 6 — Resiliencia, backups y restauración

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

### 1.3 — Configuración del servidor Odoo ✓

`odoo/config/odoo.conf` montado en el contenedor como `/etc/odoo/odoo.conf` (solo lectura).

Parámetros configurados:
- `addons_path` — las tres rutas que expone la imagen oficial (core, upgrades, extra-addons)
- `log_level = info` + `logfile` vacío → logs a stdout, capturados por Docker
- `proxy_mode = True` — necesario para cuando Traefik actúe como reverse proxy (Etapa 3)
- `workers = 0` — modo threading, adecuado para carga baja. Pendiente ajustar a `(#CPU * 2) + 1` cuando se confirme el hardware del mini PC
- `max_cron_threads = 1`
- Límites de memoria y tiempo documentados para cuando se active modo multiprocess

### 1.4 — Healthcheck PostgreSQL y arranque ordenado ✓

El servicio `db` incluye un healthcheck con `pg_isready` que verifica que PostgreSQL está listo para aceptar conexiones. El servicio `odoo` usa `depends_on: db: condition: service_healthy`, de modo que no arranca hasta que la base de datos esté operativa.

### 1.5 — Separación dev/producción con docker-compose.override.yml ✓

El `docker-compose.yml` base es la configuración de producción: no expone puertos al host (Traefik gestionará el acceso en la Etapa 3).

El `docker-compose.override.yml` añade lo necesario para desarrollo local — actualmente solo el mapeo del puerto 8069. Docker Compose lo fusiona automáticamente al ejecutar `docker compose up`, sin necesidad de flags adicionales.

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
