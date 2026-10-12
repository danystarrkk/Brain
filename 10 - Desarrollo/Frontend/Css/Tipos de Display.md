---
tags:
  - desarrollo
  - frontend
  - display
tipo: concepto
estado: terminado
---

# Tipos de Display

La propiedad `display` define **cómo se comporta un elemento en el flujo de la página**: si ocupa todo el ancho, si va al lado de otro, si deja de existir visualmente. Cada etiqueta HTML trae un display **por defecto**, y solo hace falta cambiarlo cuando el diseño lo exige.

> [!info] `display` resuelve dos preguntas a la vez
> En el modelo moderno (CSS Display Level 3), `display` codifica dos decisiones independientes:
> 1. **Participación** (*outer display*): cómo se coloca el elemento respecto de sus hermanos — `block`, `inline` o `list-item`.
> 2. **Contexto de formato** (*inner display*): qué modelo rige a sus **hijos** — `flow`, `flow-root`, `flex`, `grid`, `table`...
>
> Así `display: flex` significa *"caja de bloque cuyos hijos se ordenan con el modelo flex"* y `display: inline-grid` significa *"caja inline cuyos hijos se ordenan en grid"*. Entender esa gramática explica casi todos los valores de golpe.

## Gramática de dos valores

`display` acepta una o dos palabras; las precompuestas (`flex`, `inline-block`...) son azúcar sobre combinaciones:

| Valor | Forma equivalente | Significado |
|---|---|---|
| `block` | `block flow` | caja de bloque, hijos en flujo normal |
| `inline` | `inline flow` | caja inline, hijos en flujo normal |
| `inline-block` | `inline flow` (atómico) | inline a nivel externo, pero **una sola caja** interna |
| `flow-root` | `block flow-root` | bloque que establece contexto de formato de bloque (BFC) |
| — | `inline flow-root` | inline atómico que además establece BFC (el "inline-block moderno") |
| `list-item` | `block flow` + caja de marker | bloque + viñeta/numeración (`::marker`) |
| `flex` / `inline-flex` | `block/inline flow flex` | contexto flex para los hijos |
| `grid` / `inline-grid` | `block/inline flow grid` | contexto grid para los hijos |
| `table` / `inline-table` | `block/inline table` | modelo de tablas |
| `contents` | — | **no genera caja** |
| `none` | — | no genera caja ni aparece en el árbol |

Las combinaciones con dos palabras se usan poco a mano (p. ej. `display: block flow-root`), pero aparecen en la especificación, en las devtools y en reset modernos — leerlas como *"outer inner"* evita la confusión.

## Los dos displays por defecto: block e inline

### Display block

Ocupa **todo el ancho disponible** y se coloca uno debajo del otro, como ladrillos apilados:

![[Pasted image 20250124163054.png]]

Los tienen los headings (`h1`–`h6`), párrafos `p`, `div`, listas `ul`/`ol`, secciones... Son el default de casi todo el HTML estructural.

Características:

- Aceptan `width`, `height`, `margin` y `padding` en **todos** los lados (ojo: los márgenes verticales entre bloques **se colapsan** — toman el mayor, no se suman; detalle en [[Box Model]]).
- Ocupan todo el ancho del contenedor aunque su contenido sea corto.
- Empiezan en una línea nueva (salvo que se les quite del flujo con `float`/`position` o se cambie el display).

### Display inline

Los elementos se colocan **uno a la derecha del otro**, fluyendo con el texto:

![[Pasted image 20250124163114.png]]

Los tienen los enlaces `a`, `span`, `strong`, `em`, `img`, etc.

Características:

- **Ignoran `width`/`height`**: su tamaño lo decide el contenido.
- Los márgenes y paddings verticales **no empujan** a los demás elementos (pintan, pero no desplazan).
- Se **rompen en varias líneas**: un `<a>` largo cruza el salto de línea, a diferencia de un inline-block.
- Aceptan `vertical-align`, que es la forma clásica de alinear inline entre sí.

> [!warning] El error clásico
> Ponerle `width: 200px` a un `<a>` "no funciona" — no es un bug: los inline no aceptan ancho. Solución: `display: inline-block`.

### La pieza intermedia: `inline-block`

Combina ambos comportamientos: fluye en línea como un inline, pero acepta `width`/`height` y paddings completos como un block. Era *la* solución para tarjetas en fila antes de Flexbox, y sigue siendo útil para cosas puntuales:

```css
.badge {
  display: inline-block;
  width: 12px;
  height: 12px;
}
```

> [!tip] El hueco fantasma del inline-block
> Dos `inline-block` en HTML con un espacio/salto de línea entre ellos generan un **gap** del tamaño del espacio de texto (medio carácter). Soluciones: quitar el espacio en el HTML, `font-size: 0` en el padre (con `font-size` explícito en los hijos), o — la buena — usar [[Flexbox]] con `gap`, que no depende del whitespace.

## El display moderno: flex y grid

Los displays de layout **son hijos de esta misma familia**: reemplazan al `inline-block` (y a los float) para construir diseños.

```css
.nav { display: flex; }    /* hijos en fila/columna — ver [[Flexbox]] */
.grilla { display: grid; } /* hijos en filas y columnas — ver [[CSS Grid]] */
```

| `display`       | Comportamiento                                        |
| --------------- | ----------------------------------------------------- |
| `block`         | Ocupa todo el ancho, apilado                         |
| `inline`        | Fluye con el texto, tamaño por contenido             |
| `inline-block`  | Fluye en línea pero acepta dimensiones               |
| `flex`          | Contenedor flex (layout unidimensional)               |
| `inline-flex`   | Ídem, pero la caja es inline                          |
| `grid`          | Contenedor grid (layout bidimensional)                |
| `inline-grid`   | Ídem, pero la caja es inline                          |
| `flow-root`     | Bloque que establece BFC sin hacks de overflow/float |
| `contents`      | El elemento "desaparece": sus hijos pasan al flujo del abuelo |
| `none`          | Se oculta y **no ocupa espacio**                      |

Cambiar el display de un contenedor **cambia el modelo de sus hijos**: un `div` con `display: flex` hace de sus hijos flex items automáticamente (sin tocar el HTML), y sus márgenes verticales ya no colapsan. Es el mecanismo detrás de casi todo CSS de layout moderno.

## Contextos de formato de bloque (BFC): `flow-root` y sus compañeros

Un **BFC** es un "recinto de flujo" aislado: los floats de adentro no se escapan, los márgenes no colapsan hacia afuera, y el contenido define el alto mínimo del contenedor. Antes de `flow-root` se forzaba con hacks:

| Método para crear BFC | Problema |
|---|---|
| `overflow: hidden/auto` | Crea scrollbars o recorta contenido |
| `float: left/right` | Saca del flujo y distorsiona hermanos |
| `position: absolute` | Sacal del flujo |
| `display: inline-block` | La caja deja de ocupar todo el ancho |
| **`display: flow-root`** ✅ | Sin efectos secundarios |

```css
.contenedor-limpio {
  display: flow-root;   /* contiene floats sin overflow hack */
}
```

El BFC también explica el bug #1 de Flexbox y Grid: los hijos de un contexto de formato **no pueden ser más pequeños que su contenido** (mínimo de contenido), por eso el texto largo rompe el layout hasta que se pone `min-width: 0` — ver [[Flexbox]].

Qué establece un BFC: el elemento raíz, `flow-root`, elementos con `float ≠ none`, `overflow ≠ visible`, posicionados (`absolute`/`fixed`), `display: inline-block`, `table-cell`, `table-caption`, `table-wrapper` y los items de flex/grid.

## `display: contents`: la caja que no existe

```css
/* El wrapper deja de existir visualmente; los hijos van al layout del abuelo */
.wrapper-tema { display: contents; }
```

- Los hijos pasan a ser hijos de **layout** del abuelo (siguen siendo hijos en el DOM y en la herencia).
- La herencia de propiedades sigue pasando por el elemento (sus `--vars` y `color` heredan normal).
- Es útil para layouts: un `<div class="grid">` envolvente que no debe participar del grid del padre.
- **Riesgo de accesibilidad**: históricamente algunos navegadores sacaban del árbol de accesibilidad elementos con `display: contents` (perdiendo roles de `ul`, `li`, `button`...). Los motores actuales preservan la semántica, pero con elementos semánticos conviene verificar con el árbol de accesibilidad o evitarlo.

> [!warning] No uses `display: contents` en elementos semánticos sin verificar
> El escenario clásico de bug: `<li style="display: contents">` dejando de anunciarse como item de lista en algunos combos de navegador + lector de pantalla. En 2026 los motores principales lo manejan bien, pero el coste de un `div` extra es cero — ponderá si vale la pena.

## `display: list-item` y `::marker`

`list-item` es el display de `<li>`: caja de bloque **más** una caja de marker (bullet, número o custom content) gestionada por `::marker`:

```css
.paso { display: list-item; }
.paso::marker { content: "✔ "; color: teal; }
.lista { list-style: none; }
.lista > li { display: list-item; }   /* recupera el marker si algo lo quitó */

.paso-indicado { list-style: inside; }
```

| Propiedad | Función |
|---|---|
| `list-style-type` | Tipo de marker (circle, square, decimal, lower-alpha...) |
| `list-style-position` | `inside` (marker dentro del flujo) / `outside` (marker colgando) |
| `list-style-image` | Imagen como marker |
| `::marker` | Estiliza el marker (color, content, fuente) |
| `list-style` | shorthand de los tres |

Detalle que se nota: si cambiás `li { display: flex }`, **el marker desaparece** porque el display ya no incluye `list-item`. Para listas navegables en fila con marcadores semánticos, conservá `display: list-item` en el `li` y pone el `flex` en el `ul` (o aceptá perder el bullet visual).

## Displays de tabla

La familia completa del modelo de tablas:

| Display | Rol |
|---|---|
| `table` | La tabla (wrapper) |
| `inline-table` | Tabla inline |
| `table-caption` | El `<caption>` |
| `table-column` / `table-column-group` | Columnas (`<col>`, `<colgroup>`) |
| `table-header-group` / `table-footer-group` | `<thead>` / `<tfoot>` (repiten en paginación) |
| `table-row-group` | `<tbody>` |
| `table-row` | `<tr>` |
| `table-cell` | `<td>` / `<th>` |

Aplicaciones legítimas hoy:

- **Fijar cabeceras de tabla**: `thead { display: table-header-group; }` hace que la cabecera se repita al imprimir cada página.
- Reproducir el comportamiento de celdas (alineación vertical por defecto, tamaño según contenido) sin usar `<table>`.
- **Anti-patrón**: `display: table*` como técnica de layout (era común antes de Grid). No aporta semántica real — un `<div style="display: table">` **no** es una tabla para lectores de pantalla — y hoy Grid lo hace mejor.

## `display: none` vs `visibility: hidden` vs `content-visibility`

| Propiedad / valor | Se ve | Ocupa espacio | En árbol de a11y | Transiciones | Find-in-page |
|---|---|---|---|---|---|
| `display: none` | No | **No** | No | No (salto de layout) | No |
| `visibility: hidden` | No | **Sí** | No | Sí (discreta, se puede retar) | No |
| `content-visibility: auto` | Fuera de pantalla salteado | Sí (con tamaño) | **Sí** | — | Sí |
| `content-visibility: hidden` | No | Sí | Sí (si `hidden-until-found`) | — | Con `hidden-until-found` |
| `[hidden="until-found"]` | No | Sí | revelado bajo demanda | — | Sí |

Para animar la aparición/ocultamiento de un elemento hoy se prefiere `opacity` + `translate` (ver [[Animaciones]]), que además respeta `prefers-reduced-motion`. `visibility` sí permite transiciones discretas (con `transition-behavior: allow-discrete` se anima también el `display` en el modelo nuevo, mostrando/ocultando con cross-fade).

```css
/* Patrón moderno: ocultar con fade respetando reduced-motion */
.panel {
  opacity: 0;
  visibility: hidden;
  transition: opacity .2s, visibility 0s .2s;
}
.panel[data-abierto] {
  opacity: 1;
  visibility: visible;
  transition: opacity .2s;
}
```

## Cuándo un `display` cambia la semántica (accesibilidad)

- `display: none` / `visibility: hidden`: el contenido **desaparece del árbol de accesibilidad** — correcto para contenido realmente oculto, catastrófico si es informativo.
- `display: contents`: ver la advertencia de arriba.
- Cambios habituales (`block`, `flex`, `grid`, `inline-block`) en elementos semánticos (`<nav>`, `<ul>`, `<section>`) **conservan** su rol en los motores actuales; aun así, verificar con DevTools → pestaña Accessibility cuando el componente sea crítico.
- `display: flex` en `<ul>`: en motores pasados hubo bugs que perdían el rol de lista; hoy funciona, pero si el soporte legado importa, agregar `role="list"` es una línea de seguro.
- Nada de lo que hagas con `display` convierte un `<div>` en `<button>`: la semántica es del HTML (ver [[Html]]) — el CSS solo la respeta o la esconde.

> [!tip] No cambies el display "por si acaso"
> No es necesario modificar los displays de las etiquetas siempre y cuando no sea necesario quitar o reemplazar el que viene por defecto: cada cambio de `display` rompe el flujo heredado y debe justificarse en el diseño.

## Errores comunes

1. **`width`/`height` en inline** — el error clásico; usar `inline-block` o flex.
2. **Gap fantasma entre inline-blocks** por whitespace del HTML — ver la advertencia de inline-block.
3. **Esperar `stretch`/alto en un hijo de `display: contents`** — el elemento no genera caja; todas las propiedades de caja se ignoran.
4. **Perder el marker de `<li>` al poner `display: flex` en el li** — el marker pertenece al display `list-item`.
5. **`display: table` "para que herede mejor"** — la herencia no depende del display; solo el layout cambia.
6. **Toggle de `display` para animar** — `display` no se interpola (con `transition-behavior: allow-discrete` sí se transiciona discretamente); para fade usar `opacity`/`visibility`.
7. **Confundir BFC con contexto de posicionamiento**: `display` no crea contexto de posicionamiento — para que un hijo `absolute` tenga referencia hace falta `position` ≠ `static` (ver [[Tipos de Posiciones]]). `flow-root` crea un BFC (contención de floats/márgenes), **no** un containing block para descendientes absolutos; la excepción es `contain: layout/paint`, que sí los contiene.

## Rendimiento

- Cambiar `display` dispara **layout completo** del subtree — barato de vez en cuando, caro en bucles de scroll o animaciones.
- `content-visibility: auto` es la gran herramienta de rendimiento de la familia: el navegador **saltea layout/paint** del contenido offscreen manteniendo el tamaño aproximado (`contain-intrinsic-size`). Ideal para comentarios, secciones largas, acordeones colapsados.
- `contain` (relacionado con `flow-root` y containment) va en la misma dirección: limitar el alcance del recálculo — ver [[Efectos de Scroll]].
- `display: none` vs `visibility` en listas gigantes: `none` libera layout pero paga el reflow al volver; para contenido que aparece/desaparece mucho, `content-visibility` o `visibility` + contain.

## Cheatsheet

| Valor | Outer | Inner | Ocupa espacio | Acepta width/height | Nota |
|---|---|---|---|---|---|
| `block` | block | flow | ✅ | ✅ | default estructural |
| `inline` | inline | flow | parcial | ❌ | tamaño por contenido |
| `inline-block` | inline | flow (atómico) | ✅ | ✅ | gap por whitespace |
| `flow-root` | block | flow-root | ✅ | ✅ | BFC sin hacks |
| `list-item` | block | flow + marker | ✅ | ✅ | `::marker` |
| `flex` / `inline-flex` | block/inline | flex | ✅ | ✅ | ver [[Flexbox]] |
| `grid` / `inline-grid` | block/inline | grid | ✅ | ✅ | ver [[CSS Grid]] |
| `table` / `inline-table` | block/inline | table | ✅ | ✅ | modelo tabular |
| `contents` | — | — | ❌ | ❌ | sin caja; cuidado con a11y |
| `none` | — | — | ❌ | ❌ | ausente del árbol de a11y |

| Ocultamiento | Espacio | A11y | Animable | Uso |
|---|---|---|---|---|
| `display: none` | no | fuera | no directo | ocultar de verdad |
| `visibility: hidden` | sí | fuera | sí (discreta) | reservar espacio |
| `content-visibility: auto` | sí | dentro | — | rendimiento offscreen |
| `opacity: 0` + `pointer-events: none` | sí | **dentro** | sí | fades; requiere gestionar foco |

## Relacionadas

- [[Flexbox]] — `display: flex` en profundidad
- [[CSS Grid]] — `display: grid` en profundidad
- [[Box Model]] — cómo dimensionar cada tipo de display y colapso de márgenes
- [[Tipos de Posiciones]] — sacar elementos del flujo con `position`
- [[Animaciones]] — por qué animar `display` es distinto que animar `opacity`
- [[Html]] — el display por defecto de cada etiqueta
- [[Efectos de Scroll]] — `content-visibility` y containment en contenido largo
