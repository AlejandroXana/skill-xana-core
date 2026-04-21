---
name: Fatal array_keys(null) en wom-iCat-Favorites — single-product se corta a mitad
description: Fatal PHP 8 en woocommerce_single_product_summary del plugin Favorites cortaba el render de /product/...; el síntoma es HTTP 500 con HTML parcial (falta packing, relacionados, footer)
---

# Fatal `array_keys(null)` en wom-iCat-Favorites — single-product cortado a mitad

**Fecha**: 2026-04-21
**Entorno afectado**: `demolive.xanasystem.com` (y probablemente cualquier otro sitio Xana Core con PHP 8 + usuario sin favoritos en la cookie)
**URL concreta del bug**: `/product/white-body-ceramic-white-dove-mini-picket/`

## Síntoma

Página de producto devolvía **HTTP 500 con HTML parcial** (~45 KB, cortado a mitad del bloque de características, sin `</body></html>`). El usuario percibía que "no carga completamente": faltaban packing, productos relacionados, series relacionadas, footer.

`HEAD` respondía 200 porque no ejecuta el body PHP completo; solo `GET` disparaba el fatal.

## Causa raíz

En `plugins/wom-iCat-Favorites/wom-iCat-Favorites.php:175`:

```php
// Roto
if (in_array($post->ID, array_keys($favorites['main']))) {
```

Cuando el visitante no tiene cookie de favoritos, `$favorites['main']` es `null`. PHP 8 ya no acepta `null` en `array_keys()` → `TypeError` → muere el request en mitad del hook `woocommerce_single_product_summary`.

El callback está enganchado en `wom\icat\favorites\woocommerce_single_product_summary()`, que corre antes de packing/relacionados → todo lo de abajo no se renderiza.

## Fix

```php
// Línea 175
if (!empty($favorites['main']) && in_array($post->ID, array_keys($favorites['main']))) {
```

Aplicado in-place en el servidor. Tras editar:

```bash
docker exec php-fpm kill -USR2 1                                        # recargar OPcache
docker exec varnish varnishadm 'ban req.url ~ /product/<slug>'          # purgar cache
```

Verificación: `curl` a la URL devuelve 200 y 88 KB con `</body></html>` al final.

## Detalle importante: por qué no se propaga por git

`wp-content-xana/.gitignore` incluye `/plugins/*` — **los plugins no están en git**. Mi copia local (younginteriorsflooring) ya tenía este fix aplicado manualmente hace tiempo, pero `git pull` en demolive no traía nada porque el archivo nunca se versionó.

Cualquier fix en plugins Xana Core (wom_iCat, wom-iCat-Outlet, wom-iCat-Favorites, wom-iCat-admin, plugin-xanasettings/XanaSettings, xana-dtc-images, limpiar_zona_usuario, InicioWoman) debe **aplicarse sitio por sitio a mano** hasta que el equipo decida versionar los plugins.

## Diagnóstico — notas útiles

- `WP_DEBUG=false` + `WP_DEBUG_LOG=false` + `display_errors=0` oculta TODO: ni en `wp-content/debug.log`, ni en `/var/log/php/error.log`. Hay que activar **ambas** (`WP_DEBUG=true` + `WP_DEBUG_LOG=true`) para que los fatales lleguen a `debug.log`. `WP_DEBUG_DISPLAY=false` deja el HTML limpio.
- Comando exacto:
  ```bash
  docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && \
    wp --allow-root --skip-plugins --skip-themes config set WP_DEBUG true  --raw --type=constant && \
    wp --allow-root --skip-plugins --skip-themes config set WP_DEBUG_LOG true --raw --type=constant'
  docker exec php-fpm kill -USR2 1
  ```
  Desactivar al acabar, **no dejar debug encendido en producción**.
- Varnish **no cachea 500 con `Cache-Control: no-cache`**, pero por seguridad se puede purgar con `varnishadm 'ban req.url ~ /ruta'`.
- El log `/var/log/php/error.log` dentro del contenedor `php-fpm` tiene los warnings de startup (imagick missing) y fatales antiguos — no los fatales actuales de WP sin `WP_DEBUG_LOG`.

## Pendiente / recomendado

- **Barrer el resto de sitios** del stack (`/var/www/wordpress-stack/sites/*/wp-content/plugins/wom-iCat-Favorites/wom-iCat-Favorites.php`) y aplicar el mismo fix donde falte.
- Decisión arquitectónica pendiente: **versionar los plugins custom** o mantenerlos fuera de git (si fuera, documentar el canal oficial de distribución).