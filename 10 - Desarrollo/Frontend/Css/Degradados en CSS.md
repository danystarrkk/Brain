---
aliases: ["Degradados Css", "Gradientes CSS", "linear-gradient"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Colores CSS]]"
  - "[[Background]]"
  - "[[Animaciones]]"
  - "[[Sombras CSS]]"
  - "[[Imágenes CSS]]"
  - "[[Medidas Relativas]]"
  - "[[Efectos de Scroll]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/gradient/linear-gradient"
---

# Degradados en CSS

Un **degradado** (*gradient*) es una transición suave entre dos o más colores. En CSS **no es una imagen**: es una función que el navegador **pinta** en tiempo de render. Se declara donde se espera un `<image>` — sobre todo `background-image`, pero también `border-image`, `mask-image`, `list-style-image`… y produce un resultado **vectorial**: escalable, sin petición HTTP y nítido a cualquier DPI.

Las tres funciones degradado (`linear-gradient()`, `radial-gradient()`, `conic-gradient()`) y sus variantes `repeating-*` comparten la misma anatomía:

```css
.degradado {
  background-image: linear-gradient(
    DIRECCIÓN_O_POSICIÓN,   /* parámetro geométrico (opcional) */
    color 0%,                /* paradas de color 1…n */
    color 100%
  );
}
```

1. **La geometría**: dirección (linear), centro y tamaño (radial) o ángulo de partida (conic).
2. **Las paradas de color** (*stops*): color + posición donde empieza/termina.
3. La interpolación entre paradas ocurre automáticamente (por defecto en sRGB).

> [!info] Diferencia clave con una imagen
> Un `background-image: url(...)` descarga bytes, tiene tamaño fijo y peso HTTP. Un degradado se recalcula con cada resize de la caja, no cuesta red y pesa solo los bytes del CSS. A cambio, no se puede editar en un editor de imágenes ni cachear aparte — ver rendimiento más abajo.

## `linear-gradient()`: la función base

```css
background-image: linear-gradient();
```

### Dirección: keywords, esquinas y ángulos

```css
.de-arriba   { background: linear-gradient(to bottom, #000, #fff); }  /* defecto */
.de-abajo    { background: linear-gradient(to top, #000, #fff); }
.horizontal  { background: linear-gradient(to right, #000, #fff); }
.diagonal    { background: linear-gradient(to top right, #000, #fff); }
.por-angulo  { background: linear-gradient(135deg, #000, #fff); }
```

| Forma | Ejemplo | Equivalencia |
|---|---|---|
| `to <lado>` | `to right`, `to bottom` | `90deg`, `180deg` |
| `to <esquina>` | `to top right`, `to bottom left` | calculado según el aspecto |
| Ángulo | `0deg`, `45deg`, `135deg`, `315deg` | entero o decimal, negativos permitidos |

- **`0deg` apunta hacia arriba** y los ángulos **aumentan en sentido horario**: `90deg` = derecha, `180deg` = abajo, `270deg` = izquierda.
- Sin parámetro, el defecto es `to bottom`.
- Los negativos y los >360deg funcionan (`-45deg` = `315deg`).

> [!tip] Mental modelo de la "línea del gradiente"
> El navegador traza una **línea del gradiente** por el centro de la caja, perpendicular a la dirección pedida. El 0% vive en un extremo de esa línea y el 100% en el otro, **dimensionada para que las cuatro esquinas de la caja caigan dentro del rango**. Por eso `to top right` no es "de una esquina a la otra": en una caja ancha la transición real ocurre en una línea inclinada cuyos extremos sobresalen de las esquinas.

### Paradas de color (stops)

Los porcentajes (o longitudes) después de cada color son **paradas**: dicen en qué punto de la transición empieza o termina ese color.

```css
/* 3 colores: el amarillo ocupa el 10% central */
.bandera {
  background: linear-gradient(to right, #c60b1e 0% 45%, #ffc400 45% 55%, #c60b1e 55% 100%);
}
```

Reglas de las paradas:

- **Dos posiciones en la misma parada** (`#ffc400 45% 55%`): el color se mantiene **constante** entre esas dos posiciones — la base de los *hard stops*.
- **Colores pegados** (`#000 50%, #fff 50%`) = corte **duro** sin transición en el 50%.
- **Sin posición**: el navegador reparte uniforme; con dos paradas = `0%` y `100%`.
- **Una sola posición sin parar**: se ubica a mitad de camino entre sus vecinas. `#000 30%, #fff` → el blanco empieza al **65%** ((30+100)/2).
- La primera parada con posición > 0% se **recorta a 0%** (color sólido inicial); la última < 100% se recorta a 100%.
- **Paradas fuera de orden** (`#000 80%, #fff 20%`) = declaración inválida: el `background-image` entero se descarta y la caja queda sin degradado (error silencioso clásico).

Un ejemplo completo usando las custom properties de [[Colores CSS]]:

```css
:root {
  --negro: #212121;
  --blanco: #ffffff;
}

.hero {
  background-image: linear-gradient(to top, var(--negro) 0%, var(--blanco) 100%);
}
```

### Interpolación: dónde vive el "entre"

- Por defecto los colores se interpolean en **sRGB** (espacio de la mayoría de monitores). Con colores muy saturados esto produce **bandas** y zonas percibidas como sucias en el medio.
- La especificación moderna permite pedir interpolación en espacios perceptuales: `linear-gradient(in oklch, red, blue)` — transiciones más naturales y *hue* preservado. Soportado en los motores modernos desde 2023-2024 (Chrome 111+, Safari 16.4+, Firefox 128+).
- La interpolación de **alfa** ocurre en modo *premultiplied*: `transparent` (que es `rgb(0 0 0 / 0)`) ya no produce la franja gris histórica en motores actuales. Aun así, la práctica recomendada con espacios de color avanzados es escribir el color del fin transparente explícitamente: `rgb(255 255 255 / 0)`.

## `repeating-linear-gradient()`: patrones que se repiten

Misma sintaxis, pero el navegador toma el **último tramo** como patrón y lo repite indefinidamente:

```css
.rayas {
  background: repeating-linear-gradient(
    45deg,
    #3498db 0 15px,   /* 15px azul */
    #ecf0f1 15px 30px /* 15px claro → patrón de 30px que se repite */
  );
}
```

- El "paso" del patrón es `última posición − primera posición`. Acá: 30px.
- Para que el patrón sea **continuo**, el color final de la última parada debe coincidir con el inicial de la primera.
- Con paradas en `%`, el paso es relativo al tamaño de la caja (el patrón cambia con el resize; con `px` no).

## `radial-gradient()`: del centro hacia afuera

```css
.bola {
  background: radial-gradient(circle at 30% 30%, #fff, #3498db 70%);
}
```

Sintaxis: `radial-gradient([<forma> || <tamaño>]? [at <posición>]?, <paradas>)`.

### Forma y tamaño

| Parámetro | Valores | Significado |
|---|---|---|
| Forma | `circle` \| `ellipse` (defecto) | Círculo (mismo radio en ambos ejes) o elipse |
| Tamaño | `farthest-corner` (defecto) | Radio hasta la esquina más lejana |
| | `closest-corner` | Hasta la esquina más cercana |
| | `closest-side` | Hasta el lado más cercano |
| | `farthest-side` | Hasta el lado más lejano |
| | `50px` / `50px 80px` | Radios explícitos (2 valores = elipse) |
| Posición | `at center` (defecto), `at 30% 20%`, `at top left`… | Centro del gradiente |

- Con `circle`, **un solo valor** de tamaño es suficiente; con `ellipse` hacen falta dos (o keywords).
- `radial-gradient(circle, …)` con el defecto `farthest-corner` produce una circunferencia que toca las esquinas del rectángulo — el "resplandor" clásico de fondos.

### Repeticiones y patrones radiales

```css
.anillos {
  background: repeating-radial-gradient(circle at center, #2c3e50 0 10px, #34495e 10px 20px);
}
```

Misma lógica que el repeating lineal: el tramo entre la primera y la última parada se repite hacia afuera indefinidamente.

## `conic-gradient()`: rotación alrededor de un punto

El degradado **angular**: los colores giran alrededor del centro, como las secciones de un reloj o una torta.

```css
.reloj {
  background: conic-gradient(from 0deg, #e74c3c, #f1c40f, #2ecc71, #e74c3c);
}
```

- Sintaxis: `conic-gradient([from <ángulo>] [at <posición>]?, <paradas>)`.
- `from 0deg` (defecto) arranca **arriba** y gira **en sentido horario**; `from 90deg` empieza a la derecha.
- Las paradas miden **giro acumulado** (`0`, `25%`, `90deg`, `0.25turn`… todo vale).
- El centro por defecto es `at 50% 50%`.

### Gráfico torta (pie chart) con hard stops

```css
.pie {
  width: 160px;
  aspect-ratio: 1;
  border-radius: 50%;
  background: conic-gradient(
    #e74c3c 0 33%,      /* rojo: un tercio, corte duro */
    #3498db 33% 70%,    /* azul: 37% */
    #2ecc71 70% 100%    /* verde: el resto */
  );
}
```

### Tablero de ajedrez con `repeating-conic-gradient`

```css
.tablero {
  background: repeating-conic-gradient(#000 0% 25%, #fff 0% 50%)
              0 0 / 40px 40px;   /* tamaño del cuadrante vía background-size */
}
```

El truco: `conic` con corte duro al 25% genera **cuadrantes** alternos; `background-size: 40px 40px` convierte cada cuadrante en un damero de 20px.

### `repeating-conic-gradient()`: patrones angulares

```css
.rayos {
  background: repeating-conic-gradient(from 0deg, #f1c40f 0deg 10deg, #2c3e50 10deg 20deg);
}
```

## Múltiples capas

`background-image` acepta **varias** funciones separadas por comas; la **primera capa queda encima** de las demás:

```css
.overlay {
  background-image:
    linear-gradient(to top, rgb(0 0 0 / 0.7), transparent),
    url("hero.avif");
  background-size: cover;
  background-position: center;
}
```

Este es el patrón clásico del "overlay" de texto sobre foto — ver [[Imágenes CSS]] y [[Formatos Avif y Webp]].

- Cada capa tiene su propio `background-size/position/repeat/clip/origin` si listás varios valores (el orden de los valores define a qué capa corresponden).
- Mezclar shorthand `background:` con capas multi-línea es propenso a resetear capas sin querer — mismo cuidado que con los shorthands de [[CSS Grid]]; preferir `background-image:` explícito.

> [!warning] Degradado ≠ `background-color`
> Un gradiente vive en `background-image`. `background-color` solo acepta un color plano y **queda debajo** del gradiente (se ve solo donde el degradado es transparente). El error típico: `background: linear-gradient(...)` funciona porque `background` es shorthand que asigna `background-image`, pero `background-color: linear-gradient(...)` es inválido y no pinta nada.

## Degradados más allá del fondo

### Texto con degradado

```css
.titulo {
  background-image: linear-gradient(90deg, #8e2de2, #4a00e0);
  -webkit-background-clip: text;   /* prefijo histórico aún recomendado */
  background-clip: text;
  color: transparent;
}
```

Funciona con cualquier gradiente (incluso animados). La accesibilidad exige verificar contraste del color **más claro** contra el fondo (ver accesibilidad abajo).

### Borde con degradado

```css
.card {
  border: 2px solid transparent;
  border-radius: 12px;
  background:
    linear-gradient(#fff, #fff) padding-box,
    linear-gradient(135deg, #8e2de2, #4a00e0) border-box;
}
```

Variante legacy (requiere `border-width > 0`): `border-image: linear-gradient(135deg, #8e2de2, #4a00e0) 1;` — pero ignora `border-radius`.

### Máscaras con degradado (fundidos de contenido)

```css
.fade-out {
  mask-image: linear-gradient(to bottom, #000 60%, transparent);
  -webkit-mask-image: linear-gradient(to bottom, #000 60%, transparent);
}
```

El degradado define **dónde el contenido es visible** (negro = opaco, transparente = oculto) — el fundido de bordes de un carrusel o el "carga más" de un feed. Ver [[Efectos de Scroll]] para el patrón completo.

### Gradientes como "imágenes" de listas y bordes

Cualquier sitio que acepte `<image>` acepta degradados: `list-style-image`, `mask`, `border-image`… La pieza clave es que la función devuelve un `<image>`, no un color.

## Animar degradados

Un degradado **no es interpolable por defecto** (cambiar `to right` por `to bottom` = salto seco). Hay dos caminos:

### 1. `@property`: registrar las piezas animables

```css
@property --angulo {
  syntax: "<angle>";
  inherits: false;
  initial-value: 0deg;
}

.aurora {
  --angulo: 0deg;
  background: linear-gradient(var(--angulo), #8e2de2, #4a00e0);
  animation: girar 4s linear infinite;
}

@keyframes girar {
  to { --angulo: 360deg; }
}
```

Al registrar `--angulo` como `<angle>`, el navegador **interpola el ángulo** y repinta el gradiente cada frame (ver [[Animaciones]]). Mismo truco con `<color>` para cruzar paradas de color. `@property` es estándar en todos los motores actuales (Chrome 85+, Safari 16.4+, Firefox 128+).

### 2. Animar `background-position` (universal y gratis)

```css
.brilla {
  background-image: linear-gradient(90deg, #1e3c72, #2a5298, #1e3c72);
  background-size: 200% 100%;
  animation: brillo 3s ease-in-out infinite alternate;
}

@keyframes brillo {
  to { background-position: 100% 0; }
}
```

Sin `@property`, sin repintar el gradiente (solo mover la capa) — funciona hasta en navegadores viejos.

## Colores modernos dentro de los degradados

Ver [[Colores CSS]] para los formatos. Lo que importa acá:

- `oklch()` produce transiciones **más limpias** que `rgb/hsl`: el recorrido entre colores saturados no se ensucia tanto.
- `color-mix(in oklch, var(--primario), white 20%)` genera variantes de un tema sin definir variables extra — ideal para overlays proporcionales.
- Los porcentajes de alfa modernos (`rgb(0 0 0 / 0.7)`) hacen el código de overlays mucho más legible que `rgba()`.

## Recetas prácticas

| Receta | Núcleo |
|---|---|
| Overlay de hero | `linear-gradient(to top, rgb(0 0 0 / .7), transparent)` + foto — ver arriba |
| Glassmorphism | `linear-gradient(rgb(255 255 255 / .25), rgb(255 255 255 / .08))` + `backdrop-filter: blur(12px)` |
| Botón con brillo | Gradiente 200% + `background-position` animado |
| Texto con marca | `background-clip: text` |
| Spinner sin SVG | `conic-gradient(transparent 0 70%, #3498db 70% 100%)` + `border-radius: 50%` + rotar con `transform` |
| Fundido de feed | `mask-image: linear-gradient(...)` |
| "Mesh" (gradiente malla) | 2-3 `radial-gradient` en capas con distintos `at %` y `color-mix()` — no hay `mesh-gradient()` nativo todavía |

```css
/* Glassmorphism completo */
.tarjeta {
  border-radius: 1rem;
  border: 1px solid rgb(255 255 255 / 0.25);
  background: linear-gradient(135deg, rgb(255 255 255 / 0.25), rgb(255 255 255 / 0.08));
  backdrop-filter: blur(12px);
  color: #fff; /* contraste: texto claro sobre fondo oscurecido */
}

/* "Mesh gradient" aproximado con capas radiales */
.mesh {
  background:
    radial-gradient(at 20% 20%, #8e2de2 0, transparent 50%),
    radial-gradient(at 80% 30%, #4a00e0 0, transparent 50%),
    radial-gradient(at 50% 90%, #f857a6 0, transparent 60%),
    #1e1b4b; /* color base bajo las capas */
}
```

## Accesibilidad

- **Contraste del texto sobre degradados**: medí contra la **peor zona** del gradiente (la más clara detrás de texto oscuro, o la más oscura detrás de texto claro), no contra el promedio. La receta overlay oscuro existe justamente para garantizar contraste sobre fotos.
- **`prefers-contrast: more`**: considerá degradados planos (un solo color) para usuarios de alto contraste — media query nativa.
- No uses gradientes como **único** indicador de estado o significado (WCAG 1.4.1): siempre acompañar con texto o icono.
- Los degradados animados caen bajo `prefers-reduced-motion`: desactivar la animación (ver [[Animaciones]]).

## Rendimiento

- **Costo cero de red**: un degradado no descarga nada y se ve nítido a cualquier DPI — más liviano que una imagen de fondo equivalente.
- **Costo de pintado**: la caja se rasteriza cada vez que cambia tamaño o se repinta. Degradados **enormes** (pantalla completa) con **decenas de paradas** en **capas múltiples** sí se sienten en equipos modestos, sobre todo si se combinan con `backdrop-filter`.
- **Animar gradientes con `@property`** repinta cada frame: bien para paneles chicos; con cuidado en secciones full-screen (composición en GPU ayuda, pero el repintado es real).
- `will-change: background-position` puede ayudar al truco de background-position; no abusar.

## Errores comunes

1. **`background-color: linear-gradient(...)`** — inválido; va en `background-image` (o el shorthand `background:`).
2. **Ángulo confundido**: `90deg` es **hacia la derecha**, no hacia abajo (`180deg`). Y `to top` va de oscuro a claro *hacia arriba*.
3. **Paradas fuera de orden** → el gradiente completo desaparece silenciosamente.
4. **`transparent` en interpolaciones no premultiplicadas** (o con `in oklch`) → grisáceo raro: usar `rgb(R G B / 0)` del colorvecino.
5. **Esperar repetición**: `linear-gradient` NO se repite; para patrones usar `repeating-*` o `background-size` chico + `repeat`.
6. **`conic-gradient` sin `from`/`at`**: arranca siempre desde arriba y desde el centro — los "sectores" parecen corridos si esperabas otro origen.
7. **Olvidar `background-size/position` con capas**: la capa de degradado cubre por defecto todo; sin `background-size: cover` la foto de debajo queda mal encajada.
8. **Hard stops borrosos en pantalla chica**: cortes de 1px a escalas distintas generan *anti-aliasing* distinto — con dameros, verificar en mobile.

## Cheatsheet — funciones y parámetros

| Función | Geometría | Defecto geométrico | Uso típico |
|---|---|---|---|
| `linear-gradient()` | dirección: `to <lado/esquina>` \| `<ángulo>` | `to bottom` | Fondos, overlays, bordes |
| `repeating-linear-gradient()` | igual | igual | Rayas, patrones |
| `radial-gradient()` | `[circle\|ellipse] [tamaño] at <posición>` | `ellipse farthest-corner at center` | Resplandores, botones, sombras suaves |
| `repeating-radial-gradient()` | igual | igual | Anillos, dianas |
| `conic-gradient()` | `from <ángulo> at <posición>` | `from 0deg at 50% 50%` | Tortas, ruedas, spinner |
| `repeating-conic-gradient()` | igual | igual | Dameros, rayos |

| Parámetro | Valores | Notas |
|---|---|---|
| Dirección (lineal) | `to top/right/bottom/left`, `to top right`… , `Ndeg` | `0deg` = arriba, horario |
| Forma (radial) | `circle`, `ellipse` | defecto `ellipse` |
| Tamaño (radial) | `farthest-corner` (defecto), `closest-corner`, `closest-side`, `farthest-side`, longitudes | `circle` acepta 1 valor |
| Posición (radial/conic) | `at <lado>\|<%>\|<px>` | defecto `center` |
| Paradas | `<color> [<posición> <posición>?]` | 2 posiciones = tramo constante |
| Espacio de interpolación | `in oklch`, `in oklab`, `in shorter hue`… | moderno, 2023+ |

## Compatibilidad

Los seis tipos de degradado son compatibles con **todos los navegadores actuales** (2026). Datos históricos: `conic-gradient()` llegó en Chrome 69 (2018), Safari 12.1 y Firefox 83 (2020); `@property` en Chrome 85, Safari 16.4 y Firefox 128 (2024); la interpolación `in oklch` en 2023-2024. Los prefijos `-webkit-linear-gradient()` de la era 2011 son **legado eliminable** — la sintaxis estándar no lleva prefijo (y la vieja además usaba ángulos al revés).

## Relacionadas

- [[Colores CSS]] — formatos de color usados en los degradados (`oklch`, `color-mix`, alfa)
- [[Background]] — cómo encaja `background-image` con el resto de propiedades de fondo y sus shorthands
- [[Sombras CSS]] — otra forma de dar profundidad a los elementos
- [[Animaciones]] — `@property` y qué propiedades conviene animar
- [[Imágenes CSS]] — degradados como capa sobre fotos, `cover` y `object-fit`
- [[Efectos de Scroll]] — máscaras con degradado para fundidos de scroll
- [[Medidas Relativas]] — `%` y `px` en las paradas: qué cambia al redimensionar
