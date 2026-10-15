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
- **Skill aparte `actualizar-core`** — subir el Xana Core de un cliente de versión: procedimiento,
  fallos conocidos según la versión de origen (fatales de PHP 8, el salto de Uncode 2.11.3),
  migración del `themeXanaSettings.json` y los pasos manuales del escritorio de WordPress.
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
- `incidencias/2026-07-16-alfagres-transients-cron.md` — alfagres.com lento (load 5.5): `wp_options` a 970MB/318k transients (`xanaTransient_` filtros TTL 7d + `external_ip_address_` de WooCommerce geolocation con payloads Log4Shell); cron `delete_expired_transients` atascado 5 meses. Fix: `DISABLE_WP_CRON` + crontab `woman` con WP-CLI `--due-now`. Load 5.5→0.52
- `incidencias/2026-09-14-borrado-usuarios-sin-reasignacion.md` — borrar un usuario con "Atribuir contenido a…" dejaba los posts huérfanos o en la papelera: el hook `remove_user_from_blog` de `functions-xana.php` (borrado global desde 2021) llamaba a `wpmu_delete_user()` antes de que el core reasignara. Fix: no actuar en borrados de red y reasignar en todos los blogs antes de borrar (incluye `wp user delete --reassign`)
- `incidencias/2026-10-15-selector-idioma-mlp-cache-busqueda.md` — Keros: el selector ES/EN de la portada llevaba a `/search/FONTANA/`. La caché `nav_menu.item_filter` de MultilingualPress usa solo el path como clave y, con Redis, una búsqueda `/?s=X` contamina el menú de la portada. Solo afecta a los selectores hechos con elementos de menú `mlp_language`; el `selector__idioma` de Xana Core es seguro. Fix: desactivar esa caché, purgar Varnish y borrar las claves de Redis. Checklist de migración en `arquitectura/multisite-multiidioma.md`

### Cambios
- `incidencias/2026-09-09-profile-php-truncado-subcatalogos.md` — perfil de usuario que se corta a media página: `in_array()` sobre `get_user_meta(...,true)` vacío = fatal en PHP 8, silenciado por `WP_DISABLE_FATAL_ERROR_HANDLER`. El fix (`34037f547`) está en master; la máquina corría una rama suelta 34 commits por detrás.
- `cambios/2026-09-09-urls-core-antiguo-a-nuevo.md` — migrar enlaces del core antiguo (`?idfamilia=N`, `?ucat=N`) a las rutas nuevas por slug de familia (`/{familia}/familia/series|product/`) y a los filtros `?aspecto=` / `?estancia=`. Incluye cómo sacar el mapeo por idioma en un multisite y la trampa de los content blocks EN copiados del ES.
- `cambios/2026-10-15-menu-center-split-responsive.md` — entre 1025 y 1366 px el menú `menu-center-split` partía los enlaces en dos líneas (la flecha debajo del texto), la cabecera crecía a 120 px y el logo encogía hasta 53 px. Causa: columnas laterales fijas (160/120px), `white-space: normal` y logo con `flex-shrink: 1`. Fix en la rama `fix/menu-center-split-responsive`; en adex, replicado en el client.
- `cambios/2026-04-28-transientservice-redis-aware.md` — `wom_iCat\Transient\TransientService` ahora consulta Redis (vía API del plugin redis-cache) en `countActive`/`getAllTransients`/`deleteAll`; antes solo miraba `wp_options` y reportaba 0 con Redis activo. Fallback SQL si Redis no está.

### docs-dev, docs-user
(Vacíos por ahora — añadir según crezca la plataforma)

## Mantenimiento

- Al resolver una incidencia nueva → añadir fichero en `incidencias/YYYY-MM-DD-slug.md` y entrada en el índice de arriba.
- Al aplicar un cambio relevante (feature, refactor, migración) → añadir en `cambios/YYYY-MM-DD-slug.md`.
- Al tomar una decisión arquitectónica → añadir ADR en `decisiones/YYYY-MM-DD-slug.md` con formato: contexto, opciones consideradas, decisión, consecuencias.
- La documentación viva (arquitectura, runbooks, convenciones) se edita en sitio sin crear fechados.
- Mantener el índice de esta SKILL.md actualizado tras cada adición.