# Decisiones (ADR — Architecture Decision Records)

Decisiones arquitectónicas tomadas con su rationale. Capturar aquí para que en el futuro se pueda reconstruir por qué algo se hizo de cierta forma (y no al revés).

**Formato de archivo**: `YYYY-MM-DD-slug.md` (cronológico).

**Plantilla de cada ADR**:

```markdown
# <Título de la decisión>

**Fecha**: YYYY-MM-DD
**Estado**: propuesta | aceptada | supersededa por XXXX

## Contexto
Qué problema/situación motiva la decisión.

## Opciones consideradas
- Opción A — pros/contras
- Opción B — pros/contras
- ...

## Decisión
Qué se ha elegido y por qué.

## Consecuencias
Qué implica esto a futuro (tanto positivo como negativo).
```

**Ejemplo de cuándo añadir un ADR**: elegir un sistema de caché (Varnish vs WP Rocket), decidir migrar de sesiones PHP a user_meta, estandarizar un naming de hooks, etc.