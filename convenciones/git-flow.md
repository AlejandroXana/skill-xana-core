# Flujo git en repos Xana

Aplica a **todos** los repos del equipo: `wp-content-xana`, `wordpress-stack`, plugins individuales, etc. Si un repo no tiene las dos ramas base, hay que crearlas al iniciarse.

## Ramas base

Los proyectos "compartidos" (plataforma, plugins genéricos, tema base) trabajan con dos ramas permanentes:

- **Rama principal** (`main` o `master`, según el repo) — estado estable, lo que corre en producción. Nunca se empuja directo salvo hotfix autorizado (ver abajo).
- **`develop`** — integración. Aquí aterrizan todas las features/fixes antes de pasar a la rama principal.

### Excepción: repos exclusivos de cliente

Los repos **exclusivos de un único cliente** (p.ej. `client-xana-younginteriorsflooring-dtc`, ubicados en `themes/uncode-child/client/` o similar) **NO llevan `develop`**. Trabajan directamente sobre la rama principal (`master`) porque:
- Solo hay un entorno de producción por cliente.
- Los cambios se validan directamente en el sitio del cliente.
- Añadir `develop` sería sobrecarga sin beneficio.

Identificarlos por: prefijo `client-*` en el nombre del repo, o ubicación dentro del subdirectorio `client/` de un tema. Si dudas, preguntar al usuario.

**Detectar la rama principal del repo:**

```bash
git remote show origin | grep 'HEAD branch'
# o
git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||'
```

Repos antiguos suelen usar `master` (Bitbucket clásico); repos nuevos suelen usar `main` (GitHub actual). Ambos son válidos. Úsala tal como esté, no la renombres.

Si un repo solo tiene la rama principal, al iniciar trabajo crear `develop` a partir del HEAD actual y usar `develop` de ahí en adelante. Informar al usuario si detectas un repo sin `develop` antes de seguir.

## Flujo de trabajo estándar

```
feature/xxx  ──┐
               ├──> develop ──> <rama principal>
fix/yyy      ──┘
```

Sustituye `<rama principal>` por `main` o `master` según el repo.

1. **Salir de `develop`**: `git checkout develop && git pull`
2. **Crear rama**: `git checkout -b feature/<nombre-descriptivo>` o `fix/<nombre>`
3. **Commits** con formato `Feature: …` o `Fix: …` (sin `Co-Authored-By`)
4. **Merge a `develop`** al acabar:
   - Vía Pull Request en Bitbucket/GitHub (preferido)
   - O merge directo si el usuario lo indica
5. **Validación en entorno de prueba** (desde `develop`)
6. **Merge `develop` → rama principal** cuando todo OK → despliegue a producción

### Commits directos a `develop`

Aceptado para cambios pequeños o bien entendidos, sin necesidad de rama `feature/`. Pero **nunca** commit directo a la rama principal fuera del flujo de hotfix.

## Hotfixes en producción — push directo aceptado

Para fatales en producción, Alejandro acepta saltar la ceremonia del PR. Sustituye `<main>` por el nombre real de la rama principal del repo:

1. `git checkout develop`
2. Aplicar fix + commit `Fix: <descripción>`
3. `git push origin develop`
4. `git checkout <main>`
5. `git merge --no-ff develop -m "Merge branch 'develop'"`
6. `git push origin <main>`
7. `ssh` al servidor, `git pull --ff-only origin <main>` en la ruta del repo
8. Reload FPM si aplica: `docker exec php-fpm kill -USR2 1`

## Formato de commit (preferencia Alejandro)

- `Fix: <descripción>` para bugs
- `Feature: <descripción>` para features
- **Sin** `Co-Authored-By` ni trailers de coautoría
- Mensaje conciso, foco en el *por qué* cuando sea relevante

## Pisado de cambios in-situ

Patrón habitual en hotfixes: aplicar fix in-situ en el servidor primero (sed, editor) para recuperar servicio en segundos, después propagar por el flujo git normal. Al hacer `git pull` en el servidor después, si el archivo local tiene cambios divergentes:

```bash
git checkout -- <archivo>   # descarta cambios locales (equivalentes al commit)
git pull --ff-only origin <main-o-master>
```

## Propagación multisitio (solo `wp-content-xana`)

Como el repo `wp-content-xana` se comparte entre múltiples sitios **sin CI/CD**, tras el merge a la rama principal hay que hacer `git pull` manualmente en cada servidor afectado. Chequear el estado de todos los sitios de un servidor:

```bash
for d in /var/www/wordpress-stack/sites/*/wp-content; do
  echo "=== $d ==="
  (cd "$d" && git log --oneline -1 2>/dev/null)
done
```

## `develop` abandonada o por detrás de la rama principal

Varios repos Xana tienen `develop` desactualizada respecto a `master`/`main` (histórico: hotfixes se fueron directos a la principal sin propagarse a `develop`).

### Paso 1 — Comparar historial Y contenido

No basta con mirar el número de commits de diferencia: puede haber commits de merge vacíos, o `Feature X` + `Revert X` que se anulan. **La fuente de verdad es el tree (contenido de archivos), no el historial.**

```bash
git fetch --prune origin

# Commits distintos (puede incluir ruido: merges vacíos, feature+revert...)
git log --oneline origin/develop..origin/master   # master no está en develop
git log --oneline origin/master..origin/develop   # develop no está en master

# ✅ Fuente de verdad: ¿hay diferencia real de archivos?
git diff --stat origin/develop origin/master
# Si está vacío → contenido idéntico, NO hace falta FF aunque los hashes difieran.
```

### Paso 2 — Decidir según el caso

**Caso A — contenido distinto, develop atrás, 0 commits exclusivos en develop:**
```bash
git checkout develop
git reset --hard origin/develop       # alinea local con remoto
git merge --ff-only origin/master     # fast-forward de master → develop
git push origin develop
```

**Caso B — contenido idéntico pero hashes distintos (merges vacíos, feature+revert):**
No hacer FF. Subiría commits de ruido sin aportar nada. Trabajar directamente sobre `origin/develop`:
```bash
git checkout develop
git reset --hard origin/develop
# commitear los cambios nuevos encima
```

**Caso C — hay commits exclusivos en develop (trabajo en curso):**
NO resetear. Parar y aclarar con el usuario qué hacer con ese trabajo antes de tocar nada.

### Casos reales documentados (2026-04-23)

- **plugin_favoritos** → Caso A. Develop abandonada desde 2024-01-11, master con +10 commits de contenido real. FF aplicado.
- **wp-content-xana** → Caso B. Apariencia de +20 commits de desfase, pero tree idéntico (merges vacíos + `Feature add search` revertido). Resuelto trabajando directamente sobre `origin/develop`.

## Resumen decisión

- Por defecto: trabajar en `develop` o rama `feature/*`|`fix/*` que parte de `develop`.
- A la rama principal (`main` o `master`): solo vía merge desde `develop` (tras validación) o hotfix autorizado.
- Nunca commit directo a la rama principal sin confirmarlo.
- Comprobar qué rama principal tiene cada repo antes de asumir — depende del proyecto.
- Antes de commitear en `develop`, comprobar si está al día con la rama principal. Si no, fast-forward primero.