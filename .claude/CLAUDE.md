# CLAUDE.md — ERP Odoo Community

## Contexto del proyecto

ERP basado en Odoo Community para una asociación española pequeña (<20 usuarios, <10 simultáneos).
Desplegado en un mini PC local como servidor, gestionado con Docker Compose.

## Stack técnico

| Componente | Elección |
|---|---|
| Odoo | Community v18 |
| Base de datos | PostgreSQL (imagen oficial) |
| Proxy | Traefik |
| Backup | offen/docker-volume-backup + Google Drive (rclone) |
| OS servidor | Debian |
| CI/CD | GitHub Actions con self-hosted runner en el mini PC |
| Acceso remoto | Tailscale VPN |
| Repositorio | GitHub |

## Arquitectura Docker Compose

Cuatro servicios:
1. `odoo` — imagen oficial `odoo:18`
2. `db` — imagen oficial `postgres:15`
3. `traefik` — proxy inverso + TLS
4. `backup` — offen/docker-volume-backup con destino Google Drive

Dos volúmenes persistentes:
- `odoo_data` → filestore de Odoo
- `postgres_data` → datos de PostgreSQL

## Convenciones del proyecto

- **Variables sensibles** siempre en `.env`, nunca hardcodeadas. `.env` está en `.gitignore`
- Mantener `.env.example` actualizado con todas las variables necesarias (sin valores reales)
- Un `docker-compose.override.yml` para configuración local de desarrollo
- No modificar `docker-compose.yml` base para configuraciones locales; usar el override
- Ver `.claude/rules/` para estándares de commits y enfoque de desarrollo

## Comandos habituales

```bash
# Levantar en local (desarrollo)
docker compose up -d

# Ver logs de un servicio
docker compose logs -f odoo

# Parar todo
docker compose down

# Backup manual
docker compose run --rm backup

# Acceder a la DB
docker compose exec db psql -U odoo -d odoo

# Reiniciar solo Odoo
docker compose restart odoo

# Ver estado de todos los servicios
docker compose ps
```

## Etapas del proyecto

| Etapa | Descripción | Estado |
|---|---|---|
| 0 | Configuración inicial del servidor (OS, Docker, SSH, Tailscale) | Pendiente |
| 1 | Docker Compose funcionando en local (host de desarrollo) | Pendiente |
| 2 | Pipeline CI/CD con GitHub Actions y self-hosted runner | Pendiente |
| 3 | Despliegue en mini PC y validación completa | Pendiente |
| 4 | Acceso remoto seguro, seguridad y funcionalidades básicas | Pendiente |
| 5 | Resiliencia: auto-restart, backups y restauración | Pendiente |

## Módulos Odoo activos inicialmente

- `hr` — Empleados (módulo de validación inicial)
- `l10n_es` — Localización española

## Decisiones pendientes de confirmar

- [ ] Dominio/DDNS o estrategia de acceso remoto final (Tailscale vs exposición pública)
- [ ] Hardware exacto del mini PC (CPU) para ajustar workers de Odoo

## Restricciones importantes

- NO commitear el archivo `.env` ni ningún secreto
- NO modificar datos en producción directamente; siempre via UI o scripts controlados
- Los backups deben incluir SIEMPRE DB + filestore de forma coordinada
- Antes de actualizar versión de Odoo, validar estado de `l10n_es` en la nueva versión
