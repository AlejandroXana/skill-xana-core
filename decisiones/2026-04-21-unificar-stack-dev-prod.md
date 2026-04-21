# Unificar stack de desarrollo y producción bajo `wordpress-stack`

**Fecha**: 2026-04-21
**Estado**: aceptada — en aplicación inmediata

## Contexto

Hasta hoy, los entornos de desarrollo local del equipo corrían con un Docker distinto al de los servidores de producción. El directorio típico en local era `Documents/xana/Docker Produccion/` con un compose propio que no coincidía 1-a-1 con el stack real en servidor (`/var/www/wordpress-stack/` sobre imágenes `wordpress-stack-php`, `varnish:7-alpine`, `nginx:alpine`, `mysql:8.0`, etc.).

Consecuencias del drift:
- Problemas que solo aparecen en producción y no son reproducibles en local (o al revés).
- Runbooks que valen para un entorno pero no para el otro — ej. `docker exec php-fpm kill -USR2 1` funciona en prod pero no necesariamente en el Docker de local.
- Documentación que se bifurca (una versión "dev", otra "prod").
- Tiempo perdido en depurar incidencias producidas por la propia diferencia de stacks.

## Opciones consideradas

**A. Mantener dos stacks distintos**
- Pros: nadie tiene que cambiar nada ahora.
- Contras: perpetúa el drift, obliga a mantener documentación duplicada, bugs de paridad siguen apareciendo.

**B. Unificar usando `wordpress-stack` también en local** ← elegida
- Pros: paridad dev-prod total, un único runbook, un único docker-compose que mantener, onboarding de devs más simple.
- Contras: migración puntual del entorno local de cada dev, puede haber ajustes de ergonomía (puertos, certificados locales, hot reload).

**C. Crear un stack "xana-dev" inspirado en `wordpress-stack` pero optimizado para desarrollo**
- Pros: ergonomía de dev (ej. hot reload, sin Varnish).
- Contras: reintroduce drift de otra forma — dos stacks conceptualmente paralelos a mantener en sincronía.

## Decisión

Adoptar el repo `github.com:xana-technologies/wordpress-stack` también para desarrollo local. Los devs clonan ese repo en su máquina y lo usan como entorno único.

## Consecuencias

### Positivas
- **Paridad total** entre lo que ejecuto en mi Mac y lo que corre en los servidores demolive/productivos.
- **Runbooks simplificados**: los comandos `docker exec php-fpm …`, `docker exec mysql mysqldump …`, reload FPM con `USR2`, etc., funcionan igual en cualquier entorno.
- **Onboarding unificado**: un solo documento para explicar cómo levantar Xana Core en cualquier máquina.
- **Varnish en local**: pasamos a tener cache layer en dev, lo que permite detectar problemas de caché antes de desplegar (ver contexto en la discusión sobre sesiones PHP y bypass de Varnish que motivó parte de esta revisión).

### A vigilar
- Devs que ya tenían el `Docker Produccion/` local funcionando deben migrar. Plan: clonar `wordpress-stack`, mover su `sites/<dominio>/wp-content` existente (o hacer `git clone` limpio) y arrancar.
- Los recursos (RAM/CPU) en portátiles pueden sentir la diferencia respecto a un compose más ligero — acordar si se desactivan contenedores no críticos en dev (p.ej. traefik si hay proxy local propio).
- Documentación existente que menciona "Docker Produccion" queda obsoleta — revisar y actualizar progresivamente.

### Seguimiento

Cuando haya un setup de referencia probado, documentarlo en:
- `arquitectura/stack.md` (ya actualizado en esta misma revisión)
- `runbooks/setup-desarrollo-local.md` (pendiente — crear cuando el primer dev complete la migración y valide los pasos)
- `docs-dev/onboarding.md` (pendiente — guía de bienvenida para devs nuevos)