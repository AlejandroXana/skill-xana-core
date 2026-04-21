# Flujo git en repos Xana Core

## Flujo estándar

`develop` → `master` vía **Pull Request en Bitbucket**. Los merges habituales tienen mensaje tipo `Merged in develop (pull request #XX)`. Repo principal: `xana-technologies/wp-content-xana`.

## Hotfixes en producción — push directo aceptado

Para fatales en producción, Alejandro acepta saltar la ceremonia del PR y usar push directo:

1. `git checkout develop`
2. Aplicar fix y commit con formato `Fix: <descripción>`
3. `git push origin develop`
4. `git checkout master`
5. `git merge --no-ff develop -m "Merge branch 'develop'"`
6. `git push origin master`
7. `ssh` al servidor afectado, `git pull --ff-only origin master` en `/var/www/wordpress-stack/sites/<dominio>/wp-content`
8. Reload FPM: `docker exec php-fpm kill -USR2 1`

El log queda con mensaje `Merge branch 'develop'` en vez de `Merged in develop (pull request #XX)` — diferencia visual menor pero aceptable para hotfix.

## Formato de commit (preferencia Alejandro)

- `Fix: <descripción>` para bugs
- `Feature: <descripción>` para features
- **Sin** `Co-Authored-By` ni trailers de coautoría
- Mensaje conciso, foco en el *por qué* más que en el *qué* cuando sea relevante

## Pisado de cambios in-situ

Patrón habitual en hotfixes: aplicar fix in-situ en el servidor primero (sed, python, editor) para recuperar servicio en segundos, después propagar por el flujo git normal. Al hacer `git pull` en el servidor después, si el archivo local tiene cambios divergentes:

```bash
git checkout -- <archivo>   # descarta cambios locales (equivalentes al commit)
git pull --ff-only origin master
```

## Propagación multisitio

Como el repo `wp-content-xana` se comparte entre múltiples sitios **sin CI/CD**, tras el push hay que hacer `git pull` manualmente en cada servidor afectado. Un comando útil para chequear el estado de todos los sitios de un servidor:

```bash
for d in /var/www/wordpress-stack/sites/*/wp-content; do
  echo "=== $d ==="
  (cd "$d" && git log --oneline -1 2>/dev/null)
done
```

## Por qué importa

- Un PR por cada fatal que está tirando producción es burocracia innecesaria.
- El push directo queda registrado en el log igual, solo con formato de merge manual.
- Saber qué sitios están desfasados evita que un fix quede aplicado en unos servidores y no en otros.