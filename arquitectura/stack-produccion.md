# Stack de producción Xana Core

Infraestructura Docker y layout de sitios en los servidores de la plataforma.

## Layout del stack

- **Ruta host** de cada sitio: `/var/www/wordpress-stack/sites/<dominio>/wp-content/`
- **Ruta equivalente dentro del contenedor `php-fpm`**: `/var/www/html/<dominio>/`
- **Docker compose** en `/var/www/wordpress-stack/`
- **Contenedores típicos**:
  - `php-fpm` (imagen `wordpress-stack-php`, corre como root)
  - `nginx` (alpine)
  - `varnish` (7.x-alpine)
  - `redis` (7-alpine)
  - `traefik` (v3.x)
  - `mysql` (8.0)

## Acceso

- **SSH**: usuario `woman` (sin sudo sin contraseña — pedir al usuario si algo lo requiere)
- **Propietario de archivos** en host: `woman:woman`
- **Propietario dentro del contenedor**: `www-data`

## Repositorio

- Repo canónico: `git@bitbucket.org:xana-technologies/wp-content-xana.git`
- **Se comparte entre múltiples sitios** (demolive.xanasystem.com, demolive.digitaltilecatalog.com, younginteriorsflooring, etc.)
- Cada servidor hace `git pull` de forma independiente
- **No hay pipeline CI/CD automatizado**: un push a `master` no llega a los sitios hasta que alguien lo tira manualmente en cada uno

## Multisite

Algunos sitios son multisite (p.ej. demolive.xanasystem.com con `/` ES y `/en/`). Tras un `wp core update` hay que ejecutar:

```bash
wp --allow-root --skip-plugins --skip-themes core update-db --network
```

## Backups de BD

`mysqldump` **no existe** dentro del contenedor `php-fpm`. Para backups hay que usar el contenedor `mysql` directamente, leyendo la credencial de `wp-config.php`:

```bash
DB_PASS=$(docker exec php-fpm sh -c 'cd /var/www/html/<dominio> && wp --allow-root --skip-plugins --skip-themes config get DB_PASSWORD' | tail -1)
docker exec -e MYSQL_PWD="$DB_PASS" mysql sh -c 'mysqldump --single-transaction --quick --routines --triggers --no-tablespaces -u woman <db_name>' > backup.sql
```

Añadir `--no-tablespaces` para evitar el warning de privilegio `PROCESS` (MySQL 8).

## Por qué importa conocer este layout

- Evita confusiones como buscar código en el repo de un cliente cuando el error está en otro (distintos sitios, mismo repo compartido).
- Permite atacar incidencias (fatales, backups, actualizaciones) con comandos concretos sin tener que explorar el stack cada vez.

## Cómo aplicarlo

Cuando aparezca un error en un dominio Xana Core concreto:
1. Identificar en qué sitio/contenedor ocurre.
2. Comprobar `git rev-parse HEAD` de ese sitio vs el HEAD del repo de referencia.
3. Recordar que los fixes no se propagan solos — `git pull` sitio por sitio.
4. Para cualquier update de core en producción, hacer backup de BD con el patrón de arriba **antes** de tocar nada.