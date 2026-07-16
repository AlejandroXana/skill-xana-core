# Repos dentro de `wp-content` (inventario)

En un sitio Xana desplegado, `wp-content/` es un **repo compuesto**: el directorio raíz es un clon de `wp-content-xana` y varios plugins/temas dentro son **subrepos independientes** (cada uno con su propio `.git`). No están gestionados como git submodules — son clones yuxtapuestos.

## Inventario actual (2026-04-23)

| Ruta dentro de `wp-content` | Repo Bitbucket | Workspace | Estado |
|---|---|---|---|
| `.` (raíz) | `wp-content-xana` | `xana-technologies` | Activo |
| `themes/uncode-child/client/` | `client-xana-younginteriorsflooring-dtc` | `xana-technologies` | Activo — repo por cliente (ver excepción `client-*` en git-flow) |
| `plugins/wom-iCat-admin/` | `plugin-importacion` | `xana-technologies` | Activo — el importador PIM |
| `plugins/plugin-xanasettings/` | `plugin-xanasettings` | `xana-technologies` | Activo |
| `plugins/wom-iCat-Favorites/` | `plugin_favoritos` | `xana-technologies` | Activo |
| `plugins/gallery-wpbackery/` | `gallery-wpbackery` | `woman_on_mars` | **Deprecado — se eliminará próximamente**. Acceso desde local puede fallar si la SSH key no está añadida al workspace `woman_on_mars`. |

Los repos `xana-technologies` se clonan con la SSH key personal de cada dev (debe tener acceso al workspace). El repo en `woman_on_mars` requiere permiso aparte y en general no hace falta tocarlo.

## Convención sobre develop

Ver `convenciones/git-flow.md`. Resumen:
- Los repos **compartidos** llevan `develop` + rama principal.
- El repo del subdirectorio `client/` **NO** lleva `develop` (excepción por-cliente).
- `gallery-wpbackery` no se va a tocar → no aplica crear `develop`.

## Cómo descubrir todos los subrepos de un sitio

```bash
find /var/www/wordpress-stack/sites/<dominio>/wp-content -maxdepth 4 -type d -name '.git'
```

Cada `.git` encontrado es un repo independiente que puede requerir su propio ciclo de fetch/commit/push.

## Nota sobre `plugin-importacion` (wom-iCat-admin)

El repo es `plugin-importacion` aunque la carpeta se llama `wom-iCat-admin`. Es el importador del PIM — core del flujo de sincronización de productos. Normalmente está en una rama `feature/*` con trabajo activo (ej. visto en 2026-04-23: `feature/OptimizacionesImportacion`). Comprobar antes de tocar si hay trabajo en curso.