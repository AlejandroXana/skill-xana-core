# Fatal 500 en login: callbacks huérfanos en namespace `xana\outlet\theme`

**Fecha**: 2026-04-21
**Entorno afectado**: Sitios Xana Core con plugin `wom-iCat-Outlet` y PHP 8.x
**Detectado en**: demolive.xanasystem.com al intentar acceder a `/wp-login.php`
**Commit del fix**: `a8ffd6853` (develop) → merge `f580ab170` (master)

## Síntoma

```
Fatal error: Uncaught TypeError: call_user_func_array(): Argument #1 ($callback)
  must be a valid callback, function "\xana\outlet\theme\wp_login" not found
  or invalid function name in wp-includes/class-wp-hook.php:341
```

`/wp-login.php` devolvía HTTP 500 en cualquier intento de acceder. `/wp-admin/` funcionaba porque no dispara el hook `authenticate` hasta que alguien se loguea.

**Nota**: el fatal estaba latente desde siempre, pero antes de desactivar `display_errors` y de reiniciar FPM, los Deprecated de PHP 8 emitían output previo al fatal, de modo que el servidor devolvía `200` con HTML roto en vez de `500` limpio. Tras limpiar el debug, salió a la luz como 500 real.

## Causa raíz

En `plugins/wom-iCat-Outlet/functions-xana.php`, namespace `xana\outlet\theme`, había dos registros de hook con callbacks que **nunca existieron** en ese namespace:

```php
add_action('remove_user_from_blog', '\xana\outlet\theme\remove_user_from_blog', 10, 3);
add_action('authenticate', '\xana\outlet\theme\wp_login', 10, 3);
```

Las funciones `wp_login()` y `remove_user_from_blog()` **sí existen** pero en el tema `uncode-child/functions-xana.php`, namespace `xana\theme` — no en el plugin. El tema ya registra los hooks correctos:

```php
// themes/uncode-child/functions-xana.php (namespace xana\theme)
add_action('authenticate', '\xana\theme\wp_login', 10, 3);
```

Los registros del plugin eran **redundantes** (el tema ya cubre los hooks) y **rotos** (las funciones no están definidas en el namespace del plugin).

Origen probable: copy-paste del bloque del tema al plugin cuando se creó `wom-iCat-Outlet`, con replace `xana\theme` → `xana\outlet\theme`, pero sin reimplementar las funciones.

## Fix

Eliminar ambas líneas del plugin. El tema sigue cubriendo los hooks con el namespace correcto.

## Diagnóstico rápido en otros sitios

```bash
grep -n "xana.outlet.theme.wp_login\|xana.outlet.theme.remove_user_from_blog" \
  /var/www/wordpress-stack/sites/<dominio>/wp-content/plugins/wom-iCat-Outlet/functions-xana.php
```

Si sale algo, ese sitio aún tiene el bug. Hacer `git pull` en ese sitio.

## Propagación manual (cuando los cambios locales impiden pull)

Si el sitio tiene el archivo modificado in-situ (porque aplicamos fix rápido), para sincronizar con el repo:

```bash
cd /var/www/wordpress-stack/sites/<dominio>/wp-content
git checkout -- plugins/wom-iCat-Outlet/functions-xana.php
git pull --ff-only origin master
docker exec php-fpm kill -USR2 1
```

## Lección

Este es el **segundo bug del mismo patrón** en este plugin (ver `2026-04-17-fatal-xana-text-domain.md` como variante previa). Revisar periódicamente `functions-xana.php` del plugin outlet buscando otras funciones del namespace `xana\outlet\theme` que no estén definidas pero sí registradas como callbacks.