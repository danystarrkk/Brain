---
tags:
  - desarrollo
  - frontend
  - svg
tipo: concepto
estado: terminado
---

# Archivos SVG

**SVG** (*Scalable Vector Graphics*, "gráficos vectoriales escalables") describe imágenes con **fórmulas** — rectas, círculos, curvas y texto — en vez de una grilla de píxeles. Esa es su gran ventaja: al no tener resolución fija, **se pueden escalar y animar programáticamente** sin perder calidad, con archivos de pocos kilobytes. Y como el SVG vive embebido en el HTML, es parte del [[Document Object Model (DOM)|DOM]] y se manipula con los mismos métodos de selección.

> [!info] Vector vs raster, en una frase
> Un PNG/JPG/WebP (raster) responde "qué color en cada píxel"; un SVG responde "**cómo dibujar**" con instrucciones matemáticas. El navegador ejecuta esas instrucciones a la resolución que sea — por eso un logo SVG se ve idéntico en un favicon de 16px y en una valla de 4 metros.

| | **Raster** (PNG, JPG, AVIF, WebP) | **Vectorial** (SVG) |
|---|---|---|
| Contenido | Grilla de píxeles | Formas + texto + filtros (XML) |
| Escalado | Se pixela/borrea | Perfecto a cualquier tamaño |
| Edición de color | Requiere regenerar | CSS o atributos, en vivo |
| Peso vs detalle | Complejo = MB | Complejo = KB |
| Mejor para | Fotos, texturas, video | **Logos, iconos, ilustraciones, diagramas, UI** |

Regla práctica: fotos → raster (ver [[Formatos Avif y Webp]]); todo lo que son **formas o iconos** → SVG. Cuando una foto raster necesita reescalarse mucho, se compara contra SVG en [[Imágenes CSS]].

## Cómo incluir SVG en una página (4 caminos)

| Método | Ejemplo | ¿DOM? | ¿CSS/JS de la página? | Notas |
|---|---|---|---|---|
| **Inline** | `<svg>…</svg>` directo en el HTML | **Sí** | **Sí, total** | El ideal para iconos e interactividad; tema con `currentColor` |
| **`<img>`** | `<img src="logo.svg" alt="Logo">` | No | **No** (solo `alt`/tamaño) | Simple y cacheable; SVG aislado: sus estilos viajan dentro |
| **CSS background** | `background-image: url('logo.svg')` | No | No | Decorativo; ver [[Imágenes CSS]] |
| **`<use>` + sprite** | `<use href="#icono">` desde sprite inline | Sí (copia viva) | Sí | Reutilización sin repetir el código del SVG |

- **Inline** cuando la página controla la apariencia (tema oscuro, hover, animación con JS) o cuando se usa una sola vez y conviene un `<symbol>` de sprite.
- **`<img>`** cuando el SVG es una pieza cerrada (contenido de terceros, email, decoración que nunca cambia).
- **Sprite externo + `<use href="sprite.svg#icono">`** reduce peticiones, pero el contenido cargado por `<use>` externo **no** hereda CSS/JS de la página — motivo por el que el patrón dominante hoy es el **sprite inline** (abajo).

> [!tip] `viewBox` sí o sí
> Todo `<svg>` bien hecho lleva `viewBox`. Sin él, el SVG usa sus coordenadas "crudas" y no escala bien: se corta o queda en un tamaño raro (ver "Errores comunes").

## El sistema de coordenadas y `viewBox`

Un SVG es un lienzo con su propio sistema de coordenadas: **origen arriba-izquierda**, `x` crece hacia la derecha, `y` crece hacia **abajo** (a diferencia de las matemáticas clásicas, donde `y` sube).

```
(0,0) ───────────► x
  │   ┌─────────────────┐
  │   │  ● (10,5)       │   viewBox="0 0 20 10"  →  el lienzo mide
  │   │                 │   20 unidades de ancho y 10 de alto.
  ▼   └─────────────────┘   1 unidad = 1/20 del ancho renderizado.
  y
```

`viewBox="minX minY ancho alto"` define **qué porción del espacio vectorial se dibuja** y **cómo se mapea al tamaño del elemento**:

```html
<!-- El mismo código dibujado a 100px o a 1000px: idéntica nitidez -->
<svg viewBox="0 0 24 24" width="100" height="100"><!-- icono de 24 unidades --></svg>
```

| Atributo | Rol |
|---|---|
| `viewBox` | Ventana de coordenadas internas: qué se dibuja y a qué escala |
| `width`/`height` | Tamaño del elemento en la página (con CSS se overridea) |
| `preserveAspectRatio` | Cómo encaja el viewBox en el rectángulo: defecto `xMidYMid meet` (centrado, sin recortar, **respeta proporción**); `none` (estira); `xMidYMid slice` (llena y recorta) |
| `x`, `y` | Desplazan el lienzo dentro de un `<svg>` anidado (patrón de *viewport anidado*) |

> [!info] La potencia de `viewBox`
> Separa **diseño** (las coordenadas de adentro: "mi ícono mide 24×24") de **presentación** (el tamaño CSS de afuera). Es el mismo principio que un `.fig` de Figma exportado a cualquier dimensión. Los iconos modernos (Lucide, Material) son SVGs de `viewBox="0 0 24 24"`.

## Anatomía de un SVG: las formas básicas

Un archivo `.svg` es XML (con llaves `{}` no, con etiquetas y atributos):

```xml
<svg viewBox="0 0 200 100" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="titulo">
  <title id="titulo">Círculo y rectángulo</title>
  <rect x="10" y="10" width="80" height="80" rx="12" fill="#38bdf8" />
  <circle cx="150" cy="50" r="35" fill="none" stroke="currentColor" stroke-width="2" />
</svg>
```

- `xmlns="http://www.w3.org/2000/svg"` es el **espacio de nombres**. El HTML5 inline no lo exige; sí es obligatorio en archivos `.svg` servidos como documentos, y en HTML antiguo/XHTML.
- `role`, `aria-labelledby` y `<title>`: accesibilidad — ver la sección al final.

### Formas nativas (sin `path`)

| Elemento | Atributos clave | Dibuja |
|---|---|---|
| `<rect>` | `x y width height`, `rx ry` (redondeo) | Rectángulo (con `rx`: esquinas redondeadas) |
| `<circle>` | `cx cy r` (centro y radio) | Círculo |
| `<ellipse>` | `cx cy rx ry` | Elipse |
| `<line>` | `x1 y1 x2 y2` + `stroke` | Segmento (necesita `stroke`: el `fill` no lo ve) |
| `<polyline>` | `points="x1,y1 x2,y2 …"` | Línea quebrada abierta |
| `<polygon>` | `points="…"` | Figura **cerrada** (cierra sola al último punto) |
| `<text>` | `x y`, estilos por CSS | Texto (accesible y seleccionable) |
| `<image>` | `href`, `x y width height` | Incrusta raster dentro del SVG |

```html
<svg viewBox="0 0 100 60">
  <polygon points="0,60 50,0 100,60" fill="tomato" />          <!-- triángulo -->
  <polyline points="5,55 25,35 45,45 65,15" fill="none" stroke="#333" stroke-width="2" />
  <line x1="0" y1="59" x2="100" y2="59" stroke="#999" />
  <text x="50" y="20" text-anchor="middle" font-size="8">Hola</text>
</svg>
```

## El lenguaje `path`: dibujar lo que no existe forma para

`<path d="...">` es el comodín: **una cadena de comandos** que describe trazados arbitrarios. El atributo `d` es minúsculas-sensitive (`M` ≠ `m`): mayúsculas = coordenadas **absolutas**; minúsculas = **relativas** al punto actual.

| Cmd | Nombre | Parámetros | Qué hace |
|---|---|---|---|
| `M x y` | MoveTo | 1 punto | **Eleva** el lápiz en ese punto (empieza un sub-trazado) |
| `L x y` | LineTo | 1 punto | Línea recta hasta ahí |
| `H x` / `V y` | Horizontal/Vertical | 1 número | Línea horizontal/vertical |
| `C x1 y1 x2 y2 x y` | Cubic Bézier | 2 controles + final | Curva cúbica: los puntos de control "apalan" la curva |
| `S x2 y2 x y` | Smooth cubic | 1 control + final | Refleja el control anterior → continuidad suave |
| `Q x1 y1 x y` | Quadratic Bézier | 1 control + final | Curva cuadrática (más simple) |
| `T x y` | Smooth quad | 1 punto | Refleja el control del `Q` anterior |
| `A rx ry rot largueza-arc sweep x y` | Elliptical Arc | radios, rotación, flags, final | Arco elíptico (botones de "píldora", círculos dibujados) |
| `Z` | ClosePath | — | Cierra el sub-trazado con línea al inicio |

```html
<!-- Casa dibujada solo con path -->
<svg viewBox="0 0 100 80" width="150">
  <path d="M10 40 L50 10 L90 40" fill="none" stroke="#333" stroke-width="3" />  <!-- techo -->
  <path d="M20 40 L20 75 L80 75 L80 40 Z" fill="#fca5a5" stroke="#333" stroke-width="2" /> <!-- cuerpo -->
  <path d="M42 75 L42 55 L58 55 L58 75" fill="#7c3aed" />                       <!-- puerta -->
</svg>
```

### Receta práctica de la curva C

La curva cúbica es el 90% del diseño de iconos: `M` al inicio, una o varias `C`, `Z` si se cierra.

```
M 10 80                        → arranco abajo-izquierda
C 10 30, 60 30, 60 80          → control1 (10,30), control2 (60,30), final (60,80): una "colina"
Z                              → cierro la colina formando un semicírculo relleno
```

> [!tip] Generar `d` sin sufrir
> Los paths profesionales no se escriben a mano: se **dibujan** (Figma/Illustrator exportan a SVG) o se generan (Haikei, oramice). Leé el `d` como "recorrido del lápiz" y podés retocarlo a mano; para crear desde cero, la herramienta gráfica gana.

Reglas del `d`: comandos se concatenan sin separadores (`M10 20L30 40` es válido); números se separan por espacios o comas; múltiples pares tras un comando repiten el comando (`L10 20 30 40` = dos líneas).

## Estilizar SVG: CSS, `currentColor`, `fill` y `stroke`

El modelo pintura del SVG: **`fill`** rellena el interior; **`stroke`** dibuja el contorno (con `stroke-width`, `stroke-linecap` `butt|round|square`, `stroke-linejoin`, `stroke-dasharray`). Ambos aceptan color, `none` o `url(#gradiente)`.

- **Inline**, el SVG se estila con CSS normal (clases, selects, variables CSS, herencia):

```css
.icono { fill: currentColor; }              /* hereda el color del texto circundante */
.icono--alerta { color: crimson; }          /* al cambiar el color, cambia el icono */
.icono:hover { transform: scale(1.1); transition: transform 0.2s; }  /* ver [[Animaciones]] */
.tema-oscuro .icono { color: #e5e7eb; }     /* theming gratis */
```

- **`currentColor`** es la joya: el `fill`/`stroke` toma el valor de la propiedad CSS `color` del contexto — un icono hereda el color del texto donde se inserte. Así un sprite se usa en botones de 5 colores distintos sin duplicar archivos.
- **Variables CSS** llegan al SVG inline: `fill: var(--color-primario)`.
- En `<img>` o background, **la CSS de la página no llega**: el SVG es un documento aislado. Su apariencia debe venir de estilos **dentro** del archivo (`<style>` en el SVG) o de atributos.
- `stroke` + `fill="none"` es el combo de los iconos de línea; `vector-effect="non-scaling-stroke"` mantiene el grosor constante aunque el SVG escale (útil en diagramas).

## Patrón sprite: `<symbol>` + `<use>` (iconos reutilizados)

Un sprite inline es un SVG invisible con definiciones, seguido de `<use>` en cada icono — código una sola vez, N instancias:

```html
<!-- Sprite: una vez arriba del body -->
<svg style="display:none" aria-hidden="true">
  <symbol id="i-flecha" viewBox="0 0 24 24">
    <path d="M5 12h14M13 6l6 6-6 6" fill="none" stroke="currentColor"
          stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
  </symbol>
</svg>

<!-- Uso: en botones, navs, donde sea -->
<button><svg class="icono" width="20" height="20"><use href="#i-flecha"/></svg> Seguir</button>
<a href="#"><svg class="icono" width="16" height="16"><use href="#i-flecha"/></svg> Ver más</a>
```

- `symbol` tiene `viewBox` propio; `<use>` lo clona a cualquier tamaño sin distorsión.
- La copia **hereda CSS de la página** (patrón inline) — por eso se prefiere al sprite externo.
- Variantes de color: la herencia de `currentColor` hace que el mismo `<use>` se pinte según el contexto.
- Alternativa moderna: librerías de iconos que exportan este patrón (Lucide, heroicons) — ver el enfoque de [[Material Icons]].

## Optimización: SVGO

Los SVGs exportados de herramientas de diseño traen basura: metadatos de Figma, capas invisibles, decimal con 6 cifras, `xmlns` redundantes. **SVGO** (la herramienta de optimización estándar, detrás de `vite-plugin-svgo` y `svg-spriter`) los elimina:

```bash
npx svgo logo.svg                    # optimiza in place
npx svgo -o logo.min.svg logo.svg    # salida separada
```

- Ahorros típicos: **20–70%** del peso.
- Cuidado con presets demasiado agresivos: `removeViewBox` **rompe el escalado** (desactívalo), y ciertos `cleanupIDs` rompen animaciones/referencias `url(#id)`.
- Regla: SVGO en el build, nunca a mano; y comparar visualmente antes de publicar.

## Animación de SVG

| Motor | Para qué sirve | Nota |
|---|---|---|
| **CSS** (ver [[Animaciones]]) | Transiciones/keyframes sobre `transform`, `fill`, `stroke`, `opacity` | `transform` en SVG necesita unidad (`translateX(10px)`) o el modo SVG (`transform="translate(10,10)"`) |
| **JS** (GSAP ver [[Librería GSAP]]) | Coreografía, trazados (`stroke-dashoffset`), morphing | GSAP anima `attr`, `strokeDashoffset`, `morphSVG`… |
| **SMIL** (`<animate>`, `<animateTransform>`) | Animaciones declarativas dentro del SVG | Nativo y cacheable, pero histórico/conflictivo: solo para piezas aisladas; el estándar actual es CSS/JS |

```css
/* Trazo que se dibuja solo (receta clásica) */
.trazo {
  stroke-dasharray: 300;        /* longitud del trazo */
  stroke-dashoffset: 300;       /* oculto */
  animation: dibujar 2s ease forwards;
}
@keyframes dibujar { to { stroke-dashoffset: 0; } }
```

```js
// Mismo efecto con GSAP sobre el atributo, con control de tiempo
gsap.fromTo('.trazo', { strokeDashoffset: 300 }, { strokeDashoffset: 0, duration: 2 });
```

## Accesibilidad

- **Icono decorativo** (el texto de al lado ya lo explica): `aria-hidden="true"` en el `<svg>` — el lector de pantalla lo salta.
- **Icono informativo** (es el único portador del significado): `role="img"` + `<title>` accesible o `aria-label`:

```html
<svg role="img" aria-label="Carrito de compras" viewBox="0 0 24 24">…</svg>
<!-- o con title referenciado -->
<svg role="img" aria-labelledby="t" viewBox="0 0 24 24"><title id="t">Carrito</title>…</svg>
```

- En `<img src="icono.svg">` el equivalente es el `alt` del `<img>` (vacío si es decorativo).
- **Nunca** transmitir información **solo** por color dentro de SVG (contraste — ver [[Acceder a Html]] para el resto de la página).
- SVG es texto: el `<text>` interno es seleccionable y buscable por buscadores — ventaja sobre raster.

## Casos de uso en el desarrollo web

- **Iconos de UI** (el 90% del uso real): sprite + `currentColor`.
- **Logos**: escalado perfecto, theming por CSS, sin versiones por densidad.
- **Ilustraciones e ilustraciones interactivas**: hover, cambios de estado por clase CSS.
- **Diagramas, chart y data-viz ligeros**: generados por JS (el DOM del SVG se manipula como cualquier otro — ver el tutorial abajo).
- **Fondos/decoración** como CSS background (con sus límites: sin JS/CSS externo).
- **Gráficos responsivos**: un solo archivo sirve a móvil, desktop y retina.
- **No** es para fotos (peso explosivo) — ahí raster, ver [[Formatos Avif y Webp]].

## Tutorial: SVG reactivo con JavaScript

El ejemplo de esta nota es un avatar cuyas partes (orejas, cabeza, ojos, hocico) **siguen al cursor** del mouse o al dedo en pantallas táctiles. Funciona porque el SVG inline **es DOM**: `querySelector`, estilos y eventos funcionan igual que con HTML.

### 1. Capturar las partes del SVG

Lo primero es seleccionar los elementos por clase, como cualquier HTML:

```js
const eyes = document.querySelector('.eyes');
const head = document.querySelector('.head');
const ears = document.querySelector('.ears');
const nose = document.querySelector('.nose');
const snout = document.querySelector('.snout');
```

### 2. Estado inicial: cursor y tamaño de ventana

```js
// Posición del cursor (se actualiza en cada movimiento)
let cursorPosition = { x: 0, y: 0 };

// Tamaño de la ventana, capturado al iniciar el script
let windowWidth = window.innerWidth;
let windowHeight = window.innerHeight;

// Al redimensionar hay que volver a medir, o los cálculos quedan desfasados
const defWindow = () => {
  windowWidth = window.innerWidth;
  windowHeight = window.innerHeight;
};
```

### 3. Funciones que actualizan la posición del cursor

```js
// Movimiento del mouse: e es el evento que dispara mousemove
const mouseMov = (e) => {
  // clientX/clientY: coordenadas del cursor relativas a la ventana
  cursorPosition = { x: e.clientX, y: e.clientY };
};

// Variante táctil: el primer dedo sobre la pantalla
const touchMode = (e) => {
  cursorPosition = {
    x: e.targetTouches[0].offsetX,
    y: e.targetTouches[0].offsetY,
  };
};
```

> [!warning] El parámetro del handler es el evento
> El error clásico es declarar `function mouseMov(position)` pero usar `e` por dentro: `e` no existe en ese scope y salta un `ReferenceError`. El parámetro y el nombre usado dentro deben coincidir.

### 4. Mover un elemento según el cursor

La idea: medir dónde está el elemento, comparar su centro con la posición del cursor y aplicar un `transform` de desplazamiento:

```js
const seguirCursor = (element, valX, valY) => {
  // Dimensiones y posición del elemento respecto de la ventana:
  // x, y (borde superior izquierdo), width y height
  const elementOffset = element.getBoundingClientRect();

  // Centro exacto del elemento
  const centerX = elementOffset.x + elementOffset.width / 2;
  const centerY = elementOffset.y + elementOffset.height / 2;

  // Distancia cursor-centro, normalizada al tamaño de la ventana (en %)
  const distanceLeft = Math.round(((cursorPosition.x - centerX) * 100) / window.innerWidth);
  const distanceTop = Math.round(((cursorPosition.y - centerY) * 100) / window.innerHeight);

  // valX/valY amortiguan el movimiento: cuanto más grande, menos se mueve
  element.style.transform = `translate(${distanceLeft / valX}px, ${distanceTop / valY}px)`;
};
```

Cada parte del rostro se mueve con su propio factor de amortiguación: las orejas apenas se mueven (`0` = casi fijas), los ojos casi siguen al cursor (`1.8`), y nariz y hocico quedan en el medio:

```js
const seguir = () => {
  if (ears) seguirCursor(ears, 0, 0);
  if (head) seguirCursor(head, 6, 6);
  if (eyes) seguirCursor(eyes, 1.8, 1.8);
  if (nose) seguirCursor(nose, 3, 3);
  if (snout) seguirCursor(snout, 3, 3);
};

seguir(); // una primera posición inicial
```

Los `if (ears)` protegen contra elementos ausentes: `querySelector` retorna `null` si la clase no existe, y llamar un método sobre `null` lanzaría un error.

> [!info] Sobre `transform` en elementos SVG
> El `element.style.transform` con `translate(Xpx, Ypx)` funciona porque los navegadores modernos aplican el modo CSS a los elementos SVG; la sintaxis SVG nativa sería `translate(X,Y)` **sin unidades** (unidades = usuario del viewBox, no píxeles). Elegir un modo y no mezclarlos evita desfasajes.

### 5. Vincular los eventos

Por último se conectan las funciones a los eventos del [[Eventos|DOM]]:

```js
window.addEventListener('resize', defWindow);  // al cambiar el tamaño de la ventana
window.addEventListener('mousemove', mouseMov); // al mover el mouse
window.addEventListener('touchmove', touchMode); // al mover el dedo (móvil)
```

> [!tip] Rendimiento en `mousemove`
> Los handlers de `mousemove`/`scroll` se disparan decenas de veces por segundo. Si el cálculo es pesado, limita la frecuencia con `requestAnimationFrame` o con *throttling*, o la página se sentirá lenta en dispositivos modestos — el patrón exacto está en [[Librería Parallax]].

### Ideas para extender el ejemplo

- Cambiar la opacidad o el color de alguna parte según la distancia al cursor.
- Sumar `deviceorientation` para que el rostro siga la inclinación del celular.
- Animar con CSS las propiedades que no dependen del cursor (parpadeo, respiración), y dejar el `transform` de posición para JavaScript.
- Mover toda la coreografía a GSAP (`gsap.to` + `quickTo` para suavizado) — ver [[Librería GSAP]].

## Errores comunes

1. **SVG que no aparece o aparece tiny**: falta `viewBox`, o `width`/`height` fijos muy chicos. Síntoma clásico: el icono se corta a la esquina.
2. **`viewBox` con dimensiones distintas al dibujo**: si las formas viven en coordenadas 0-100 y el `viewBox` dice `0 0 24 24`, todo queda fuera del lienzo. El `viewBox` debe contener las coordenadas del contenido.
3. **Icono invisible por `fill`**: las formas por defecto se rellenan **negro**; los `<line>` no se ven porque no tienen `stroke`. Y un `fill="none"` heredado deja invisible lo que esperabas relleno.
4. **`rx` sin `ry`** (o al revés): si solo definís uno, el otro lo copia; pero si los dos son 0, no hay redondeo.
5. **CSS de la página que no aplica**: es esperado en `<img>`/background — el SVG es un documento aislado. Para controlarlo, inline o estilos dentro del archivo.
6. **`currentColor` que no hereda**: el `<svg>` está en `<img>` (aislado) o alguien puso `fill` explícito en el path y gana sobre la herencia.
7. **Sprites externos sin CORS/servidos con MIME equivocado**: `<use href="sprite.svg#id">` falla silenciosamente en algunos setups — el sprite inline es el camino seguro.
8. **Metadatos de Figma sin optimizar**: archivos con capas `defs`, nombres raros y KB muertos → SVGO.
9. **Olvidar `alt`/`aria`**: icono informativo sin nombre accesible = inexistente para lectores de pantalla.
10. **Diseñar fotos en SVG**: una foto vectorializada puede pesar 10× más que su JPEG — el SVG es para formas.

## Cheatsheet

| Tema | Regla |
|---|---|
| Vector vs raster | Formas/iconos → SVG; fotos → AVIF/WebP |
| Escalar | Siempre `viewBox`; `width`/`height` los pone CSS |
| Coordenadas | Origen arriba-izquierda; `y` hacia abajo |
| `preserveAspectRatio` | Defecto `xMidYMid meet`; `slice` llena recortando |
| `fill` / `stroke` | Relleno / contorno; ambos `none` para solo línea |
| Iconos | `fill: currentColor` + heredar `color` del contexto |
| Reutilización | Sprite inline: `<symbol id>` + `<use href="#id">` |
| `<img>` vs inline | `<img>` = aislado; inline = CSS/JS/DOM total |
| Formas básicas | `rect`, `circle`, `ellipse`, `line`, `polyline`, `polygon` |
| Trazado libre | `path d="M L H V C S Q T A Z"` (mayúsculas absolutas, minúsculas relativas) |
| Curvas | `C` = cúbica (2 controles); `Q` = cuadrática (1 control) |
| Optimizar | SVGO (`removeViewBox: false`) |
| Animar | CSS/`transform`, GSAP; SMIL solo para piezas aisladas |
| Acceso | Decorativo: `aria-hidden`; informativo: `role="img"` + label |

## Relacionadas

- [[Document Object Model (DOM)]] — el SVG embebido es parte del árbol y se manipula igual
- [[Eventos]] — `mousemove`, `touchmove` y compañía para interactividad
- [[Acceder a Html]] — selección de elementos del SVG
- [[Animaciones]] — CSS sobre SVG (transiciones, keyframes)
- [[Librería GSAP]] — animación avanzada de SVG (`attr`, trazados, morphing)
- [[Librería Parallax]] — capas vectoriales ligeras para escenas de profundidad
- [[Imágenes CSS]] — el SVG como background y su aislamiento
- [[Formatos Avif y Webp]] — cuándo raster gana (fotos)
- [[Material Icons]] — bibliotecas de iconos que usan este patrón sprite
