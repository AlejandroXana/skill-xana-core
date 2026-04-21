# wp-cli y OPcache en el stack Xana Core

Dos gotchas recurrentes al operar sobre sitios Xana Core.

## 1. wp-cli con fatal en init

Si hay algún fatal que se dispara durante `do_action('init')` (desfase WP/WooCommerce, plugin con namespace mal resuelto, constante indefinida, etc.), wp-cli no puede arrancar porque carga el entorno completo de WordPress.

**Fix**: usar `--skip-plugins --skip-themes` para evitar que se carguen. El contenedor `php-fpm` corre como root, así que además `--allow-root`. Patrón:

```bash
docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && wp --allow-root --skip-plugins --skip-themes <comando>'
```

Con esto se puede ejecutar `core update`, `db export`, `config get/set`, etc. incluso con el sitio tirado.

## 2. OPcache de FPM no se purga desde CLI

`opcache_reset()` ejecutado por CLI **NO limpia** el OPcache del proceso FPM. Son procesos distintos con pools de memoria separados. Tras editar un archivo PHP en producción, las peticiones HTTP siguen sirviendo la versión antigua cacheada por FPM hasta que:

- Pasa el `opcache.revalidate_freq` (típicamente 120s en este stack).
- Se reinicia FPM.

**Fix**: reload graceful de workers de FPM con señal `USR2` al master:

```bash
docker exec php-fpm kill -USR2 1
```

Espera ~2 segundos antes de testear. Si `USR2` no funciona en ese stack concreto, `docker restart php-fpm` como último recurso (corta conexiones abiertas).

## 3. Reproducir el stack real de un fatal que devuelve 500 con body vacío

Si el sitio tiene `WP_DEBUG_DISPLAY=false` el fatal no aparece en HTML, solo un 500 sin body. Para ver el error real:

1. Activar temporalmente debug:
   ```bash
   docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && wp --allow-root --skip-plugins --skip-themes config set WP_DEBUG true --raw --type=constant'
   docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && wp --allow-root --skip-plugins --skip-themes config set WP_DEBUG_DISPLAY true --raw --type=constant'
   docker exec php-fpm kill -USR2 1
   ```
2. Hacer la petición (`curl` o navegador) y capturar el mensaje.
3. Volver a desactivar:
   ```bash
   docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && wp --allow-root --skip-plugins --skip-themes config set WP_DEBUG false --raw --type=constant'
   docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && wp --allow-root --skip-plugins --skip-themes config set WP_DEBUG_DISPLAY false --raw --type=constant'
   docker exec php-fpm kill -USR2 1
   ```

**Ojo**: si `wp-config.php` tiene `@ini_set('display_errors', 1)` (línea típica ~87), los errores se muestran aunque `WP_DEBUG_DISPLAY` sea false. Cambiar ese `1` a `0`.

## Por qué importa

Sin este patrón:
- Se intenta `wp core update` y falla porque un plugin lanza fatal en init → bucle de "no puedo actualizar porque algo falla, no puedo arreglar porque no puedo actualizar".
- Se edita un archivo, se confirma que está bien, se prueba la URL y sigue fallando → se cambia más código sin necesidad porque el cambio real ya estaba bien pero FPM cacheaba la versión antigua.