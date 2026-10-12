---
tags:
  - desarrollo
  - frontend
  - css
tipo: concepto
estado: terminado
---

# Estilos para escribir CSS

A medida que crece un proyecto, el CSS se vuelve difícil de mantener: selectores que se pisan, nombres repetidos y specificity descontrolada. Las **metodologías para escribir CSS** no son frameworks ni librerías: son **convenciones** — reglas de nombrado, organización y prioridad que hacen predecible el resultado de la cascada a los 3, 30 y 3.000 componentes.

> [!info] Los tres problemas que toda metodología ataca
> 1. **Alcance global**: CSS no tiene namespaces — `.titulo` de la home choca con `.titulo` del panel de admin.
> 2. **Colisión de nombres**: dos equipos nombran lo mismo distinto, o lo distinto igual.
> 3. **Guerra de especificidad**: para sobreescribir se sube el nivel del selector (`.nav ul li a`), y eso escala hacia arriba hasta llegar al `!important`.
> Toda metodología es una forma distinta de meter a esos tres problemas en caja. Elegir una es elegir **dónde** se paga el costo: en disciplina de nombrado (BEM), en cantidad de clases en el HTML (utility-first) o en infraestructura de build (módulos).

Las grandes familias:

| Familia | Exponentes | Idea central |
|---|---|---|
| **Convención de nombres** | BEM, SMACSS, OOCSS | Selectores planos (0–1–0) + nombres que declaran la relación con el componente |
| **Capas por nivel de abstracción** | ITCSS | Archivos ordenados de lo genérico a lo específico, con presupuesto de especificidad |
| **Atómica / utility-first** | Atomic CSS, Tailwind | Clases de una sola propiedad que se componen en el HTML |
| **Aislamiento por build** | CSS Modules, scoped CSS, CSS-in-JS | El compilador impide que los estilos se fugen entre componentes |
| **Aislamiento nativo** | Shadow DOM, `@scope`, `@layer` | El navegador separa contextos de estilos |

En proyectos reales se **combinan**: ITCSS ordena los archivos, BEM nombra los componentes, utilities afinan puntualmente y los módulos del framework aíslan cada vista.

## BEM: Bloques, Elementos, Modificadores

BEM (*Block Element Modifier*, nacido en Yandex, 2010) es una metodología para escribir CSS reutilizable y ordenado. Su aporte central son las **convenciones de nombres**, que evitan las colisiones entre estilos de partes distintas de la app y mantienen la especificidad plana (todas las clases = 0–1–0, ver [[Especificidad]]).

El modelo mental: cada pieza de UI es un **bloque** (unidad independiente), que se compone de **elementos** (partes sin sentido propio) y admite **modificadores** (variantes del mismo bloque).

### Bloques

Un bloque es un contenedor con significado propio: si una sección **se entiende por sí sola** y no depende de otras para su apariencia, es un bloque. Un bloque puede anidarse dentro de otro (`.menu .menu__item` no, pero `.card` dentro de `.grid` sí — el bloque no sabe nada de su contenedor).

![[Pasted image 20250124164325.png]]

```css
.card {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}
```

### Elementos

Los elementos son **partes del bloque**: dependen de él y no tienen significado aislado. Se nombran con el nombre del bloque, **doble guion bajo** `__` y el nombre del elemento:

![[Pasted image 20250124164347.png]]

```css
.card__titulo {
  font-size: 1.25rem;
}

.card__imagen {
  width: 100%;
  border-radius: 8px 8px 0 0;
}
```

> [!warning] Anidar elementos es una anti-patrón
> BEM prohíbe `.card__titulo__icono`: un elemento pertenece **directo** a su bloque. Si necesitás anidar, creá un bloque nuevo (`.icono`) reutilizable dentro de varios elementos. La regla existe porque `block__a__b` acopla dos componentes entre sí — el día que el ícono se usa fuera de la card, el nombre miente.

Tampoco se anidan bloques en los nombres (`.card__header__titulo`): si una parte necesita dos niveles, son dos componentes.

### Modificadores

Un modificador es una **bandera** que indica que el bloque o el elemento tiene una variante de diseño. Doble guion bajo también:

![[Pasted image 20250124164425.png]]

```css
.card--oscura {
  background: #111827;
  color: #e5e7eb;
}

.boton--activo {
  font-weight: bold;
}

/* Modificador de elemento: una sola pieza del bloque cambia */
.card__titulo--xl {
  font-size: 2rem;
}
```

En HTML se combinan las clases — el modificador **nunca va solo**:

```html
<div class="card card--oscura">
  <h3 class="card__titulo">Título</h3>
</div>
```

| Parte        | Separador | Ejemplo              |
| ------------ | --------- | -------------------- |
| Bloque       | —         | `.card`              |
| Elemento     | `__`      | `.card__titulo`      |
| Modificador  | `--`      | `.card--oscura`      |
| Elem + mod   | `--`      | `.card__titulo--xl`  |

### Variantes de sintaxis BEM

La especificación oficial (bem.methodology) permite dos estilos de nombres:

```css
/* Estilo "nested" (separador único, el de la mayoría de tutoriales) */
.card__titulo
.card--oscura

/* Estilo "flat" de Yandex (elemento con un solo guion bajo) */
.card_titulo
.card--oscura
```

Los dos son válidos; lo que **no** es válido es mezclarlas en un mismo proyecto. Elegí una y documentala.

### BEM con Sass: la combinación clásica

El atajo real de BEM es el anidamiento con `&` — el CSS resultante sigue siendo plano aunque el Sass esté anidado:

```scss
.card {
  border: 1px solid #e5e7eb;

  &--oscura {
    background: #111827;
  }

  &__titulo {
    font-size: 1.25rem;

    &--xl {
      font-size: 2rem;
    }
  }

  &__imagen {
    width: 100%;
  }
}
```

Compila a `.card`, `.card--oscura`, `.card__titulo`, `.card__titulo--xl`, `.card__imagen` — selectores planos, especificidad 0–1–0. Ver [[Sintaxis de Sass]] y [[Mixins]].

> [!warning] El anidamiento profundo contamina la especificidad
> `&` anidado genera selectores planos; pero si anidás sin `&` (`.card { .titulo { } }`) compila a `.card .titulo` = 0–2–0 y ahí empezó la guerra. Regla BEM: **todo selector compila a una sola clase**.

### Fortalezas y costos de BEM

| A favor | En contra |
|---|---|
| Especificidad plana: sobreescribir = añadir clase, nunca subir selector | Verbosidad en el HTML (`class="card card--oscura"`) |
| Colisiones casi imposibles (nombre = ruta del componente) | Nombrar mal cuesta: renombrar un bloque es buscar-reemplazar |
| Depurable: buscás `.card__titulo` y está en un solo lado | No resuelve por sí solo los valores compartidos (espaciados, paleta) |
| Funciona sin build, con cualquier preprocesador | — |

BEM no impide duplicar estilos entre bloques hermanos; para eso están los objetos de ITCSS o los mixins de Sass.

## OOCSS: separar estructura de piel

OOCSS (*Object-Oriented CSS*, Nicole Sullivan, ~2009) es la metodología más antigua de las modernas y el DNA oculto de Bootstrap. Sus **dos principios**:

1. **Separar estructura de skin**: el layout no lleva colores ni bordes, y viceversa.

```css
/* Mal: estructura y skin mezclados */
.btn-rojo-grande {
  padding: 1rem 2rem;
  background: #e74c3c;
  border-radius: 8px;
}

/* OOCSS: objeto + skin por separado */
.btn {
  display: inline-block;
  padding: 1rem 2rem;
  border-radius: 8px;
}

.btn--rojo {
  background: #e74c3c;
  color: #fff;
}
```

2. **Separar container de content**: un objeto no debe depender de su posición en la página.

```css
/* Mal: el componente sabe que vive en la sidebar */
.sidebar .lista {
  font-size: 0.875rem;
}

/* OOCSS: el componente define su tamaño; el container lo configura */
.lista {
  font-size: 0.875rem;
}
```

El ejemplo canónico es el **media object** (avatar + texto) de Facebook: un objeto reutilizable en timelines, comentarios y listas sin duplicar CSS. Ver la receta en [[Flexbox]].

> [!info] Legado
> OOCSS como "metodología con nombre" casi no se usa sola hoy, pero sus principios están dentro de BEM (skin/modificador) y de utility-first (clases orthogonalmente combinables). Leer CSS de Bootstrap 3/4 es leer OOCSS.

## SMACSS: categorías y prefijos

SMACSS (*Scalable and Modular Architecture for CSS*, Jonathan Snook, 2012) clasifica **cada regla** en una de cinco categorías y usa prefijos de nombre para hacerlas reconocibles al ojo:

| Categoría | Qué es | Prefijo | Ejemplo |
|---|---|---|---|
| **Base** | Estilos por defecto de elementos | — | `a { color: #333 }`, `body { font-family: ... }` |
| **Layout** | Estructuras de página (grillas, sidebars) | `l-` / `layout-` | `.l-sidebar`, `.layout-grid` |
| **Module** | Componentes reutilizados (nav, card) | `m-` opcional | `.m-nav`, `.card` |
| **State** | Estados visuales conmutables | `is-` / `has-` | `.is-active`, `.is-open`, `.has-error` |
| **Theme** | Variables de apariencia (skin) | `theme-` | `.theme-dark` |

```css
/* Base */
a {
  color: #2563eb;
}

/* Layout */
.l-sidebar {
  display: grid;
  grid-template-columns: 260px 1fr;
}

/* Module */
.nav {
  display: flex;
  gap: 1rem;
}

/* State: se agrega/quita con JS, describe un momento, no un diseño */
.nav__link.is-active {
  font-weight: 700;
}
```

Reglas de SMACSS: no usar IDs como selectores, mantener la especificidad baja, y que los estados (`.is-*`) se manejen **agregando y quitando clases** desde JS — nunca con selectores de atributo en cascada.

> [!tip] SMACSS es la metodología "de transición"
> Para equipos que vienen de CSS sin método, los prefijos `l-` / `is-` dan orden inmediato sin la disciplina completa de BEM. Es común ver SMACSS para el layout de página + BEM dentro de cada módulo.

## ITCSS: la capa invertida

ITCSS (*Inverted Triangle CSS*, Harry Roberts, ~2013) no toca el HTML: **ordena los archivos CSS** de lo más genérico y alcance global a lo más específico y acotado, de modo que cada capa solo puede sobreescribir a las anteriores. Se dice "triángulo invertido" porque el alcance (cuánto DOM toca) es amplio arriba y estrecho abajo:

```
┌──────────────────────────────────────────────┐
│ 1. Settings   variables, tokens (sass vars)  │  ← 0 tocar DOM
│ 2. Tools      mixins, functions              │  ← 0 tocar DOM
│ 3. Generic    reset, normalize, box-sizing   │  ← todo el DOM
│ 4. Elements   estilos de tags puros (a, h1)  │  ← todo el DOM
│ 5. Objects    layout puro, sin skin (media)  │  ← clases genéricas
│ 6. Components UI concreta (card, nav, btn)   │  ← clases propias
│ 7. Utilities  utilidades de una línea        │  ← ganan a todo
└──────────────────────────────────────────────┘
```

La consecuencia operativa es la **inversión de la cascada**: como los componentes van después que los elementos, `.card` puede sobreescribir `h2 {}` sin ningún truco. Y las utilities van al final por eso: su trabajo es ganar limpiamente.

```scss
// Estructura de carpetas tipo ITCSS con Sass (7-1 es su pariente cercano)
@import "settings/variables";
@import "tools/mixins";
@import "generic/reset";
@import "elements/typography";
@import "objects/media";
@import "components/card";
@import "components/nav";
@import "utilities/text-utils";
```

Ver [[Separar Sass en Archivos]] para la variante 7-1 completa.

| Capa | Ejemplo concreto | Specificidad permitida |
|---|---|---|
| Settings | `$color-primario`, tokens | — |
| Tools | `@mixin sombra($n)` | — |
| Generic | `*, *::before { box-sizing: border-box }` | 0–0–1 / 0–1–0 |
| Elements | `body`, `h1`, `a` | 0–0–1 |
| Objects | `.media`, `.banda` | 0–1–0 |
| Components | `.media--avatar`, `.card` | 0–1–0–1 aceptable |
| Utilities | `.text-center`, `.mt-2` | 0–1–0 + `!important` permitido |

> [!tip] Presupuesto de especificidad
> Regla de Heydon Pickering (CSS Guidelines): cada componente puede "gastar" un poco de especificidad extra (un elemento encadenado, un atributo), pero **las utilities siempre ganan** porque van al final. Si un componente necesita `!important`, el problema está en el orden de las capas, no en el `!important`.

ITCSS es perfecto compañero de los **cascade layers** nativos (ver más abajo): la capa `@layer` es ITCSS ejecutado por el navegador.

## Utility First (CSS atómico)

En lugar de componentes semánticos, se crean **clases diminutas con una sola propiedad** que describen exactamente lo que hacen, y se componen en el HTML:

![[Pasted image 20250124164451.png]]

```css
.text-center { text-align: center; }
.mt-2 { margin-top: 0.5rem; }
.p-4 { padding: 1rem; }
.bg-azul { background-color: #3498db; }
```

```html
<div class="text-center mt-2 p-4 bg-azul">...</div>
```

**Cómo evita las specificity wars**: todas las clases valen 0–1–0 y hacen una sola cosa — no hay nada que sobreescribir, solo agregar otra clase. El orden del CSS generado decide limpiamente.

- **Contra**: mucho CSS generado (aunque con purging de no usados, como hace Tailwind, el CSS final es mínimo) y ruido en el HTML.
- **A favor**: diseño 100% consistente (espaciados y colores de un sistema) sin specificity wars — todas las clases empatan y el orden del CSS decide limpiamente.
- **Costo real**: el HTML deja de ser autodescriptivo (`class="flex gap-4 p-6"` exige saber el sistema) y el "estilo" vive repartido entre docenas de archivos en vez de un CSS localizable.

El exponente moderno de esta filosofía es [[TailwindCSS - Sintaxis Básica]], que además aporta `@apply`, design tokens y purgado por build.

> [!info] "Atomic" ≠ "Atomic Design"
> **Atomic CSS** es la metodología de estilos de esta sección. **Atomic Design** (Brad Frost) es otra cosa: una taxonomía de **componentes** (atoms → molecules → organisms → templates → pages) para el HTML/JS. Se confunden por el nombre; no se usan juntas necesariamente.

## Módulos: aislar estilos por componente

La tercera vista: definir el **contexto principal** y seleccionar los elementos dentro de este, en lugar de selectores globales sueltos:

![[Pasted image 20250124164515.png]]

```css
.contenedor-principal .titulo {
  color: #111827;
}

.contenedor-principal .item {
  padding: 1rem;
}
```

En frameworks modernos este concepto evolucionó a varios mecanismos de **aislamiento real**:

| Mecanismo | Cómo aísla | Quién lo usa | Notas |
|---|---|---|---|
| **CSS Modules** | El build hashea los nombres: `.card__titulo` → `.card_titulo_a1b2c` | React, Vue, Astro, SvelteKit | `styles.titulo` en el JS/TS; `:global()` para escape |
| **Scoped CSS** (Vue SFC) | El build añade un atributo único: `.titulo[data-v-3fa4]` | Vue 2/3 | `:deep(.hijo)` para componentes hijos |
| **Svelte** | Aislado por defecto, sin configuración | Svelte | El más limpio: scoped nativo del compilador |
| **Angular** | Encapsulación emulada (`_ngcontent-xxx`) | Angular | `ViewEncapsulation.Emulated` por defecto |
| **Shadow DOM** | Aislamiento real de cascada del navegador | Web Components | CSS global **no** entra; hereda solo `color`, `font`... |
| **`@scope`** (CSS nativo) | `@scope (.card) to (.contenido) { }` | Navegadores 2024+ | Scoping sin build; delimita por subárbol y "hasta" |

```css
/* CSS Module: el punto de composición */
.card {
  composes: sombra from "./sombras.module.css";
  padding: 1rem;
}
```

```css
/* @scope: scoping nativo, con límite de bajada */
@scope (.tarjeta) to (.pie) {
  h3 {
    color: #111827;
  } /* solo afecta h3 dentro de .tarjeta, sin bajar a .pie */
}
```

### CSS-in-JS: el espectro completo

| Estilo | Ejemplos | Mecanismo | Costo |
|---|---|---|---|
| **Runtime CSS-in-JS** | styled-components, emotion, goober | Genera `<style>` en ejecución, en el cliente | JS bundle + costo de ejecución + FOUC en SSR |
| **Zero-runtime** | vanilla-extract, Linaria, Panda CSS | El CSS se extrae en build time; el JS solo son referencias | Requiere build, sin costo en runtime |
| **CSS Modules + utilidades** | Modules + Tailwind | Nada en runtime | El camino más común en 2026 |

La lección de la historia: el CSS-in-JS de runtime (2015–2020) perdió rendimiento contra CSS extraído en build (2021+); incluso styled-components recomienda extraer a hojas estáticas en SSR. Ver [[ReactJS - Fundamentos]] para el contexto de componentes.

## Convenciones de nombrado, cara a cara

| Convención | Sintaxis | Especificidad | Escala bien con… |
|---|---|---|---|
| BEM | `.block__elem--mod` | 0–1–0 | Componentes medianos, Sass, equipos mixtos |
| SMACSS | `.l-layout`, `.is-state` | 0–1–0 | Layout de página + estados |
| OOCSS | objetos `.media`, skins `.btn--rojo` | 0–1–0 | Sistemas de diseño |
| Utility | `.mt-2`, `.text-center` | 0–1–0 (empate por orden) | Design systems, purgado, Tailwind |
| Global anidado (sin método) | `.nav ul li a` | 0–1–3+ | Nada: es el origen de las wars |

**Qué elegir**

- Proyectos pequeños/médios: BEM con especificidad plana (o utility-first si el equipo ya piensa en ese sistema).
- Prototipos y equipos que ya piensan en utilidades: Tailwind.
- SPAs con componentes (React, Vue): CSS Modules o scoped, con utilidades puntuales.
- Design systems grandes: ITCSS (orden de capas) + BEM (nombres) + utilities (afinado final) — la combinación completa.
- Web components: Shadow DOM nativo; recordá que el CSS global no entra.
- Lo que **no** funciona en ninguno: selectores largos anidados con `!important` de por medio — ver [[Especificidad]].

## Cascade layers: la cascada, ahora con jerarquía

Desde 2022 (Chrome 99, Firefox 97, Safari 15.4) el navegador entiende capas de cascada — la ITCSS ejecutada nativamente:

```css
@layer theme, base, components, utilities;

@layer base {
  a { color: #2563eb; }
}

@layer components {
  .nav__link { color: #111827; }
}

@layer utilities {
  .text-blue { color: #2563eb; }
}
```

Reglas que gobiernan todo lo anterior:

- El orden de la declaración `@layer a, b, c;` fija la prioridad: **gana la capa más tarde**, sin importar especificidad.
- **Los estilos sin capa ganan a todos los layered** (por diseño: lo sin capa es "lo urgente/lo tuyo").
- `!important` **invierte** el orden dentro de las capas: en importante, la primera capa gana.
- `.nav__link` (0–1–0) en `components` gana a `a` (0–0–1) sin capa… no: `a` sin capa gana. Hay que poner **todo** en capas para que el orden mande.

```css
/* Tailwind v4 es literalmente esto por dentro */
@layer theme, base, components, utilities;
```

### `:where()` e `:is()`: especificidad a medida

```css
/* :where() aporta especificidad 0 — ideal para resets configurables */
:where(.sidebar) a {
  color: inherit;
} /* 0–1–1… no: 0–0–1 — .sidebar no suma */

/* :is() suma la especificidad del argumento más específico */
:is(.card, .panel) h2 {
  font-size: 1.25rem;
} /* 0–1–1 */
```

Patrón de configuración: `.tema { --enlace: #2563eb }` + `:where(.tema) a { color: var(--enlace) }` — cualquier regla posterior gana sin pelear.

## Organización de proyectos

La estructura de carpetas **es** parte de la metodología — un archivo `estilos.css` de 3.000 líneas es la anti-metodología por excelencia:

```
css/
├── main.scss              @imports en orden ITCSS
├── settings/              tokens, breakpoints, variables
├── tools/                 mixins, funciones, placeholders
├── generic/               reset, normalize (ver [[Normalize]])
├── elements/              tipografía base, listas, links
├── objects/               grillas, bandas, media object
├── components/            card, nav, boton, formulario
└── utilities/             utilidades de una propiedad
```

Con módulos (React/Vue), el modelo es **colocación**: cada componente trae su CSS junto al código, y los estilos globales quedan solo para base/tokens:

```
src/
├── styles/       global.css (tokens, reset, tipografía)
└── components/
    └── Card/
        ├── Card.tsx
        └── Card.module.css
```

## Errores comunes

1. **Modificar en el HTML sin la base**: `class="card--oscura"` solo no pinta nada — el modificador se agrega **junto** a la clase base.
2. **Anidar elementos BEM** (`block__a__b`) o bloques en nombres: acopla componentes; extraé un bloque nuevo.
3. **`!important` en utilities de producción**: en ITCSS/Tailwind las utilities ganan por orden de capa, no por `!important`; si lo necesitás, el orden está mal.
4. **Methodology sin tokens**: BEM sin variables compartidas duplica colores en 30 modificadores; los design tokens (custom properties) van en la capa Settings.
5. **Utility-first sin purga**: servir 3.000 clases sin JIT/purge es el contraejemplo clásico — con Tailwind el build solo emite las usadas.
6. **CSS-in-JS runtime en SPAs con mucho CSS**: el costo de ejecución por componente escala peor que CSS estático; medir antes de adoptar.
7. **Mezclar metodologías sin plan**: Tailwind dentro de componentes BEM funciona (utilities = capa final), pero BEM + OOCSS + SMACSS "porque sí" solo agrega prefijos.
8. **IDs como selectores** (`#header`): especificidad 1–0–0, imposible de bajar; SMACSS y BEM los prohíben.

## Accesibilidad y rendimiento

- **Consistencia = accesibilidad**: un sistema de tokens (espaciados, tamaños de toque ≥ 44px, contraste) aplicado por metodología evita pantallas incoherentes; ver [[Colores CSS]] para contraste.
- **Especificidad plana = depuración accesible**: poder apagar una regla desde DevTools sin romper diez cosas acelera los ajustes de accesibilidad.
- Purgado/JIT (Tailwind, esbuild) reduce el CSS inicial decenas de veces — el CSS es render-blocking; menos bytes = FCP antes.
- CSS-in-JS runtime añade trabajo en el hilo principal en cada render; el CSS de build es cacheable estáticamente.

## Cheatsheet — metodologías

| Metodología | Año | Autor | Núcleo | Specificidad | Úsala cuando |
|---|---|---|---|---|---|
| OOCSS | 2009 | N. Sullivan | Estructura/skin, container/content | Plana | Sistemas de diseño, objetos reutilizables |
| BEM | 2010 | Yandex | `block__elem--mod` | 0–1–0 | Componentes con nombre claro |
| SMACSS | 2012 | J. Snook | 5 categorías + prefijos | Baja | Layout de página + estados |
| ITCSS | 2013 | H. Roberts | Capas de lo genérico a específico | Presupuestada | Proyectos grandes con Sass |
| Utility/Atomic | 2015+ | Airbnb, Tailwind | Clases de una propiedad | Empate por orden | Design systems, velocidad |
| CSS Modules | 2015+ | React ecosist. | Hash por build | Local | SPAs con componentes |
| Scoped (Vue/Svelte) | 2016+ | frameworks | Atributo/compilador | Local | Vue y Svelte |
| CSS-in-JS runtime | 2016+ | styled-components | `<style>` en ejecución | Local (runtime) | Legado; hoy preferir zero-runtime |
| Zero-runtime | 2020+ | vanilla-extract, Panda | Extracción en build | Local | Máximo rendimiento con DX de JS |
| Cascade layers | 2022 | W3C | `@layer` nativo | Por capa | Todo proyecto nuevo |

## Compatibilidad

BEM, SMACSS, OOCSS e ITCSS son **convenciones**: funcionan en cualquier navegador porque son CSS plano. Lo único con requisitos de soporte son las primitivas modernas que las acompañan: `@layer` y `:where()`/`:is()` (2022), `@scope` (2024–2025, usar con `@supports`), CSS Modules y scoped CSS (dependen del build, no del navegador).

## Relacionadas

- [[Especificidad]] — el problema que estas metodologías buscan controlar
- [[TailwindCSS - Sintaxis Básica]] — utility-first en la práctica
- [[Sintaxis de Sass]] — anidación y organización de archivos
- [[Separar Sass en Archivos]] — la variante 7-1 de ITCSS
- [[Mixins]] — la capa Tools de ITCSS
- [[Selectores en CSS]] — herramientas de selección disponibles
- [[CSS Introducción]] — la cascada que gobierna todo esto
- [[Normalize]] — la capa Generic, punto de partida de ITCSS
- [[Crear archivo para CSS]] — cómo estructurar y enlazar la hoja
- [[ReactJS - Fundamentos]] — el contexto de componentes donde viven los módulos
