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
- [x] Etapa 1.5 — Puerto 8069 en base compose, acceso delegado a Tailscale ACLs
- [x] **Etapa 1 — Docker Compose en local (entorno de desarrollo)**
- [x] Etapa 2.1 — VM Debian 12 con Vagrant + libvirt, Docker y Tailscale provisionados
- [x] Etapa 2.2 — Deploy key SSH configurada, repositorio clonado en la VM
- [x] Etapa 2.3 — Stack desplegado en la VM y acceso validado desde el host
- [x] Etapa 2.4 — Persistencia verificada tras halt/up de la VM
- [x] **Etapa 2 — Servidor simulado con Vagrant y despliegue del stack**
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
│       ├── development-approach.md
│       └── error-solving.md
├── odoo/
│   └── config/
│       └── odoo.conf           # Configuración de Odoo
├── backup/                     # Configuración del servicio de backup (Etapa 6)
├── docker-compose.yml          # Stack principal
├── Vagrantfile                 # VM para servidor simulado (Etapa 2)
└── .env.example                # Variables de entorno requeridas (sin valores reales)
```

> **Nota:** El archivo `.env` nunca se commitea. Copiar `.env.example` como `.env` y rellenar los valores antes de levantar el stack.

---

## Etapa 1 — Docker Compose en local

### 1.1 — Servicios Odoo + PostgreSQL ✓

`docker-compose.yml` define dos servicios:
- `db` — PostgreSQL 15 con volumen persistente `postgres_data`
- `odoo` — Odoo 18 con volumen persistente `odoo_data`, expuesto en el puerto 8069

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

> El volumen se pierde únicamente si se ejecuta `docker compose down -v` o se borra manualmente. De ahí la criticidad del backup planificado en la Etapa 6.

### 1.3 — Configuración del servidor Odoo ✓

`odoo/config/odoo.conf` montado en el contenedor como `/etc/odoo/odoo.conf` (solo lectura).

Parámetros configurados:
- `addons_path` — las tres rutas que expone la imagen oficial (core, upgrades, extra-addons)
- `log_level = info` + `logfile` vacío → logs a stdout, capturados por Docker
- `proxy_mode = True` — preparado para si en el futuro se añade Traefik como proxy inverso
- `workers = 0` — modo threading, adecuado para carga baja. Pendiente ajustar a `(#CPU * 2) + 1` cuando se confirme el hardware del mini PC
- `max_cron_threads = 1`
- Límites de memoria y tiempo documentados para cuando se active modo multiprocess

### 1.4 — Healthcheck PostgreSQL y arranque ordenado ✓

El servicio `db` incluye un healthcheck con `pg_isready` que verifica que PostgreSQL está listo para aceptar conexiones. El servicio `odoo` usa `depends_on: db: condition: service_healthy`, de modo que no arranca hasta que la base de datos esté operativa.

### 1.5 — Puerto 8069 en base compose ✓

El puerto 8069 se expone directamente en `docker-compose.yml`. El control de acceso se delega a las ACLs de Tailscale a nivel de red, no al binding de puertos. No existe fichero override — si en el futuro se necesitan overrides de desarrollo, se crea en ese momento.

---

## Etapa 2 — Servidor simulado con Vagrant

### 2.1 — VM Debian 12 con Vagrant ✓

`Vagrantfile` en la raíz del repositorio define una VM Debian 12 (bookworm) con provider libvirt:
- 2 CPUs, 2 GB RAM
- Docker + Docker Compose plugin provisionados automáticamente (método oficial Docker para Debian)
- Tailscale instalado automáticamente
- Carpeta sincronizada deshabilitada — el despliegue se gestiona vía SSH

Para levantar la VM:

```bash
vagrant up       # crea y provisiona la VM
vagrant ssh      # acceso SSH a la VM
vagrant halt     # apagar la VM (los datos persisten)
vagrant destroy  # eliminar la VM y su disco
```

### 2.2 — Deploy key SSH ✓

El acceso al repositorio privado desde el servidor se gestiona mediante una Deploy Key SSH, que es la práctica estándar para servidores permanentes.

```bash
# Dentro de la VM
ssh-keygen -t ed25519 -C "odoo-server" -f ~/.ssh/github_deploy -N ""
cat ~/.ssh/github_deploy.pub
# → añadir en GitHub: repo → Settings → Deploy keys
```

Configuración SSH en la VM (`~/.ssh/config`):

```
Host github.com
  IdentityFile ~/.ssh/github_deploy
  IdentitiesOnly yes
```

### 2.3 — Despliegue del stack ✓

```bash
# Dentro de la VM
git clone git@github.com:USUARIO/REPO.git ~/erp
cd ~/erp
cp .env.example .env
# editar .env con los valores reales
nano .env
docker compose up -d
```

Odoo accesible desde el host en `http://<IP-VM>:8069` (la IP de la interfaz libvirt, visible con `hostname -I` dentro de la VM).

### 2.4 — Persistencia verificada ✓

Los volúmenes Docker (`postgres_data`, `odoo_data`) persisten entre reinicios de la VM:

```bash
vagrant halt && vagrant up  # los datos del ERP se mantienen
```

Los datos se pierden únicamente con `vagrant destroy`, equivalente a formatear el disco del servidor.

---

## Etapa 3 — Acceso remoto vía Tailscale

> *Pendiente de documentar*

---

## Etapa 4 — Pipeline CI/CD

> *Pendiente de documentar*

---

## Etapa 5 — Despliegue en mini PC real

> *Pendiente de documentar*

---

## Etapa 6 — Resiliencia, backups y restauración

> *Pendiente de documentar*

---

## Licencia

Copyright (c) Asociación. Todos los derechos reservados.
Ver [LICENSE](LICENSE) para los términos completos.
