# ERP Odoo Community

ERP basado en Odoo Community 18 para uso interno de la asociación.
Desplegado en VPS Hetzner Cloud (CPX22) mediante Docker Compose con acceso remoto seguro vía VPN.

---

## Arquitectura

### Enfoque actual: acceso por Tailscale

Todos los accesos al servidor pasan por Tailscale. La granularidad entre usuarios se gestiona mediante una política de acceso (ACL) en Tailscale:

- **Usuarios ERP**: acceso únicamente al puerto 8069 (Odoo)
- **Admins**: acceso completo al servidor (Tailscale SSH + todos los puertos)

```
        Tailscale (ACL: solo puerto 8069)
       ┌─────────────────────────────────────────┐
       │                                         ▼
  Usuario ERP                  ┌─────────────────────────────────────────────┐
                               │            VPS Hetzner (Debian)             │
                               │                                             │
  Admin                        │  ┌──────────┐     ┌─────────────────┐       │
       │                       │  │   Odoo   │────▶│   PostgreSQL    │       │
       └──── Tailscale SSH ────│─▶│  :8069   │     │     (db)        │       │
             (admins)          │  └──────────┘     └─────────────────┘       │
                               │        │                                    │
                               │        ▼                                    │
                               │  ┌──────────────────────────┐               │
                               │  │  docker-volume-backup    │─▶ Google Drive│
                               │  └──────────────────────────┘               │
                               └─────────────────────────────────────────────┘
```

Política de acceso Tailscale (ver [`.tailscale/policy.hujson`](.tailscale/policy.hujson)):

```json
{
    "tagOwners": {
        "tag:server":   ["autogroup:admin"],
        "tag:erp-user": ["autogroup:admin"],
        "tag:ci":       ["autogroup:admin"]
    },
    "grants": [
        { "src": ["autogroup:admin"], "dst": ["tag:server"], "ip": ["*"] },
        { "src": ["tag:erp-user"],    "dst": ["tag:server"], "ip": ["tcp:8069"] },
        { "src": ["tag:ci"],          "dst": ["tag:server"], "ip": ["tcp:22"] }
    ],
    "ssh": [
        {
            "action": "accept",
            "src":    ["autogroup:admin"],
            "dst":    ["tag:server"],
            "users":  ["root", "autogroup:nonroot"]
        }
    ]
}
```

### Alternativa documentada: acceso público con proxy inverso

En esta alternativa los usuarios ERP acceden al servidor a través de internet mediante un dominio, sin necesidad de instalar Tailscale. El acceso admin sigue siendo por Tailscale SSH.

Con un VPS la granularidad es aún más sencilla: no hay router ni port forwarding, el VPS tiene IP pública directa. Solo se expone el puerto 443 al público.

**Flujo de un usuario ERP:**

```
Navegador
    │
    │  https://erp.asociacion.es
    ▼
  DNS
    │  resuelve la IP pública del VPS
    ▼
Traefik :443 (VPS)
    │  termina TLS (Let's Encrypt), enruta a Odoo
    ▼
Odoo :8069
```

**Flujo de un admin:**

```
Admin → Tailscale SSH → VPS
```

**Stack adicional necesario:**

| Elemento | Detalle |
|---|---|
| Dominio | Nombre de dominio propio o DNS |
| DNS | Registro apuntando a la IP pública del VPS |
| Traefik | Proxy inverso + TLS con Let's Encrypt |

---

### Componentes del stack

| Componente | Tecnología |
|---|---|
| Aplicación | Odoo Community 18 |
| Base de datos | PostgreSQL 15 |
| Backup | offen/docker-volume-backup → Google Drive |
| Acceso remoto | Tailscale VPN |
| CI/CD | GitHub Actions (runner alojado en GitHub) |
| OS servidor | Debian 12 |
| Servidor | Hetzner Cloud CPX22 (escalable a CPX32) |

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
- [x] Etapa 3.1 — Cuenta Tailscale y dispositivos conectados al tailnet
- [x] Etapa 3.2 — Política de acceso (ACL) con tag:server y tag:erp-user
- [x] Etapa 3.3 — Conexión del servidor al tailnet con auth key
- [x] Etapa 3.4 — Validación de acceso remoto desde dispositivo externo
- [x] **Etapa 3 — Validación de acceso remoto vía Tailscale**
- [x] Etapa 4.1 — Política Tailscale actualizada con tag:ci (acceso SSH al servidor)
- [x] Etapa 4.2 — OAuth client en Tailscale (setup manual, ver sección 4.2)
- [x] Etapa 4.3 — Clave SSH dedicada para el CI (setup manual, ver sección 4.3)
- [x] Etapa 4.4 — Secretos configurados en GitHub (setup manual, ver sección 4.4)
- [x] Etapa 4.5 — Workflow deploy.yml creado
- [x] **Etapa 4 — Pipeline CI/CD con GitHub Actions**
- [ ] Etapa 5.1 — VPS CPX22 creado en Hetzner con Debian 12
- [ ] Etapa 5.2 — Docker instalado en el VPS
- [ ] Etapa 5.3 — Tailscale instalado, nodo conectado al tailnet con tag:server
- [ ] Etapa 5.4 — Tailscale SSH activado en el VPS
- [ ] Etapa 5.5 — Hetzner Firewall configurado (puerto 22 cerrado al público)
- [ ] Etapa 5.6 — Clave SSH del CI añadida al VPS
- [ ] Etapa 5.7 — Secretos de GitHub actualizados (SERVER_HOST, SERVER_USER)
- [ ] Etapa 5.8 — Primer despliegue automático validado
- [ ] **Etapa 5 — Despliegue en VPS Hetzner**
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
├── .tailscale/
│   └── policy.hujson           # Política de acceso Tailscale (ACL)
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
- `workers = 0` — modo threading, adecuado para desarrollo. Para producción en CPX22 (2 vCPUs): `workers = 5`. Se activará en Etapa 5.
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

### 3.1 — Cuenta Tailscale y dispositivos conectados ✓

Se crea una cuenta Tailscale (tailnet personal). Los dispositivos se conectan vía SSO (Google/GitHub). El servidor se conecta mediante **auth key** para evitar autenticación interactiva.

Dispositivos del tailnet:
- Desktop personal — host de desarrollo y administración (admin)
- Móvil Android — dispositivo de prueba (conectado como admin para esta etapa; ver nota más abajo)
- VM / servidor — conectado con auth key y `tag:server`

### 3.2 — Política de acceso (ACL) ✓

La política se define antes de generar la auth key del servidor, ya que el tag debe existir en la política para poder asignarlo. Archivo versionado en [`.tailscale/policy.hujson`](.tailscale/policy.hujson).

Dos tags:
- `tag:server` — dispositivos servidor. Accesibles desde admins en cualquier puerto y desde `tag:erp-user` solo en el 8069.
- `tag:erp-user` — usuarios ERP. Solo pueden iniciar conexiones al puerto 8069 del servidor.

Aplicar: copiar el contenido del archivo y pegarlo en `login.tailscale.com/admin/acls`.

### 3.3 — Conexión del servidor al tailnet ✓

El Vagrantfile ya instala Tailscale en la VM. Para conectarla al tailnet:

```bash
# Generar auth key en: login.tailscale.com/admin/settings/keys
# Opciones: reusable, no ephemeral, pre-approved, tag: tag:server

# Dentro de la VM
sudo tailscale up --auth-key=<auth-key>
```

La VM aparece en el tailnet con `tag:server` y recibe una IP fija `100.x.x.x`.

### 3.4 — Validación de acceso remoto ✓

Acceso verificado desde el móvil (Pixel 6a) con datos móviles (fuera de la red local) a `http://<IP-tailscale-VM>:8069`. Odoo responde correctamente a través de la VPN.

**Nota sobre `tag:erp-user`:** La app Tailscale en Android no admite autenticación con auth key, solo SSO. Por tanto no es posible etiquetar un dispositivo personal como `tag:erp-user` sin una segunda cuenta. La política está definida y lista; la validación de acceso restringido se realizará en Etapa 5 cuando los usuarios reales de la asociación se incorporen al tailnet con sus propias cuentas.

---

## Etapa 4 — Pipeline CI/CD

### Diseño

Cada push a `main` lanza un workflow de GitHub Actions que conecta el runner temporalmente al tailnet de Tailscale para desplegar en el servidor vía SSH.

```
push to main
    │
    ▼
[GitHub runner ubuntu-latest]
    ├── actions/checkout@v4
    ├── tailscale/github-action@v4  →  nodo efímero tag:ci se une al tailnet
    ├── tailscale ping $SERVER_HOST  →  verifica conectividad (hasta 3 min)
    ├── SSH al servidor (IP Tailscale)
    │     ├── git pull
    │     ├── regenerar .env desde secrets de GitHub
    │     └── docker compose up -d --remove-orphans
    └── nodo efímero se destruye automáticamente
```

El runner nunca es accesible desde internet: solo existe en el tailnet durante la ejecución del job. El acceso al servidor está limitado al puerto 22 mediante la política ACL (`tag:ci` → `tag:server` → `tcp:22`).

### 4.1 — Actualización de la política Tailscale ✓

Se añade `tag:ci` para los nodos efímeros del runner. El tag está definido en [`.tailscale/policy.hujson`](.tailscale/policy.hujson) con acceso exclusivo al puerto 22 del servidor.

Aplicar la política actualizada: copiar el contenido del archivo y pegarlo en `login.tailscale.com/admin/acls`.

### 4.2 — OAuth client en Tailscale

En `login.tailscale.com/admin/settings/oauth-clients`, crear un OAuth client con el scope **Auth Keys → Write**.

Guardar el Client ID y el Client secret — solo se muestran una vez.

### 4.3 — Clave SSH dedicada para el CI

Generar un par de claves exclusivo para el runner (no reutilizar la deploy key del repositorio):

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions -N ""
```

Añadir la clave pública al VPS (paso de Etapa 5):

```bash
# Dentro del VPS
echo "<contenido de github_actions.pub>" >> ~/.ssh/authorized_keys
```

### 4.4 — Secretos en GitHub

En el repositorio → **Settings → Secrets and variables → Actions**, añadir:

| Secret | Valor |
|---|---|
| `TS_OAUTH_CLIENT_ID` | Client ID del OAuth client de Tailscale |
| `TS_OAUTH_SECRET` | Client secret del OAuth client de Tailscale |
| `SSH_PRIVATE_KEY` | Contenido de `~/.ssh/github_actions` (clave privada) |
| `SERVER_HOST` | IP Tailscale del servidor (`100.x.x.x`) |
| `SERVER_USER` | Usuario SSH en el VPS (ej: `debian`) |
| `POSTGRES_PASSWORD` | Contraseña de PostgreSQL |

### 4.5 — Workflow ✓

El archivo [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) define el pipeline completo. Se activa automáticamente en cada push a `main`.

---

## Etapa 5 — Despliegue en VPS Hetzner

### 5.1 — Crear el VPS en Hetzner

En la consola de Hetzner Cloud (`console.hetzner.cloud`), crear un nuevo servidor con:

- **Imagen**: Debian 12
- **Tipo**: CPX22 (escalable a CPX32 si se requiere más capacidad)
- **Región**: Falkenstein o Nuremberg (EU, baja latencia desde España)
- **SSH key**: añadir la clave pública del admin durante la creación

El VPS arranca con acceso root por SSH a su IP pública.

### 5.2 — Instalar Docker

```bash
# Conectar al VPS por SSH (acceso inicial vía IP pública)
ssh root@<IP-pública-VPS>

# Instalación oficial Docker para Debian
apt-get update
apt-get install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 5.3 — Instalar Tailscale y conectar al tailnet

```bash
# Instalación oficial Tailscale para Debian
curl -fsSL https://tailscale.com/install.sh | sh

# Generar auth key en: login.tailscale.com/admin/settings/keys
# Opciones: reusable, no ephemeral, pre-approved, tag: tag:server
sudo tailscale up --auth-key=<auth-key>
```

El VPS aparece en el tailnet con `tag:server` y recibe una IP fija `100.x.x.x`.

### 5.4 — Activar Tailscale SSH

```bash
sudo tailscale set --ssh
```

A partir de aquí los admins acceden al VPS directamente con:

```bash
ssh root@<nombre-nodo-tailscale>
# o por IP Tailscale
ssh root@100.x.x.x
```

Sin necesidad de claves SSH para los admins — la autenticación la gestiona Tailscale.

### 5.5 — Cerrar puerto 22 al tráfico público

En la consola de Hetzner → **Firewalls** → crear regla que bloquee el puerto 22 entrante desde internet. Todo el acceso SSH pasa a requerir conexión al tailnet.

El CI/CD sigue funcionando porque su nodo efímero (`tag:ci`) se une al tailnet antes de hacer el SSH.

### 5.6 — Preparar el VPS para el CI/CD

```bash
# Crear el directorio de trabajo y clonar el repositorio
# (deploy key SSH, igual que en Etapa 2 con la VM)
ssh-keygen -t ed25519 -C "odoo-server" -f ~/.ssh/github_deploy -N ""
cat ~/.ssh/github_deploy.pub
# → añadir en GitHub: repo → Settings → Deploy keys
```

```
# ~/.ssh/config
Host github.com
  IdentityFile ~/.ssh/github_deploy
  IdentitiesOnly yes
```

```bash
git clone git@github.com:USUARIO/REPO.git ~/erp
```

Añadir también la clave pública del CI (generada en Etapa 4.3):

```bash
echo "<contenido de github_actions.pub>" >> ~/.ssh/authorized_keys
```

### 5.7 — Actualizar secretos de GitHub

Actualizar en el repositorio → **Settings → Secrets and variables → Actions**:

| Secret | Valor |
|---|---|
| `SERVER_HOST` | IP Tailscale del VPS (`100.x.x.x`) |
| `SERVER_USER` | Usuario SSH en el VPS (ej: `debian`) |

El resto de secretos (`TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET`, `SSH_PRIVATE_KEY`, `POSTGRES_PASSWORD`) no cambian si ya estaban configurados de Etapa 4.

### 5.8 — Primer despliegue automático

Mergear `feat/cicd-pipeline` a `main`. El pipeline se activa automáticamente y despliega el stack en el VPS por primera vez.

---

## Etapa 6 — Resiliencia, backups y restauración

> *Pendiente de documentar*

---

## Licencia

Copyright (c) Asociación. Todos los derechos reservados.
Ver [LICENSE](LICENSE) para los términos completos.
