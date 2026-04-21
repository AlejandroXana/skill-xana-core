# Update WP 5.9.12 → 6.9.4 en demolive.xanasystem.com (desfase con WooCommerce 10)

**Fecha**: 2026-04-21
**Entorno afectado**: demolive.xanasystem.com (multisite con `/` y `/en/`)
**Versiones de partida**:
- WordPress core: **5.9.12** (enero 2022, obsoleto)
- WooCommerce: **10.6.1** (muy reciente)
- PHP: 8.3.30

## Síntoma

Tras resolver la incidencia de `XANA_THEME_TEXT_DOMAIN` (ver `2026-04-17-fatal-xana-text-domain.md`), aparece nuevo fatal:

```
Fatal error: Uncaught Error: Call to undefined function
  Automattic\WooCommerce\Blocks\wp_register_script_module()
  in plugins/woocommerce/src/Blocks/AssetsController.php:88
```

## Causa raíz

La función `wp_register_script_module()` se introdujo en **WordPress core 6.5** (abril 2024). WooCommerce 10 la usa. Este sitio tenía WP 5.9.12, donde la función no existe → fatal al cargar WooCommerce.

Causa más profunda: alguien actualizó WooCommerce sin actualizar WordPress core. WP 5.9 ni siquiera soporta oficialmente PHP 8.x (soporte desde WP 6.0).

## Fix

Actualizar WP core de 5.9.12 a 6.9.4 (última) mediante wp-cli.

### Pasos ejecutados

1. **Backup de BD** (con contenedor `mysql` porque `mysqldump` no está en `php-fpm`):
   ```bash
   DB_PASS=$(docker exec php-fpm sh -c 'cd /var/www/html/demolive-xanasystem-com && wp --allow-root --skip-plugins --skip-themes config get DB_PASSWORD' | tail -1)
   docker exec -e MYSQL_PWD="$DB_PASS" mysql sh -c \
     'mysqldump --single-transaction --quick --routines --triggers --no-tablespaces -u woman core_demo_live_xana' \
     > /var/www/wordpress-stack/backups/demolive-xanasystem-com/backup-$(date +%F-%H%M%S).sql
   ```
   Resultado: dump íntegro de 49 MB con 179 tablas.

2. **Update de core** con `--skip-plugins --skip-themes` (imprescindible porque el fatal de Woo bloquea `init`):
   ```bash
   docker exec php-fpm sh -c 'cd /var/www/html/demolive-xanasystem-com && wp --allow-root --skip-plugins --skip-themes core update'
   ```

3. **Update de BD del multisite**:
   ```bash
   docker exec php-fpm sh -c 'cd /var/www/html/demolive-xanasystem-com && wp --allow-root --skip-plugins --skip-themes core update-db --network'
   ```
   Actualiza db_version 51917 → 60717 en los 2 sitios (`/` y `/en/`).

4. **Reload FPM** para que OPcache recargue:
   ```bash
   docker exec php-fpm kill -USR2 1
   ```

### Efectos secundarios detectados durante el update

- **Debug display activo** (`WP_DEBUG_DISPLAY=true`) exponía los deprecated de PHP 8 al HTML. Desactivado:
  ```bash
  wp config set WP_DEBUG false --raw --type=constant
  wp config set WP_DEBUG_DISPLAY false --raw --type=constant
  wp config set WP_DEBUG_LOG false --raw --type=constant
  ```
- **`wp-config.php:87` tenía `@ini_set('display_errors', 1)`** que forzaba mostrar errores aun con WP_DEBUG_DISPLAY=false. Cambiado a 0.
- Tras quitar debug, afloró un **fatal latente en `/wp-login.php`** por callbacks huérfanos del plugin outlet — ver incidencia `2026-04-21-fatal-callbacks-outlet.md`.

## Resultado

- WP core en 6.9.4, sitio sin fatal de Woo.
- Home `/` → 302 a `/my-account2/` (redirect configurado del sitio, normal).
- `/wp-login.php` → 200 tras resolver la incidencia relacionada de callbacks huérfanos.
- `/en/wp-login.php` → 200.

## Pendiente / recomendado

Otros sitios del stack en WP 5.9 con WooCommerce moderno tendrán el mismo fatal. Barrido rápido:

```bash
for d in /var/www/wordpress-stack/sites/*/wp-content; do
  DOM=$(basename $(dirname $d))
  VER=$(grep wp_version "$d/../wp-includes/version.php" 2>/dev/null | head -1)
  echo "$DOM  $VER"
done
```

Cualquiera con `$wp_version = '5.x'` es candidato a este mismo bug en cuanto se active WooCommerce.

## Lección

- Nunca actualizar plugins críticos (WooCommerce, WordPress sec) sin verificar compatibilidad con el core.
- Un fatal enmascarado por deprecated/display_errors puede aparecer con un 500 limpio al limpiar el debug; es buena señal (significa que antes había algo tapado).
- El `opcache_reset()` por wp-cli **no** limpia OPcache de FPM — usar `kill -USR2 1` (ver runbook `wp-cli-y-opcache.md`).