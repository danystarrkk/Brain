---
tags:
  - desarrollo
  - frontend
  - pseudo-codigo
tipo: concepto
estado: terminado
---

# Pseudo Código

Los selectores **pseudo** ("pseudocódigo" de CSS) seleccionan elementos **sin necesidad de clases ni IDs extra en el HTML**: se definen en el momento y según el estado o la utilidad que se necesite. Se dividen en dos familias que se distinguen por la cantidad de dos puntos:

| Familia          | Sintaxis  | Selecciona                                            |
| ---------------- | --------- | ----------------------------------------------------- |
| **Pseudo-clase** | `:estado` | Un elemento en un **estado** o de un **tipo** concreto |
| **Pseudo-elemento** | `::parte` | Una **parte virtual** del elemento (no existe en el HTML) |

Las pseudo-clases responden a la pregunta *"¿en qué estado está el elemento?"*; los pseudo-elementos a *"¿qué parte del elemento estilizo?"*. Ambas viven dentro de un **selector compuesto** y se comportan como un simple punto o etiqueta para efectos de especificidad — detalle completo más abajo.

## Pseudo-clases: un solo `:`

Estilizan estados dinámicos o categorías de elementos. El ejemplo clásico: un enlace que cambia **al haber entrado ya una vez** o al recibir foco:

![[Pasted image 20250124164638.png]]

En la imagen, `.navegacion` es la clase contenedora, `a` el enlace dentro, y con `:focus` definimos la pseudo-clase. Dentro de una pseudo-clase podemos declarar **tantas propiedades como haga falta**: solo es cuestión de acomodar todo a gusto.

```css
.navegacion a {
  color: #333;
}

.navegacion a:hover {
  color: #0066cc; /* el mouse está sobre el enlace */
}

.navegacion a:focus {
  color: #e67e22; /* el enlace tiene foco de teclado */
  outline: 2px solid #e67e22;
}

.navegacion a:visited {
  color: #7f5af0; /* enlace ya visitado */
}
```

### Pseudo-clases estructurales (posición en el árbol)

Seleccionan por dónde está el elemento respecto de sus hermanos y padres:

| Pseudo-clase | Significado | Ejemplo de uso |
|---|---|---|
| `:root` | El elemento raíz (`<html>`) — ideal para custom properties | `[data-theme]` + tokens |
| `:first-child` | Primer hijo de su padre | `.menu li:first-child` |
| `:last-child` | Último hijo de su padre | `.lista li:last-child { border: 0 }` |
| `:only-child` | Único hijo de su padre | avisos sin separación |
| `:nth-child(an+b)` | Fórmula sobre **todos** los hermanos | zebra, ritmos de 3 |
| `:nth-last-child(an+b)` | Ídem, contando **desde el final** | últimos 2 elementos |
| `:first-of-type` | Primer hijo de **su mismo tipo** de etiqueta | `.card h2:first-of-type` |
| `:last-of-type` | Último de su tipo | quitar borde al último `p` |
| `:only-of-type` | Único hijo de su tipo | título único |
| `:nth-of-type(an+b)` | Fórmula entre hijos **del mismo tipo** | zebra segura |
| `:nth-last-of-type(an+b)` | Ídem desde el final | — |
| `:empty` | Sin hijos **ni texto** (ni siquiera espacio en blanco) | estado "sin mensajes" |
| `:has(selector)` | El elemento **contiene** lo que matchea el selector | card inválida, sombra al final |
| `:scope` / `:host` | Alcance actual (`@scope`/shadow DOM) | componentes |

> [!warning] `:nth-child` vs `:nth-of-type`: la trampa clásica
> `li:nth-child(2)` cuenta **todos** los hijos del padre y después filtra los `li`. `li:nth-of-type(2)` cuenta solo los `li`. En HTML con etiquetas mixtas dentro de un contenedor (un `div` con `img`, `p`, `p`), las dos dan resultados distintos. Regla: **striped de listas homogéneas → `nth-child`; contenido mezclado → `nth-of-type`**.

### Anatomía de `:nth-child(an+b)`

La fórmula `an + b` describe una secuencia de posiciones: empieza en `b` y salta de `a` en `a`. Todo es evaluable a mano:

| Selector | Posiciones que matchea | Lectura |
|---|---|---|
| `:nth-child(3)` | 3 | solo el tercero |
| `:nth-child(odd)` / `(2n+1)` | 1, 3, 5... | impares |
| `:nth-child(even)` / `(2n)` | 2, 4, 6... | pares |
| `:nth-child(3n)` | 3, 6, 9... | cada tercero |
| `:nth-child(3n+1)` | 1, 4, 7... | cada tercero, empezando en el 1 |
| `:nth-child(-n+3)` | 1, 2, 3 | **los 3 primeros** (fórmula inversa) |
| `:nth-child(n+4)` | 4, 5, 6... | **del cuarto en adelante** |
| `:nth-child(-n+4):not(:first-child)` | 2, 3, 4 | los primeros sin el primero |

La fórmula `n` crece de 0 a infinito; `an+b` es *"b más a por cada vuelta"*. El truco inverso `-n+b` es la forma estándar de seleccionar **los primeros N**; `n+b` selecciona **desde N en adelante**. Para contar **desde el final** se usa `:nth-last-child`.

### Variante `:nth-child(an+b of S)` — conteo filtrado

Selectors Level 4 permite filtrar por selector dentro del conteo, sin depender del tipo de etiqueta:

```css
/* Cada segundo .producto, aunque haya textos o imágenes intercalados */
.item:nth-child(even of .producto) { background: #f6f6f6; }

/* El primer .destacado entre sus hermanos */
.item:first-of-type(...)  /* equivalente práctico: */
.item:nth-child(1 of .destacado) { border-color: gold; }
```

Es el sustituto moderno de los hacks con clases extra tipo `.item.item--even`.

### Pseudo-clases de estado e interacción

| Pseudo-clase | Significado | Nota clave |
|---|---|---|
| `:hover` | Cursor sobre el elemento | En táctil "se queda pegado"; ver advertencia abajo |
| `:active` | Mientras se presiona (click/tecla) | Botones: feedback inmediato |
| `:focus` | Elemento con foco (cualquier motivo) | No distinguir teclado de click es la vieja UX |
| `:focus-visible` | Foco **merecedor** de indicador (heurística del navegador: teclado sí, click a menudo no) | Reemplazo moderno del patrón `:focus:not(:focus-visible)` |
| `:focus-within` | El elemento **o cualquiera de sus hijos** tiene foco | Resaltar la card cuando un input interno está activo; equivale a `.card:has(:focus)` |
| `:visited` | Enlace ya visitado | Solo `<a href>`; **restringido por privacidad** (ver accesibilidad) |
| `:link` | Enlaces **no** visitados con `href` | Raramente necesario |
| `:any-link` | `a` o `area` con o sin href visitado | `:any-link` = `:link, :visited` |
| `:target` | Elemento al que apunta el `#fragment` de la URL | `#aviso:target { ... }` — anclas con énfasis |
| `:defined` | Web component ya definida | Evitar FOUC de custom elements |
| `:modal` | Diálogo abierto con `showModal()` | Oscurecer el resto sin clases extra |
| `:picture-in-picture` | Elemento en modo PiP | Video flotante |
| `:dir(ltr)` / `:dir(rtl)` | Dirección del texto (atributo `dir`, no CSS) | Soporte limitado; alternativa `[dir="rtl"]` |

### Pseudo-clases de formulario (todas las de estado de control)

El arsenal completo — la base de una UI de formularios sin JavaScript (ver [[Formularios]]):

| Pseudo-clase | Significado |
|---|---|
| `:checked` | Checkbox/radio/select marcado |
| `:indeterminate` | Estado intermedio (checkbox "algunos hijos", progress) |
| `:disabled` / `:enabled` | Control deshabilitado / habilitado |
| `:default` | El valor por defecto de su grupo (submit inicial, radio preseleccionado) |
| `:required` / `:optional` | Tiene o no el atributo `required` |
| `:valid` / `:invalid` | Cumple o no las restricciones de validación |
| `:user-valid` / `:user-invalid` | Ídem, pero **solo después de que el usuario interactuó** — evita el rojo al cargar el formulario |
| `:in-range` / `:out-of-range` | Valor dentro/fuera de `min`/`max` |
| `:read-only` / `:read-write` | Solo lectura vs editable |
| `:placeholder-shown` | Mostrando el placeholder (input vacío) |
| `:autofill` | Autocompletado por el navegador | 
| `:blank` | Vacío (más preciso que `:placeholder-shown`) | 

```css
/* Patrón de UX: no pintar errores hasta que el usuario escribe */
.campo input:user-invalid {
  border-color: red;
}

/* Etiqueta asterisco solo en los requeridos */
.campo:has(:required)::after { content: " *"; color: red; }

/* El submit responde al estado del formulario completo */
.formulario:not(:valid) button[type="submit"] { opacity: .5; pointer-events: none; }
```

### Pseudo-clases funcionales: `:is()`, `:where()`, `:not()`, `:has()`

Aceptan un **selector** como argumento y componen lógica:

```css
:is(h1, h2, h3) { text-wrap: balance; }        /* agrupa sin repetir contexto */

:where(a, button):focus-visible {               /* where = especificidad 0
  outline: 2px solid currentColor; }            /* utilísimo para "base" */

.menu li:not(:first-child) { margin-left: 1rem; }

.formulario:has(input:invalid) { border-color: red; }
```

| Función | Argumento | Especificidad resultante | Selector forgiving |
|---|---|---|---|
| `:is(...)` | lista de selectores | la **mayor** de la lista | ✅ (los inválidos se descartan) |
| `:where(...)` | lista de selectores | **siempre 0** | ✅ |
| `:not(...)` | lista de selectores | la **mayor** de la lista | ❌ (lista inválida = selector inválido) |
| `:has(...)` | selectores **relativos** | la **mayor** de la lista | ❌ |
| `:nth-child(... of S)` | selector | la **mayor** de S | ❌ |

> [!tip] El truco de `:where()` para capas de base
> Como `:where()` puntúa 0, cualquier utilidad o componente posterior le gana sin `!important`. Es la herramienta para escribir estilos base reemplazables: `:where(h1, h2, h3) { margin-block: 1.5rem }` crea un estilo por defecto que cualquier `.clase` posterior pisa limpiamente. Detalle completo en [[Especificidad]].

### `:has()`: la pseudo-clase que faltaba ("el selector hacia atrás")

`:has()` invierte la dirección de los selectores: permite estilizar **un ancestro o hermano previo** según lo que contenga o le siga. Es relativo al elemento que lo usa:

```css
/* Contenedor de campo con input inválido */
.campo:has(input:invalid) { border-color: red; }

/* Card que contiene una imagen: layout distinto */
.card:has(> img) { grid-template-rows: 200px 1fr; }

/* El párrafo anterior a un destacado recibe ritmo extra */
p:has(+ .destacado) { margin-bottom: 2rem; }

/* Lista vacía: estado "sin elementos" */
ul:not(:has(li)) { color: #999; }

/* Menú: borde solo cuando alguien está tabulando dentro */
.menu:has(a:focus-visible) { outline: 2px solid dodgerblue; }

/* Cabecera con sombra cuando el contenido de abajo ya no está al tope */
.encabezado:not(:has(~ .principal:first-child))  /* (variante compleja; en la práctica se usa con scroll) */
```

Selector relativo admitido: `:has(> img)` (hijo directo), `:has(+ p)` (hermano adyacente), `:has(~ p)` (hermanos siguientes), `:has(img)` (cualquier profundidad).

> [!info] Rendimiento de `:has()`
> Es la primera pseudo-clase que depende de mutaciones del DOM: el navegador la reevalúa cuando cambia el árbol. En la práctica es rápida (los casos patológicos son DOMs gigantes con miles de `:has()`); no hay que temerla, pero tampoco es gratis dentro de selectores de bucle en listas enormes. Soporte: Safari 15.4, Chrome 105, Firefox 121.

## Pseudo-elementos: dos puntos dobles `::`

No seleccionan un estado, sino una **parte inexistente en el HTML** del elemento. El ejemplo rey: contenido decorativo antes/después del elemento:

```css
.advertencia::before {
  content: "⚠ ";
  font-size: 1.2em;
}

.enlace-exterior::after {
  content: " ↗";           /* añade un glifo al final del enlace */
  font-size: 0.85em;
}

input::placeholder {
  color: #999;             /* el texto de ayuda del input */
}

.parrafo::first-letter {
  font-size: 3rem;         /* capitular */
  float: left;
}
```

> [!warning] `content` solo existe en pseudo-elementos
> `::before` y `::after` **ignoran sus estilos si no tienen `content`** (aunque sea vacío `""`). Y ese contenido es decorativo: los lectores de pantalla no lo leen por defecto, así que nunca pongas ahí información importante.

### Reglas de la sintaxis

- **Un solo pseudo-elemento por selector compuesto**, y debe ir **al final**: `p::before:hover` es inválido; `p:hover::before` es válido.
- La forma de un solo `:` (`:before`, `:first-line`) es **legado CSS 2.1** y sigue funcionando **solo** para `::before`, `::after`, `::first-letter`, `::first-line` — en código nuevo, siempre doble dos puntos.
- El pseudo-elemento hereda el elemento del selector: `.tarjeta h2::after` es el `::after` del `h2` dentro de `.tarjeta`.

### Catálogo completo de pseudo-elementos

| Pseudo-elemento | Sobre qué aplica | Detalle |
|---|---|---|
| `::before` / `::after` | Cualquier elemento con `content` | Generan un hijo inline al principio/final |
| `::first-line` | Solo contenedores de bloque | La primera línea **formateada**; cambia con el resize |
| `::first-letter` | Bloques cuyo contenido empiece en texto | Capitulares; admite propiedades limitadas |
| `::marker` | Items de lista (`list-item`) | El bullet/número; admite `color`, `content`, fuente |
| `::placeholder` | Inputs/textarea | El texto de ayuda; estilo limitado por navegador |
| `::selection` | Texto seleccionado | Solo propiedades de color/relleno permitidas |
| `::backdrop` | Elementos de la capa superior (`dialog`, fullscreen, popover) | La capa detrás del diálogo |
| `::file-selector-button` | `<input type="file">` | El botón nativo "Examinar" |
| `::cue` / `::cue-region` | `<video>` con subtítulos | Estilo de cues de `<track>` |
| `::part(nombre)` | Shadow DOM | Expone partes del componente (`exportpart`) |
| `::slotted(selector)` | Shadow DOM | Slotea contenido proyectado |
| `::details-content` | `<details>` | El contenedor plegable (nuevo) |
| `::spelling-error` / `::grammar-error` | Errores del corrector nativo | Soporte muy limitado |
| `::view-transition-*` | View Transitions API | Grupos/clip-paths de la transición |

```css
/* Capitular editorial */
.article > p:first-of-type::first-letter {
  float: left;
  font-size: 3.5rem;
  line-height: 1;
  padding-inline-end: 0.5rem;
  color: #7f5af0;
}

/* Bullet personalizado sin tocar list-style */
.lista-tareas li::marker { content: "☐ "; color: #94a3b8; }
.lista-tareas li.hecha::marker { content: "✓ "; color: #16a34a; }

/* Selection acorde al tema */
::selection { background: oklch(80% 0.12 250); color: black; }

/* Tooltip puro CSS con attr() */
.boton[data-tooltip]::after {
  content: attr(data-tooltip);
  position: absolute;
  bottom: 100%;
  background: #111;
  color: #fff;
  font-size: 0.75rem;
  padding: 0.25rem 0.5rem;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.15s;
}
.boton[data-tooltip]:hover::after,
.boton[data-tooltip]:focus-visible::after { opacity: 1; }

/* Fondo del backdrop de un modal nativo */
dialog::backdrop { background: rgb(0 0 0 / 0.5); backdrop-filter: blur(2px); }
```

### Mecánica del contenido generado (`content`)

`content` acepta mucho más que texto:

| Valor | Ejemplo | Efecto |
|---|---|---|
| `""` / `none` | `content: ""` | Caja vacía (útil con `width`/`height` o como ancla de posicionamiento) |
| Cadena | `content: "Nuevo: "` | Texto literal (escapar `"` con `\201B` o comillas simples) |
| `attr(atributo)` | `content: attr(href)` | Inserta el valor de un atributo (solo texto plano) |
| `attr(x, fallback)` | `content: attr(data-icon, "•")` | Fallback si no existe el atributo |
| `url(...)` | `content: url(icon.svg)` | Imagen (solo `normal`/`none` en otros contextos) |
| `counter()` / `counters()` | `content: counter(item) "."` | Contadores CSS (`counter-reset/step/increment`) |
| `open-quote` / `close-quote` | en `q::before/::after` | Citas con `quotes` |
| `no-open-quote` / `no-close-quote` | — | Ajusta la pila de citas o el anidamiento de contadores |

Los **contadores** son el motor detrás de las listas numeradas propias, las secciones del TOC y los "paso 1, paso 2" de recetas:

```css
.seccion { counter-increment: seccion; }
.seccion > h2::before { content: counter(seccion) ". "; }

.pasos { counter-reset: paso; }
.pasos > li { counter-increment: paso; }
.pasos > li::before { content: "Paso " counter(paso) ": "; font-weight: bold; }

/* Contadores anidados: 1.1, 1.2... */
.capitulo { counter-reset: apartado; }
.capitulo h3::before { content: counter(capitulo) "." counter(apartado) " "; }
.capitulo h3 { counter-increment: apartado; }
```

> [!warning] Contenido generado y accesibilidad
> El contenido de `::before`/`::after` **no es fiable para lectores de pantalla**: algunos navegadores lo exponen en el árbol de accesibilidad, otros no. Nunca pongas ahí información, instrucciones ni texto esencial. Para números decorativos de lista está bien; para el nombre de un botón, no. Y recordá que el contenido generado **sí** es seleccionable y copiable con el mouse, lo que puede produir copias raras.

## Especificidad

Las pseudo-clases se puntúan como **clases** (0–1–0) y los pseudo-elementos como **elementos** (0–0–1) — detalle completo en [[Especificidad]]. Las funcionales puntúan según su argumento: `:is()`/`:not()`/`:has()` toman el valor **máximo** de su lista; `:where()` es siempre **0**.

| Selector | Puntuación |
|---|---|
| `a:hover` | 0–1–1 |
| `.menu li:nth-child(2)` | 0–2–1 |
| `#nav a:focus-visible` | 1–1–1 |
| `p::before` | 0–0–2 |
| `:where(h1, h2) .a` | 0–1–0 (`:where` no suma) |
| `:is(h1, #x) p` | 1–0–1 (toma el máximo: `#x`) |

## Errores comunes

1. **`::before` sin `content`**: la caja no genera estilos visibles. Síntoma: "la clase no funciona". Fix: `content: ""` aunque no lleve texto.
2. **Información en contenido generado**: texto crítico en `::after` que los lectores de pantalla omiten. Fix: HTML real; contenido generado solo decorativo.
3. **Confundir `:nth-child` con `:nth-of-type`** en contenedores mixtos — ver la advertencia estructural.
4. **`:hover` como única vía de información**: en táctil se activa con el toque y queda "pegado"; el dato debe estar siempre visible o también en `:focus-visible`. Además existe `@media (hover: hover)` para aplicar el hover solo a dispositivos con cursor real.
5. **Esperar que `:visited` estilice libremente**: por privacidad, los navegadores solo permiten cambiar **colores** y a menudo los **aproximan**; `getComputedStyle` miente después de visitar. No diseñes funciones dependientes de `:visited`.
6. **Pseudo-elemento en medio del selector**: `a::after:hover` inválido. El pseudo-elemento cierra el selector compuesto; lo que venga después debe ir en el cuerpo de la regla.
7. **`:focus` sin `:focus-visible`**: outline permanente al hacer click daña la UX; outline ausente con teclado daña la accesibilidad. Patrón correcto: estilizar `:focus-visible` para el indicador y `:focus` solo si hace falta.
8. **Asumir que `:empty` ignora espacios**: `<li> </li>` **no** es `:empty` (el texto de espacio cuenta). Es un bug frecuente con formateo de HTML "bonito".

## Accesibilidad

- **Nunca escondas el foco**: si usás `:focus { outline: none }`, reemplazá con `:focus-visible` con contraste suficiente.
- **Estados de formulario no solo por color**: `:invalid` en rojo debe ir con texto o icono, no solo color (WCAG 1.4.1) — ver [[Formularios]].
- **`:checked` también se aplica a custom controls**: los `input` visuales con `appearance: none` + `:checked` siguen siendo accesibles; si los reemplazás por `<div>`, perdés teclado y rol — mejor CSS sobre elementos nativos.
- **Tooltip con `::after` no es accesible por sí mismo**: no recibe foco ni se anuncia; si el tooltip es informativo, usá `aria-describedby` en HTML (el `::after` puede quedarse como refuerzo visual).
- Verificá con el árbol de accesibilidad que el contenido generado no se cuela ni duplica texto.

## Rendimiento

- Las pseudo-clases de estado (`:hover`, `:checked`, `:nth-child`) son evaluadas en el momento de match: costo nulo.
- `:has()` introduce re-evaluación ante mutaciones del DOM — en la práctica rápido; medir solo con DOMs muy grandes.
- `::selection` y estilos de pseudo-elementos comunes no tienen costo medible; los pseudo-elementos con `filter` o `backdrop-filter` sí pintan cada repaint (ver [[Sombras CSS]] para costos de pintura).

## Compatibilidad

- `:has()` y `:nth-child(... of S)`: navegadores actuales (Safari 15.4+, Chrome 105+, Firefox 121+ para `:has()`).
- `:user-valid`/`:user-invalid`: Chromium/Firefox recientes; Safari desde 17.
- `::details-content`, `::backdrop` en popover: specs nuevas (2023–2025), ir con progressive enhancement.
- Forma legada `:before`/`:after`: soportada para siempre; migrar al escribir CSS nuevo.

## Cheatsheet

| Concepto | Sintaxis | Especificidad | Estado |
|---|---|---|---|
| Pseudo-clase simple | `:hover` | 0–1–0 | universal |
| Pseudo-clase estructural | `:nth-child(2n)` | 0–1–0 | universal |
| Pseudo-clase de formulario | `:checked`, `:invalid` | 0–1–0 | universal |
| `:is()` / `:where()` / `:not()` / `:has()` | funcional | máx. arg. / 0 / máx. arg. / máx. arg. | moderno (2021–2023) |
| `:focus-visible` | `:focus-visible` | 0–1–0 | universal desde 2021 |
| Pseudo-elemento | `::before` | 0–0–1 | universal |
| `::marker`, `::selection`, `::placeholder` | `::x` | 0–0–1 | universal |
| `::backdrop`, `::file-selector-button` | `::x` | 0–0–1 | moderno |
| `::part()` / `::slotted()` | `::part(x)` | 0–0–1 | shadow DOM |
| `content` | solo en pseudo-elementos | — | universal |

## Relacionadas

- [[Selectores en CSS]] — el resto de selectores y cómo se combinan
- [[Especificidad]] — cómo puntúa el navegador estos selectores
- [[Anatomía de CSS]] — la estructura de la regla donde viven
- [[Formularios]] — pseudo-clases de estado en inputs y validación visual
- [[Estilos para escribir CSS]] — cómo los pseudo-elementos conviven con BEM (¿`::before` o `.icon`?)
- [[Animaciones]] — transiciones disparadas por estados pseudo
- [[Tipos de Posiciones]] — posicionamiento del contenido generado (`attr()` + `absolute`)
