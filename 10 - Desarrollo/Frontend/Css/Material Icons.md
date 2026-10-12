---
tags:
  - desarrollo
  - frontend
  - material-icons
tipo: concepto
estado: terminado
---

# Material Icons

Los **Material Icons** (hoy renombrados **Material Symbols**) son el set de iconos oficial de Google, gratuito y de código abierto. La forma clásica de usarlos es como **fuente tipográfica**: cada icono es un glifo de una fuente, y el nombre del icono se escribe como texto. Como cualquier fuente externa, se carga con un `<link>` en el HTML — ver [[Fuentes Externas]].

> [!info] Cómo funciona por dentro: ligaduras tipográficas
> Una fuente de iconos no es un set de imágenes: es un archivo de texto donde los glifos tienen **nombres de ligadura**. Cuando el motor de texto ve la secuencia `h-o-m-e` y existe una ligadura `home` en la fuente activa, la sustituye (vía la tabla GSUB de OpenType) por **un solo glifo** antes de pintar. De ahí las consecuencias prácticas:
> - Sin la fuente cargada, el usuario ve **la palabra** `home` — no un ícono vacío.
> - Cualquier cosa que rompa el token de texto (`text-transform: uppercase`, `letter-spacing`, recorte de whitespace, separadores) **impide el match** de la ligadura.
> - El lookup es más lento que un simple glifo por codepoint; los sets grandes llevan miles de ligaduras (de ahí el peso del archivo — ver Rendimiento).

Google sirve dos generaciones del set:

| | Material Icons (clásico) | Material Symbols (2022+) |
|---|---|---|
| Tipo | Fuente estática, 1 file por peso | Fuente **variable** con 4 ejes |
| Estilos | `Icons`, `Outlined`, `Round`, `Sharp`, `Two tone` | `Outlined`, `Rounded`, `Sharp` (+ `Filled` por eje) |
| Relleno/grosor | Fijos por variante | `FILL`, `wght` interpolables |
| Carga | `icon?family=Material+Icons` | `css2?family=Material+Symbols+…` |
| Peso del archivo | 1 archivo por peso | 1 archivo cubre todos los pesos |

## 1. Cargar la fuente en el HTML

Agregamos la referencia en el `<head>`, **antes que nuestro CSS**, para no tener conflictos:

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <!-- Fuente de los iconos -->
  <link
    href="https://fonts.googleapis.com/icon?family=Material+Icons"
    rel="stylesheet"
  />
  <link rel="stylesheet" href="css/estilos.css" />
  <title>Material Icons</title>
</head>
```

Ese endpoint `icon?family=` sirve la variante clásica **Material Icons** (con relleno). Los cinco sets históricos tienen su propio `family`:

```html
<!-- family=Material+Icons → .material-icons (relleno completo) -->
<!-- family=Material+Icons+Outlined → .material-icons-outlined -->
<!-- family=Material+Icons+Round / +Sharp / +TwoTone -->
```

### Versión moderna: Material Symbols

Google unificó el set en **Material Symbols**, que además permite elegir **peso, grado y filled** variables vía la API `css2` (misma infra que las fuentes de [[Fuentes Externas]]):

```html
<link
  href="https://fonts.google.com/icon?family=Material+Icons" rel="stylesheet"
/>
<link
  href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@24,400,0,0"
  rel="stylesheet"
/>
```

Anatomía del parámetro de ejes: `family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@24,400,0,0` — la lista de ejes va en orden alfabético **con las mayúsculas primero** (`FILL`, `GRAD`, después `opsz`, `wght`), y los valores `@24,400,0,0` en ese mismo orden. Rangos y variantes:

| Eje | Rango | Significado | Ejemplo |
|---|---|---|---|
| `opsz` | 24–48 | **Tamaño óptico**: a 24 los detalles se simplifican; a 48 se añaden finuras | `@24,…` para UI, `@48,…` para display |
| `wght` | 100–700 | **Grosor** (peso del trazo) | `300` light, `400` regular, `500` medium |
| `FILL` | 0 / 1 | 0 = contorno, 1 = relleno | relleno en estado activo |
| `GRAD` | −50 – 200 | **Gradación óptica**: contraste del trazo para tamaños chicos | −50 = más fino, 200 = más pesado |

```css
.material-symbols-outlined {
  font-family: "Material Symbols Outlined";
  font-weight: normal;
  font-style: normal;
  line-height: 1;
  letter-spacing: normal;
  text-transform: none;
  display: inline-block;
  white-space: nowrap;
  word-wrap: normal;
  direction: ltr;
  -webkit-font-smoothing: antialiased;
  font-feature-settings: "liga";
  font-variation-settings: "FILL" 0, "wght" 400, "GRAD" 0, "opsz" 24;
}
```

> [!tip] Dos formas de controlar los ejes
> - `font-weight: 600` funciona para el eje `wght` (mapeo directo del navegador).
> - `font-variation-settings: "FILL" 1, "GRAD" 200` para los ejes **sin** propiedad CSS propia (`FILL`, `GRAD` son ejes personalizados; `opsz` lo maneja `font-optical-sizing` automáticamente).
> Desventaja de `font-variation-settings`: pisa el `font-weight` normal — si lo usás, declará `wght` adentro también, como arriba.

Se pueden pedir **rangos** para abrir todos los ejes: `family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@10..48,100..700,0..1,-50..200` — un único archivo variable donde cualquier valor se usa en tiempo real.

## 2. Definir la clase base en el CSS

La hoja de Google trae el `@font-face` pero **casi ningún estilo útil**: definimos la clase que configura la fuente y neutraliza propiedades heredadas que distorsionan los glifos:

```css
.material-icons {
  font-family: "Material Icons";
  font-weight: normal;
  font-style: normal;
  font-size: 24px; /* Tamaño preferido de icono */
  display: inline-block;
  line-height: 1;
  text-transform: none;
  letter-spacing: normal;
  word-wrap: normal;
  white-space: nowrap;
  direction: ltr;

  /* Soporte para todos los navegadores WebKit */
  -webkit-font-smoothing: antialiased;
  /* Soporte para Safari y Chrome */
  text-rendering: optimizeLegibility;
  /* Soporte para Firefox */
  -moz-osx-font-smoothing: grayscale;
  /* Soporte para navegadores legacy */
  font-feature-settings: "liga";
}
```

Por qué cada línea es necesaria:

| Declaración | Qué evita |
|---|---|
| `font-family` | Sin ella se pinta el nombre del icono con la fuente del texto |
| `font-weight/style: normal` | Los heredados (`b` de un título) disparan búsqueda de caras que no existen o sintetizan |
| `text-transform: none` | MAYÚSCULAS rompen el match de ligaduras (`HOME` ≠ `home`) |
| `letter-spacing: normal` | El tracking intercalado letras entre letras de la ligadura |
| `white-space: nowrap` | Nombres multi-palabra (`arrow_back` es un token, pero `insert chart` en sets antiguos) |
| `direction: ltr` | Bidi en layouts RTL reordena el token |
| `line-height: 1; font-size fija` | La caja del icono hereda la caja de texto y descentra todo |
| `font-feature-settings: "liga"` | Activa las ligaduras en motores donde el usuario las desactivó por defecto |

> [!info] ¿Por qué `font-feature-settings: "liga"`?
> Los iconos se activan mediante **ligaduras** (ligatures): el navegador "fusiona" el texto del nombre del icono en un solo glifo. Sin las ligaduras activas, verías el nombre en lugar del icono.

## 3. Usarlos en el HTML

Solo hay que escribir el nombre del icono como contenido del elemento y asignarle la clase `material-icons`. Los nombres se encuentran en la [galería oficial](https://fonts.google.com/icons):

```html
<i class="material-icons">home</i>
<i class="material-icons">search</i>
<i class="material-icons">settings</i>
<span class="material-icons">delete</span>
```

```html
<!-- Con atributos de acceso -->
<i class="material-icons" aria-hidden="true">home</i>
<button>
  <i class="material-icons" aria-hidden="true">delete</i> Eliminar
</button>
```

Los nombres usan **snake_case** (`arrow_back`, `add_shopping_cart`) y hay que copiarlos exactos de la galería — un `arrow-back` no existe y pinta el texto literal.

## Ajustar tamaño y color: son texto

Al ser una fuente, el icono se estila **como cualquier texto**:

```css
.material-icons.md-18 { font-size: 18px; }
.material-icons.md-24 { font-size: 24px; } /* defecto */
.material-icons.md-36 { font-size: 36px; }
.material-icons.md-48 { font-size: 48px; }

.material-icons.rojo { color: #e74c3c; }
.material-icons.alinear-derecha { float: right; }
```

- **Color = `color`**: los glifos se pintan con el color del texto y no tienen partes — no hay "relleno" y "borde" separados (las variantes Two tone son dos glifos apilados, no estilables).
- **Tamaño escalable**: al ser vectorial, escala sin pixelear; el límite es `opsz` (a tamaños grandes pedí `opsz` alto para detalles).
- **Alineación**: el glifo vive en una caja de línea; para centrarlo en botones usá flex o `vertical-align: middle` — nunca `line-height` mágicas en el padre.

## Recetas

### Botón con icono + texto (patrón correcto)

```css
.boton {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;          /* separación real, no márgenes */
  padding: 0.5rem 1rem;
}

.boton .material-icons { font-size: 18px; }
```

```html
<button class="boton">
  <i class="material-icons" aria-hidden="true">add</i>
  Crear
</button>
```

### Toggle filled/outlined en hover (solo Symbols)

```css
.material-symbols-outlined {
  font-variation-settings: "FILL" 0, "wght" 400, "GRAD" 0, "opsz" 24;
  transition: font-variation-settings 150ms ease;
}

.boton:hover .material-symbols-outlined {
  font-variation-settings: "FILL" 1, "wght" 400, "GRAD" 0, "opsz" 24;
}
```

`font-variation-settings` **es animable** (interpolación de ejes): el relleno "se llena" suavemente. Ver [[Animaciones]].

### Spinner de carga (el icono de la ruleta)

```css
.spin {
  display: inline-block;
  animation: rotar 1s linear infinite;
}

@keyframes rotar {
  to { transform: rotate(360deg); }
}
```

El icono `progress_activity` de Material Symbols está pensado para esto; rotá con `transform` (composición, no repaint).

### Icono de dos colores con `mask`

```css
.icono-dos-tonos {
  background-color: #2563eb;              /* color 1 = tinta */
  mask-image: url("data:image/svg+xml,…"); /* o la fuente como mask */
  mask-size: contain;
  mask-repeat: no-repeat;
}
```

Fuera de las variantes Two tone, `mask` + `background-color` tiñe **cualquier** fuente/icono con color libre (ver [[Imágenes CSS]] para el patrón de mask).

### Actualizar el icono desde JS (sin librerías)

```js
el.textContent = "settings"; // el mismo mecanismo de ligadura re-renderiza
el.classList.add("giro");    // disparar animación quitando/reponiendo la clase
```

## Icon fonts vs SVG sprite vs librerías

| Criterio | Fuente de iconos (Material) | SVG inline / sprite `<use>` | Librería (Lucide, Heroicons, Iconify…) |
|---|---|---|---|
| Peticiones | 1 (la fuente) | 0 si es inline | 0 (bundle) o 1 (CDN Iconify) |
| Color | `color` heredado | `currentColor` heredado | `currentColor` |
| Estilo por CSS | Solo tamaño/color/grosor (ejes) | Cualquier trazo | Cualquier trazo |
| Animación por trazo | Limitada (los ejes ayudan) | `stroke-dasharray` y SMIL | Igual que SVG |
| Cambio dinámico | Cambiar texto (rápido) | Swap de `<use href>` | Swap de componente |
| A11y | Riesgo: el lector borra la ligadura | `aria-hidden` limpio | `aria-hidden` limpio |
| Tamaño del set | Decenas–cientos de KB si no subseteas | Bytes por icono usado | Tree-shake por icono |
| JS | No | No | Sí (en React: react-icons/Heroicons) |

```html
<!-- SVG sprite nativo: la alternativa sin fuentes -->
<svg style="display:none">
  <symbol id="i-home" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M3 9.5 12 3l9 6.5V21H3z" />
  </symbol>
</svg>
<svg class="icono" aria-hidden="true"><use href="#i-home"></use></svg>
```

Regla práctica 2026: en apps React/Vue con set propio, **librería de componentes** (tree-shakeable); en HTML estático o email-ish, **fuente de iconos subseteada**; SVG sprite cuando se necesita máximo control por trazo. Ver [[Archivos SVG]] para el formato.

## Self-hosting y subseteado

Google Fonts sirve el set completo, pero se puede descargar y servir desde tu dominio (privacidad + una sola conexión, ver [[Fuentes Externas]]):

1. Descargar la familia (botón *Get font* en la galería) o extraer de la hoja `css2` la URL del `.woff2`.
2. Subsetear con `pyftsubset` a los iconos usados. **Truco de las ligaduras**: subsetear por el *texto* de los nombres incluye todas sus ligaduras:

```bash
pyftsubset "MaterialSymbolsOutlined[wght].ttf" \
  --text="home search settings delete arrow_back add" \
  --flavor=woff2 --output-file=iconos.woff2
```

3. Declarar y usar:

```css
@font-face {
  font-family: "IconosSitio";
  src: url("/f/iconos.woff2") format("woff2");
  font-display: block; /* iconos: block corto evita ver palabras sueltas */
}

.ic {
  font-family: "IconosSitio";
  font-feature-settings: "liga";
}
```

> [!tip] Alternativa radical: borrar las ligaduras
> Los nombres en claro solo existen para legibilidad humana. Con `fonttools` se puede **reemplazar cada ligadura por un solo carácter** (p. ej. `home` → `a`): el archivo pierde decenas de KB de tablas de sustitución y el renderizado es más rápido. Costo: el HTML queda `class="ic">a</span>` sin auto-documentación — trade-off de equipos grandes, no de tutoriales.

## Accesibilidad

- **Icono decorativo junto a texto**: `aria-hidden="true"` — sin él, el lector de pantalla deletrea "home" o "search" en medio del botón.
- **Icono como único contenido**: el control necesita nombre accesible; el icono va oculto:

```html
<button aria-label="Cerrar">
  <i class="material-icons" aria-hidden="true">close</i>
</button>
```

- **Icono informativo aislado** (no botón): `role="img"` + `aria-label` en el propio elemento — no `aria-hidden`.
- **Contraste**: los iconos que comunican estado (éxito/error) necesitan 3:1 contra el fondo (requisito de no-texto, WCAG 1.4.11) — ver [[Colores CSS]].
- **Target táctil**: el ícono de 18px dentro de un botón no es el target; el botón debe medir ≥ 44×44 px.
- Durante la carga, un lector de pantalla puede leer la ligadura **antes** de que la fuente aplique la sustitución — otro motivo para `aria-hidden` sistemático en decorativos.

## Rendimiento

- **Peso**: el set completo ronda 100–200 KB por estilo/variante (WOFF2). Con `css2` puedes pedir rangos abiertos (más flexible, archivo más grande) o valores fijos (instancia estática por URL, archivo menor). Para producción: subseteado a los ~20–50 iconos reales del proyecto = pocos KB.
- **`font-display`**: el endpoint clásico no aplica `swap` — hasta ~3 s puede haber FOIT (o FOUT con el nombre en texto). Self-hosteando, elegí `block` corto: ver un "home" de texto suelto es peor que 200 ms de vacío en un ícono.
- **Ligaduras vs codepoint**: escribir el glifo por codepoint privado (`content: "\e88a"`) evita el lookup de ligadura; es lo que hacen los codegen de producción. Como contenido HTML, el nombre es más mantenible.
- **Cachear agresivo**: la fuente es inmutable por versión; `Cache-Control: immutable` la hace gratis para las visitas siguientes.
- No confundir con iconos-bitmap: una fuente vectorial escala gratis donde un PNG necesitaría `image-set` (ver [[Imágenes CSS]]).

## Errores comunes

1. **Se ve el nombre del icono, no el ícono**: la fuente no cargó, la clase no coincide con la familia (`material-icons` vs `Material Symbols Outlined`), o las ligaduras están desactivadas (`liga` off). Diagnosticar en ese orden.
2. **`text-transform: uppercase` heredado** (del `h1` del header): rompe las ligaduras — por eso la clase base lo neutraliza.
3. **`letter-spacing` heredado** (del párrafo): interrumpe el match token a token.
4. **Nombre mal escrito** (`arrow-back`): pinta el literal; copiar exacto de la galería.
5. **Eje fuera de rango**: `wght@900` en un pedido fijo no existe (tope 700); el navegador no errora, solo recorta.
6. **URL `css2` con ejes desordenados**: los ejes van alfabéticos, mayúsculas primero; mal ordenado, el endpoint devuelve 400.
7. **Tamaño con `line-height` del padre**: el icono hereda cajas de línea grandes y "flota"; fijar `line-height: 1` en la clase base.
8. **FOIT largo en iconos críticos** (menú de nav): subsetear + `font-display` adecuado, o usar SVG inline para esos.

## Cheatsheet — Material Icons / Symbols

| Concepto | Valores | Defecto | Nota |
|---|---|---|---|
| Endpoint clásico | `icon?family=Material+Icons` | — | 1 estilo fijo por family |
| Endpoint moderno | `css2?family=Material+Symbols+Outlined:…@…` | — | ejes mayúscula-primero |
| Clase clásica | `.material-icons` | — | familia `"Material Icons"` |
| Clase moderna | `.material-symbols-outlined` (`…rounded`, `…sharp`) | — | familia idéntica al nombre |
| Eje `wght` | 100–700 | 400 | vía `font-weight` o variation-settings |
| Eje `FILL` | 0 / 1 | 0 | contorno / relleno |
| Eje `GRAD` | −50–200 | 0 | gradación óptica |
| Eje `opsz` | 24–48 | 24 | `font-optical-sizing` lo automatiza |
| Ligaduras | nombre en snake_case como texto | — | requiere `liga` + token intacto |
| Color | `color` | heredado | un solo tinte por glifo |
| Tamaño | `font-size` | 24px | vectorial, escala libre |
| A11y decorativo | `aria-hidden="true"` | — | siempre junto a texto visible |
| A11y informativo | `role="img"` + `aria-label` | — | ícono aislado con significado |
| Subset | `pyftsubset --text="…"` | — | incluye las ligaduras pedidas |
| Fallback cargando | `font-display: block` | endpoint clásico sin swap | evita ver palabras sueltas |

## Compatibilidad

Las fuentes de iconos (WOFF2 + ligaduras) son universales desde 2016+. Material Symbols variable pide soporte de fuentes variables (2018+); en navegadores viejos se ve el eje por defecto (400, contorno) — degradación elegante. El endpoint `css2` con rangos funciona en los tres motores; `font-variation-settings` también (Chrome 62+, Firefox 62+, Safari 11+).

## Relacionadas

- [[Fuentes Externas]] — el mismo patrón de carga de fuentes externas, `@font-face` y `font-display`
- [[Html]] — ubicación correcta de los `<link>` en el documento
- [[Archivos SVG]] — la alternativa vectorial sin fuente
- [[Animaciones]] — rotar/tintar iconos con `transform` y variaciones
- [[Estilos para escribir CSS]] — utilities `.md-24` como ejemplo de utility-first
- [[TailwindCSS - Sintaxis Básica]] — cómo vivirían estas clases en un sistema utility
- [[Colores CSS]] — contraste mínimo de iconos informativos
