# 2026-07-16 — alfagres.com lento: wp_options reventada por transients + cron atascado

## Síntoma

alfagres.com iba muy lento (load **5.53 en 2 vCPU**, los 5 workers php-fpm `max_children=5` en estado R, pool a tope). MySQL en cambio idle → cuello de botella **PHP/CPU**, no BD. Además el usuario reportaba que **no podía borrar +100k transients** (ni desde el panel del plugin ni de ninguna forma).

## Diagnóstico

Tabla `WrQa3_options` (multisite, prefijo `WrQa3_`, sitio `/var/www/html/nueva_alfagres`) inflada a **970 MB / 318.549 filas**, casi todo transients:

- **`xanaTransient_filtros_*` (82k) + `resultados` (21k)** — cache del tema, TTL default **7 días** (`set_xana_transient`, 604800). El borrado *lazy* on-access SÍ los recicla porque el nombre (combinación de filtros) **se repite** con el tráfico.
- **`external_ip_address_*` (81k)** — los crea **WooCommerce** (`WC_Geolocation::get_external_ip_address`, `class-wc-geolocation.php:108/134`) con `woocommerce_default_customer_address = geolocation`. Un transient por IP visitante (TTL 1 día). Nombre **único por IP** → el borrado lazy **NUNCA** los alcanza (nadie repite el mismo). Muchos con **payloads Log4Shell** (`${jndi:ldap://...}`) inyectados en cabeceras `X-Forwarded-For` (WooCommerce mete el valor de la cabecera en el nombre del transient sin validar que sea IP). Log4Shell no afecta a PHP, pero es un **vector de inflado ilimitado** de la tabla.
- **~113k transients ya expirados** sin limpiar.

### Por qué NADA los borraba (los 3 frenos rotos a la vez)

1. **Panel del plugin (`TransientService::deleteAll`) / `wom-iCat-admin`:** hacen `SELECT` masivo + `delete_transient()` **uno a uno en la request web** → timeout de PHP-FPM. Además el importador de alfagres (HEAD `20d5e44`, feb-2024) **no tiene** el paso `deleteTransients` (es mejora de abril-2026 no desplegada aquí), y aunque lo tuviera solo cubre `xanaTransient_`, no los `external_ip`.
2. **Cron `delete_expired_transients`:** atascado en "now" **5 meses**. El WP-Cron por-visitas (`DISABLE_WP_CRON` no definido) hereda el límite HTTP y el self-join sobre 970 MB moría antes de completar → nunca se reprogramaba.
3. **Borrado lazy:** solo actúa sobre nombres que se re-piden → inútil para `external_ip_address_*`.

Círculo vicioso: hay tantos transients que el cron que debería borrarlos no puede con ellos.

## Fix aplicado

1. Limpieza de `xanaTransient_*` por lotes SQL (`DELETE ... LIKE '_transient_xanaTransient_%' LIMIT 5000` en bucle) con backup previo (`mysqldump --where`).
2. **Configurar WP-Cron real:**
   ```bash
   wp config set DISABLE_WP_CRON true --raw --type=constant
   # crontab de woman (posee ficheros + pool, sin sudo):
   */5 * * * * /usr/local/bin/wp cron event run --due-now --path=/var/www/html/nueva_alfagres --quiet >/dev/null 2>&1
   ```
3. Forzar `wp cron event run delete_expired_transients` a mano: **completó en ~9s** (por WP-CLI **no hay `max_execution_time`**) y borró ~160k. La tabla pasó a **4.195 filas**, y el evento se **reprogramó solo** (ya no se atasca).

**Resultado: load 5.53 → 0.52.**

## Lecciones

- **`OPTIMIZE TABLE` pendiente:** InnoDB NO libera los 970 MB en disco al borrar filas (queda hueco reutilizable). La tabla sigue pesando 970 MB con 4k filas hasta reconstruirla. No se lanzó (decisión del usuario).
- **Orden correcto:** limpiar la tabla ANTES de confiar en el cron; el cron solo funciona porque la query ya cabe en su tiempo.
- **WP-CLI > WP-Cron por-visitas** para tareas pesadas (sin límite HTTP).
- **Caducar ≠ borrarse** en WP: un transient expirado permanece hasta que (a) se re-pide ese nombre exacto, o (b) corre `delete_expired_transients`.
- **Cura de raíz pendiente:** desactivar geolocation de WooCommerce (corta los `external_ip_address_*`), bajar TTL de filtros Xana, y sanear constantes `XANA_THEME_TEXT_DOMAIN`/`WOM_ICAT_OUTLET_CONF_REGISTRO` (warnings en CLI php7.4; serían fatales en php8 y romperían el cron).

## Warnings vistos en CLI (relacionado con incidencia 2026-04-17)

`Use of undefined constant XANA_THEME_TEXT_DOMAIN` (`wom-iCat-Outlet/functions-xana.php:5`) y `WOM_ICAT_OUTLET_CONF_REGISTRO` (`wom_iCat/wom-iCat.php:28`). Misma familia que `2026-04-17-fatal-xana-text-domain.md`. En php7.4 no rompen; en php8 sí.
