# Fatal PHP 8: `XANA_THEME_TEXT_DOMAIN` indefinido en namespace

**Fecha**: 2026-04-17
**Entorno afectado**: Todos los sitios Xana Core con PHP 8.x y plugin `wom-iCat-Outlet`
**Commit del fix**: `fe63e96d9` — "Feature: Correcciones php8 args"

## Síntoma

```
Fatal error: Uncaught Error: Undefined constant "xana\outlet\theme\XANA_THEME_TEXT_DOMAIN"
  in wp-content/plugins/wom-iCat-Outlet/functions-xana.php:5
```

Sitio caído desde la línea 5 del plugin, antes incluso de cargar WordPress entero.

## Causa raíz

En `plugins/wom-iCat-Outlet/functions-xana.php`, línea 5, había:

```php
namespace xana\outlet\theme;

if (!XANA_THEME_TEXT_DOMAIN) {   // ← bug
    define('XANA_THEME_TEXT_DOMAIN', 'xana-webcatalog');
}
```

Dos problemas combinados:

1. **PHP 8 cambió el comportamiento**: en PHP 7, `if (!UNDEFINED_CONSTANT)` emitía warning y trataba la constante como string. En PHP 8 es **Error fatal**.
2. **Namespace**: al estar dentro de `namespace xana\outlet\theme`, PHP resuelve `XANA_THEME_TEXT_DOMAIN` como `\xana\outlet\theme\XANA_THEME_TEXT_DOMAIN` (no la constante global). Aunque se hubiera definido a nivel global, este check sin `defined()` no la encontraría.

## Fix

Patrón canónico WordPress:

```php
if (!defined('XANA_THEME_TEXT_DOMAIN')) {
    define('XANA_THEME_TEXT_DOMAIN', 'xana-webcatalog');
}
```

`defined()` recibe un string, no resuelve namespace, y devuelve bool — es el check correcto.

## Verificación

```bash
# En cada sitio desplegado con Xana Core
head -10 /var/www/wordpress-stack/sites/<dominio>/wp-content/plugins/wom-iCat-Outlet/functions-xana.php
# Línea 5 debe decir: if (!defined('XANA_THEME_TEXT_DOMAIN')) {
```

Si no es así, ese sitio **no tiene el fix pulled** aunque exista en el repo.

## Propagación

El fix se aplicó en el repo `wp-content-xana`, pero como no hay CI/CD, **cada servidor lo recibe sólo cuando alguien hace `git pull` en él**. Esto provocó que el mismo fatal reapareciera en otros sitios (p.ej. demolive.xanasystem.com, ver incidencia del 2026-04-21) durante semanas porque nadie había tirado pull.

**Checklist al detectar el fatal**:
1. `git log --oneline plugins/wom-iCat-Outlet/functions-xana.php` en el sitio afectado.
2. Si no aparece el commit `fe63e96d9`, hacer `git pull --ff-only origin master`.
3. Reload FPM: `docker exec php-fpm kill -USR2 1`.

## Lección

El mismo patrón `if (!CONSTANT)` sin `defined()` puede estar en más sitios del plugin. Ver la incidencia relacionada sobre callbacks huérfanos (`2026-04-21-fatal-callbacks-outlet.md`) — es otra variante del mismo problema de copy-paste en el plugin outlet.