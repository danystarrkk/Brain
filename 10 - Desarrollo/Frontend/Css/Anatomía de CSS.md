---
tags:
  - desarrollo
  - frontend
  - css
tipo: concepto
estado: terminado
---

# Anatomía de CSS

Toda regla de CSS se compone de un **selector** (qué elemento afecta) y un bloque de **declaraciones** (qué cambia). Entender esta estructura —y los tokens con los que se construye— es el cimiento para leer, escribir y depurar cualquier hoja de estilo. Esta nota desarma el lenguaje hasta su nivel más bajo: qué es un *token*, qué es una *regla*, qué pasa cuando el navegador encuentra un error y cómo se conecta todo con la cascada ([[Especificidad]]).

> [!info] Un poco de contexto
> CSS (*Cascading Style Sheets*) nació en 1996 (CSS1) para separar presentación de contenido en HTML. CSS2 (1998) agregó posicionamiento y z-index; a partir de ahí el estándar se **modularizó**: hoy no existe "CSS3" como versión única, sino módulos independientes que evolucionan por separado (Selectors, Color, Flexbox, Grid, Animations…). Por eso en MDN cada propiedad tiene su propia página y su propia fecha de compatibilidad.

## La estructura básica

```css
p {
  color: blue;
}
```

- **`p`** es el **selector**: apunta a todas las etiquetas `<p>` del HTML (ver [[Etiquetas]] y [[Selectores en CSS]] para la lista completa de selectores).
- **`color: blue;`** es una **declaración**, formada por:
  - **propiedad** (`color`): qué aspecto queremos controlar.
  - **valor** (`blue`): cómo queda ese aspecto.
- Cada declaración termina en **punto y coma `;`**. La única declaración sin `;` es la última *antes* de la llave de cierre, pero escribirla siempre es mejor hábito: agregar una nueva declaración después ya no obliga a recordar el `;` anterior.

```css
/* Varias declaraciones en un mismo bloque */
.card {
  color: #1a1a2e;    /* texto */
  padding: 1rem;     /* espacio interior */
  border-radius: 8px; /* esquinas redondeadas */
}
```

### Selector y bloque

```
.card { color: blue; }
└─┬─┘ └────────┬────────┘
selector    bloque de declaraciones
```

El selector puede ser simple (`p`), combinado (`.card .titulo`) o incluir pseudo-clases y pseudo-elementos (`a:hover`, `p::first-line`). Entre selectores se usan **comas** para aplicar las mismas declaraciones a varios elementos a la vez:

```css
h1, h2, h3 {
  font-family: Georgia, serif;
}
```

> [!tip] La coma separa selectores, no "elementos"
> `h1, h2` son **dos reglas distintas** que comparten bloque — el navegador los trata como si cada uno tuviera su propia copia de las declaraciones. Esto importa con la cascada: dos selectores en coma no "se suman" en especificidad; cada uno compite por su cuenta (ver [[Especificidad]]).

### Anatomía de una regla completa

```css
@media (min-width: 40rem) {     /* at-rule + bloque condicional */
  .card > .titulo:not(.oculto)  /* selector con combinador y pseudo-clase */
  {                              /* bloque de declaraciones */
    color: var(--texto);         /* propiedad: valor (función var) */
    font-size: 1.25rem; !important
  }
}
```

Los cinco componentes posibles de una regla:

| Componente | Ejemplo | Rol |
|---|---|---|
| **Selector** | `.card > .titulo` | qué elementos matchea |
| **Bloque** | `{ ... }` | agrupa declaraciones |
| **Propiedad** | `color` | el aspecto a controlar |
| **Valor** | `var(--texto)` | el estado del aspecto |
| **Declaración** | `propiedad: valor;` | la unidad mínima (una sola, terminada en `;`) |

## Propiedades: kebab-case y alias de JS

Las propiedades CSS usan **guion medio** (`background-color`, `font-size`). Al llevarlas a JavaScript hay que usar corchetes o camelCase:

```js
elemento.style.backgroundColor = 'blue'; // JS convierte a background-color
```

En CSS las propiedades **son case-insensitive** (`COLOR: Blue` funciona), pero el estándar es escribirlas en minúsculas — excepto las **custom properties**, que sí distinguen mayúsculas: `--MiColor` y `--micolor` son variables distintas.

### Shorthand vs longhand

Muchas propiedades son **atajos** que descomponen en sub-propiedades:

```css
/* shorthand: todo junto */
.a { background: #fff url("hero.avif") no-repeat center / cover; }

/* longhand: cada sub-propiedad explícita */
.b {
  background-color: #fff;
  background-image: url("hero.avif");
  background-repeat: no-repeat;
  background-position: center;
  background-size: cover;
}
```

> [!warning] El shorthand reinicia lo que no menciona
> `background: url("x.png")` **borra** el `background-color` anterior y viceversa: los shorthands escriben *todas* sus sub-propiedades, y las no mencionadas vuelven a su valor inicial. Regla: o usás el shorthand completo, o usá longhand. Lo mismo aplica para `font`, `border`, `flex`, `grid-area`, `transition`…

También existen **shorthands lógicos** (`margin-inline`, `padding-block`, `border-inline-start`) que mapean según la dirección de escritura — ver [[Medidas CSS]] para el sistema lógico completo.

## Valores: keywords, números con unidad y funciones

```css
.box {
  color: blue;              /* keyword */
  width: 300px;             /* número + unidad */
  opacity: 0.8;             /* número puro (sin unidad) */
  margin: 0 auto;           /* una keyword + una función */
  width: calc(100% - 40px); /* función: calc() */
}
```

> [!warning] Unidad obligatoria
> Las longitudes necesitan unidad (`10px`, `2rem`, `50%`). **Excepción: `0`** — `margin: 0` es válido, pero `margin: 0px` también lo es. Unidades relativas como `rem`, `em` o `vw` se detallan en [[Medidas CSS]] y [[Medidas Relativas]].

Los tipos de valor que existen en CSS (tokens del lenguaje):

| Tipo | Ejemplos | Dónde se usa |
|---|---|---|
| **keyword** | `auto`, `inherit`, `none`, `center` | casi todas las propiedades |
| **longitud** | `10px`, `2rem`, `50%`, `10vw` | tamaños, espacios, offsets |
| **número** | `0.5`, `3` (adimensionales) | `opacity`, `line-height`, `flex-grow` |
| **ángulo** | `45deg`, `0.5turn`, `1rad` | `rotate`, gradientes |
| **tiempo** | `300ms`, `1.5s` | `transition`, `animation` |
| **string** | `"Helvetica"` | `font-family`, `content` |
| **color** | `#fff`, `rgb(0 0 0)`, `oklch(...)` | ver [[Colores CSS]] |
| **función** | `calc()`, `var()`, `url()`, `clamp()` | valores calculados |
| **lista** | `1px solid black` | `border`, `box-shadow` |

### Funciones de valor más comunes

```css
.ancho {
  width: clamp(300px, 50vw, 900px);  /* fluido con límites */
}
.fondo {
  background-image: url("hero.avif"); /* recurso, entre comillas */
}
.color {
  color: rgb(255 0 0 / 80%);         /* canal alfa con barra */
}
.variable {
  color: var(--texto, #111827);       /* fallback si --texto no existe */
}
```

| Función | Qué hace | Nota |
|---|---|---|
| `calc()` | aritmética mezclando unidades | requiere espacios alrededor de `+` y `-` |
| `min()` / `max()` / `clamp()` | límites y valores fluidos | `clamp(min, ideal, max)` |
| `var()` | inyecta custom property | con fallback opcional |
| `url()` | referencia un recurso | relativa al **archivo CSS**, no al HTML |
| `attr()` | lee un atributo del HTML | solo contenido en `content`, uso real muy limitado |
| `counter()` | numera listas automáticamente | `content: counter(item) ". "` |
| `env()` | variables del dispositivo | `env(safe-area-inset-top)` en móviles |
| `color-mix()` | mezcla colores en un espacio | ver [[Colores CSS]] |

> [!warning] `url()` se resuelve contra el CSS, no contra el HTML
> Si el HTML está en `/paginas/index.html` y el CSS en `/css/styles.css`, un `url("foto.png")` dentro del CSS busca `/css/foto.png`. Es la causa #1 de imágenes de fondo que no cargan.

## Valores globales: `inherit`, `initial`, `unset`, `revert`

Toda propiedad acepta estas cuatro keywords especiales, y entenderlas explica gran parte del comportamiento "mágico" de la cascada:

| Keyword | Significado |
|---|---|
| `inherit` | toma el valor **computado del padre** |
| `initial` | vuelve al **valor inicial del estándar** (ignora herencia y cascada) |
| `unset` | `inherit` si la propiedad es heredable, `initial` si no lo es |
| `revert` | vuelve al valor que tendría **sin las hojas del autor** (respeta la hoja del usuario/navegador) |

```css
.un-nodo { all: unset; }  /* reset total del subárbol: la "bomba nuclear" */
```

`all` es un shorthand que aplica cualquier combinación de las anteriores a **todas** las propiedades a la vez (`all: initial`, `all: revert`).

## Herencia: qué propiedades se propagan

Muchas propiedades **no se heredan por defecto** (como `border`), pero las tipográficas y de color sí: `color`, `font-family`, `font-size`, `line-height`, `letter-spacing`, `text-align`, `visibility`, `cursor`, `list-style`…

```css
nav { color: gray; }
nav a { }               /* los enlaces NO heredan color: siempre fueron azules */
nav a { color: inherit; } /* AHORA sí: toma el color del nav */
```

Por eso los `<a>` nacen azules aunque el padre sea gris: `color` es heredable, pero el navegador trae un **estilo UA** (user-agent stylesheet) que pinta los enlaces. Toda cascada tiene tres hojas: UA → usuario → autor; las nuestras (autor) son las del sitio. El detalle está en [[Especificidad]].

## Cascada, orden y especificidad (resumen)

> [!info] La última declaración gana
> Si dos reglas con el mismo selector declaran la misma propiedad, se aplica **la última** que aparezca en la hoja de estilo. Este es el punto de partida de la cascada y la [[Especificidad]]: reglas más específicas ganan aunque aparezcan antes.

Orden de resolución (simplificado): **origen e importancia > especificidad > orden de aparición**. El día a día de la cascada completo —incluidas capas `@layer` y `!important` invertido— vive en [[Especificidad]].

## Comentarios

```css
/* Comentario de una o varias líneas */
/* multilinea:
   ocupa varias líneas
   y el navegador lo ignora por completo */
// Los comentarios de dos barras NO existen en CSS puro
```

Los comentarios de dos barras (`//`) solo funcionan en preprocesadores como Sass — ver [[Sintaxis de Sass]]. Desde CSS Nesting nativo (2023) tampoco sirven dentro de reglas anidadas: el parser moderno los rechaza, otro motivo para no usarlos.

> [!tip] Comentarios como marcadores de sección
> En hojas grandes se estila separar bloques con banner comments para navegar en DevTools:
> ```css
> /* ===== HEADER ===== */
> ```

## Múltiples valores en una propiedad

Muchas propiedades aceptan de 1 a 4 valores que se leen como caras de un reloj (`arriba derecha abajo izquierda`):

```css
padding: 1rem 2rem 1rem 2rem;  /* 4 valores */
padding: 1rem 2rem;            /* 2 valores: vertical | horizontal */
padding: 1rem;                 /* 1 valor: los 4 lados */
```

La regla de repetición: con 1 valor van los 4 lados; con 2, el primero es vertical (arriba/abajo) y el segundo horizontal; con 3, falta el izquierdo (hereda del derecho); con 4, en orden horario desde arriba-izquierda. El mismo patrón se repite en `margin`, `border-radius` (con `/` para el radio horizontal y vertical) y `background-position`.

## At-rules: las reglas que contienen otras reglas

Los bloques que empiezan con `@` no son declaraciones sino **directivas**. Se dividen en dos familias:

| At-rule | Bloque | Para qué |
|---|---|---|
| `@media` | `{ reglas }` | condiciona por viewport/dispositivo — ver [[Responsive Web Design]] |
| `@supports` | `{ reglas }` | condiciona por soporte de propiedad |
| `@layer` | `{ reglas }` | ordena capas de la cascada |
| `@container` | `{ reglas }` | condiciona por tamaño del contenedor padre |
| `@scope` | `{ reglas }` | limita selectores a un subárbol |
| `@keyframes` | `{ fotogramas }` | define una animación — ver [[Animaciones]] |
| `@font-face` | `{ reglas }` | declara una fuente web — ver [[Fuentes Externas]] |
| `@property` | `{ reglas }` | da tipo a una custom property (habilita interpolarla) |
| `@import` | declaración | importa otro CSS (**serializa la descarga**, ver [[Crear archivo para CSS]]) |
| `@page` | `{ reglas }` | estilos de impresión |
| `@charset` | declaración | codificación del archivo (hoy casi innecesario) |
| `@namespace` | declaración | para XML/XHTML con namespaces |

```css
@supports (backdrop-filter: blur(4px)) {
  .header { backdrop-filter: blur(4px); }
}

@property --rotacion {
  syntax: "<angle>";
  inherits: false;
  initial-value: 0deg;
}
```

## Errores: qué pasa cuando algo está mal escrito

El parser de CSS es **notoriamente tolerante**: un error rara vez rompe la hoja completa. Reglas del juego:

1. **Declaración inválida → se descarta solo esa declaración.** `widht: 100px;` (typo) no aplica nada, pero las vecinas siguen funcionando. Por eso un error puede pasar *inadvertido*: simplemente "no hace nada".
2. **Selector inválido → se descarta toda la regla.** `a..link { }` no compila; el bloque entero se pierde.
3. **Bloque sin cerrar → todo lo siguiente puede quedar comido.** Una llave `}` de más cierra reglas prematuramente y las siguientes caen en contexto equivocado. Síntoma típico: mediaqueries enteras que "no funcionan".
4. **Un `@media` mal escrito no consume las reglas siguientes** — el error se recupera en la siguiente regla válida.

> [!tip] Diagnóstico rápido en DevTools
> En la pestaña *Sources*/*CSS* el CSS parseado se muestra con las declaraciones inválidas tachadas. Si una regla no aplica, buscá primero: ¿existe en el árbol de CSSOM con la declaración tachada?, ¿el selector matchea en *Elements* → *Styles*?, ¿quién la anula (tachada por otra regla más específica)?

```css
/* Errores concretos y su síntoma */
.widht: 100px;        /* typo → declaración descartada en silencio */
.card { color: red    /* falta } y ; → bloque abierto, se "come" lo siguiente */
```

## Custom properties (variables CSS)

Sintaxis propia: empiezan con `--`, se leen con `var()` y su valor es **cualquier token** (se validan en uso, no en definición):

```css
:root {
  --gap: 1rem;
  --color-marca: #3498db;
}
.item {
  padding: var(--gap);
  color: var(--color-marca, #111); /* fallback si no existe */
}
```

Se heredan por defecto, se pueden redefinir por scope (temas) y, con `@property`, tipificarse para que el navegador las interpole en animaciones — ver [[Colores CSS]] para el patrón de paleta completa.

## Cheatsheet — anatomía de una regla

| Pieza | Ejemplo | Error típico |
|---|---|---|
| Selector | `.card .titulo` | typo en clase → no matchea (regla completa inútil) |
| Lista de selectores | `h1, h2, h3` | usar espacio en vez de coma (`h1 h2` = descendiente) |
| Bloque | `{ ... }` | llave sin cerrar → se come el resto del archivo |
| Declaración | `color: red;` | faltar `;` → la siguiente declaración puede morir |
| Propiedad | `font-size` | typo (`font_size`, `fontsize`) → descartada en silencio |
| Valor | `1rem`, `center` | número sin unidad en longitudes → descartada |
| Comentario | `/* ... */` | `//` no existe en CSS → error de sintaxis |
| At-rule | `@media (...) { }` | paréntesis sin cerrar → reglas siguientes caen mal |
| Shorthand | `background: ...` | reinicia sub-propiedades no mencionadas |
| Keyword global | `inherit` / `unset` | confundirlas → herencia inesperada |

## Relacionadas

- [[CSS Introducción]] — qué es CSS y para qué sirve
- [[Selectores en CSS]] — todos los selectores disponibles
- [[Especificidad]] — qué regla gana cuando compiten varias
- [[Medidas CSS]] — unidades absolutas y sistema lógico de medidas
- [[Medidas Relativas]] — `rem`, `em`, `%`, `vh`, `vw`
- [[Colores CSS]] — los valores de tipo color en profundidad
- [[Crear archivo para CSS]] — cómo se enlaza este lenguaje al HTML
- [[Estilos para escribir CSS]] — cómo organizar reglas a escala
- [[Sintaxis de Sass]] — el preprocesador que extiende esta sintaxis
