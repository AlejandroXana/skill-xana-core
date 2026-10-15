# Menú `menu-center-split` responsive entre 1025 y 1366 px

**Rama:** `fix/menu-center-split-responsive` (wp-content-xana, desde `develop` `2091d3062`).
**Fichero:** `themes/uncode-child/scss/_elements/_navbars/horizontal-menu-center-split.scss`
(+ `css/style.css` y `css/style.min.css` compilados con `sass` 1.99).
**Detectado en:** adexusa.com (Core v.7.4.3), 15-oct-2026.

## Síntoma

Entre 1025 px (por debajo sale la hamburguesa) y ~1380 px:

- La flecha (`.fa-angle-down`) de cada desplegable cae debajo del texto.
- Los textos de dos palabras (`ABOUT US`, `DEALER AREA`) se parten en dos líneas.
- La cabecera pasa de 99 a 120 px de alto y empuja el contenido.
- El logo encoge: 125 px a 1100 y **53 px a 1030**.

## Causa (Core)

1. `.menu-center-split { grid-template-columns: 160px 3fr 160px }` (y `120px 3fr 160px`
   a ≤1200): las columnas laterales son fijas, aunque la izquierda suele ir vacía
   y la derecha solo lleva ~50 px de iconos. La central se queda con lo que sobra.
2. Los enlaces de primer nivel tienen `white-space: normal`.
3. `.menu-item-logo` tiene `flex-grow: 0` pero `flex-shrink: 1`: es lo primero que encoge.

## Cambio

```scss
grid-template-columns: minmax(max-content, 1fr) minmax(max-content, 3fr) minmax(max-content, 1fr);
// (se elimina la variante de ≤1200px)
.menu-group > .menu-item > a { white-space: nowrap; }
.menu-item-logo { flex-shrink: 0; }
.custom-logo-link { @media (max-width: 1200px) { max-width: 120px; } }
```

Mantiene la proporción 1:3:1 cuando sobra espacio; cuando no, cada columna se queda
con su contenido y el menú no salta de línea. El submenú y el megamenú no cambian.

Verificado inyectando el CSS en adexusa.com a 1920, 1440, 1366, 1200, 1100 y 1030 px:
una línea, sin scroll horizontal, desplegables bien.

## Riesgo

Un cliente con muchos enlaces o etiquetas largas que hoy «cabe» partiendo líneas, con
`nowrap` necesitará más ancho. Si no le llega, la página tendrá scroll horizontal
entre 1025 px y el ancho en que entre. Al subir un cliente con `menu-center-split`,
revisar a 1030 px. El ajuste fino (paddings, logo) va en el client: ver adex.

## Clientes

- **adex**: fix replicado en `client-xana-adexusa` (bloque estructural + ajuste propio
  1025–1399 px) para no tener que subirle el Core. Al subirlo, borrar el bloque 1.
- **samboro** (copia de pruebas con center-split + megamenús): pendiente de revisar.
