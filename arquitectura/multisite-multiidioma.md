# Multisite por idioma

## Resumen

**Los sitios del catálogo Xana Core usan WordPress Multisite como estrategia de multiidioma**: cada idioma es un blog independiente dentro de la misma instancia multisite, no una traducción dentro de un único sitio.

Cada blog del multisite representa **un idioma**: `/es/`, `/en/`, `/fr/`, etc. El prefijo de tablas base (ej. `lcblvb_`) es compartido, pero cada blog tiene su set: `lcblvb_posts`, `lcblvb_2_posts`, `lcblvb_3_posts`…

## Rationale

Decisión tomada hace tiempo (precede al uso actual de plugins como WPML/Polylang). En aquel momento era la mejor forma de soportar multiidioma con:
- Separación limpia de contenido por idioma.
- Permalinks nativos sin reescrituras.
- Aislamiento de opciones/menús/widgets por idioma.
- Importadores PIM por idioma (las tablas `wp_icat_*` almacenan datos con el campo `idioma`).

No se migra a plugins de traducción estándar porque el flujo de importación, el theme (`uncode-child`) y varios plugins custom (`wom-iCat-*`, `multilingualpress`) están acoplados a este modelo.

## Integración con MultilingualPress

El plugin `multilingualpress` es el que conecta los blogs entre sí (relaciones de traducción entre posts equivalentes en distintos idiomas). La función `check_current_language_importer()` del plugin wom-iCat-admin detecta el idioma vía `multilingualpress_site_settings` o por `get_locale()` si no es multisite.

## Implicaciones operativas

### Backups y migración
- Dump de BD **siempre completo** (no por blog): el multisite comparte tabla `lcblvb_users`, `lcblvb_usermeta`, `lcblvb_blogs`, `lcblvb_site`, `lcblvb_sitemeta`.
- `search-replace` de URLs debe aplicarse con `wp search-replace --network --all-tables` (WP-CLI) o similar para tocar todas las tablas por-blog.
- Casos que requieren multi-dominio: cada idioma puede estar en un dominio distinto (`cliente.com` / `cliente.es` / `cliente.fr`) o en subpath (`/en/`, `/fr/`). Esto se configura en `lcblvb_blogs` y opcionalmente en plugins de domain mapping.

### Desarrollo local
- Al descargar un sitio al stack local, mantener `define('MULTISITE', true)` y demás constantes de multisite en `wp-config.php`.
- Hacer `search-replace` considerando todos los dominios por idioma presentes en `lcblvb_blogs`, no solo el principal.
- Si en producción usan subdominios (`en.cliente.com`), se pueden emular en local con `en.cliente.local` + entradas en `/etc/hosts` + routing en Traefik.

### Taxonomías compartidas vs por-blog
- Las taxonomías de WooCommerce (categorías, etiquetas) son **por blog** (un set por idioma).
- Los IDs de PIM (`idPim`) son globales, pero se replican como post WP distinto en cada blog.
- El plugin importador traduce `idPim` → `post_id` usando `get_translated_id()` o consultas contra `wp_icat_*`.

### Prefijo de tablas
- Las tablas WP respetan el prefijo del sitio (ej. `lcblvb_`): siempre usar `{$wpdb->posts}`, `{$wpdb->postmeta}`, nunca `wp_posts` hardcodeado.
- Las tablas PIM (`wp_icat_*`) sí tienen prefijo fijo `wp_icat_` (son compartidas entre todos los blogs del multisite).

## Casos conocidos

Sitios demolive/producción documentados en incidencias: `demolive.xanasystem.com`, `demolive.digitaltilecatalog.com`, `younginteriorsflooring.com`, `test11.digitaltilecatalog.com`.