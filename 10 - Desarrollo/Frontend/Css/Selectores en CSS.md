---
tags:
  - desarrollo
  - frontend
  - css
tipo: concepto
estado: terminado
---

# Selectores en CSS

Un selector define **a qué elementos aplica** una regla. Existen muchas formas de escribirlos; este repaso cubre todos los tipos, cómo se combinan, qué conviene usar en cada caso y las herramientas modernas (`:is()`, `:where()`, `:has()`, `@scope`) que cambiaron la forma de escribir selectores mantenibles.

## Selector de elemento (tag)

Selecciona por nombre de etiqueta. Es lo más básico: todos los elementos de ese tipo:

```css
p {
  color: blue;
}
```

![[Pasted image 20250124161224.png]]

Con lo que vemos en la imagen, **todos** los párrafos pasan a ser azules: es tan abstracto que toma todas las etiquetas `p`, y lo mismo con cualquier otra etiqueta.

> [!warning] Máxima alcance, mínima especificidad
> Un selector de elemento llega a todos los `<p>` del documento (incluidos los que no querías) y puntúa 0–0–1 — la especificidad más baja útil. Ver [[Especificidad]].

Elementos especiales que funcionan como tipo: `html`, `body` (cuidado: estilos en `body` se heredan a todo), `:root` (pseudo-clase raíz, mejor para tokens).

## Selector de clase

Selecciona por el atributo `class`, con punto adelante. Las clases son **reutilizables**: se pueden usar las veces que queramos, en la misma página o en varias:

```css
.destacado {
  background: yellow;
}
```

![[Pasted image 20250124161246.png]]

Una clase puede estar en muchos elementos, y un elemento puede tener varias clases (`<p class="destacado urgente">`). Es el selector **por defecto** del CSS moderno y la base de metodologías como BEM — ver [[Estilos para escribir CSS]].

## Selector de ID

Funciona igual que la clase pero con `#` adelante, y **no es reutilizable**: cada id debe ser único en la página, por lo que solo puede aplicar a un elemento:

```css
#principal {
  max-width: 1200px;
}
```

![[Pasted image 20250124161315.png]]

> [!warning] No uses IDs para estilos
> Su especificidad (1–0–0) es tan alta que pisarlos desde otras hojas es un calvario, y la unicidad impide reutilizarlos. Regla práctica: **ids para JavaScript/`label`, clases para CSS**. El único uso CSS legítimo que se le perdona al `#` es como ancla de posicionamiento (`anchor-name`) o `:target`.

## Selector de atributo

Selecciona elementos según el valor (o presencia) de un atributo:

```css
input[type="text"] { border: 1px solid #ccc; }
a[target="_blank"] { color: #7f5af0; }
a[href^="https"] { padding-right: 1em; }   /* ^ = empieza con */
a[href$=".pdf"] { color: red; }            /* $ = termina con */
a[href*="github"] { font-weight: bold; }   /* * = contiene */
```

![[Pasted image 20250124161348.png]]

Al principio parece extraño, pero con tiempo y práctica se comprende y se usa bastante (especialmente con formularios y enlaces externos).

### Todos los operadores de atributo

| Sintaxis | Significado | Ejemplo |
|---|---|---|
| `[attr]` | el atributo **existe** (cualquier valor) | `input[required]` |
| `[attr="val"]` | valor **exacto** | `[type="checkbox"]` |
| `[attr~="val"]` | el valor es una **lista** que contiene `val` como palabra | `[class~="btn"]` equivale a `.btn` |
| `[attr\|="val"]` | el valor es exacto o empieza con `val-` | `[lang\|="es"]` matchea `es`, `es-AR` |
| `[attr^="val"]` | **empieza** con | `[href^="https://"]` |
| `[attr$="val"]` | **termina** con | `[src$=".webp"]` |
| `[attr*="val"]` | **contiene** en cualquier posición | `[data-testid*="modal"]` |

Detalles que casi nadie conoce:

- **Sensibilidad a mayúsculas**: por defecto depende del atributo y del doctype. Se fuerza con los modificadores **`i`** (insensible) y **`s`** (sensible): `[href^="HTTPS" i]`, `[lang="es-AR" s]`.
- **Comillas recomendadas**: `[type=text]` funciona, pero `[class=foo bar]` no; usar comillas siempre evita sorpresas y es más legible.
- **Sintaxis de escape**: `[data-icon="attr\\:value"]`, `[href^="https://"]` — los caracteres estructurales van escapados.
- **`data-*` como API de styling**: el HTML moderno usa atributos datos para variantes (`data-estado="cargando"`, `data-tema="oscuro"`), y CSS los estiliza sin tocar clases:

```css
[aria-pressed="true"] { background: #dbeafe; }   /* botón toggle activo */
[data-estado="cargando"] { opacity: 0.6; pointer-events: none; }
[data-tema="oscuro"] .card { background: #1e293b; }
[hidden] { display: none; }                       /* refuerzo defensivo */
```

## Combinadores: cómo se relacionan los selectores

### Descendiente (espacio)

Selecciona cualquier descendiente de un elemento, a cualquier profundidad — por ejemplo, la clase `nombre` **dentro de** la clase `cliente`:

```css
.cliente .nombre {
  font-weight: bold;
}
```

![[Pasted image 20250124161421.png]]

Busca la clase `cliente` y, dentro de todo su árbol, aplica a los `.nombre`.

### Hijo directo (`>`)

Solo modifica los hijos **inmediatos** de una clase, id o etiqueta:

```css
.cliente > p {
  margin: 0;
}
```

![[Pasted image 20250124161442.png]]

Dice: modificará a todos los párrafos que sean **hijos directos** de `.cliente` — los nietos quedan afuera. Se puede usar con cualquier etiqueta.

### Hermanos

```css
h2 + p { margin-top: 0; }  /* + : el hermano inmediato siguiente */
h2 ~ p { color: #555; }    /* ~ : todos los hermanos siguientes */
```

| Combinador | Nombre        | Relación                              |
| ---------- | ------------- | ------------------------------------- |
| ` `        | Descendiente  | Cualquier nivel por dentro            |
| `>`        | Hijo          | Nivel inmediato                       |
| `+`        | Hermano adyacente | El siguiente directo               |
| `~`        | Hermanos      | Todos los que siguen                  |

### Combinadores relativos (dentro de `:has()` y `@scope`)

`:has()` admite que el argumento **empiece por combinador**, lo que lo vuelve relativo al elemento base: `:has(> img)` (hijo directo img), `:has(+ p)` (le sigue un `p`), `:has(~ p)` (siguen `p`). Es "el descendiente al revés": el `+` y `~` ahora pueden seleccionar **hacia atrás** (el hermano previo o el ancestro) vía `p:has(+ .destacado)`.

## Selector universal

```css
* { box-sizing: border-box; }   /* todo elemento (0 puntos de especificidad) */
```

`*` no tiene especificidad propia (suma 0), pero **el elemento que matchea sí conserva la suya** al heredar. Variantes modernas:

```css
:where(*) { margin: 0; }          /* reset sin especificidad */
:where(*)::before, :where(*)::after { box-sizing: border-box; }

/* Universal bajo un contexto */
.menu > * { flex: 1; }            /* todos los hijos directos del menú */
```

## Selector de lista (coma)

La coma **agrupa selectores independientes** que comparten el mismo bloque de declaraciones. No es un combinador: la regla se aplica a cada selector por separado, con la especificidad de **cada uno**, no una combinada:

```css
h1, h2, h3, .titulo-seccion { text-wrap: balance; }
```

> [!warning] La lista de selectores es estricta
> Un selector inválido en la lista invalida **toda la regla**: `.card, , .btn` (con la coma doble) silencia las dos reglas legítimas. La excepción son las listas *dentro* de `:is()`/`:where()`, que son "forgiving": ahí los selectores inválidos se descartan y el resto funciona.

## Pseudo-clases y pseudo-elementos

`:hover`, `:nth-child()`, `:has()` y `::before` viven en su propia nota, con catálogo completo — ver [[Pseudo Código]]. Resumen de puntación para ubicarlos: pseudo-clase = 0–1–0, pseudo-elemento = 0–0–1.

## Herramientas modernas: `:is()`, `:where()`, `:has()`

### `:is()` — agrupar sin repetir contexto

```css
/* Antes: tres reglas con contexto repetido */
nav a:hover, header a:hover, footer a:hover { color: blue; }

/* Después */
:is(nav, header, footer) a:hover { color: blue; }
```

Su especificidad es la del **argumento más específico** de la lista: `:is(#nav, .menu)` puntúa como `#nav`.

### `:where()` — el mismo agrupador con especificidad cero

Idéntico a `:is()` salvo que **siempre puntúa 0**. Es la herramienta para estilos base sobreescribibles:

```css
/* Base reemplazable: cualquier .clase posterior le gana sin !important */
:where(h1, h2, h3) { margin-block: 1.25rem; font-weight: 600; }
:where(a) { color: inherit; text-decoration: underline; }

.hero h1 { font-weight: 800; }   /* gana por orden, no por especificidad */
```

> [!tip] El patrón de capas con `:where()`
> Los resets modernos y las bases de design systems usan `:where()` para que el "estilo por defecto" tenga especificidad 0 y cualquier componente o utilidad lo pise naturalmente. Es la alternativa sana al `!important` — ver [[Especificidad]] y [[Normalize]].

### `:has()` — estilizar según lo que haya adentro o alrededor

```css
/* La card cambia de layout si tiene imagen */
.card:has(> img) { grid-template-rows: 200px auto; }

/* El campo se marca en rojo si su input es inválido */
.campo:has(:invalid) { border-color: red; }

/* El párrafo previo a un destacado gana ritmo */
p:has(+ .destacado) { margin-bottom: 2rem; }

/* El menú se dibuja cuando alguien tabula dentro */
.menu:has(a:focus-visible) { outline: 2px solid dodgerblue; }

/* Estado vacío */
.listado:not(:has(> *)):not(:empty)::after {
  content: "Sin resultados";
  color: #94a3b8;
}
```

Patrones de producto reales con `:has()`:

| Patrón | Selector | Efecto |
|---|---|---|
| Formulario con errores | `.formulario:has(:user-invalid)` | banner "revisá los campos" |
| Modal que oscurece | `body:has(dialog[open])` | `overflow: hidden` del body sin JS |
| Sticky shadow al hacer scroll | `body:not(:has(:root .scrolled))`* | sombra en cabecera al cambiar estado |
| Sidebar que se contrae | `.layout:has(.sidebar[colapsada])` | una sola regla en el contenedor |
| Carrito con items | `.carrito:has(.item)` | oculta el "vacío" |
| Grid con sidebar | `.grid:has(> .sidebar)` | cambia `grid-template-columns` |

\* el estado de scroll se agrega con una línea de JS o con scroll-driven animations (ver [[Efectos de Scroll]]) — `:has()` lee el DOM, no el scroll.

## `@scope`: aislar estilos por alcance

Las hojas CSS globales no tienen bordes: `.titulo` estiliza cualquier `.titulo` de la página. `@scope` delimita selectores a un subárbol — un sistema de encapsulado intermedio entre clases globales y web components:

```css
@scope (.popover) {
  :scope { box-shadow: 0 8px 24px rgb(0 0 0 / 0.15); }  /* la raíz del scope */
  button { font: inherit; }                              /* solo botones dentro */
  .titulo { font-weight: 600; }                          /* solo .titulo del scope */
}

/* Con límite de parada: aplica hasta encontrar .fin */
@scope (.articulo) to (.pie-articulo) {
  a { color: rebeccapurple; }
}
```

| Concepto | Significado |
|---|---|
| Raíz del scope | El selector entre paréntesis inicial; delimita el subárbol |
| `:scope` | La raíz misma del scope (como `:root` pero del scope) |
| Límite de parada (`to`) | El alcance se corta al encontrar el selector final (como "hasta") |
| Proximidad | Si dos scopes matchean, gana el **más cercano** al elemento (regla de proximidad, aún parcial en algunos motores) |

Casos de uso: estilos de widgets insertados dinámicamente sin prefijar cada clase, aislamiento de componentes de terceros, "shadow DOM sin shadow DOM". Soporte: Chrome 118, Firefox 128, Safari 17.4 (2023–2024) — usar con progressive enhancement.

## Recetas prácticas

### 1. Filas alternas y ritmos con `nth-child` / `nth-of-type`

```css
/* Zebra clásica */
.tabla tbody tr:nth-child(even) { background: #f6f7f9; }

/* Separadores cada 4 filas (patrón de listas tipo editorial) */
.feed > li:nth-child(4n + 1) { border-top: 2px solid #333; }

/* Los 3 primeros destacados */
.destacado:nth-child(-n + 3) { font-size: 1.25rem; }
```

### 2. Tarjetas y layouts reactivos al contenido

```css
/* Fila con imagen: miniatura a la izquierda */
.fila:has(> img) { display: grid; grid-template-columns: 96px 1fr; gap: 1rem; }

/* Último item del grupo sin borde */
.item:not(:last-child) { border-bottom: 1px solid #e2e8f0; }
/* (preferible a .item:last-child { border: 0 } por menor alcance) */
```

### 3. Enlaces por destino

```css
a[href^="http"]:not([href*="mi-dominio.com"])::after {
  content: " ↗";
  font-size: 0.8em;
}
a[href^="mailto:"] { color: inherit; }
a[download]::before { content: "⬇ "; }
```

### 4. Estados de formulario sin clases extra

```css
input:placeholder-shown + label { transform: translateY(0.5rem); font-size: 0.8rem; }
input:user-invalid { border-color: #dc2626; }
input:valid:not(:placeholder-shown) { border-color: #16a34a; }
```

### 5. Agrupación temática con `:is()`

```css
:is(h1, h2, h3, h4) { text-wrap: balance; }
:is(.alerta, .exito, .info) { padding: 1rem; border-radius: 8px; }
:is(button, [role="button"], a.boton):focus-visible { outline: 2px solid currentColor; }
```

### 6. Base con `:where()` + componentes con clase

```css
:where(.tabla) th { text-align: start; font-weight: 600; }   /* base */
.tabla--compacta th { padding: 0.25rem; }                    /* pisa la base sin pelear */
```

## Recomendaciones de uso

1. **Preferí clases**: reutilizables, especificidad plana (0–1–0) y fáciles de mantener. Selectores de 4+ segmentos son una señal de que falta una clase (o de que BEM te resolvería la vida — ver [[Estilos para escribir CSS]]).
2. **Evitá IDs para CSS** y selectores larguísimos: cuantos más elementos encadenás, más difícil es reutilizar y sobreescribir.
3. **No estilizas desde JavaScript**: agregá/quitar clases y dejá el estilo en el CSS.
4. **Usá `>` con intención**: `nav > ul > li` es más seguro que `nav ul li` cuando el HTML anida widgets, pero también más frágil ante refactorizaciones de markup. Elegir un estilo y ser consistente.
5. **Conocé la puntuación de cada selector** — [[Especificidad]] decide qué regla gana; `:where()` es tu válvula de escape para bajarla.

## Errores comunes

1. **Regla silenciada por lista inválida**: una coma de más invalida toda la regla. Síntoma: "ninguno de los dos selectores funciona".
2. **`nth-child` esperando `nth-of-type`** (o viceversa) — ver [[Pseudo Código]].
3. **Especificar de más** (`.layout .sidebar .widget .titulo`): especificidad alta = imposible de sobreescribir desde componentes. Fix: clases planas, `:where()` para bases.
4. **Atributos sin comillas y con espacios**: `[class=foo bar]` no matchea. Fix: comillas siempre.
5. **Confundir `~` con `+`**: `+` es el **inmediato** siguiente; `~` todos los siguientes. Con `h2 + p`, dos `p` seguidos tras el `h2` solo matchean el primero.
6. **Depender de la proximidad de `@scope` en motores viejos**: verificar soporte o degradar gracefully.
7. **Selectores que asumen HTML exacto**: `.menu > ul > li > a` rompe con cualquier refactor del markup. Los selectores son un contrato con el HTML — cambiar el HTML rompe CSS, y eso es esperable; documentar el contrato.

## Accesibilidad

- Los selectores no modifican el árbol de accesibilidad: estilizar con CSS **nunca** es sustituto de semántica correcta (un `<div role="button">` estilizado como botón no es un botón).
- `:focus-visible` debe estar presente en todo elemento interactivo — no cubrirlo con selectores menos específicos.
- Cuidado con `:has()` para revelar contenido: si la revelación es solo visual (CSS), el contenido ya existe en el DOM y el teclado puede alcanzar zonas "ocultas" — combinar con `[hidden]`/`inert` cuando corresponda (ver [[Tipos de Posiciones]] para `inert` y top layer).
- Contenido generado con `::after` + `attr()`: poco fiable para lectores de pantalla — ver [[Pseudo Código]].

## Compatibilidad

| Herramienta | Chrome | Firefox | Safari |
|---|---|---|---|
| `:is()` / `:where()` | 88 | 84 | 14 |
| `:not()` simple | 4 | 1 | 9 |
| `:has()` | 105 | 121 | 15.4 |
| `:nth-child(of S)` | 111 | 112 | 9 |
| `@scope` | 118 | 128 | 17.4 |
| Atributos `i`/`s` | 49 | 46 | 9.1 |

## Cheatsheet

| Selector | Ejemplo | Especificidad | Nota |
|---|---|---|---|
| Universal | `*` | 0 | suma 0 |
| Tipo | `p`, `h1` | 0–0–1 | alcance total |
| Clase | `.card` | 0–1–0 | el default |
| ID | `#main` | 1–0–0 | evitá en CSS |
| Atributo | `[type="text"]` | 0–1–0 | 7 operadores + `i`/`s` |
| Descendiente | `.a .b` | suma | cualquier nivel |
| Hijo | `.a > .b` | suma | nivel inmediato |
| Adyacente | `.a + .b` | suma | siguiente directo |
| Hermanos | `.a ~ .b` | suma | todos los siguientes |
| Lista | `h1, .a` | por selector | inválido rompe todo |
| Pseudo-clase | `:hover` | 0–1–0 | ver [[Pseudo Código]] |
| Pseudo-elemento | `::before` | 0–0–1 | ver [[Pseudo Código]] |
| `:is()` | `:is(h1, .a)` | máx. arg. | forgiving |
| `:where()` | `:where(h1)` | **0** | truco de bases |
| `:not()` | `:not(.a)` | máx. arg. | estricto |
| `:has()` | `.a:has(> img)` | máx. arg. | selector relativo |
| `@scope` | `@scope (.w) {}` | según selector | aislamiento por alcance |

## Relacionadas

- [[Especificidad]] — la puntuación que decide qué selector gana
- [[Anatomía de CSS]] — cómo se estructura selector + declaración
- [[Pseudo Código]] — pseudo-clases y pseudo-elementos en profundidad
- [[Estilos para escribir CSS]] — BEM y naming para selectores mantenibles
- [[CSS Grid]] — selectores de hijos directos sobre `grid-template-areas`
- [[Normalize]] — por qué `:where(*)` se usa en los resets modernos
- [[Efectos de Scroll]] — combinaciones de `:has()` con estados de scroll
