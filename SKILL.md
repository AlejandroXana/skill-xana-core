---
name: xana-core
description: Base de conocimiento de la plataforma Xana Core (WordPress/WooCommerce personalizado del equipo Xana Technologies). Usar cuando se trabaje en sitios Xana Core, cuando aparezcan repos como `wp-content-xana`, plugins `wom_iCat*`/`wom-iCat-Outlet`/`XanaSettings`, temas `uncode-child`, constantes `XANA_*`, namespaces `xana\\theme` o `xana\\outlet\\theme`, rutas `/var/www/wordpress-stack/sites/*`, o dominios demolive (demolive.xanasystem.com, demolive.digitaltilecatalog.com, younginteriorsflooring, etc.). Contiene arquitectura del stack, runbooks operativos, incidencias resueltas, convenciones del equipo y material base para documentación de desarrolladores y usuarios.
---

# Xana Core — Base de conocimiento

Plataforma WordPress/WooCommerce personalizada desarrollada por Xana Technologies. Repo canónico: `git@bitbucket.org:xana-technologies/wp-content-xana.git` (compartido entre múltiples sitios).

## Cómo navegar esta skill

Antes de responder al usuario sobre cualquier tema de Xana Core, identificar qué categoría aplica y leer el fichero relevante. **No leer todo de golpe** — ir por el índice y cargar solo lo que necesitas.

```
arquitectura/   → Cómo funciona la plataforma (stack, componentes, integraciones)
incidencias/    → Bugs/fatales ya resueltos (nombre: YYYY-MM-DD-slug.md)
cambios/        → Features/refactors/migraciones aplicados (nombre: YYYY-MM-DD-slug.md)
decisiones/     → ADRs: decisiones arquitectónicas con rationale (nombre: YYYY-MM-DD-slug.md)
runbooks/       → Procedimientos operativos repetibles (updates, backups, comandos)
convenciones/   → Normas del equipo (git, commits, formato, estilo)
docs-dev/       → Material base para documentación de desarrolladores
docs-user/      → Material base para documentación de usuarios finales
```

## Índice actual

### Arquitectura
- `arquitectura/stack.md` — stack Docker unificado dev+prod (`wordpress-stack`), layout, repos involucrados, backups, multisite
- `arquitectura/multisite-multiidioma.md` — los sitios del catálogo usan WP Multisite con un blog por idioma. Rationale histórico e implicaciones operativas.
- `arquitectura/repos-wp-content.md` — inventario de repos git dentro de `wp-content/` (wp-content-xana + subrepos por plugin/tema), workspaces Bitbucket, y cuáles están deprecados.
- `arquitectura/cache-invalidation.md` — 4 capas de cache (Varnish, Redis, transients `xanaTransient_*`, WP Rocket) y cómo se invalidan al final de `syncPimDatabase`. Incluye la convención del prefijo `xanaTransient_` y el patrón `X-Xana-No-Cache` para bypass selectivo por respuesta desde el backend (páginas personalizadas tipo wishlist/carrito sin hardcodear URLs en VCL).

### Runbooks
- `runbooks/wp-cli-y-opcache.md` — patrón `--skip-plugins --skip-themes` y recarga FPM con `kill -USR2 1`

### Convenciones
- `convenciones/git-flow.md` — flujo develop→master vía PR, push directo aceptado para hotfixes

### Decisiones (ADR)
- `decisiones/2026-04-21-unificar-stack-dev-prod.md` — adoptar `wordpress-stack` también en local para eliminar drift dev/prod

### Incidencias
- `incidencias/2026-04-17-fatal-xana-text-domain.md` — `if (!XANA_THEME_TEXT_DOMAIN)` → fatal PHP 8
- `incidencias/2026-04-21-fatal-callbacks-outlet.md` — callbacks huérfanos `\xana\outlet\theme\wp_login` → 500 en login
- `incidencias/2026-04-21-update-wp-demolive-xanasystem.md` — desfase WP 5.9 vs WooCommerce 10, update core con wp-cli
- `incidencias/2026-04-21-fatal-favorites-array-keys-null.md` — `array_keys(null)` en wom-iCat-Favorites corta single-product a mitad (faltaba packing y relacionados)

### Cambios
- `cambios/2026-04-28-transientservice-redis-aware.md` — `wom_iCat\Transient\TransientService` ahora consulta Redis (vía API del plugin redis-cache) en `countActive`/`getAllTransients`/`deleteAll`; antes solo miraba `wp_options` y reportaba 0 con Redis activo. Fallback SQL si Redis no está.

### docs-dev, docs-user
(Vacíos por ahora — añadir según crezca la plataforma)

## Mantenimiento

- Al resolver una incidencia nueva → añadir fichero en `incidencias/YYYY-MM-DD-slug.md` y entrada en el índice de arriba.
- Al aplicar un cambio relevante (feature, refactor, migración) → añadir en `cambios/YYYY-MM-DD-slug.md`.
- Al tomar una decisión arquitectónica → añadir ADR en `decisiones/YYYY-MM-DD-slug.md` con formato: contexto, opciones consideradas, decisión, consecuencias.
- La documentación viva (arquitectura, runbooks, convenciones) se edita en sitio sin crear fechados.
- Mantener el índice de esta SKILL.md actualizado tras cada adición.