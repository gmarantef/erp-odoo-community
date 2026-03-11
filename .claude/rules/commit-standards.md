# Commit Standards

Este proyecto sigue la especificación **Conventional Commits 1.0.0**.
Referencia oficial: https://www.conventionalcommits.org/en/v1.0.0/

## Formato

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

## Tipos permitidos

| Tipo | Cuándo usarlo |
|---|---|
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de bug |
| `docs` | Cambios solo en documentación |
| `style` | Cambios de formato sin efecto en lógica (espacios, comas...) |
| `refactor` | Refactorización sin nueva funcionalidad ni fix |
| `test` | Añadir o corregir tests |
| `chore` | Tareas de mantenimiento (deps, config, CI...) |
| `ci` | Cambios en la pipeline CI/CD |
| `revert` | Revertir un commit anterior |

## Scopes del proyecto

| Scope | Qué cubre |
|---|---|
| `docker` | docker-compose.yml, Dockerfiles, override |
| `odoo` | Configuración de Odoo (odoo.conf, módulos) |
| `db` | PostgreSQL, migraciones |
| `traefik` | Configuración del proxy |
| `backup` | Servicio de backup, scripts de restauración |
| `ci` | GitHub Actions workflows |
| `infra` | Scripts de servidor, Tailscale, OS |
| `env` | Variables de entorno, .env.example |

## Reglas

- La `<description>` en **inglés**, en imperativo presente ("add", "fix", "update" — no "added", "fixed")
- Máximo 72 caracteres en la primera línea
- El body explica el **por qué**, no el qué
- BREAKING CHANGE se indica en el footer con `BREAKING CHANGE: <descripción>`

## Ejemplos válidos

```
feat(docker): add offen/docker-volume-backup service

chore(env): add Google Drive credentials variables to .env.example

fix(odoo): correct database host reference in odoo.conf

ci: add self-hosted runner deployment workflow

docs: update etapa 1 setup instructions
```

## Ejemplos inválidos

```
# Sin tipo
updated docker compose

# Descripción en pasado
feat(docker): added backup service

# Demasiado vago
fix: fixed bug

# En español
feat(odoo): añadir módulo de empleados
```
