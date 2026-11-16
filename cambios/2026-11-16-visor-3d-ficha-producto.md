# Visor 3D en la ficha de producto (2026-11-16)

Rama `feature/visor-3d-ficha` (desde develop) en el worktree `wp-content-xana-visor3d`. Basado en el visor que hizo un compañero
replicando el de Porcemall (artifact "Visor 3D Egeo Ivory", three.js 0.170).

## Qué es
No hay modelo 3D: la pieza es un `RoundedBoxGeometry` a escala real (ancho × alto × grosor) con la foto de la
cara como textura frontal; canto y dorso genéricos. Controles (sin miniaturas de caras: Alejandro no quiere otra "galería" bajo el visor): "Mezclar" (en una pieza, otra
cara al azar con giro; en colocadas, nuevo reparto), una pieza / colocadas
con junta en rejilla fija 2×2 / 4×4 / 6×6 y colocación recta, trabada ½ o trabada ⅓ (filas desplazadas, la junta
va por filas; al cambiar se deslizan) a elegir. Caras al azar sin dos iguales vecinas (con trabada una pieza toca
hasta 6: búsqueda con vuelta atrás; 2 caras en trabada es imposible → reparto con menos repeticiones; si hay menos caras que
piezas se repiten, si hay más se eligen al azar; con 2 caras, tablero de ajedrez; botón "Mezclar" para barajar de nuevo), cotas mm + pulgadas, luz estudio / ventana, pantalla completa. El acabado del PIM fija la rugosidad
(pulido/lappato brilla, satinado medio, resto mate).

## Configuración por cliente (`themeXanaSettings.json`)
```json
"fichaProducto": {
  "galeria_modo": "imagenes",            // "imagenes" (por defecto) | "3d" | "ambos"
  "visor3d": { "grosor_defecto_mm": 10, "junta_mm": 2, "logo_dorso": 123, "texto_dorso": "…" }  - Dorso como Porcemall: nervaduras anchas y lisas (12 mm de cara, surco de 4 mm) y el logo del cliente marcado en la cerámica (sin color,
  bajorrelieve con bumpMap) en franjas repetidas cada ~18 cm; logo discreto (alto máx. 26 mm o 9 % del lado corto).
  Logo = `logo_dorso` (ID de adjunto o URL; `""` lo quita) → `custom_logo` → opciones de Uncode `_uncode_logo`,
  `_uncode_logo_light`, `_uncode_logo_dark` (solo si son imágenes). La silueta sale del alfa del PNG/SVG (o del
  contraste con el fondo): necesita mismo origen o CORS. Sin logo usa `texto_dorso` / `blogname` como silueta.
  Lleva luz propia (emissiveMap) porque el RoomEnvironment es oscuro por detrás.
 dorso salía gris.
- `ambos`: selector "Imágenes / Vista 3D" encima de la galería; three.js solo se descarga al pulsar 3D.
- `3d`: arranca el visor; si falla (sin WebGL, textura bloqueada por CORS…) se ve la galería, que siempre va en el HTML.
- Filtros: `xana_visor3d_data` (ajustar datos o devolver null para desactivar en un producto),
  `xana_visor3d_tamano_textura` (por defecto `2048x2048`, cae a `full` si no existe).

## De dónde salen los datos
- Caras: `wp_icat_Media` con `tipoEntidad='producto'`, `tipoMedia='imagen'`, `tipoImagen='caras'`,
  `idEntidad = wp_icat_Productos.idProducto` (no idWP), ordenadas por `Orden`. El importador ya las traía; antes
  el Core no las usaba (la galería usa featured + `_product_image_gallery`).
- Medidas: `wp_icat_Productos.idFormato` → `wp_icat_Formatos.Ancho/Alto` en **cm** (×10 = mm).
- Grosor: `Formatos.Profundidad` (siempre 0 en General) → `Productos.thickness_gen` (texto, "12 mm (…)", solo en
  algunos clientes; por eso se hace `SELECT p.*`) → valor por defecto. Si es supuesto, el visor no pinta la cota
  de grosor ni lo pone en el HUD.
- La pieza se orienta según la proporción de la foto (si la foto es apaisada y el formato vertical, se gira).
- Sin visor (galería normal): pieza especial, sin caras o formato fuera de 2–400 cm.

## Ficheros
- `plugins/wom_iCat/includes/visor3d/visor3d.php` — modo, datos, encolado, render (se autoincluye por `includes/`).
- `plugins/wom_iCat/xanaTemplates/singleProduct/Product/Gallery/visor3d-default.php` — envuelve la galería normal.
- `customSingleProduct.php` → `custom_gallery_content()` llama a `xana_visor3d_render()` antes de la galería.
- `plugins/wom_iCat/assets/visor3d/` — `src/visor3d.js` (módulo), `dist/visor3d.js` (bundle con three.js,
  511 KB / 134 KB gzip, se commitea), `visor3d-loader.js`, `visor3d.css`. Compilar: `npm install && npm run build`.

## Probado (local General, Chevron Artwood Grey, 3 caras, 24"×48")
Modos `ambos` y `3d`, cambio de cara, colocadas 2×2, luz, vuelta a imágenes (Swiper se recalcula), móvil 375 px
sin scroll horizontal, producto sin caras y pieza especial → galería normal sin cargar assets del visor.

## Pendiente / ojo
- Las animaciones van con requestAnimationFrame: en una pestaña oculta (o el panel del navegador de Claude oculto)
  se paran y la cola de acciones espera; al volver a la vista siguen. Al probar, mostrar el panel.
- Fase 2: cambio de formato entre productos hermanos (misma serie/color/acabado) con la animación de recorte de la demo.
- Si un cliente sirve uploads desde S3/CDN, el bucket necesita CORS para que WebGL use las texturas (si no, cae a galería).
- El nginx local se cae si `php74-cmp` está parado (vhost cmp-antes): `docker start php74-cmp && docker restart nginx`.
