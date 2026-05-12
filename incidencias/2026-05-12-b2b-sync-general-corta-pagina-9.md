# Incidencia 2026-05-12 — b2b-sync se corta a los 450 productos en general

**Sitio**: clientportal.generalceramic.com (B2B General Ceramic, no es Xana Core estándar; instalación WP no dockerizada en `/var/www/html/general/`).

**Plugin afectado**: `wp-content/plugins/b2b-sync/` (repo `bitbucket.org:xana-technologies/sync-products-xana-b2b`). Sincroniza productos contra `api.xanasystem.com/v1/products`.

## Síntoma

La sincronización procesa 9 páginas (50 productos/página = 450 productos) y se detiene. Estado en BD tras el corte:

```
xana_b2b_sync_status = running     <- queda atascado
xana_b2b_sync_page = 10            <- update_option SÍ se ejecutó
xana_b2b_sync_updated_products = 450
xana_b2b_sync_max_products = 636
```

No hay `PHP Fatal` ni timeout en `debug.log` ni en `sync.log`. El último log es:
```
Fin de bloque: 450 de 636 productos sincronizados
```

Y luego silencio total hasta el día siguiente.

## Causa raíz

El plugin encadena batches con:
```php
update_option('xana_b2b_sync_page', $pageNumber+1);
wp_schedule_single_event(time(), 'xana_b2b_handle_batch_event');
```

El sitio depende **exclusivamente de wp-cron por loopback HTTP** (sin `DISABLE_WP_CRON`, sin crontab del usuario `woman`). En horario nocturno (3 AM UTC), el único motor que genera loopback requests es **Action Scheduler** procesando su cola (`action_scheduler_run_queue` cada minuto).

Cada página procesada por b2b-sync genera ~50 tareas `woocommerce_run_product_attribute_lookup_update_callback` (una por producto). Mientras Action Scheduler tenga cola, hace requests cada minuto y de paso wp-cron se dispara, ejecutando el siguiente `xana_b2b_handle_batch_event`.

**El día del incidente**, tras la página 9 (03:18:44), Action Scheduler vació su cola a las 03:20:07 y dejó de hacer requests. Como no había tráfico humano hasta las ~04:13, el evento `xana_b2b_handle_batch_event` programado para la página 10 quedó huérfano. Cuando wp-cron volvió a despertar, el evento ya no existía en la lista (ejecutado en un loopback abortado o desprogramado por alguna limpieza).

**Por qué a veces sí completa**: si la cantidad de productos cambiados es mayor, Action Scheduler tiene cola para más tiempo y mantiene wp-cron activo hasta el final. Es una carrera entre "termina la cola de WC" y "termina la sincronización".

## Cómo confirmar el patrón

```bash
# Estado actual de las opciones
wp --allow-root --skip-plugins --skip-themes option get xana_b2b_sync_status
wp --allow-root --skip-plugins --skip-themes option get xana_b2b_sync_page
wp --allow-root --skip-plugins --skip-themes cron event list | grep xana

# Última línea del log (debe ser "Fin de bloque: X de Y" sin nada después)
tail -1 wp-content/plugins/b2b-sync/sync.log

# Confirmar que Action Scheduler dejó de tener cola justo después del corte
wp --allow-root --skip-plugins --skip-themes db query \
  'SELECT last_attempt_gmt, hook FROM bif2ew_actionscheduler_actions \
   WHERE last_attempt_gmt >= "2026-05-12 03:00:00" ORDER BY last_attempt_gmt DESC LIMIT 20'
```

## Mitigaciones

### Quick fix manual para reanudar (estado actual)
El status es `running`, así que el botón "Sync Products" del admin devolverá *"Ya hay una sincronización en curso"*. Hay que:
1. Pulsar **Clean Sync Args** (resetea status a `completed` y page a 1).
2. Pulsar **Sync Products** — empezará desde cero (no continúa desde página 10).

Alternativamente, reanudar desde la página 10 sin perder progreso:
```bash
wp --allow-root --skip-plugins --skip-themes eval \
  "wp_schedule_single_event(time(), 'xana_b2b_handle_batch_event');"
wp --allow-root --skip-plugins --skip-themes cron event run xana_b2b_handle_batch_event
```

### Fix definitivo (recomendado)
Cron del sistema cada 1–5 min y desactivar el wp-cron HTTP:

`wp-config.php`:
```php
define('DISABLE_WP_CRON', true);
```

`crontab -e` del usuario woman:
```
*/5 * * * * cd /var/www/html/general && /usr/local/bin/wp --allow-root --skip-plugins --skip-themes cron event run --due-now > /dev/null 2>&1
```

### Fix robusto (cambio en el plugin)
Migrar el encadenamiento de batches a Action Scheduler:
```php
as_enqueue_async_action('xana_b2b_handle_batch_event', [], 'xana-b2b-sync');
```
Action Scheduler tiene su propio sistema de ejecución, locks y reintentos — no depende de loopback HTTP.

Además, añadir en el cron diario una comprobación: si `xana_b2b_sync_status === 'running'` desde hace > N minutos sin progreso, reprogramar el batch siguiente en lugar de no hacer nada.

## Cómo aplicarlo

Cuando un sitio Xana / B2B con sincronizaciones por batch se quede atascado a mitad:
1. Comprobar `xana_b2b_sync_page` vs `xana_b2b_sync_max_products / page_size` — el corte siempre cae en el límite de una página.
2. Confirmar que `wp cron event list` no muestra el evento single pendiente.
3. Confirmar en `bif2ew_actionscheduler_actions` que la actividad del cron se cortó cuando Action Scheduler vació su cola.
4. Si el patrón coincide, la causa es wp-cron HTTP sin disparador externo: aplicar fix de crontab del sistema.
