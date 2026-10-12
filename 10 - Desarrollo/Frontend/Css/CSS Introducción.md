---
tags:
  - desarrollo
  - frontend
  - css-introduccion
tipo: concepto
estado: terminado
---

# CSS Introducción

**CSS** (*Cascading Style Sheets*, hojas de estilo en cascada) es el lenguaje encargado del **aspecto visual** de una página: colores, tipografía, espacios, layouts, animaciones y adaptación a distintos dispositivos. Mientras [[Html]] se ocupa de la **estructura y el significado** del contenido (qué es cada cosa), CSS se ocupa de cómo **se ve** (cómo se ve cada cosa). La tercera pata del trío es JavaScript, que agrega **comportamiento**.

> [!info] Un poco de historia
> CSS fue propuesto por Håkon Wium Lie en 1994 (trabajando con Tim Berners-Lee en el CERN) y co-diseñado con Bert Bos; la especificación 1.0 salió en 1996. El problema que resolvía: el HTML de la época mezclaba estructura y presentación (`<font>`, `align=`, `<center>`), y cada navegador inventaba sus propias etiquetas. Separar "qué" de "cómo" fue la decisión que hizo posible la web moderna — y por eso el HTML semántico actual sigue siendo la base sobre la que CSS trabaja bien.

Concretamente, CSS controla:

- Tamaños y tipos de fuentes, interlineado, espaciado
- Colores (texto, fondos, bordes) — ver [[Colores CSS]]
- Espacios, márgenes y el [[Box Model|modelo de caja]]
- Posicionamiento y layouts ([[Flexbox]], [[CSS Grid]], [[Tipos de Posiciones]])
- Adaptación a distintos dispositivos ([[Responsive Web Design]])
- Animaciones y transiciones ([[Animaciones]])
- Incluso cosas que parecen de JS hoy: menús, tooltips y modales nativos con `:hover`, `:focus-within`, `dialog` y `:has()`.

## Anatomía mínima de una regla

La sintaxis no es complicada, aunque al principio pueda resultar extraña:

```css
p {
  color: blue;
}
```

Un **selector** (`p`), un bloque entre llaves y **declaraciones** `propiedad: valor;` separadas por punto y coma. Variantes mínimas:

```css
/* varias propiedades */
p { color: blue; line-height: 1.6; }

/* varias reglas juntas */
h1, h2 { font-weight: 700; }

/* comentarios */
/* esto se ignora */
```

La estructura completa — pseudo-clases, pseudo-elementos, unidades, anidamiento nativo — se detalla en [[Anatomía de CSS]]; para apuntar a los elementos correctos, [[Selectores en CSS]].

## Cómo llega el CSS al navegador

Hay cuatro formas de entregar estilos, de mejor a peor en la práctica:

| Método | Ejemplo | Cuándo usarlo |
|---|---|---|
| **Externo** (recomendado) | `<link rel="stylesheet" href="css/styles.css">` | Siempre; cacheable, mantenible, paralelizable |
| **Interno** | `<style> h1 { color: teal; } </style>` | Estilos críticos de una sola página, CSS generado por JS |
| **En línea** | `<p style="color: gray;">` | Ajustes puntuales o valores calculados por JS |
| **`@import`** | `@import url("otra.css");` al inicio del CSS | Evitar: serializa la descarga (ver abajo) |

```html
<head>
  <link href="css/styles.css" rel="stylesheet"> <!-- externo -->
  <style> h1 { color: teal; } </style>          <!-- interno -->
</head>
<body>
  <p style="color: gray;">En línea</p>
</body>
```

- El atributo `style=""` tiene **prioridad de cascada máxima** (por encima de cualquier selector salvo `!important` ajeno) — por eso "casi imposible de mantener": ver [[Especificidad]].
- `@import` debe ser la **primera** regla del archivo y hace que el navegador espere a descargar ese CSS antes de pedir el siguiente. La alternativa moderna es cargar ambos con `<link>` en paralelo, o precargar con `rel="preload" as="style"` — ver [[Preload]].
- El CSS crítico (el de "arriba de la página") a veces se inlinea en el HTML para pintar antes, y el resto se carga como archivo — patrón *critical CSS*.
- El atributo `media` del `<link>` condiciona **dónde aplica** la hoja: `media="print"` solo al imprimir (hojas de estilo de impresión), `media="(min-width: 768px)"` solo si la condición se cumple — ver [[Responsive Web Design]].
- Para precargar sin bloquear el render: `<link rel="preload" href="styles.css" as="style" onload="this.rel='stylesheet'">` — ver [[Preload]].

```css
/* hoja de impresión: el CSS que nadie escribe y todos necesitan */
@media print {
  nav, footer, .anuncios { display: none; }
  body { color: #000; background: #fff; font-size: 12pt; }
  a[href^="http"]::after { content: " (" attr(href) ")"; }
}
```

## ¿Por qué "en cascada"?

Se llama así porque cuando **varias reglas compiten** por el mismo elemento y propiedad, el navegador aplica un orden de resolución — la *cascade*:

1. Primero filtra qué reglas **aplican** (origen e importancia).
2. Luego resuelve conflictos por **[[Especificidad|especificidad]]** del selector.
3. A igualdad, por **orden en el archivo** (gana la última declarada).

```css
p {
  color: blue;
}

p {
  color: red; /* gana: mismo selector y misma especificidad, pero aparece después */
}
```

> [!info] Cascada ≠ "gana siempre el último"
> La última regla solo gana cuando la especificidad es la misma. `#titulo { color: red }` posterior a `p { color: blue }` no pinta de azul los párrafos con ese id: los selectores más específicos dominan — ver [[Especificidad]] para la cuenta completa.

### Los orígenes de una hoja de estilo (orden completo)

No todo el CSS vive en "nuestro archivo": el navegador trae su propio CSS, el usuario puede imponer los suyos, y las animaciones y transiciones se intercalan. De **menos a más** prioridad:

| # | Estilo | Quién lo define |
|---|---|---|
| 1 | CSS normal del **user agent** | El navegador (`user-agent.css`) |
| 2 | CSS normal del **usuario** | Preferencias del usuario (fuentes grandes, alto contraste) |
| 3 | CSS normal del **autor** | Tu CSS |
| 4 | **Animaciones** | `@keyframes` en curso |
| 5 | `!important` del **autor** | Tu CSS con `!important` |
| 6 | `!important` del **usuario** | CSS de accesibilidad del usuario |
| 7 | `!important` del **user agent** | El navegador |
| 8 | **Transiciones** | Propiedades en transición (ganan todo temporalmente) |

Detalle clave: `!important` **invierte el orden por origen** (dentro del mundo important, el autor le gana al usuario normal, pero el usuario important le gana al autor important — es lo que permite que un usuario con necesidades de accesibilidad pueda forzar estilos por encima del sitio).

### Cascade layers: orden explícito con `@layer`

La capa más moderna de la cascada deja **declarar el orden a mano**, por encima de la especificidad:

```css
@layer base, componentes, utilidades;

@layer base {
  a { color: blue; }        /* cualquier enlace */
}
@layer componentes {
  .btn { color: teal; }
}
@layer utilidades {
  .text-red { color: red; } /* 0-1-0 pero gana: capa posterior */
}
```

Las capas sin declarar van **antes** que las declaradas; dentro de la misma capa aplica especificidad normal. Ver [[Especificidad]] para la interacción completa.

## Herencia: el otro mecanismo de propagación

Ciertas propiedades (`color`, `font-family`, `line-height`, `visibility`…) **heredan**: si no se definen en un hijo, toman el valor del padre, sin selector de por medio. No compite con la cascada — es un canal distinto que solo interviene cuando ninguna regla (por específica que sea) define la propiedad en el elemento.

```css
body { font-family: system-ui; }
/* todos los descendientes "heredan" system-ui, sin escribir selectores */
```

Toda propiedad entiende cinco **palabras globales** que intervienen en cascada + herencia:

| Palabra | Significado |
|---|---|
| `initial` | El valor del **especificación** (ignora herencia y cascada previa) |
| `inherit` | Tomar el valor **computado del padre** |
| `unset` | `inherit` si la propiedad hereda; `initial` si no |
| `revert` | Volver al valor del **origen anterior** al nuestro (del autor: al user agent) — "deshacer mi hoja de estilo" |
| `revert-layer` | Igual pero volviendo a la **capa anterior** de `@layer` |

```css
.botón { all: unset; } /* reset nuclear: todas las propiedades a unset */
```

## El CSS como sistema: modelo de caja, valores y layout

Detrás de cada regla hay un modelo que conviene tener claro:

- **Modelo de caja**: todo elemento es una caja de contenido + padding + border + margin; `box-sizing` cambia el cálculo del ancho — ver [[Box Model]].
- **Valores y unidades**: longitudes absolutas (`px`) y relativas (`rem`, `em`, `%`, `vw`, `vh`/`dvh`, `ch`) — ver [[Medidas CSS]] y [[Medidas Relativas]].
- **Colores**: `named`, hex, `rgb()`, `hsl()`, y los modernos `oklch()`/`color-mix()` — ver [[Colores CSS]].
- **Los cuatro contextos de layout**: flujo normal (bloque/inline), flex (1 dimensión), grid (2 dimensiones) y posicionados (`relative/absolute/fixed/sticky`) — ver [[Flexbox]], [[CSS Grid]] y [[Tipos de Posiciones]].

## Responsive: adaptar sin romper

El principio: **CSS puede consultar las capacidades del entorno** y aplicar reglas condicionales:

```css
/* media query clásica: pantalla ancha */
@media (min-width: 768px) {
  .layout { grid-template-columns: 240px 1fr; }
}

/* rango moderno (sin min/max sueltos) */
@media (768px <= width <= 1024px) { /* ... */ }

/* preferencia del usuario */
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; }
}
```

Y desde 2023, las **container queries** consultan el ancho del **contenedor** y no de la pantalla — la pieza que faltaba para componentes reutilizables. Detalle completo en [[Responsive Web Design]].

## La caja de herramientas moderna

El CSS actual va mucho más allá de `color` y `margin`:

| Herramienta | Para qué | Nota |
|---|---|---|
| Custom properties (`--var`) | Variables en runtime, temas | Ver [[Colores CSS]] |
| Funciones `clamp()`, `min()`, `max()` | Tamaños fluidos sin media queries | `clamp(1rem, 2vw, 1.5rem)` |
| `@layer` (cascade layers) | Orden explícito de la cascada | Ver arriba |
| `:has()` | Selector "padre" según sus hijos | `.form:has(:invalid)` |
| Anidamiento nativo | CSS con estructura de carpeta | Con `&`; ojo con la especificidad |
| Container queries | Adaptar al contenedor | Ver [[Responsive Web Design]] |
| `@property` | Tipar custom properties para animarlas | Ver [[Animaciones]] |
| Subgrid | Hijos alineados con la grilla padre | Ver [[CSS Grid]] |

## Debugging: las DevTools son parte del aprendizaje

- **Inspector de elementos**: clic derecho → *Inspect* muestra todas las reglas que aplican (con las tachadas y **por qué** están tachadas).
- Pestaña **Styles**: muestra el orden de cascada efectivo; el hover sobre un selector revela su **puntuación de especificidad**.
- Panel **Computed**: qué valor computado ganó para cada propiedad, y si vino por herencia.
- **Toggle forces**: forzar `:hover`, `:focus`, `@media` activos sin tocar el código.
- **Lighthouse/Performance**: para auditar rendimiento y accesibilidad del CSS cargado.

Ciclo de depuración de un estilo que "no aplica":

1. ¿La regla **se carga**? (pestaña Network / Sources, o el panel de fuentes en Styles).
2. ¿El **selector apunta** al elemento? (Clases con erratas, jerarquía de HTML distinta a la imaginada).
3. ¿Está **tachada**? La causa escrita al lado lo dice: especificidad, `!important` de otra regla, o capa (`@layer`).
4. ¿El elemento **tiene ese display/estado**? (`:hover` no existe en touch, `display: none` rompe todo lo demás).
5. ¿Es un problema de **herencia**? (el hijo no hereda porque algo definió la propiedad directamente).

> [!tip] Cómo se aprende bien CSS
> CSS no es difícil: es un **sistema pequeño** con reglas de resolución (cascada, especificidad, herencia) y un modelo de caja. Lo que lo hace confuso al principio es no conocer esas reglas. Camino recomendado: [[Anatomía de CSS]] → [[Selectores en CSS]] → [[Especificidad]] → [[Box Model]] → [[Flexbox]] y [[CSS Grid]] → [[Responsive Web Design]]. Cada nota enlazada lleva ejemplos ejecutables.

## Performance y seguridad (lo mínimo que hay que saber)

- **Rendimiento**: el navegador parsea CSS en un árbol de estilo, calcula la cascada (*style recalc*), reubica cajas (*layout*), repinta (*paint*) y compone (*composite*). Animar propiedades de caja (`width`, `top`) dispara layout+paint cada frame; `transform`/`opacity` van directo a composición — ver [[Animaciones]].
- **Paralelismo**: `<link rel="stylesheet">` bloquea el render pero se descarga en paralelo con otros CSS; `@import` rompe ese paralelismo.
- **Seguridad**: el navegador ejecuta lo que el CSS dice — un campo de usuario sin sanear metido en un estilo puede exfiltrar datos (selectores de atributos + `url()` de fondos, ver [[Inyecciones CSS]]). Regla: nunca concatenar entrada de usuario dentro de CSS sin escapar.

## Preprocesadores y frameworks: el CSS que llega al navegador

Todo lo anterior es lo que ejecuta el navegador **tal cual**. Encima de eso hay dos capas habituales en proyectos reales:

| Capa | Ejemplos | Qué aporta | Qué llega al navegador |
|---|---|---|---|
| Preprocesadores | Sass (ver [[Qué es Sass]]) | Variables compiladas, anidación, mixins (ver [[Mixins]]) | CSS plano generado |
| Utilidades atómicas | Tailwind (ver [[TailwindCSS - Sintaxis Básica]]) | Clases con significado en el HTML, especificidad plana | CSS generado de solo lo usado |
| El navegador | — | cascada, especificidad, herencia | **el CSS final, sin excepciones** |

La clave: cualquiera de las dos capas se **compila a CSS** y compite en la misma cascada. Entender la especificidad y el orden no es "vanidad académica": es leer el CSS final que el navegador realmente aplica.

## Mapa rápido: concepto → nota

| Si querés saber… | Andá a |
|---|---|
| Cómo se escribe una regla | [[Anatomía de CSS]] |
| Cómo apuntar a elementos | [[Selectores en CSS]] |
| Qué regla gana en un conflicto | [[Especificidad]] |
| Cómo funcionan las cajas | [[Box Model]] |
| Colores y variables | [[Colores CSS]] |
| Layouts modernos | [[Flexbox]], [[CSS Grid]] |
| Adaptar a pantallas | [[Responsive Web Design]] |
| Cómo organizar CSS a escala | [[Estilos para escribir CSS]] |
| Preprocesadores y utilidades | [[Qué es Sass]], [[TailwindCSS - Sintaxis Básica]] |

## Relacionadas

- [[Anatomía de CSS]] — estructura detallada de una regla
- [[Selectores en CSS]] — cómo apuntar a los elementos
- [[Especificidad]] — qué regla gana en un conflicto
- [[Crear archivo para CSS]] — enlazar CSS desde el HTML
- [[Colores CSS]] — valores de color y variables
- [[Box Model]] — la caja: la primera abstracción real de CSS
- [[Flexbox]] y [[CSS Grid]] — los dos modelos de layout
- [[Responsive Web Design]] — media queries y container queries
- [[Estilos para escribir CSS]] — metodologías para escalar CSS
- [[Qué es Sass]] — preprocesadores: variables compiladas y anidación
- [[Inyecciones CSS]] — la cara oscura de meter datos de usuario en estilos
