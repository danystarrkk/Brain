---
aliases: ["Position", "Posicionamiento CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Box Model]]"
  - "[[Flexbox]]"
  - "[[Efectos de Scroll]]"
  - "[[Tipos de Display]]"
  - "[[Sombras CSS]]"
  - "[[Responsive Web Design]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/position"
---

# Tipos de Posiciones

La propiedad `position` define **cómo se sitúa un elemento** dentro del flujo de la página y respecto a qué punto se miden sus desplazamientos (`top`, `right`, `bottom`, `left`). Tiene cuatro valores principales — más el `static` por defecto — y son la base de menús pegajosos, botones flotantes y modales. Dominarlas es entender tres máquinas: el **containing block** (de dónde se mide), el **stacking context** (quién pinta encima de quién) y la **capa superior** (*top layer*, que se salta todas las reglas).

## Los valores de `position` de un vistazo

| Valor      | ¿Sale del flujo? | Referencia de `top/left/...`            | Uso típico                        |
| ---------- | ----------------- | --------------------------------------- | --------------------------------- |
| `static` (defecto) | No        | No admite desplazamientos              | Todo el contenido normal          |
| `relative` | No                | Su **posición original**                | Contexto de posicionamiento, corregir a mano |
| `absolute` | **Sí**            | El ancestor posicionado más cercano     | Overlays, badges, menús           |
| `fixed`    | **Sí**            | La **ventana** (viewport)               | Headers, botones flotantes, barras |
| `sticky`   | No (hasta pegarse) | La ventana, dentro de su contenedor     | Índices, encabezados de tabla     |

## `static`: el comportamiento por defecto

No necesita declararse. Los elementos estáticos fluyen normal (block apilados, inline en línea) y **ignoran `top`/`left`**: para poder usarlos hay que cambiar el `position`. También ignoran `z-index` (excepto items de flex/grid — ver stacking más abajo).

## `relative`

No saca al elemento del flujo (sigue ocupando su lugar), pero **desplaza su posición visual** respecto a donde debería estar, y — clave — **pasa a ser el punto de referencia** de sus hijos `absolute`:

```css
h1 {
  position: relative; /* Desbloquea el uso de los demás atributos */
  left: 10rem;        /* También admite %, rem, vw... */
  top: 2rem;
}
```

> [!tip] El patrón relative + absolute
> Un padre `position: relative` + un hijo `position: absolute` es el combo más usado del posicionamiento: el hijo se ubica **dentro de su padre**, sin afectar al resto del layout.

Notas de `relative`:

- El espacio del elemento **sigue reservado**: los hermanos no se mueven. Dos `relative` desplazados pueden superponerse entre sí.
- Con `relative` también se crea un stacking context **solo si** `z-index` ≠ `auto` (ver la sección de stacking).
- Es el valor correcto para micro-correcciones (`top: -2px` para optical alignment) — pero si el desplazamiento es grande, casi siempre falta flex/grid (ver [[Flexbox]]).

## `absolute`

Ubica el elemento a partir del **elemento padre posicionado** más cercano (el ancestor con `position` distinto de `static`). Si **ningún** ancestor está posicionado, la referencia es el `body`/documento inicial — ese es el requisito que más confunde:

```css
.contenedor {
  position: relative; /* origen de coordenadas para los hijos */
}

.contenedor .badge {
  position: absolute;
  top: 0;
  right: 0;           /* pegado a la esquina superior derecha del contenedor */
}
```

Como sale del flujo, los elementos siguientes **no reservan espacio** para él: se superpone.

Detalles finos:

- La referencia es el **padding-box** del ancestro posicionado, no su content-box: `top: 0` lo pega al borde interior, por encima del `padding` del padre. Con `border` visible hay que compensar a mano.
- Sin ancestro posicionado, la referencia es el **initial containing block** (el viewport anclado al inicio del documento): `absolute` con `top: 0` se va al tope del *documento*, no de la ventana (eso sería `fixed`).
- Hijos de un `absolute` con `position: static` fluyen dentro de él normalmente — el patrón tarjeta-absoluta con contenido es válido.
- % de `top`/`bottom` se mide sobre la **altura** del containing block; `left`/`right` sobre el ancho. En `absolute` con `height: auto`, `top` + `bottom` definidos **estiran** el elemento entre ambos.

## `fixed`

Se mantiene **fijo en ese punto de la pantalla**, sin importar el scroll:

```css
h1 {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
}
```

Es muy útil para botones flotantes o barra de navegación anclados a un punto de la pantalla que no se mueven al hacer scroll. Como sale del flujo, deja un hueco donde estaba: casi siempre va acompañado de un margen en el contenido (por ejemplo `body { padding-top: 60px }`).

> [!warning] `fixed` deja de ser "de la ventana" con `transform`, `filter` o `contain`
> Si **cualquier ancestro** tiene `transform`, `perspective`, `filter`, `backdrop-filter`, `will-change` de alguno de estos o `contain: paint/layout`, ese ancestro se convierte en el containing block del `fixed`: el elemento "se atasca" dentro del ancestro y deja de ser fijo respecto del viewport. Síntoma clásico: el modal o el botón flotante se mueve con el contenido tras animar un ancestro con `transform`. Fix: sacar el `transform` del ancestro, o montar el `fixed` fuera de ese árbol (o usar `<dialog>`/top layer).

Comportamiento en móvil: `fixed` respecto del viewport *visual* hace que el elemento salte cuando la barra de URL del navegador se oculta/ensancha. Con `100vh`/`top: 0` se ven temblores; la medida correcta es `dvh` — ver [[Medidas Relativas]] y [[Responsive Web Design]].

## `sticky`

El híbrido: se comporta **como `relative` hasta que** llega el punto de anclaje durante el scroll, y entonces "se pega" a la ventana **dentro de los límites de su contenedor**:

```css
.indice {
  position: sticky;
  top: 1rem;   /* se pega cuando llegue a 1rem del borde superior */
}
```

Suele usarse en encabezados de sección, tablas largas o índices. Casos prácticos de pegado con scroll en [[Efectos de Scroll]].

### Mecánica exacta de `sticky`

1. **No sale del flujo**: reserva su espacio; los hermanos no se reordenan.
2. Requiere **al menos un `top`/`bottom`/`left`/`right` ≠ `auto`**: sin ancla, no hace nada (se comporta como `relative`).
3. Se mantiene dentro de su **containing block** (el padre en flujo): al llegar al borde del padre, deja de pegarse y lo acompaña hacia arriba. El "viaje" de sticky es exactamente la altura del padre menos la del propio elemento.
4. El scrollport de referencia es el **ancestro con scroll más cercano**: si algún ancestro intermedio tiene `overflow: hidden/auto/scroll/clip`, *ese* es el contenedor de scroll — y si no puede scrollear (p. ej. `overflow: hidden` sin altura limitada), el sticky **nunca se pega**. Síntoma: "el sticky no funciona"; causa #1: un `overflow: hidden` de por medio.
5. En flex/grid, el item sticky necesita `align-self: start` (o su eje debe tener tamaño de contenido): con `stretch` el item llena la fila entera y no hay "viaje" posible para pegarse.

```css
/* Sidebar sticky dentro de un grid */
.layout {
  display: grid;
  grid-template-columns: 240px 1fr;
  gap: 2rem;
}
.sidebar { position: sticky; top: 1rem; align-self: start; }
```

> [!warning] Sticky tapado por cabeceras
> Un `top: 0` sticky queda debajo de una cabecera `fixed`. Solución: `top: var(--altura-header)`, o mejor, compensar todos los anclajes con `scroll-padding-top` en el scroller:
>
> ```css
> html { scroll-padding-top: 72px; }   /* anclas y sticky respetan la cabecera */
> ```

## Desplazamientos: `top`, `right`, `bottom`, `left` y `inset`

Funcionan con cualquier unidad (`px`, `rem`, `%`...) y miden **hacia dónde se empuja el elemento desde su referencia**:

- Con `absolute`/`fixed`: son la coordenada exacta respecto a la referencia.
- Con `relative`: son un desplazamiento sobre su posición original (el espacio original **sigue ocupado**).
- Con `sticky`: son la **distancia de anclaje** al borde del scrollport.

Si definís `top` y `bottom` a la vez en un elemento no absoluto (o con altura fija), gana `top`; en `absolute` con `height: auto`, ambos pueden estirarlo.

### `inset`: el shorthand moderno

```css
.overlay { inset: 0; }              /* top/right/bottom/left: 0 */
.overlay { inset: 1rem auto; }      /* top/bottom: 1rem; left/right: auto */
.overlay { inset-block: 0; }        /* top + bottom */
.overlay { inset-inline: 0; }       /* left + right (respeta dirección de escritura) */
.overlay { inset-block-start: 0; }  /* "top" lógico */
```

| Propiedad | Física | Lógica (writing modes / RTL) |
|---|---|---|
| `top` / `bottom` | eje vertical | `inset-block-start/end` |
| `left` / `right` | eje horizontal | `inset-inline-start/end` |

Sobredeterminación: si `left` + `right` + `width` no caben, en `direction: ltr` se **ignora `right`** (y en RTL `left`); si `top` + `bottom` + `height` no caben, se ignora `bottom`. Usar `margin-inline: auto` para centrar en vez de pelear con la sobredeterminación.

## Containing block: la tabla completa

De qué se mide cada valor según el tipo de posicionamiento:

| Posicionamiento | Containing block de `absolute` | Containing block de `fixed` | De `%` y `top/...` |
|---|---|---|---|
| `static` | — | — | content-box del ancestor block |
| `relative` | — | — | su propio containing block normal |
| `absolute` | padding-box del ancestor **posicionado** más cercano (o ICB) | padding-box del ancestor posicionado más cercano (o viewport) | el containing block resultante |
| `fixed` | — | viewport — **salvo** ancestro con transform/filter/perspective/contain/will-change → ese ancestro | ídem |
| `sticky` | — | — | su containing block normal; anclas contra el scrollport |

**Regla general para `absolute`:** buscar hacia arriba el primer ancestro con `position ≠ static` (o con `will-change: position/transform`, que el navegador trata como posicionado); si no hay ninguno, el initial containing block del documento.

## `z-index`: quién va arriba

Los elementos posicionados se apilan; `z-index` (solo con `position` ≠ `static`, salvo items de flex/grid) define el orden:

```css
.modal-overlay {
  position: fixed;
  inset: 0;
  z-index: 100; /* por encima de todo lo que no tenga z-index */
}
```

> [!warning] `z-index` no es infinito ni global
> Sin `position` no funciona. Y los valores gigantes (`99999`) son anticipo de problemas: en proyectos serios se usan escalas de z-index definidas en custom properties para no pelear después. El `inset` de arriba es el atajo moderno de `top/right/bottom/left: 0`.

### El árbol de stacking contexts

`z-index` **no compite en una lista global**: cada valor se compara **dentro de su stacking context**. Cuando un elemento crea un SC, todos sus descendientes quedan encerrados: su `z-index: 9999` nunca puede superar a un hermano del padre del SC.

Qué **crea** un stacking context:

| Causa | Detalle |
|---|---|
| Raíz del documento | `<html>` es el SC raíz |
| `position ≠ static` + `z-index ≠ auto` | el combo clásico |
| Item de flex/grid con `z-index` ≠ `auto` | **aunque sea `position: static`** |
| `opacity < 1` | cualquier transparencia, aunque sea 0.999 |
| `transform ≠ none` | incluye `translate`/`scale` de animaciones |
| `filter ≠ none` | incl. `drop-shadow` (ver [[Sombras CSS]]) |
| `perspective ≠ none` | y `transform-style` 3D |
| `backdrop-filter ≠ none` | glassmorphism — ver [[Background]] |
| `mix-blend-mode ≠ normal` | y `isolation: isolate` (creado a propósito) |
| `contain: paint` / `layout` | containment crea SC |
| `mask ≠ none` | máscaras |
| `will-change` de cualquiera de los anteriores | el navegador lo trata como si ya existiera |
| `column-count/column-width ≠ auto` | multicolumna |
| `element()` / `view-transition` | casos especiales |

```css
/* La trampa clásica: "mi z-index: 9999 no funciona" */
.padre { opacity: 0.99; }          /* crea SC sin que lo veas */
.padre .modal { z-index: 9999; }   /* ...así que 9999 vive DENTRO de ese mundo */
.otro { z-index: 50; }             /* pero fuera del SC de .padre, gana */
```

Orden de pintura **dentro** de un stacking context (simplificado):

1. Fondo/bordes del propio SC
2. SCs con `z-index` negativo
3. Fondos de descendientes en flujo (no posicionados)
4. Flotados
5. Contenido en línea (texto)
6. Posicionados con `z-index: auto` o `0` y items flex/grid con `z-index: auto` (en orden del DOM)
7. SCs con `z-index` positivo (menor a mayor)

> [!info] `z-index: auto` vs `z-index: 0`
> | | `z-index: auto` | `z-index: 0` |
> |---|---|---|
> | ¿Crea stacking context? | **No** | **Sí** |
> | Hijos con z-index | participan en el SC del **abuelo** | quedan encerrados en el elemento |
> | Orden vs hermanos | pinta en orden de DOM (fila 6) | pinta en orden de DOM (fila 6) |
>
> En el 99% de los casos son visualmente idénticos; la diferencia aparece cuando un **descendiente** tiene `z-index` negativo o necesita escapar/apilarse de forma independiente. Dato útil: `z-index: 0` es la forma de *aislar* el stacking de un subárbol sin tocar opacidad ni transform.

### Escala de z-index para proyectos

```css
:root {
  --z-base: 0;
  --z-sticky: 10;      /* cabeceras sticky */
  --z-dropdown: 100;
  --z-overlay: 200;    /* fondos de modal */
  --z-modal: 300;
  --z-toast: 400;
  --z-tooltip: 500;
}

.header { z-index: var(--z-sticky); }
.dropdown { z-index: var(--z-dropdown); }
```

Y si aparece el conflicto irresoluble (un `transform` heredado encapsula tu overlay), la respuesta moderna no es subir el número: es **usar top layer** — ver abajo.

## Tabla de decisión: `fixed` vs `absolute` vs `sticky`

| Necesito... | Mejor opción | Por qué | Alternativa |
|---|---|---|---|
| Barra siempre visible al hacer scroll | `sticky top: 0` | no deja hueco, respeta el flujo | `fixed` si debe flotar sobre todo |
| Badge/esquina de una card | `absolute` + padre `relative` | superposición localizada | grid con `place-self` + `z-index` |
| Botón flotante "siempre a mano" | `fixed` | vive en la ventana, no en el documento | sticky dentro de una columna lateral |
| Capa/modal de fondo | `<dialog>` (top layer) | foco, backdrop y stacking gratis | `fixed` + `z-index` (a mano) |
| Tooltip sobre un botón | `absolute` relativo al botón | se mueve con el botón | top layer + anchor positioning |
| Cabecera de tabla pegada | `sticky top: 0` en `thead`/`th` | respeta el scroll de la tabla | — |
| Encabezado que se oculta al bajar | `sticky` + estado con JS o scroll-driven | más robusto que animar `fixed` | ver [[Efectos de Scroll]] |
| Corrección óptica de 1–2px | `relative` + `top/left` | no rompe el flujo | `translate` (va a compositor) |

## Top layer: `dialog`, `popover` y compañía

Existe una **capa por encima de todo** el stacking normal. Los elementos de top layer se pintan sobre cualquier `z-index` del documento, **no necesitan números**, y el navegador les da comportamiento de modal de fábrica:

| Elemento/API | Cómo entra a top layer | Extras |
|---|---|---|
| `dialog[open]` | `showModal()` | backdrop, foco atrapado, `::backdrop`, cierre con Esc, `inert` del resto |
| `dialog` (no modal) | `show()` | sin backdrop ni bloqueo |
| `[popover]` | `showPopover()` / atributo `popovertarget` | cierre con Esc o click afuera, `:popover-open` |
| Fullscreen | `requestFullscreen()` | `::backdrop` propio |
| `<select>` nativo, `<input type=date>` | el navegador | — |
| View transitions | durante la transición | `::view-transition-*` |

```html
<button onclick="document.querySelector('dlg').showModal()">Abrir</button>
<dialog id="dlg">
  <form method="dialog"><button>Cerrar</button></form>
</dialog>
```

```css
dialog::backdrop { background: rgb(0 0 0 / 0.5); }
dialog { border: 0; border-radius: 12px; box-shadow: var(--shadow-4); }

/* Popover anclado (2024–2025) */
.menu-btn { anchor-name: --menu-btn; }
.menu {
  position: fixed;
  position-anchor: --menu-btn;
  margin-top: 8px;                 /* separación respecto del ancla */
}
```

La API de anclaje (`anchor-name`, `position-anchor`, `position-area`/`anchor()`) está llegando a soporte amplio (Chrome primero, Firefox/Safari 2024–2025) y elimina los cálculos JS de "posicionar dropdown y voltearlo si no cabe".

> [!tip] Regla moderna de modales
> Un modal ya no se construye con `position: fixed` + `z-index: 9999` + JS de foco: se construye con `<dialog>` (o popover). Te regala stacking correcto, foco atrapado, `inert`, Esc y `::backdrop` — todo lo que antes se programaba a mano.

## Errores comunes

1. **Olvidar que `absolute` busca al ancestor posicionado**: sin un `relative` padre, el elemento se escapa hasta el `body` (initial containing block). Síntoma: "el badge sale del documento".
2. **`fixed` sin dejar espacio**: el contenido de abajo queda tapado. Fix: `padding`/`margin` en el flujo.
3. **Un `transform`/`filter` invisible encapsula el `fixed`**: modal o FAB que se mueven con el contenido. Fix: revisar ancestros con `transform` (incluso `transform: none` aplicado por frameworks) o mudar el elemento a top layer.
4. **`sticky` que nunca se pega**: causas típicas — (a) falta `top`/ancla, (b) un ancestro con `overflow: hidden/auto`, (c) el padre tiene la misma altura que el sticky (sin viaje), (d) `align-self: stretch` en flex/grid.
5. **`z-index` en `static`**: no hace nada (salvo items flex/grid con `z-index`). Síntoma: "el número no pinta".
6. **Números mágicos de z-index**: `9999` contra `10000` es una carrera perdida. Fix: escala de custom properties + top layer para lo crítico.
7. **Usar `position` donde basta `flex`/`gap`**: centrar o alinear con Flexbox es más robusto que con desplazamientos — ver [[Flexbox]] y [[Tipos de Display]].
8. **Anclas tapadas por cabecera sticky**: `#seccion` al hacer click queda debajo de la barra. Fix: `scroll-padding-top` en `html` y/o `scroll-margin` en los targets.
9. **`fixed` + `100vh` en móvil**: salta con la barra de URL. Fix: `100dvh` (ver [[Medidas Relativas]]).

## Accesibilidad

- **El orden del DOM manda**: superponer con `absolute` elementos que tabulan fuera de secuencia confunde la navegación; el teclado recorre el DOM, no el `z-index` (ver [[Flexbox]] para el principio general).
- **Modales: usar `<dialog>` o implementar foco + `inert`**: un overlay `fixed` sin gestión de foco deja tabular "por detrás" del fondo. `showModal()` hace todo: atrapa foco, marca el resto como `inert` y anuncia el diálogo.
- **Indicadores de foco visibles** en todo lo posicionado: dropdowns y tooltips deben abrirse también con foco (`:focus-within`, `:focus-visible`), no solo con hover — ver [[Pseudo Código]].
- **Sticky/fixed que tapan foco**: el foco puede quedar invisible bajo una cabecera fija; `scroll-padding-top` lo corrige también para el foco por teclado.
- **`prefers-reduced-motion`**: ocultar/mostrar navegaciones con animación debe respetarse — ver [[Animaciones]].

## Rendimiento

- `z-index` y `position` en sí mismos son baratos: el costo aparece cuando un SC grande se repinta (sombra, filtro) — ver [[Sombras CSS]].
- `position: fixed` + eventos JS de scroll para animar (clásico "nav que se esconde") es el patrón caro: cada scroll dispara JS + estilos. Alternativas: `position: sticky` puro, o scroll-driven animations (`animation-timeline: scroll()`, ver [[Efectos de Scroll]]) que van por el hilo de composición.
- `backdrop-filter` en overlays fijos recalcula el desenfoque del fondo en cada frame — reservarlo a superficies pequeñas.
- `transform`/`filter` que crean SC "gratis" pueden romper layers esperados; `will-change` debe usarse con medición, no preventivamente (ver [[Animaciones]]).

## Cheatsheet

| Propiedad | Valores | Default | Notas |
|---|---|---|---|
| `position` | `static`, `relative`, `absolute`, `fixed`, `sticky` | `static` | `fixed`/`absolute` salen del flujo |
| `top`/`right`/`bottom`/`left` | longitud, `%`, `auto` | `auto` | ignorados en `static` |
| `inset` | shorthand `top right bottom left` | `auto` | `inset: 0` = cubrir el containing block |
| `inset-block`/`inset-inline` | shorthand lógico | `auto` | writing-mode / RTL safe |
| `z-index` | entero, `auto` | `auto` | requiere posición ≠ static o item flex/grid; crea SC salvo con `auto` |
| `isolation` | `isolate`, `auto` | `auto` | crea SC a propósito (mix-blend) |

| Valor | Sale del flujo | Referencia | Crea SC | Hueco |
|---|---|---|---|---|
| `static` | no | — | no | — |
| `relative` | no | posición original | solo si `z-index` ≠ auto | no |
| `absolute` | sí | ancestor posicionado / ICB | solo si `z-index` ≠ auto | sí |
| `fixed` | sí | viewport (o ancestor transformado) | solo si `z-index` ≠ auto | sí |
| `sticky` | no (hasta pegarse) | scrollport, limitado por padre | solo si `z-index` ≠ auto | no |

| Top layer | API | Backdrop | Foco | Esc/click-out |
|---|---|---|---|---|
| `dialog` modal | `showModal()` | ✅ | atrapado + `inert` | Esc |
| `dialog` simple | `show()` | ❌ | normal | no |
| `popover` | `showPopover()` / `popovertarget` | ❌ | normal | Esc + click afuera |
| fullscreen | `requestFullscreen()` | ✅ | navegador | Esc |

## Relacionadas

- [[Box Model]] — el espacio que cada elemento ocupa antes de moverlo
- [[Flexbox]] — alternativa declarativa a desplazar a mano
- [[Efectos de Scroll]] — efectos que aprovechan `sticky` y `fixed`
- [[Tipos de Display]] — cómo los elementos entran (o no) al flujo
- [[Sombras CSS]] — por qué las capas y el orden se notan en las sombras
- [[Animaciones]] — `transform` como alternativa barata a desplazar con `top/left`
- [[Responsive Web Design]] — `dvh`, safe areas y viewport en elementos fijos
- [[Background]] — `backdrop-filter` en overlays y glassmorphism
