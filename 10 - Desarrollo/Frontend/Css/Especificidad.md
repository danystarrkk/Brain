---
aliases: ["Specificity", "Cascada y especificidad"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Selectores en CSS]]"
  - "[[Anatomía de CSS]]"
  - "[[Estilos para escribir CSS]]"
  - "[[Sintaxis de Sass]]"
  - "[[CSS Introducción]]"
  - "[[TailwindCSS - Sintaxis Básica]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/Specificity"
---

# Especificidad

La **especificidad** es la puntuación que asigna el navegador a cada selector para decidir **qué regla gana** cuando varias apuntan al mismo elemento y la misma propiedad. Es la segunda ronda de la cascada (la primera es el **origen e importancia** de la hoja de estilo; ver [[CSS Introducción]]): ganan los estilos `!important` y los de mayor origen primero, y **dentro de esa pelea**, decide la especificidad.

> [!info] Dónde vive la especificidad en el orden de resolución
> Para una propiedad disputada, el navegador aplica en orden: **1)** origen + importancia (user agent < user < autor < `!important`…), **2)** **especificidad del selector**, **3)** orden de aparición (gana la última). Es decir: la especificidad solo desempata entre reglas del **mismo origen e importancia**. Un `!important` del autor gana siempre a cualquier selector normal, por muy específico que sea.

## El ejemplo clásico

```css
h1.titulo span {
  color: green;
}

.titulo span {
  color: red;
}
```

La regla 1 apunta al `span` dentro de un `h1` que tenga la clase `titulo`; la regla 2, al `span` dentro de cualquier `titulo`. Por la cascada pura "gana la última", pero la regla 2 es **más genérica**: su especificidad es menor y **no puede** pisar a la primera. El `span` dentro de `h1.titulo` queda **verde**.

## Cómo se calcula

Cada selector se puntúa como una **tupla de 3 niveles** (A, B, C), como un número de 3 dígitos:

| Nivel | Componente | Ejemplos | Puntos |
|---|---|---|---|
| A | **IDs** (`#`) | `#header` | 1–0–0 |
| B | **Clases, atributos, pseudo-clases** | `.card`, `[type="text"]`, `:hover` | 0–1–0 |
| C | **Elementos y pseudo-elementos** | `p`, `div`, `::before` | 0–0–1 |

```css
p                     { } /* 0–0–1 */
p span                { } /* 0–0–2 */
.titulo               { } /* 0–1–0 */
h1.titulo             { } /* 0–1–1 */
#nav .item a          { } /* 1–1–1 — gana a todo lo anterior */
```

Se comparan **de izquierda a derecha**: quien gana en A no se discute; solo si empatan A se mira B, y así. A igualdad total, empata la **última declarada** en el archivo.

> [!warning] Los números no son decimales
> `0–1–0` (una clase) le gana a `0–0–12` (doce elementos): primero se miran las clases. Y ninguna cantidad de clases alcanza a un id: 1 id (1–0–0) gana a 50 clases (0–50–0). El orden de importancia es **ID > clase/atributo/pseudo-clase > elemento/pseudo-elemento**.

### La notación de cuatro columnas (especificación W3C)

La especificación usa una tupla de **4** valores `(a, b, c, d)` donde `a` es el nivel del **estilo en línea**:

| Valor | Nivel | Ejemplo | Tupla W3C |
|---|---|---|---|
| a | Estilo **en línea** (`style=""`) | `<p style="color:red">` | 1–0–0–0 |
| b | IDs | `#nav` | 0–1–0–0 |
| c | Clases/atributos/pseudo-clases | `.item:hover` | 0–0–1–0 |
| d | Elementos/pseudo-elementos | `p::before` | 0–0–0–1 |

En la práctica se usa la notación de 3 columnas y el inline style se recuerda como *"por encima de cualquier selector"* (salvo un `!important` ajeno, que lo pisa).

## Puntuación de cada tipo de selector

| Selector | Specificity | Nota |
|---|---|---|
| `*` (universal) | 0–0–0 | **No suma nada** |
| `p`, `div`, `h1` | 0–0–1 | tipo |
| `::before`, `::after` | 0–0–1 | pseudo-**elemento** = tipo |
| `p::first-line` | 0–0–2 | tipo + pseudo-elemento |
| `.card`, `[disabled]`, `:hover`, `:nth-child(2n)`, `:focus-visible` | 0–1–0 | pseudo-**clase** = clase |
| `input[type="text"]` | 0–1–1 | atributo cuenta como clase |
| `.card:hover` | 0–2–0 | se suman las partes |
| `nav .item a` (contexto) | 0–1–2 | todas las partes se suman |
| `#nav` | 1–0–0 | id |
| `a#logo` | 1–0–1 | id + tipo |

### `:is()`, `:not()`, `:has()`, `:where()`: la especificidad de los selectores funcionales

| Función | Aporta | Regla |
|---|---|---|
| `:is()` | La de su **argumento más específico** | `:is(h1, #tit)` → 1–0–0 |
| `:not()` | La de su argumento **más específico** | `:not(.a, .b.c)` → 0–2–0 |
| `:has()` | La de su argumento **más específico** | `:has(.foto)` → 0–1–0 |
| `:where()` | **Siempre 0** | `:where(.reset) h1` → 0–0–1 |

```css
:where(.reset) h1 { }   /* 0–0–1: el :where() no suma */
:is(.a, #b) span { }    /* 1–0–1: toma el más alto de .a y #b */
.form:has(:invalid) { } /* 0–2–0: :invalid dentro aporta 0–1–0 */
```

> [!tip] `:where()` es la herramienta de arquitectura
> Un reset o estilos base con `:where()` quedan **fácilmente pisables**: su especificidad es 0, así que cualquier regla normal del proyecto gana sin trucos. Es el motivo por el que resets modernos (como modern-normalize) envuelven sus selectores en `:where()`.

### Duelos frecuentes: quién gana

| Duelo | Ganador | Motivo |
|---|---|---|
| `#nav .item` vs `.nav .item.activo` | `#nav .item` (1–1–0 vs 0–2–0) | El id decide en la primera columna |
| `a` vs `a:hover` | `a:hover` | pseudo-clase > tipo |
| `.btn` vs `button` | `.btn` | clase > tipo |
| `.a .b` vs `.c` | `.a .b` (0–2–0 vs 0–1–0) | contexto suma |
| `#x` vs 100 × `.y` | `#x` | 1–0–0 > 0–100–0 |
| `:where(#x)` vs `p` | `p` (0–0–1 vs 0–0–0) | `:where` no suma |
| `:is(#x)` vs `p` | `:is(#x)` (1–0–0 vs 0–0–1) | `:is` toma el máximo |
| `.a` (en capa `base`) vs `.b` (en capa `utilidades`) | `.b` | la capa decide **antes** que la especificidad |

### Una pelea resuelta paso a paso

```css
@layer base, utilidades;

@layer base {
  p { color: gray; }            /* 0–0–1 */
}
p.legal { color: teal; }        /* sin capa, 0–1–0 */
@layer utilidades {
  .texto-rojo { color: red; }   /* 0–1–0 */
}
```

Un `<p class="legal texto-rojo">` se resuelve así:

1. Todas son **normales del autor** → misma ronda de origen.
2. Sin `!important` → compite especificidad + capa.
3. Capas primero: lo **sin capa** (`.legal`, 0–1–0) le gana a **cualquier** declaración en capa (la capa `base` 0–0–1 y la `utilidades` 0–1–0 pierden ambas).
4. **Ganador: `teal`**. Ni la utilidad roja (misma especificidad, pero en capa) ni el `gray` de base pueden cambiarlo sin otra declaración.

### Anidamiento (CSS nativo y Sass)

Las reglas anidadas se resuelven al selector **completo**, así que **heredan la especificidad del padre**:

```css
.card {                    /* 0–1–0 */
  h2 { color: navy; }     /* se resuelve como .card h2 → 0–1–1, no 0–0–1 */
  &:hover { }             /* .card:hover → 0–2–0 */
}
```

Cada nivel de anidamiento suma — el motivo por el cual la anidación profunda de preprocesadores dispara la especificidad sin que te des cuenta (ver [[Sintaxis de Sass]]).

## `!important`: la banda que salta la cascada

`!important` no es especificidad: es una **marca de importancia** que compite en un nivel aparte (ver el orden completo en [[CSS Introducción]]).

```css
.alerta { color: red !important; } /* gana contra cualquier selector normal */
```

Cómo funciona en la práctica:

- Dentro del mismo origen, entre declaraciones `!important`, **también aplica especificidad** y orden: el `!important` más específico gana.
- Para **deshacer** un `!important` necesitás **otro** `!important` de mayor especificidad (o posterior a igual). Es una escalada que no termina sola.
- `!important` del **usuario** le gana al `!important` del autor — así un usuario puede forzar estilos de accesibilidad por encima del sitio.
- Las **transiciones** ganan temporalmente incluso a `!important` (lo que ves mientras la animación corre es el valor interpolado).

**Usos legítimos acotados**: estilos de accesibilidad del usuario, utilidades puntuales ("esta clase SIEMPRE rompe todo"), y overrides de librerías de terceros que no controlás. Todo lo demás se resuelve mejor con arquitectura (ver abajo). Ver [[Estilos para escribir CSS]] para metodologías que evitan este problema desde el diseño del naming.

## Cascade layers: orden explícito ANTES de la especificidad

Las capas (`@layer`) resuelven el orden **por encima** de la especificidad y son la respuesta moderna a la escalada:

```css
@layer base, componentes, utilidades;   /* orden declarado: gana el final */

@layer base {
  a { color: blue; }        /* cualquier enlace, 0–0–1 pero pierde por capa */
}
@layer componentes {
  .btn { color: teal; }     /* 0–1–0 */
}
@layer utilidades {
  .text-red { color: red; } /* 0–1–0 pero gana: capa posterior */
}
```

Reglas de prioridad (dentro del **origen autor**):

| Tipo de declaración | Orden de prioridad (menor → mayor) |
|---|---|
| Normal | capa 1 → capa 2 → … → **sin capa** (lo último, lo no-agrupado gana todo lo normal) |
| `!important` | **sin capa** → … → capa 2 → capa 1 (**el orden de capas se invierte**) |

Dos consecuencias importantes:

1. Dentro de **una misma capa**, todo lo de arriba aplica igual (especificidad, orden).
2. Un `!important` en la **primera** capa le gana a un `!important` en la última: es la forma de fijar "lo que nunca se toca" al principio de la arquitectura.

> [!tip] La arquitectura completa en tres reglas
> 1. Base (resets, tipografía) en `@layer base` **envuelta en `:where()`** → especificidad 0.
> 2. Componentes en `@layer componentes` → especificidad plana BEM.
> 3. Utilidades en `@layer utilidades` → todas 0–1–0, última capa.
> Cualquier conflicto se resuelve por **orden de capa**, no por pelea de selectores. Es exactamente el modelo de Tailwind (ver [[TailwindCSS - Sintaxis Básica]]).

## Herencia: el otro mecanismo (no es especificidad)

Confundir cascada con herencia es común. La **herencia** es el flujo de ciertas propiedades (`color`, `font-family`, `line-height`…) desde el padre a los hijos **sin selector de por medio**. No compite con la especificidad: es un canal distinto que solo aplica si ninguna regla (aunque sea genérica) define la propiedad en el hijo.

```css
body    { font-family: Georgia; }
#app   { color: navy; }        /* #app 1–0–0 ... */
#app p { }                     /* 1–0–1: los párrafos heredan el color navy
                                  sin necesidad de que esta regla lo diga */
```

- La especificidad de un selector **no se hereda**: que `#app` tenga 1–0–0 no ayuda a sus descendientes — lo que heredan es el **valor computado**, no la puntuación.
- `inherit` (y el resto de palabras globales `initial`, `unset`, `revert`, `revert-layer`) intervienen justo acá: fuerzan el flujo de herencia cuando la cascada no lo haría (tabla completa en [[CSS Introducción]]).

## Estrategias para ganar batallas sin escalar

Hay dos direcciones para resolver un conflicto, y la tercera es reescribir la arquitectura:

1. **Subir la especificidad** del selector propio: `.nav .item.activo` en vez de `.activo`. Funciona, pero deuda técnica: cada regla que quieras sobreescribir después necesita aún más puntos.
2. **Bajar la de la competencia**: envolver los estilos ajenos en `:where()` (si controlás el CSS) o usar capas.
3. **Reorganizar con `@layer`**: el orden de importancia se declara una vez y todos los conflictos futuros se resuelven por capa.

```css
/* ejemplo integrado: reset pisable + componente plano + utilidad ganadora */
@layer base, componentes, utilidades;

@layer base {
  :where(button) { background: #eee; border: 1px solid #999; }  /* 0–0–1 */
}
@layer componentes {
  .boton--primario { background: teal; }   /* 0–1–0 */
}
@layer utilidades {
  .bg-rojo { background: crimson !important; } /* capa más alta, y por las dudas */
}
```

## Debugging en DevTools

Las DevTools muestran la puntuación de especificidad al hacer hover sobre un selector en la pestaña Styles, y en el panel Computed podés ver qué regla ganó para cada propiedad:

1. Clic derecho al elemento → *Inspeccionar*.
2. Pestaña **Styles**: reglas tachadas con la razón (`overridden by…`); hover sobre el selector muestra su tupla.
3. Panel **Computed**: expandí la propiedad disputeada: muestra la cadena de herencia y la regla ganadora.
4. Para `@layer`, Styles muestra el nombre de capa bajo cada regla — muy útil para detectar orden mal declarado.
5. `Ctrl/Cmd + Shift + C` para inspeccionar al vuelo un elemento difícil de capturar (hover states).

## Errores comunes y malentendidos

1. **"Gana la última regla"** sin mirar especificidad — la orden solo desempata a especificidad igual.
2. **Contar elementos y clases como si fueran decimales** (`0–1–0` > `0–0–12`).
3. **Pensar que la especificidad vence a `!important`**: importancia/origen decide **antes**; la especificidad solo ordena dentro del mismo nivel.
4. **Subir especificidad "por las dudas"**: `#main .nav ul li a` acumula puntos que después obligan a hacks para sobreescribir. Si el selector necesita más de ~2 combinadores, probablemente el problema es de naming (ver [[Estilos para escribir CSS]]).
5. **Olvidar que el estilo en línea está por encima de todos los selectores** (salvo `!important`): depender de `style=""` en HTML complica cualquier theming.
6. **Sorpresa con `:where()`**: `.a:where(#b) { }` es 0–1–0, no 1–1–0 — o al revés, creer que `:where` no aporta nada cuando se usa **como argumento** de `:is()` (ahí sí suma).
7. **Anidar "solo por legibilidad"** en Sass/CSS nativo y multiplicar especificidad sin querer — el selector resuelto suma el padre completo.
8. **Atributos como clases**: `[type="checkbox"]` es 0–1–0 — quien diseña estrategias "sin clases" con atributos no gana especificidad.
9. **Pensar que `*` protege**: aporta 0–0–0, no pisa nada.
10. **Mezclar capas y especificidad al revés**: esperar que `.utilidad` (0–1–0) gane a un selector sin capa (0–0–1) — la capa sin declarar **gana** a cualquier capa en declaraciones normales, aunque su selector sea menos específico.

## Cheatsheet — especificidad de todo

| Selector | Tupla (A–B–C) | Nota |
|---|---|---|
| `*` | 0–0–0 | nada |
| `li` | 0–0–1 | tipo |
| `ul li` | 0–0–2 | contexto suma |
| `li::marker` | 0–0–2 | pseudo-elemento = tipo |
| `.btn` | 0–1–0 | clase |
| `:hover`, `:nth-child()`, `:not(.x)` | 0–1–0 | pseudo-clase / :not con arg. clase |
| `[aria-current="page"]` | 0–1–0 | atributo = clase |
| `.btn:hover` | 0–2–0 | suma |
| `#app` | 1–0–0 | id |
| `#app .nav a` | 1–1–1 | suma |
| estilo en línea `style=""` | 1–0–0–0 (W3C) | por encima de cualquier selector |
| `:where(...)` | lo que sea, **+0** | envoltorio transparente |
| `:is()/:not()/:has(...)` | máx. de sus argumentos | nunca suma "de más" |

| Mecanismo | Cuándo decide | Dónde se ve |
|---|---|---|
| Origen + importancia | 1ª ronda | Ver [[CSS Introducción]] |
| **Especificidad** | 2ª ronda (mismo origen/importancia) | esta nota |
| Orden de aparición | 3ª ronda (empate) | última regla gana |
| `@layer` | **Antes** de la especificidad | orden declarado; `!important` lo invierte |
| Herencia | Si nadie definió la propiedad | canal separado, sin selectores |

## Compatibilidad

Specificidad, `:is()`, `:not()`, `:where()`, `:has()` y `@layer` son estándar en todos los motores modernos (2026). Históricamente, `:has()` fue el último en llegar (Chrome 105, Safari 15.4, Firefox 121, 2023-2024) — antes se simulaba con JS. `:where()` existe desde 2020-2021 en los tres motores.

## Relacionadas

- [[Selectores en CSS]] — todos los selectores y su sintaxis
- [[Anatomía de CSS]] — estructura de las reglas
- [[CSS Introducción]] — el lugar exacto de la especificidad en la cascada completa
- [[Estilos para escribir CSS]] — BEM y compañía para mantener la especificidad plana
- [[Sintaxis de Sass]] — por qué la anidación de Sass dispara la especificidad
- [[TailwindCSS - Sintaxis Básica]] — utilidades 0–1–0 en la última capa: especificidad resuelta por arquitectura
