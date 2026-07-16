# Invalidación de caches tras sync del importador

Un sitio Xana típico tiene **4 capas de caching** delante del código PHP. Cada una se invalida de forma distinta al final de `syncPimDatabase()` (en `wom-iCat-admin/functions/class-serializer.php`). Todas están integradas como `profile_step` al final de la sync.

## Tabla de capas

| Capa | Qué cachea | Invalidación | Paso del importer |
|---|---|---|---|
| **Varnish** | HTML completo (fuera de PHP) | Señal HTTP externa (BAN por host) | `deleteVarnishCache` → `xana_purge_varnish()` |
| **Redis Object Cache** | Resultados de queries WP (`posts`, `post_meta`, `terms`, etc.) | Automática vía hooks WP (`clean_post_cache`, `wp_cache_delete`…) al usar `wp_insert_post/update_post/delete_post` | Ningún paso explícito — ocurre en cada operación |
| **Transients Xana** (filtros, listados) | Resultados de queries custom del tema/plugins (prefijo `xanaTransient_*`) | SELECT + `delete_transient()` de todos los `xanaTransient_*` | `deleteTransients` |
| **WP Rocket** (cuando está activo) | HTML con optimizaciones (CSS/JS) | `rocket_clean_domain()` | `deleteWPRocketCache` |

Orden en el `profile_step` al final de `syncPimDatabase()`:
```php
$this->profile_step( 'dropTemporaryTables',   function() { $this->dropTemporaryTables(); } );
$this->profile_step( 'deleteTransients',      function() { $this->deleteTransients(); } );
$this->profile_step( 'deleteWPRocketCache',   function() { $this->deleteWPRocketCache(); } );
$this->profile_step( 'deleteVarnishCache',    function() { $this->deleteVarnishCache(); } );
```

## Detalle por capa

### Varnish

- Introducido 2026-04-23 (commit en `feature/PurgaVarnishTrasSync` de `plugin-importacion`).
- El VCL del stack (`varnish/default.vcl`) acepta método HTTP `BAN` con header `X-Ban-Host` desde la ACL interna (redes RFC1918) y ejecuta `ban("req.http.host == " + host)`.
- `xana_purge_varnish($host = null)` (definida en `wom-iCat-admin.php`) envía la petición. Sin argumentos usa `home_url()` del blog actual → en multisite con `switch_to_blog()` (caso Xana multiidioma) se purga cada blog en su iteración correspondiente.
- Endpoint configurable vía constante `XANA_VARNISH_ENDPOINT` en `wp-config.php`. Por defecto `http://varnish` (servicio Docker).
- Falla silencioso si Varnish no responde (sitios dev sin cache): la sync sigue.

### Redis Object Cache

- No necesita intervención explícita del importador.
- El plugin Redis Object Cache (drop-in `object-cache.php`) intercepta los hooks estándar de WordPress:
  - `clean_post_cache($post_id)` → borra `posts:$id`, `post_meta:$id`, relaciones.
  - `wp_cache_delete($key, $group)` → borra la key concreta.
- El importador toca los posts vía funciones nativas de WP (`wp_insert_post`, `wp_update_post`, `wp_delete_post`, `update_post_meta`, `wp_set_object_terms`…), que disparan esos hooks.
- **Riesgo a vigilar:** si un dev escribe SQL directo sobre `wp_posts`/`wp_postmeta` (no vía WP API), Redis queda desincronizado. El importer actualmente usa las APIs, así que no hay problema documentado.
- Si alguna vez se ve data stale tras sync y Varnish ya purgado, el recurso nuclear es `wp_cache_flush()`. No añadido por defecto — flush total es costoso y la invalidación selectiva ya cubre los casos normales.

### Transients Xana (filtros y listados)

**Convención del equipo (2026-04-23):** todos los transients custom de sitios Xana usan el prefijo `xanaTransient_`.

Implementación en `wom_iCat/`:
- Legacy: `set_xana_transient($name, $value, $expiration = 604800)` en `includes/functionsTransients.php`.
- Moderno: `\Xana\Transient\TransientService::set($name, $value, $expiration)` en `src/Transient/TransientService.php`. Constante `TRANSIENT_PREFIX = 'xanaTransient_'`.

Ambos aplican el prefijo internamente, así que desde el caller se pasa el nombre "limpio" (sin prefijo). En BD aparece como `_transient_xanaTransient_<nombre>`.

Flag global `xana_transients_enabled` (option de WP): si es `'on'` los transients funcionan, si no los getters devuelven `false` (bypass de cache). Útil para debugging.

El método `deleteTransients()` de `syncPimDatabase()` hace:
```sql
SELECT option_name FROM wp_options
WHERE option_name LIKE '_transient_xanaTransient%'
   OR option_name LIKE '_transient_timeout_xanaTransient%'
```
y llama `delete_transient()` sobre cada nombre.

**Regla para dev:** **nunca** llamar `set_transient()` directo en código Xana. Usar siempre los helpers. Si se usa `set_transient()` directo sin prefijo `xanaTransient_`, ese transient **no se limpia en sync** y puede servir datos stale.

`TransientService` además expone un `deleteAll()` propio (mismo efecto que el método del importer) por si hay que purgar transients fuera del flujo de sync.

### WP Rocket

- Solo se ejecuta si `rocket_clean_domain()` existe (plugin activo).
- No todos los sitios Xana usan WP Rocket. Cuando se usa, coexiste con Varnish (Rocket genera HTML estático, Varnish lo sirve más rápido todavía).

## Multiidioma y purga

`syncPimDatabase()` se llama una vez por cada blog del multisite vía `switch_to_blog()` (línea 1347 de `class-serializer.php`). Por tanto los 4 `profile_step` de invalidación se ejecutan **N veces** (una por idioma).

Para Varnish con **subpath mode** (`cliente.com/en/`, `cliente.com/fr/`): todos los blogs comparten host → el BAN es el mismo repetido N veces (redundante pero correcto). Varnish lo consolida en la ban list.

Para Varnish con **subdominio o dominios separados** (`en.cliente.com`, `cliente.fr`): cada iteración genera un BAN distinto → cada dominio se purga en su vuelta.

Micro-optimización no aplicada: `xana_purge_varnish()` podría llevar una variable `static` con hosts ya purgados en la misma request para evitar repetición. Decisión consciente de no añadirla hasta que haya un caso que lo justifique.

## Bypass selectivo por respuesta — header `X-Xana-No-Cache`

Hay páginas que **pertenecen a un visitante concreto** y no se pueden cachear: wishlist, carrito, zona de cuenta, páginas que leen `$_COOKIE` para renderizar HTML personalizado.

La tentación es añadir reglas hardcodeadas al VCL (`if (req.url ~ "^/wishlist")`). Eso **NO funciona** para la plataforma Xana porque cada cliente puede llamar a la misma página de forma distinta: `/wishlist/`, `/favoritos/`, `/request-a-quote/`, `/en/my-wishlist/`, `/fr/favoris/`… Cada `wom_icat_fav_page_settings[favorites_page]` es un ID de post distinto por blog.

### Patrón general (funciona para cualquier plugin Xana)

**1. El backend (plugin/tema) envía el header cuando sabe que la página no es cacheable:**

```php
add_action('template_redirect', 'mi_plugin_maybe_no_cache');
function mi_plugin_maybe_no_cache() {
    if (headers_sent()) return;

    // Detectar que estamos renderizando la página personalizada.
    // Ejemplo: leer el ID de página de la config del plugin.
    $settings = get_option('mi_plugin_settings');
    $page_id  = ! empty($settings['page']) ? intval($settings['page']) : 0;

    if ($page_id && is_page($page_id)) {
        header('X-Xana-No-Cache: 1');
    }
}
```

**2. Varnish (`varnish/default.vcl`) respeta el header:**

```vcl
sub vcl_backend_response {
    # Backend puede pedir "no cachear esta respuesta" con X-Xana-No-Cache.
    # Usado por páginas con contenido específico por visitante (wishlist,
    # carrito, etc.). La URL varía por cliente/idioma, por eso el bypass
    # lo dispara el backend y no una regla hardcoded en el VCL.
    if (beresp.http.X-Xana-No-Cache) {
        set beresp.ttl = 0s;
        set beresp.uncacheable = true;
        return (deliver);
    }
    ...
}

sub vcl_deliver {
    # Limpiar header interno antes de enviar al cliente.
    unset resp.http.X-Xana-No-Cache;
    ...
}
```

### Por qué este patrón

- **Cero duplicación de configuración:** el plugin ya sabe qué página es la personalizada (lo guarda en sus settings); el VCL se limita a respetar la señal.
- **Resistente a cambios:** si un cliente cambia el slug de su página de wishlist, nada se rompe.
- **Multiidioma nativo:** cada blog del multisite tiene su propia `favorites_page`; el hook se ejecuta en el contexto del blog correcto tras `switch_to_blog()` o resolución normal por URL.
- **Reutilizable:** el mismo header sirve para cualquier otro plugin Xana que tenga páginas personalizadas (carrito custom, zona de cliente, presupuestos, etc.). No hace falta tocar el VCL otra vez.
- **No se filtra al cliente:** `vcl_deliver` borra el header antes de enviar la respuesta al navegador. Detalle interno del stack.

### Caso aplicado (2026-04-23)

Plugin `wom-iCat-Favorites`:
- Hook `template_redirect` → `\wom\icat\favorites\maybe_send_no_cache_header()`.
- Lee `wom_icat_fav_page_settings['favorites_page']` y manda header si `is_page($id)`.

Cuando quieras hacer lo mismo en otro plugin Xana: **no toques el VCL**, solo añade el hook con la lógica de detección de tu página.

### Opción alternativa (no recomendada)

Hardcodear múltiples patrones de URL en VCL (`req.url ~ "^/(wishlist|favoritos|cart|checkout)"`). Frágil, no cubre multiidioma automáticamente, y cada sitio nuevo puede requerir cambiar el stack. Usar solo como workaround temporal si por algún motivo el backend no puede enviar el header.

## Para añadir una nueva capa de cache

Si en el futuro se añade otra capa (ej. Cloudflare, fastly, LiteSpeed Cache, etc.):

1. Añadir función global en `wom-iCat-admin.php` estilo `xana_purge_<capa>($host = null)` con logging vía `error_log()` y fallback silencioso.
2. Añadir método privado `delete<Capa>Cache()` en `class-serializer.php` que llame a la función global.
3. Añadir `profile_step('delete<Capa>Cache', ...)` al final de `syncPimDatabase()` en el mismo bloque que los anteriores.

Formato consistente: todos los pasos de purga agrupados al final, justo antes del `profile_finish()` y la reprogramación del cron de miniaturas.