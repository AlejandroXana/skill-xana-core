# 2026-10-15 · Keros: el selector de idioma llevaba a una búsqueda

## Síntoma

En la portada de keros.com (`/` y `/en/`) los enlaces ES/EN del menú apuntaban a
`/search/FONTANA/` y `/en/search/FONTANA/`; en `/en/`, a una búsqueda de spam en tailandés.
El resto de páginas cambiaban de idioma bien. Los `hreflang` del `<head>` eran correctos.

## Causa

MultilingualPress 3.9.1 cachea en el object cache el resultado de `NavMenu\ItemFilter::filterItems()`,
que es lo que rellena las URLs de los elementos de menú de tipo `mlp_language`.
La clave solo usa el **path** de la URL e ignora la query string
(`src/multilingualpress/NavMenu/ServiceProvider.php`, `integrateCache()`):

```php
$url = parse_url(add_query_arg([]), PHP_URL_PATH);
$key .= substr(md5(serialize(array_keys($postArrays)) . $url), -12, 10);
```

`/?s=FONTANA` y `/` tienen el mismo path, así que la primera búsqueda que entra en la portada
(normalmente de un bot) guarda un menú con enlaces a esa búsqueda y se sirve a todo el mundo.
Sin object cache persistente esa caché muere al terminar la petición y no se nota.
**Aparece al migrar a Docker**, que trae Redis.

Encima, Varnish cachea la portada contaminada (TTL 12 h).

## A quién afecta

Solo a los sites cuyo selector de idioma son **elementos de menú `mlp_language`**
(Apariencia → Menús → «Idiomas»). El selector propio de Xana Core (`selector__idioma`,
shortcode `multisite_language_selector_widget` en `wom_iCat/includes/functionsMLP.php`) usa
`Translations::searchTranslations()`, cuya caché (`api.translation`) mete en la clave todos los
argumentos (tipo de contexto y término buscado), así que no choca.

Barrido del 15-oct-2026 en todos los catálogos con Redis: **solo Keros** tenía elementos
`mlp_language` (Smalticeram, Palorosa, Aphelion, Codicer, Mayolica, Newker, Qualis, Adex y Samboro: 0).

## Arreglo

1. Desactivar la caché del menú de MLP (network option; equivale a desmarcarla en
   Escritorio de red → MultilingualPress → Caché):
   ```bash
   wp eval 'update_network_option(0, "multilingualpress_internal_cache_setting", [
     "api" => ["api.translation"=>true,"api.content_ids"=>true,"api.content_relations"=>true,
               "api.has_site_relations"=>true,"api.all_relations"=>true,"api.related_site_ids"=>true],
     "database" => ["database.table_list"=>true],
     "nav_menu" => ["nav_menu.item_filter"=>false],
   ]);'
   ```
   Hay que guardar el array completo: si la opción no existe, MLP usa los valores por defecto
   (todo a `true`).
2. Purgar las portadas en Varnish:
   ```bash
   docker exec varnish varnishadm ban 'req.http.host == "<dominio>" && req.url ~ "^/(en/)?$"'
   ```
3. Borrar las claves huérfanas de Redis (en Keros eran 17.323, una por término buscado):
   ```bash
   redis-cli --scan --pattern "<salt>*mlp_*ItemFilter:filter_items*" | tr "\n" "\0" | xargs -0 -r -n 500 redis-cli UNLINK
   ```
   ⚠️ Sin `tr`/`xargs -0`, `xargs` se come las `\` de `Inpsyde\MultilingualPress\...` y
   `UNLINK` devuelve 0 sin borrar nada.

## Comprobación

```bash
docker exec nginx curl -s -H "Host: <dominio>" -H "X-Forwarded-Proto: https" "http://127.0.0.1/?s=PRUEBA" >/dev/null
docker exec nginx curl -s -H "Host: <dominio>" -H "X-Forwarded-Proto: https" http://127.0.0.1/ \
  | grep -oE 'mlp-language-nav-item[^>]*><a[^>]*href="[^"]*"' | grep -o 'href="[^"]*"'
```
Los enlaces no pueden contener `/search/`.

## Prevención

- Checklist de migración a Docker: ver «Selector de idioma y caché de MultilingualPress» en
  `arquitectura/multisite-multiidioma.md`.
- Check automático en el script de la skill `diagnostico-servidor`.
