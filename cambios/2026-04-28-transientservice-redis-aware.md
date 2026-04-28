# 2026-04-28 — TransientService Redis-aware

## Contexto

El plugin `wom_iCat` (clase `Xana\Transient\TransientService`) cachea queries pesadas de filtros y agregaciones en transients prefijados `xanaTransient_*`. La sección admin del plugin (`/wp-admin/admin.php?page=xana_plugin_transients`) los lista, cuenta y permite borrarlos.

`countActive`, `getAllTransients` y `deleteAll` consultaban directamente `wp_options` con SQL. Cuando Redis Object Cache está activo, WordPress redirige `set_transient/get_transient` a Redis (vía `wp_cache_*`) y **no escribe en `wp_options`** → el panel admin reportaba 0 transients aunque internamente el cache funcionaba bien.

## Cambio aplicado

Hacer las tres funciones Redis-aware con fallback SQL.

**Helpers privados nuevos:**
- `getRedisInstance()` — devuelve la instancia Redis del plugin redis-cache si está activa (`wp_using_ext_object_cache()` + método `redis_instance()` + `redis_status()`); si no, `null`.
- `scanXanaTransientKeys()` — `SCAN` paginado (no `KEYS`, que bloquea Redis) con patrón `*:transient:xanaTransient_*` (cubre cualquier blogid + `WP_CACHE_KEY_SALT` configurado). Devuelve `[fullKey => transientName]` o `null` si Redis no está.

**Métodos modificados:**
- `countActive()` — si SCAN devuelve array → `count()`; si null → SQL original.
- `getAllTransients()` — itera keys, lee valor con `get_transient(...)` (que enruta a Redis), TTL con `$redis->ttl($fullKey)`. Fallback SQL.
- `deleteAll()` — itera keys, `delete_transient(...)` por cada uno. Fallback SQL.

**No tocados:** `set`, `get`, `deleteSpecific`, `isEnabled`, `setEnabled`, `formatTransientName` — la API de transients de WP ya enruta sola.

## Verificación

En `demolive.xanasystem.com` (Redis ON, drop-in `Valid`):
- Pre-patch: `countActive() = 0`, panel admin vacío, pero `redis-cli SCAN '*xanaTransient*'` mostraba 2+ keys reales en Redis DB 3.
- Post-patch: `countActive() = 54` (acumulando con el tráfico real), `getAllTransients()` devuelve nombres + TTL correctos.

En `demolive.digitaltilecatalog.com` (Redis OFF): cae al fallback SQL, comportamiento idéntico al original.

## Archivos y commits

- Repo `wp-content-xana`, archivo `plugins/wom_iCat/src/Transient/TransientService.php` (+114 / -6).
- Commit en `develop`: `f7425bd1b` — `Feature: TransientService Redis-aware (countActive/getAllTransients/deleteAll)`.
- Merge a `master`: `8cc6ed547` — `Merge branch 'develop'`.
- Propagado a los dos demos del servidor (`164.92.137.227`) con el patrón "pisado in-situ" del git-flow (`git checkout --` + `git pull --ff-only`).

## Pendiente / siguiente paso

Cuando aparezca otra cookie/sesión propia de un plugin Xana que requiera no-cache (similar al patrón ya documentado para `wom_icat_favorites` en VCL), revisar también si el plugin lista/cuenta algo desde `wp_options` y aplicar el mismo patrón Redis-aware.
