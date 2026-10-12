---
tags:
  - desarrollo
  - frontend
  - responsive-design
tipo: concepto
estado: terminado
---

# Responsive Web Design

**Responsive Web Design (RWD)** es la técnica para que una web se adapte a cualquier dispositivo: móviles, tablets, relojes, televisores, monitores. El punto de partida es una realidad: **son los usuarios quienes eligen con qué dispositivo navegan**; no podemos imponerles una resolución ni un navegador. Además, Google penaliza en posicionamiento a los sitios que no son responsive.

La solución se apoya en tres pilares:

1. **Vistas fluidas** — medidas relativas en vez de pixeles fijos (`%`, `rem`, `vw`) — ver [[Medidas CSS]] y [[Medidas Relativas]].
2. **Layouts flexibles** — `flex` y `grid` en vez de flotados rígidos — ver [[Flexbox]] y [[CSS Grid]].
3. **Media queries** — puntos de quiebre donde el CSS se ajusta.

> [!info] De dónde viene todo esto
> El término lo acuñó **Ethan Marcotte en 2010** (artículo *Responsive Web Design* en A List Apart), uniendo tres técnicas que ya existían por separado: layouts fluidos de porcentaje, media queries de CSS3 e imágenes flexibles. La alternativa histórica —múltiples sitios (`m.` + escritorio)— está muerta: un solo HTML adaptable ganó por costo, SEO y mantenimiento.

## Pilar 1: medidas fluidas

Un layout fluido no tiene anchos fijos: todo se expresa como proporción del espacio disponible.

```css
/* ❌ Rígido: se rompe en 320px y sobra en 2560px */
.contenido { width: 1140px; }

/* ✅ Fluidos: escala siempre */
.contenido { max-width: 1140px; width: 100%; padding-inline: 1rem; }
```

Patrones de fluidez, de menor a mayor control:

| Patrón | Ejemplo | Comportamiento |
|---|---|---|
| Porcentaje | `width: 50%` | Proporción fija al padre |
| `max-width` + `width: 100%` | contenedores de página | Escala hasta un techo |
| `min()` / `max()` | `width: min(90%, 600px)` | Techo o suelo en una expresión |
| `clamp()` | `clamp(2rem, 4vw, 3.5rem)` | Valor entre mínimo y máximo, fluido en el medio |
| `calc()` compuesto | `width: calc(100% - 2 * var(--gap))` | Combina unidades |
| Grid auto-fit | `repeat(auto-fit, minmax(260px, 1fr))` | Filas que se reorganizan solas |
| Flex wrap | `flex: 1 1 250px` con `flex-wrap` | Equivalente flex (ver [[Flexbox]]) |

```css
/* Tipografía fluida sin media queries: la escala crece con la pantalla */
h1 { font-size: clamp(1.75rem, 4vw + 0.5rem, 3rem); }
p { font-size: clamp(1rem, 0.9rem + 0.3vw, 1.125rem); }

/* Espaciado que respira */
.seccion { padding-block: clamp(3rem, 8vh, 6rem); }
```

`clamp(min, ideal, max)` es la herramienta fluida por excelencia: si el valor ideal (`4vw`) cae fuera de los límites, se queda en el mínimo o el máximo. Con eso muchas "media queries de estética" dejan de hacer falta.

## Pilar 2: layouts que fluyen

Antes de cortar con un breakpoint, el layout debe reacomodarse solo:

```css
.tarjetas {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(260px, 100%), 1fr));
  gap: 1rem;
}
```

```css
.barra {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 1rem;
}
```

La regla mental: **`grid`/`flex` + `gap` + medidas relativas cubren el 80% de la adaptación; los breakpoints cubren el 20% restante** (cambios de composición, no de tamaño).

## Pilar 3: media queries

Su sintaxis define una condición de pantalla que debe cumplirse para que las reglas apliquen:

![[Pasted image 20250124165031.png]]

Tenemos `media` y, entre paréntesis, la condición. En el ejemplo hay un `min-width`, que define un tamaño mínimo de pantalla: **las reglas se ejecutan cuando la pantalla tenga al menos 768px**.

### Anatomía completa

```css
@media media-type and (media-feature: valor) and (...) { ... }
/*         ↑ opcional              ↑ condición        ↑ se pueden encadenar */
```

| Parte | Valores posibles | Ejemplo |
|---|---|---|
| **Tipo de medio** | `all`, `screen`, `print`, `speech` | `@media print { ... }` |
| **Operadores** | `and`, `or` (coma), `not` | `@media (width >= 768px) and (hover: hover)` |
| **Condición** | ver tabla de features abajo | `(min-width: 768px)` |
| **Sintaxis de rango** (moderna) | `(400px <= width <= 700px)`, `(width >= 768px)` | reemplaza a `min-`/`max-` |

### Features más útiles

| Feature | Valores | Uso real |
|---|---|---|
| `width` / `height` | longitud | el breakpoint clásico (`min-width`, `max-width`, rangos) |
| `orientation` | `portrait` / `landscape` | layouts por orientación (raro; casi siempre basta `width`) |
| `resolution` | `2dppx`, `192dpi` | imágenes nítidas (mejor `srcset`, ver [[Imágenes CSS]]) |
| `hover` | `hover` / `none` | ¿hay cursor real? → guardar `:hover` decorativo |
| `pointer` | `fine` / `coarse` | mouse vs dedo → tamaño de targets táctiles |
| `any-hover` / `any-pointer` | igual | considera **todos** los punteros del dispositivo (tablet con ratón) |
| `prefers-color-scheme` | `light` / `dark` | tema claro/oscuro — ver [[Colores CSS]] |
| `prefers-reduced-motion` | `reduce` / `no-preference` | respetar accesibilidad de movimiento — ver [[Animaciones]] |
| `prefers-contrast` | `more` / `less` / `no-preference` | ajustes de contraste del sistema |
| `forced-colors` | `active` / `none` | modo de alto contraste forzado del SO |
| `update` | `fast` / `slow` | pantallas que no repintan rápido (e-ink) |
| `display-mode` | `standalone`, `browser` | app instalada (PWA) |
| `aspect-ratio` | `16/9` | superficies cinematográficas |

```css
/* Solo dispositivos con cursor real: nada de hover en táctiles */
@media (hover: hover) and (pointer: fine) {
  .tarjeta:hover { transform: translateY(-2px); }
}

/* Tema oscuro */
@media (prefers-color-scheme: dark) {
  :root { --fondo: #0f172a; --texto: #e2e8f0; }
}

/* Movimiento reducido */
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}

/* Impresión: sin chrome, sin quiebres dentro de tarjetas */
@media print {
  .nav, .pie { display: none; }
  .tarjeta { break-inside: avoid; }
}
```

### Breakpoints de referencia

No hay valores mágicos, pero estos son puntos de partida habituales:

| Px     | Dispositivo típico                          |
| ------ | ------------------------------------------- |
| 360–480px  | Teléfono                            |
| 768px  | Tablet                                      |
| 1024–1140px | Laptop y computadora de escritorio     |
| 1400px | Pantallas grandes                           |

> [!tip] Definí los breakpoints desde el contenido
> Los breakpoints salen de **dónde se rompe tu diseño**, no de un modelo de dispositivo específico. No existen "las medidas para iPhone y iPad": mirá tu contenido y elegí el punto donde deje de verse bien. Dos o tres breakpoints bien elegidos cubren el 99% de los casos; diez breakpoints son deuda.

### Mobile-first: escribí primero el móvil

El enfoque recomendado es partir del diseño móvil (una columna, todo visible) e ir **añadiendo** complejidad con `min-width`. Lo opuesto (`max-width`) obliga a declarar el diseño completo y luego "quitar" — más código y más peleas con la cascada.

```css
/* ✅ Mobile-first: lo base + agregos */
@media (min-width: 768px) { ... }

/* ❌ Desktop-first: lo completo + restas */
@media (max-width: 767px) { ... }
```

Por qué mobile-first gana:

| Criterio | Mobile-first (`min-width`) | Desktop-first (`max-width`) |
|---|---|---|
| Código móvil | Base limpia, sin overrides | Base + "quitamos" estilos |
| Cascada | Los agregos ganan por orden natural | Los overrides deben pisar la base |
| Rendimiento móvil | Sirve lo mínimo necesario | Analiza reglas desktop que nunca aplican |
| Prioridad de contenido | Fuerza a decidir qué es esencial | Tiende a conservar todo |

Detalle técnico: con `min-width: 768px` el breakpoint **incluye** 768px; si escribís el siguiente como `min-width: 769px` dejas un hueco de 1px. Convención: 768 / 1024 / 1280 (o 48em / 64em / 80em si medís en `em`).

> [!info] `px` o `em`/`rem` en las media queries
> En las media queries, las unidades relativas se resuelven contra el **tamaño inicial de fuente del navegador** (16px por defecto), **no** contra la fuente del usuario ni la del elemento. Eso significa que usar `rem` en breakpoints **no** los hace más accesibles para quien agrandó el texto — el cambio real de tamaño de texto no mueve los breakpoints de ancho. La elección `px` vs `em` aquí es cosmética; lo que sí importa es que el contenido *dentro* de los breakpoints use `rem` (ver [[Medidas CSS]]).

## Buenas prácticas con media queries

- Reescribí **solo lo que cambia**: no dupliques selectores completos, solo la propiedad afectada.
- Colocá cada media query **justo debajo de la propiedad/regla** que modifica, así se identifican de un vistazo (o agrupalas por tema si tu equipo lo prefiere — lo importante es la convención única).
- Usá `rem`/`%` dentro del contenido donde puedas: respeta el tamaño de fuente que eligió el usuario.
- Evitá breakpoints "por dispositivo": ni `@media (device-width: 768px)` (es el ancho del *dispositivo*, no de la ventana) ni listas de modelos.
- Cuidado con `100vw`: incluye el ancho del scrollbar vertical y produce scroll horizontal fantasma en escritorio. Usá `100%` o `calc(100vw - (100vw - 100%))`.

## Más allá de las media queries: diseño fluido

Antes de escribir un breakpoint, preguntate si necesitas uno. Muchas cosas se resuelven fluidas, sin cortes — ver los patrones de `clamp()` y `auto-fit` de arriba. Otras herramientas del diseño intrínseco:

```css
/* Imágenes que nunca desbordan */
img, video { max-width: 100%; height: auto; }

/* Proporción fija sin saltos de layout */
.miniatura { aspect-ratio: 16 / 9; object-fit: cover; }   /* ver [[Imágenes CSS]] */

/* Columnas de texto óptimas sin breakpoints */
.prosa { max-width: 65ch; }        /* ~65 caracteres por línea */

/* Unidades del viewport dinámico (móvil con barra de URL) */
.hero { min-height: 100dvh; }      /* svh/lvh/dvh, ver [[Medidas Relativas]] */
```

### Container queries: responsive por componente

Las media queries miden la **ventana**; las *container queries* miden el **contenedor del componente**. Un widget se adapta a su espacio real, sirva donde sirva (sidebar, modal, página completa):

```css
.tarjeta {
  container-type: inline-size;   /* crea un query container (contención de inline-size) */
  container-name: tarjeta;       /* opcional: nombrarlo */
}

@container tarjeta (min-width: 400px) {
  .tarjeta .contenido {
    display: grid;
    grid-template-columns: 120px 1fr;
    gap: 1rem;
  }
}

/* Unidades derivadas del container */
@container (min-width: 400px) {
  .titulo { font-size: 2cqi; }   /* 2% del ancho del container */
}
```

| Unidad | Significado |
|---|---|
| `cqi` / `cqb` | 1% del inline-size / block-size del container |
| `cqw` / `cqh` | 1% del ancho / alto del container |
| `cqmin` / `cqmax` | 1% de la dimensión menor / mayor |

Gotchas reales:

- `container-type: inline-size` aplica **contención de inline-size**: el ancho del contenido no puede influir en el ancho del contenedor. Con `container-type: size` además se contienen la altura y la anchura mínima — el contenido ya no define la caja, por eso casi nadie lo usa.
- El container más cercano que matchee el nombre (o el más cercano si no hay nombre) es el que resuelve la query.
- Los containers no se anuncian: si el componente se apila en móvil, verificá que el `@container` también cubra ese caso.

Soporte: Chrome 105+, Safari 16+, Firefox 110+ (2022–2023) — ya usable en proyectos actuales.

## Viewport: la pieza que lo habilita todo

Sin el meta viewport, **nada de lo anterior funciona en móvil**:

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

| Atributo | Efecto | Recomendación |
|---|---|---|
| `width=device-width` | El ancho del layout = el CSS px del dispositivo | Siempre |
| `initial-scale=1` | Sin zoom inicial | Siempre |
| `maximum-scale=1` / `user-scalable=no` | **Bloquea el zoom del usuario** | ❌ Prohibido por accesibilidad (WCAG 1.4.4) |
| `viewport-fit=cover` | Extiende el layout bajo notch/islas | Solo con `env(safe-area-inset-*)` |
| `initial-scale` + `interactive-widget=resizes-content` | Cómo afecta el teclado virtual | Widgets/apps de formulario |

Con `viewport-fit=cover` hay que respetar las áreas seguras del sistema:

```css
.boton-flotante {
  bottom: max(1rem, env(safe-area-inset-bottom));
}
```

> [!warning] Síntoma del viewport ausente
> Sin el meta, los móviles simulan una pantalla de ~980px y muestran la versión "escritorio" reducida: las media queries de `min-width` aplican todas a la vez, los textos salen diminutos. Si "el responsive no funciona pero en escritorio sí", primero mirá el `<head>` — ver [[Html]].

## Flujo de trabajo y testing

1. **Diseñar en el navegador, no en un modelo fijo**: DevTools → modo dispositivo (Ctrl+Shift+M) con anchos arbitrarios.
2. **Probar de menor a mayor**: 320 → 375 → 768 → 1024 → 1440 → 2560px.
3. **Probar con texto grande**: DevTools → Rendering → text zoom, o `rem` + zoom del navegador al 200%.
4. **Probar container queries** con el resize del contenedor, no solo de la ventana.
5. **Probar real**: un dispositivo físico o al menos emulador con gestos reales — el hover táctil y el scroll-bounce no se emulan bien en escritorio.

## Errores comunes

1. **Anchos fijos** (`width: 960px`): la causa #1 de scroll horizontal. Fix: `max-width` + `width: 100%`.
2. **Breakpoints por dispositivo** en vez de por contenido: proliferación de cortes arbitrarios. Fix: cortar donde se rompe *tu* diseño.
3. **Ocultar en vez de adaptar**: `display: none` masivo en móvil es una señal de que el layout no está pensado para fluir. El contenido esencial debe estar en todos los tamaños (WCAG reflow).
4. **Olvidar el meta viewport** — ver la advertencia de arriba.
5. **Desktop-first heredado con huecos**: `max-width: 767px` + `min-width: 1024px` deja sin estilos entre 768 y 1023. Fix: mobile-first o rangos explícitos `(400px <= width <= 700px)`.
6. **Imágenes sin limitar**: una imagen de 1200px dentro de un contenedor de 320 desborda el layout. Fix: `img { max-width: 100%; height: auto }` + `srcset`/`sizes` — ver [[Imágenes CSS]].
7. **Contenido ancho no fluido**: tablas, bloques de código e iframes fijos rompen el móvil. Fix: `overflow-x: auto` en el bloque, `min-width: 0` en contenedores flex/grid (ver [[Flexbox]]), y tablas envueltas en un scroller accesible.
8. **`100vw` + scrollbar**: scroll horizontal fantasma en escritorio. Fix: `100%`.
9. **Hover-only en táctil**: funciones escondidas tras `:hover` son invisibles en móvil. Fix: `@media (hover: hover)` + fallback táctil.

## Accesibilidad

- **Reflow (WCAG 1.4.10)**: el contenido debe funcionar a 320 CSS px de ancho (equivale a ~400% de zoom en 1280) **sin** scroll horizontal bidireccional. Es el requisito que transforma RWD en accesibilidad real.
- **Zoom de texto 200% (1.4.4)**: por eso todo en `rem`, nunca todo en `px`.
- **Targets táctiles (2.5.8, antes 2.5.5)**: mínimo 24×24px, recomendado 44×44; `gap` generoso entre botones (ver [[Flexbox]]).
- **`prefers-reduced-motion`**: los "micro" efectos responsive deben respetarse; ver [[Animaciones]].
- **Nada crítico solo con hover**: en `pointer: coarse` la información debe estar visible o alcanzable con foco/tap.
- **Modo forzado de colores**: `forced-colors: active` elimina sombras y muchos fondos; no diseñes jerarquía solo con color — ver [[Sombras CSS]].
- **Áreas seguras**: con `viewport-fit=cover`, los controles fijos deben usar `env(safe-area-inset-*)` o quedan bajo el notch (ver [[Tipos de Posiciones]]).

## Rendimiento

- Las media queries **no cuestan**: se evalúan en el match de estilos; tener 30 breakpoints no es un problema. El costo real está en las **imágenes y fuentes**.
- Serví imágenes adaptadas: `srcset` + `sizes` (o `<picture>`) evita descargar un hero de 2000px en un teléfono — ver [[Imágenes CSS]].
- Fuentes variables cubren muchos pesos sin descargas extra — ver [[Fuentes Externas]].
- `content-visibility` y `loading="lazy"` complementan el responsive en páginas largas.
- Evitá `vh` clásico en móvil: salta con la barra de URL; usá `dvh` (ver [[Medidas Relativas]]).

## Cheatsheet

| Herramienta | Sintaxis / valor | Mide | Default |
|---|---|---|---|
| Media query clásica | `@media (min-width: 768px)` | ventana | — |
| Media query en rangos | `@media (width >= 768px)` | ventana | — |
| Container query | `@container (min-width: 400px)` | contenedor | — |
| Declarar container | `container-type: inline-size \| size` | — | `normal` |
| Unidades de container | `cqi cqb cqw cqh cqmin cqmax` | container | — |
| Unidades de viewport | `vw vh svw svh lvw lvh dvw dvh` | ventana / viewport estable / dinámico | — |
| Valor fluido | `clamp(min, ideal, max)` | — | — |
| Grid fluida | `repeat(auto-fit, minmax(min(260px,100%), 1fr))` | — | — |
| Meta viewport | `width=device-width, initial-scale=1` | — | simulado ~980px |
| Área segura | `env(safe-area-inset-*)` | dispositivo | `0px` |
| Feature de input | `hover`, `pointer`, `any-hover`, `any-pointer` | punteros | — |
| Preferencia del sistema | `prefers-color-scheme/-motion/-contrast`, `forced-colors` | usuario | según SO |

## Relacionadas

- [[Medidas Relativas]] — `vh`, `vw`, `%`, `dvh` y `clamp()` en detalle
- [[Medidas CSS]] — `px` vs `em` vs `rem` en un diseño adaptable
- [[Flexbox]] — layouts unidimensionales que fluyen
- [[CSS Grid]] — layouts bidimensionales con áreas responsivas y `auto-fit`
- [[Imágenes CSS]] — `srcset`, `sizes`, `object-fit` y proporciones
- [[Html]] — el meta viewport y la estructura semántica adaptable
- [[Tipos de Posiciones]] — elementos fijos/sticky y el problema de la barra de URL móvil
- [[Animaciones]] — `prefers-reduced-motion` en transiciones responsivas
- [[Colores CSS]] — `prefers-color-scheme` y tokens de tema
