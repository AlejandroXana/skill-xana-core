# Stack Docker de Xana Core

Stack unificado para **desarrollo local y producción**. Desde el 2026-04-21 es el único stack del equipo (ver ADR `decisiones/2026-04-21-unificar-stack-dev-prod.md`). Lo que funciona en local debe comportarse igual en servidor.

Repositorio del stack:
- `https://github.com/xana-technologies/wordpress-stack`

## Layout

- **Ruta host** de cada sitio: `/var/www/wordpress-stack/sites/<dominio>/wp-content/` (en servidores) o la ruta equivalente donde cada dev clone `wordpress-stack` en su máquina.
- **Ruta equivalente dentro del contenedor `php-fpm`**: `/var/www/html/<dominio>/`
- **Docker compose** en la raíz del repo `wordpress-stack/`
- **Contenedores típicos**:
  - `php-fpm` (imagen `wordpress-stack-php`, corre como root)
  - `nginx` (alpine)
  - `varnish` (7.x-alpine)
  - `redis` (7-alpine)
  - `traefik` (v3.x)
  - `mysql` (8.0)

## Acceso (servidores)

- **SSH**: usuario `woman` (sin sudo sin contraseña — pedir al usuario si algo lo requiere)
- **Propietario de archivos** en host: `woman:woman`
- **Propietario dentro del contenedor**: `www-data`

## Repositorios involucrados

Xana Core se divide en **tronco común** (todos los catálogos) y **rama DTC** (catálogos derivados, ver sección siguiente). Solo se listan los repos que pertenecen al ecosistema Xana Core; los de B2B, CDI/Newker y otros productos de Xana Technologies están fuera de scope.

### Infraestructura

| Repo | Contiene | Remote |
|---|---|---|
| `wordpress-stack` | Docker compose, VCL Varnish, nginx.conf, Dockerfiles, scripts del stack | `github.com:xana-technologies/wordpress-stack` |

Nota: existe un repo antiguo `bitbucket.org:xana-technologies/docker-wp` usado en stacks de dev locales históricos. **Es legacy** — el oficial unificado dev+prod es `wordpress-stack` (ver ADR `2026-04-21-unificar-stack-dev-prod.md`).

### Tronco común (todo Xana Core)

| Repo | Se clona en | Contiene |
|---|---|---|
| `bitbucket.org:xana-technologies/wp-content-xana` | `sites/<dominio>/wp-content/` | Tema `uncode-child` y código base. **Sin plugins** (en `.gitignore`) |
| `bitbucket.org:xana-technologies/plugin-xanasettings` | `wp-content/plugins/XanaSettings/` (o `plugin-xanasettings/`) | Configuración global del tema Xana |
| `bitbucket.org:xana-technologies/plugin-importacion` | `wp-content/plugins/wom-iCat-admin/` | Importación de catálogos |
| `bitbucket.org:xana-technologies/plugin_favoritos` | `wp-content/plugins/wom-iCat-Favorites/` | Favoritos del usuario en catálogo |

### Rama DTC (exclusivo)

Un **DTC (Digital Tile Catalog)** es un catálogo que "cuelga" de un cliente proveedor: el proveedor sube su producto y ofrece a los DTCs la opción de republicar los mismos productos en la web del DTC. No todos los catálogos Xana Core son DTC — sí lo son si usan el plugin siguiente:

| Repo | Se clona en | Contiene |
|---|---|---|
| `github.com:xana-technologies/plugin-dtc-templates` | `wp-content/plugins/xana-dtc-images/` (o `plugin-dtc-templates/`) | Plantillas/imágenes específicas de la rama DTC |

### Clientes (personalización por catálogo)

Por cada catálogo que lanzamos hay un repo `client-xana-<nombre>` que se clona dentro de `wp-content/themes/uncode-child/client/`. Contiene SCSS, `themeXanaSettings.json`, traducciones y `client-functions.php` (ver `CLAUDE.md` del cliente para guía).

Ejemplos actuales: `client-xana-alfagres`, `client-xana-viterra`, `client-xana-tilecenter-dtc`, `client-xana-catalogo-digital`, `client-xana-matteo-kitchens`, `client-xana-laterrastone-dtc`.

Cada catálogo nuevo → nuevo repo `client-xana-<nombre>` en `bitbucket.org:xana-technologies/`.

## Multisite

Algunos sitios son multisite (p.ej. demolive.xanasystem.com con `/` ES y `/en/`). Tras un `wp core update` hay que ejecutar:

```bash
wp --allow-root --skip-plugins --skip-themes core update-db --network
```

## Propagación de cambios

**No hay pipeline CI/CD automatizado**. Un push a `master` de `wp-content-xana` no llega a los sitios hasta que alguien hace `git pull` en cada uno. Lo mismo aplica a `wordpress-stack`: un cambio en el compose/VCL requiere pull + `docker compose up -d` manual en cada entorno (local o servidor).

## Backups de BD

`mysqldump` **no existe** dentro del contenedor `php-fpm`. Para backups usar el contenedor `mysql` directamente, leyendo la credencial de `wp-config.php`:

```bash
DB_PASS=$(docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && wp --allow-root --skip-plugins --skip-themes config get DB_PASSWORD' | tail -1)
docker exec -e MYSQL_PWD="$DB_PASS" mysql sh -c 'mysqldump --single-transaction --quick --routines --triggers --no-tablespaces -u woman <db_name>' > backup.sql
```

Añadir `--no-tablespaces` para evitar el warning de privilegio `PROCESS` (MySQL 8).

## Por qué importa

- Como el stack es idéntico en dev y prod, lo que reproduces en local tiene garantías de comportarse igual desplegado — se acaba el *"en mi máquina funcionaba"*.
- Los runbooks y comandos de esta skill aplican a cualquier entorno; no hay casos especiales de "esto solo funciona en producción".
- Evita confusiones entre sitios (mismo `wp-content-xana` compartido entre múltiples).

## Cómo aplicarlo

Cuando aparezca un error en un dominio Xana Core concreto:
1. Identificar en qué sitio/contenedor ocurre (local o servidor, da igual — el stack es el mismo).
2. Comprobar `git rev-parse HEAD` del `wp-content` de ese sitio vs el HEAD del repo de referencia.
3. Recordar que los fixes no se propagan solos — `git pull` sitio por sitio.
4. Para cualquier update de core en producción, hacer backup de BD con el patrón de arriba **antes** de tocar nada.