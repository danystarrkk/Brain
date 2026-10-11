---
aliases: ["Efecto de Scroll", "Scroll snap", "scroll-snap"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Animaciones]]"
  - "[[Responsive Web Design]]"
  - "[[Tipos de Posiciones]]"
  - "[[Degradados en CSS]]"
  - "[[Box Model]]"
  - "[[Flexbox]]"
  - "[[CSS Grid]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/CSS_scroll_snap_module"
---

# Efectos de Scroll

El navegador pasó de "la página scrollea y punto" a exponer **todo el comportamiento de scroll en CSS**: encajar en puntos (snap), desplazamiento suave, barras de progreso ligadas al scroll, sticky, contención de scroll y más — **sin JavaScript** en la mayoría de los casos. Esta nota cubre el arsenal completo, con el **scroll snap** como pieza central.

| Efecto | Propiedad principal | Para qué |
|---|---|---|
| Encaje del scroll | `scroll-snap-*` | Secciones/carruseles que "encajan" |
| Scroll suave | `scroll-behavior: smooth` | Navegación por anclas animada |
| Animaciones ligadas al scroll | `animation-timeline: scroll()/view()` | Progreso, parallax, reveals |
| Elemento pegajoso | `position: sticky` | Headers, columnas de tabla |
| Contención | `overscroll-behavior` | Evitar scroll-chaining en modales/listas |
| Barras de scroll | `scrollbar-color`, `scrollbar-width` | Estilizar sin hacks |
| Estabilidad de layout | `scrollbar-gutter: stable` | Evitar saltos de layout |

## Base: el contexto de scroll (`overflow`)

Nada de lo anterior funciona sin entender **qué elemento es el contenedor de scroll**: el que tiene un `overflow` que crea *scroll container*.

```css
.panel {
  overflow-y: auto;   /* scroll vertical si el contenido no cabe */
}
```

| Valor | Comportamiento | Crea scroll container |
|---|---|---|
| `visible` (defecto) | Desborda visible | No |
| `hidden` | Recorta, **pero se puede scrollear por teclado/JS** | Sí |
| `clip` | Recorta de verdad, sin scroll posible | No |
| `auto` | Scroll según haga falta | Sí |
| `scroll` | Scroll siempre visible (aunque no haga falta) | Sí |

- **Regla de mezcla**: si en un eje ponés `visible` y en el otro algo que no sea `visible`/`clip`, el `visible` **se computa como `auto`**. Por eso `overflow-x: hidden` en `body` convierte el eje vertical en scrolleable implícito.
- `clip` (vs `hidden`) es el recorte "honesto": el contenido no scrolleable, útil para efectos decorativos.
- `touch-action: pan-y pinch-zoom` controla qué gestos táctiles consume el navegador en esa zona (y cuáles llegan al JS).

> [!tip] El scroll de la página
> El scroll principal de la página vive en el elemento raíz (`html`). Las propiedades de scroll de la ventana se declaran en `html` (o `body` donde la cascada lo permita): `html { scroll-behavior: smooth; scroll-snap-type: y mandatory; }`.

## Scroll suave con `scroll-behavior`

```css
html {
  scroll-behavior: smooth;
}
```

- Convierte en animados los desplazamientos por **anclas** (`<a href="#seccion">`), `window.scrollTo()` y `scrollIntoView()` sin comportamiento explícito.
- Un `scrollIntoView({ behavior: "instant" })` o un enlace con `style="scroll-behavior: auto"` lo salta a nivel individual.
- Es la línea mínima que hace que toda la navegación interna se sienta "de app".

```css
@media (prefers-reduced-motion: reduce) {
  html { scroll-behavior: auto; } /* algunos usuarios encuentran el scroll suave desorientador */
}
```

## Scroll snap: el encaje del desplazamiento

El **scroll snap** permite que al bajar o deslizar, el navegador "encaje" automáticamente en puntos definidos — como las pantallas de una app móvil. Es CSS puro, sin JavaScript.

### Las dos piezas

1. En el **contenedor** (el que scrollea): qué eje snap y con qué fuerza.
2. En cada **sección** (hijos): dónde se engancha.

#### 1. El contenedor

```css
html {
  scroll-snap-type: y mandatory;
}
```

| Parte | Valores | Significado |
|---|---|---|
| Eje | `x`, `y`, `block`, `inline`, `both` | Dónde ocurre el encaje |
| Fuerza | `mandatory` | **Obliga** a terminar siempre en un punto de snap (ideal para landing pages tipo presentación) |
| | `proximity` | Solo encaja si el punto está **cerca** — suave, menos "forzado" |
| | `none` (defecto) | Sin snap |

> [!warning] `mandatory` en contenedores impredecibles
> Con contenido de altura variable (FAQs, listas), `mandatory` puede atrapar al usuario en una sección cuyo contenido no cabe en pantalla: el navegador insiste en encajar y el contenido que sobra queda inaccesible. Para scrolls de lectura, preferí `proximity`.

#### 2. Cada sección

```css
.servicios,
.navegacion-principal {
  scroll-snap-align: center;
  scroll-snap-stop: always;
}
```

| Propiedad | Valores | Significado |
|---|---|---|
| `scroll-snap-align` | `start` \| `center` \| `end` | Qué parte de la sección se engancha al borde del scrollport |
| `scroll-snap-stop` | `normal` (defecto) \| `always` | `always` impide que un gesto rápido **salte dos secciones** de una |

```css
/* Variante común: encajar al inicio de cada sección */
.seccion {
  scroll-snap-align: start;
}
```

### Los ajustes finos: `scroll-margin` y `scroll-padding`

Los dos se usan cuando **otra cosa** (header fijo, padding) hace que el encaje "crudo" quede mal:

```css
/* CONTENEDOR: recorta el área de snap → como si el viewport tuviera padding */
html {
  scroll-snap-type: y mandatory;
  scroll-padding-top: 72px;   /* encaja dejando libre el header fijo de 72px */
}

/* SECCIÓN: aleja su punto de snap de su borde real */
.seccion {
  scroll-snap-align: start;
  scroll-margin-top: 24px;    /* respira 24px arriba del texto */
}
```

- `scroll-padding` (del contenedor) responde *"¿a qué zona del viewport se alinean los puntos?"*.
- `scroll-margin` (de la sección) responde *"¿desde qué punto exacto de esta caja se alinea?"*.
- El resultado práctico de ambos: `elemento.getBoundingClientRect().top + scrollY − scrollPaddingTop + scrollMarginTop`.
- Vale también para los enlaces ancla: el navegador usa el mismo `scroll-margin` para posicionar el destino bajo headers fijos — un solo ajuste para snap **y** anclas.

### Landing page a pantalla completa

```css
html {
  scroll-snap-type: y mandatory;
  scroll-padding-top: 0;
}

.pantalla {
  height: 100vh;              /* o 100dvh en móvil, ver Medidas Relativas */
  scroll-snap-align: start;
  display: grid;
  place-items: center;        /* contenido centrado en cada sección */
}
```

### Carrusel horizontal sin JS

```css
.carrusel {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  gap: 1rem;
  scrollbar-width: none;      /* oculta la barra (accesibilidad: ver abajo) */
}

.carrusel .card {
  flex: 0 0 80%;      /* tarjeta fija, visiblemente sobresale la siguiente */
  scroll-snap-align: center;
}
```

- Para que las secciones snappeen, **necesitan tamaño definido en el eje del snap**: un hijo `flex: 1` sin base no tiene puntos distintos.
- Un `scroll-snap-align: start` + `scroll-padding-inline` en el carrusel deja las tarjetas alineadas al borde con "peek" de la siguiente.
- El mismo patrón funciona con [[CSS Grid]] (`grid-auto-flow: column` + `grid-auto-columns: 80%`).

### Snap + navegación suave

Combinado con `scroll-behavior`, los enlaces internos también encajan animadamente:

```css
html {
  scroll-behavior: smooth;
}

nav a {
  scroll-snap-align: start;
}
```

El navegador **recalcula el snap point más cercano** después de cualquier scroll programático: no hay que "lanzar" el snap a mano.

## `position: sticky` dentro de zonas de scroll

El tercer efecto de scroll clásico: elementos que quedan pegados mientras dura su contenedor — ver [[Tipos de Posiciones]] para la propiedad completa.

```css
.encabezado-tabla {
  position: sticky;
  top: 0;
  z-index: 1;
  background: #fff;
}
```

> [!warning] El asesino del sticky: un ancestro con `overflow`
> `position: sticky` **no funciona** si cualquier ancestro entre el elemento y el scrollport tiene `overflow: hidden/auto/clip` — el "viewport de sticky" se convierte en ese ancestro (que rara vez scrollea). Es el 90% de los "sticky roto". diagnóstico: subir por el árbol en DevTools hasta encontrar el `overflow` culprit.

## `overscroll-behavior`: contener el scroll

Cuando el usuario llega al final de una lista interna, por defecto el scroll "se encadena" al contenedor padre (y al final al rubber-band del navegador). `overscroll-behavior` corta esa cadena:

```css
.modal__cuerpo {
  overflow-y: auto;
  overscroll-behavior: contain; /* no escarba el fondo de la página al llegar al final */
}
```

| Valor | Efecto |
|---|---|
| `auto` (defecto) | Cadena de scroll normal |
| `none` | Ni encadena ni efectos de borde (rubber band) |
| `contain` | El scroll no se propaga al padre, pero el comportamiento de borde del navegador se mantiene |

También acepta formas lógicas (`overscroll-behavior-inline`).

## Barras de scroll y estabilidad de layout

```css
:root {
  scrollbar-gutter: stable;                        /* reserva el espacio de la barra */
  scrollbar-color: #888 #f1f1f1;                   /* estándar: pulgar y riel */
  scrollbar-width: thin;                           /* auto | thin | none */
}
```

- `scrollbar-color`/`scrollbar-width` son **estándar** y ya funcionan en los motores modernos (largo tiempo en Firefox; Chrome/Safari desde 2024).
- El legacy `-webkit-scrollbar` (Chrome/Safari viejos) ofrece más piezas (`::-webkit-scrollbar-thumb`, `-track`, `-width`) pero es no-estándar y no tematizable por media query.
- `scrollbar-gutter: stable` evita el **salto de layout** cuando aparece/desaparece la barra (útil en modales y `overflow-y: auto` condicional).
- Ocultar barras (`scrollbar-width: none`) ahorra píxeles pero **esconde el affordance**: hacerlo solo en carruseles donde la siguiente tarjeta visible comunica el desplazamiento.

## Anclaje de scroll: evitar saltos de contenido

Cuando imágenes o fuentes cargan **encima** de la posición actual, el contenido salta. El *scroll anchoring* del navegador compensa, y puede controlarse:

```css
.scroll-area {
  overflow-y: auto;
  overflow-anchor: auto;  /* defecto: compensar saltos; none lo desactiva */
}
```

- Firefox lo puso por defecto en 2018; Chrome lo reactivó por defecto desde 2024 (M129) tras un periodo desactivado.
- Convivir con JS que scrollea a posición exacta: a veces conviene `overflow-anchor: none` en ese contenedor.

## Animaciones ligadas al scroll (sin JS)

El módulo *scroll-driven animations* permite que una animación la **gobierne el progreso del scroll** en vez del reloj:

```css
/* barra de progreso de lectura, sin una línea de JavaScript */
.progreso {
  position: fixed;
  top: 0;
  left: 0;
  height: 4px;
  width: 100%;
  transform-origin: 0 50%;
  background: #8e2de2;
}

@supports (animation-timeline: scroll()) {
  .progreso {
    animation: crecer linear;
    animation-timeline: scroll(root block); /* progreso del scroll de la página */
  }
}

@keyframes crecer {
  from { transform: scaleX(0); }
  to   { transform: scaleX(1); }
}
```

Y con `view()`, animaciones por **visibilidad de un elemento** — el "reveal" de secciones sin observers:

```css
.seccion {
  animation: aparecer linear;
  animation-timeline: view();
  animation-range: entry 10% cover 25%; /* entra → se revela antes de la mitad */
}

@keyframes aparecer {
  from { opacity: 0; transform: translateY(24px); }
  to   { opacity: 1; transform: none; }
}
```

- Ejes y rangos: `scroll(<eje>)` con `block/inline/x/y`; rangos de view: `entry`, `exit`, `cover`, `contain` (con `%`); `view-timeline-inset` ajusta el "viewport efectivo".
- **Por qué es mejor que un listener de JS**: el progreso lo consume el compositor; no hay trabajo en el hilo principal por cada frame, y la animación "se detiene sola" fuera de pantalla. Es la razón por la que el scroll con JS puro se siente envarado en páginas largas — ver rendimiento y [[Animaciones]].
- **Compatibilidad**: Chromium 115+ (2023) y posteriormente el resto de motores. Siempre con guard `@supports (animation-timeline: …)`: nunca hacer que la **información** dependa de la animación (la barra de progreso puede ser decorativa; el contenido, no).

```css
@media (prefers-reduced-motion: reduce) {
  .seccion { animation: none; }
}
```

## Accesibilidad y UX

- **No elimines el scroll libre** en contenido informativo: `mandatory` en páginas normales frustra a usuarios con teclado o lector de pantalla (pueden quedar "prisioneros" de una sección).
- Respetá `prefers-reduced-motion`: para algunos usuarios el snap y el scroll suave son desorientadores — ver [[Animaciones]] para el patrón de media query.
- Teclado: los scrolls por teclado (PageDown, flechas, Home/End) **respetan el snap**, pero con `mandatory` pueden sentirse "elásticos"; probar con teclado siempre.
- Probar en móvil: el snap se siente natural con touch, pero puede ser brusco con rueda de ratón (cada notch del mouse = un salto grande); `proximity` mitiga.
- Ocultar la barra de scroll solo donde el contexto comunica el desbordamiento (carrusel con "peek"); nunca en el scroll principal.
- Los elementos sticky deben mantener contraste y no tapar contenido al quedar fijos (background opaco, ver [[Box Model]] para el espacio que ocupan).

## Rendimiento

- **Efectos CSS vs listeners de JS**: todo lo de esta nota tiene implementación nativa. Un `addEventListener("scroll", …)` en el hilo principal con trabajo por evento (layout, medición) es la receta del scroll envarado; si hace falta JS, usar `passive: true` y rAF-throttle.
- **Animaciones de scroll**: `animation-timeline` va por el compositor; `position: sticky` reubica capas (barato); `backdrop-filter`/`blur` sobre zonas scrolleables sí cuesta pintado.
- **Páginas largas**: `content-visibility: auto` + `contain-intrinsic-size` en secciones fuera de pantalla le dice al navegador que no calcule nada hasta que se acerquen — el "lazy rendering" nativo.
- Snap, sticky y smooth **no causan reflows por sí solos**: son alineación y desplazamiento, no cambio de tamaño de cajas.

## Errores comunes

1. **`mandatory` atrapa al usuario** en contenido de altura variable — usar `proximity` en scrolls de lectura.
2. **Secciones sin tamaño definido** en el eje del snap: no hay puntos distintos y el snap no hace nada.
3. **Sticky roto por un `overflow` ancestor** — ver la advertencia de la sección sticky.
4. **Confundir `scroll-margin` con `scroll-padding`**: el primero es de la sección (su punto de enganche), el segundo del contenedor (el área efectiva del viewport). Header fijo = `scroll-padding-top` en el contenedor.
5. **`overflow-x: hidden` en `body`** convierte el eje vertical en scrolleable implícito y puede quebrar `sticky`/`100vh` — revisar qué elemento es realmente el scrollport.
6. **Ocultar la barra y perder el affordance** del scroll principal.
7. **Esperar snap tras `scrollTo` con destino arbitrario**: el navegador encaja al punto más cercano tras el salto; para posiciones "libres" usar snap `proximity` o quitarlo de ese contenedor.
8. **Animaciones de scroll sin `@supports`**: en motores sin soporte la animación queda estática en `from` (invisible). Siempre doble guarda: `@supports` + el contenido legible sin animación.

## Cheatsheet — propiedades de scroll

| Propiedad | Dónde | Valores | Defecto |
|---|---|---|---|
| `overflow` / `overflow-x/y` | cualquier caja | `visible`, `hidden`, `clip`, `auto`, `scroll` | `visible` |
| `overscroll-behavior` | contenedor scrolleable | `auto`, `none`, `contain` | `auto` |
| `scroll-behavior` | contenedor (o `html`) | `auto`, `smooth` | `auto` |
| `scroll-snap-type` | contenedor de snap | `<eje> mandatory\|proximity\|none` | `none` |
| `scroll-snap-align` | sección | `start`, `center`, `end` | `none` |
| `scroll-snap-stop` | sección | `normal`, `always` | `normal` |
| `scroll-padding` | contenedor | longitudes / `%` | `0` |
| `scroll-margin` | sección | longitudes / `%` | `0` |
| `scrollbar-color` | contenedor | `<pulgar> <riel>`, `auto` | `auto` |
| `scrollbar-width` | contenedor | `auto`, `thin`, `none` | `auto` |
| `scrollbar-gutter` | contenedor | `auto`, `stable`, `stable both-edges` | `auto` |
| `overflow-anchor` | contenedor | `auto`, `none` | `auto` |
| `animation-timeline` | elemento animado | `auto`, `scroll()`, `view()` | `auto` |
| `view-timeline` / `view-timeline-inset` | elemento observado | nombre + inset | `none` / `auto` |

## Compatibilidad

Scroll snap, sticky, smooth, overscroll y las barras estándar son **totalmente compatibles** con los navegadores actuales (2026). El módulo de animaciones ligadas al scroll (`animation-timeline`) llegó primero a Chromium (115, 2023) y luego al resto de motores — proteger siempre con `@supports`, que es exactamente como se escribe el feature de forma progresiva.

## Relacionadas

- [[Animaciones]] — keyframes, `@property` y `prefers-reduced-motion`
- [[Tipos de Posiciones]] — `sticky` en profundidad y el resto de posicionamientos
- [[Responsive Web Design]] — secciones full-screen y carruseles adaptables
- [[Degradados en CSS]] — máscaras con degradado para fundidos de contenido
- [[Flexbox]] — el layout de los carruseles horizontales
- [[CSS Grid]] — galerías con `auto-fit` que a veces combinan con snap
- [[Box Model]] — por qué un sticky necesita background y espacio propio
