# Borrar un usuario no reasigna su contenido (multisite)

**Fecha:** 2026-09-14
**Fichero:** `themes/uncode-child/functions-xana.php` (repo `wp-content-xana`)
**Afecta a:** todos los catálogos multisite (un blog por idioma)

## Síntoma

Al borrar un usuario eligiendo "Atribuir todo el contenido a…" (Red → Usuarios → Borrar),
el contenido NO pasa al usuario elegido. Los posts se quedan con `post_author` apuntando
a un ID que ya no existe, o acaban en la papelera.

## Causa

Desde 2021 (`e5ed863b7`, "eliminar un usuario de todos los sites a la vez"), el tema
engancha `remove_user_from_blog` para convertir "quitar del site" en borrado de toda la red
con `wpmu_delete_user()`.

Ese hook del core se dispara **antes** de que `remove_user_from_blog()` reasigne:

1. Red → Usuarios llama a `remove_user_from_blog($id, $blog, $reassign)` site a site.
2. El hook borra al usuario en toda la red en el primer site. `wpmu_delete_user()` manda
   a la papelera lo que tenga en cada blog.
3. El core intenta reasignar, pero `get_userdata()` ya no encuentra al usuario y devuelve
   `WP_Error('user_does_not_exist')` sin reasignar nada.

Con WP-CLI (`wp user delete X --reassign=Y` sin `--network`) se rompía de otra forma:
`wp_delete_user()` reasigna solo en el blog actual y luego llama a `remove_user_from_blog()`
**sin** `$reassign`, así que en los demás idiomas el contenido se perdía.

## Fix

- Si el borrado ya es de red (`is_network_admin()`, o el usuario pasó por `wpmu_delete_user`),
  el hook no hace nada y deja al core reasignar site a site. Se marca por usuario, no con
  `did_action()`: con `wp user delete 5 6 --reassign` el 6 se habría saltado.
- En un borrado a nivel de site con reasignación, se reasigna en **todos** los blogs del
  usuario antes de llamar a `wpmu_delete_user()`.
- El `$reassign` que llega a `delete_user` (lo usa `wp_delete_user()`) se guarda y se recupera
  en el hook.
- Los super admin se ignoran: `wpmu_delete_user()` no los borra y el hook reasignaba en balde.

"Eliminar del sitio" (sin reasignación) sigue borrando al usuario de toda la red, como antes.

## Verificación (demolive-xanasystem-local, WP 6.9.4)

| Escenario | Antes | Después |
|---|---|---|
| Red → Usuarios → Borrar con reasignación | posts huérfanos | reasignados en ambos idiomas |
| `remove_user_from_blog()` con reasignación desde un site | posts huérfanos | reasignados en ambos idiomas |
| `wp user delete v1 v2 --reassign=h` | solo reasigna en el blog actual | reasignados en ambos idiomas |
| "Eliminar del sitio" sin reasignación | borrado global, posts a la papelera | igual (sin cambios) |

## Pendiente conocido

- `remove_user_from_blog()` sigue devolviendo `user_does_not_exist` en el borrado desde un site,
  porque el usuario ya no existe cuando el core continúa. `users.php` lo ignora; WP-CLI igualmente
  muestra "Success". Viene de 2021.
- El endpoint `wom_iCat/includes/endpoints/deleteUser.php` llama directamente a `wpmu_delete_user()`
  y no admite reasignación. Tiene `permission_callback => __return_true` y valida el JWT a mano
  (ver `reference_endpoints_rest_xana_sin_permiso`).
