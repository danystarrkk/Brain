---
tags:
  - desarrollo
  - frontend
  - normalize
tipo: concepto
estado: terminado
---

# Normalize

Los navegadores no arrancan de cero: cada uno trae su propia **hoja de estilos por defecto** (user-agent stylesheet) y, por eso, un mismo HTML se ve distinto en Chrome, Firefox y Safari (márgenes de `h1`, tamaños de `ul`, espaciados de `formularios`...). **Normalize.css** resuelve esto aplicando una capa inicial de estilos que **unifica los puntos de partida** entre navegadores, sin borrar el diseño de cada uno.

> [!info] Normalize vs Reset
> - **Reset** (reseteo agresivo): borra *todo* estilo por defecto (`margin: 0` en todo, sin listas, sin headings especiales) y te obliga a reconstruir cada elemento desde cero.
> - **Normalize** (lo que usamos acá): preserva lo útil de los estilos por defecto y **solo corrige inconsistencias y bugs conocidos**. Es menos destructivo y parte de un terreno conocido.

## Qué es la hoja de estilos del usuario-agent (UA)

Cada navegador incluye una hoja de estilos propia, escrita por su equipo, que da formato "razonable" al HTML semántico: `h1` grande y con márgenes, `ul` con viñetas e indentación, `button` con borde, `em` en cursiva. Esa hoja **no es parte de CSS ni del sitio**: es comportamiento por defecto del navegador, y **varía entre motores de render**:

| Elemento | Chrome/Safari (Blink/WebKit) | Firefox (Gecko) | IE/Edge legado (EdgeHTML) |
|---|---|---|---|
| `h1` | `font-size: 2em; margin: 0.67em 0` | `font-size: 2em; margin: 0.67em 0` | margen distinto en versiones viejas |
| `ul` | `padding-inline-start: 40px` | `padding-left: 40px` | idem, distinto entre versiones |
| `button` | padding mínimo, `appearance` propia | padding distinto | fuente del sistema |
| `img` | `inline`, sin border | ídem | ídem |
| `nav ul` | **sin** viñetas (Chrome lo hace) | con viñetas | con viñetas |

La conclusión práctica: **el punto de partida ya es distinto antes de escribir una línea de CSS**. Normalize.equaliza ese punto de partida.

## Cómo entra en la cascada (por qué el orden importa)

La cascada resuelve conflictos por **origen** primero y por especificidad después. Los orígenes, de menor a mayor prioridad (sin `!important`):

| Origen | Quién lo define | Ejemplo |
|---|---|---|
| Hoja UA (user-agent) | El navegador | `h1 { font-size: 2em }` |
| Hoja del usuario | Configuración del usuario / extensiones | tipografía del lector |
| Hoja del autor (author) | Tu CSS | `estilos.css` |

Un autor stylesheet **gana siempre** sobre la hoja UA, aunque sea un selector de elemento. Entonces, ¿por qué hace falta normalize si mi CSS ya gana? Porque normalize no compite con tus reglas: **igualiza la base entre navegadores para que tus reglas apliquen sobre el mismo terreno en todos**. Sin eso, `h1 { margin: 2rem }` se ve igual en todos, pero cada detalle que *no* tocás (tamaño de `fieldset`, padding de `ul`, `line-height` de `html`) sigue distinto en cada motor.

Además, el orden de carga importa para tus **propias** reglas de igual especificidad (ver [[Especificidad]]): si normalize va después de tu CSS, sus reglas de elemento (0–0–1) pueden pisar reglas tuyas de igual especificidad que vengan antes. Por eso: **siempre primero normalize, después lo tuyo**.

### En la era de `@layer`: normalize dentro de una capa

CSS Cascade Layers permite declarar capas explícitas y ganar control total del orden sin pelear con especificidad:

```css
@layer reset, base, componentes, utilidades;

@layer reset {
  /* normalize.css entera acá */
}

@layer base {
  body { font-family: system-ui; }
}
```

Regla clave: las declaraciones **sin capa** ganan a las de cualquier capa (en origen autor, nivel normal). Así tus utilidades sin capa (o en la capa `utilidades` si van todas juntas) siempre pisan el reset sin usar `!important`. El orden de las capas se declara una vez y el CSS de Tailwind (ver [[TailwindCSS - Sintaxis Básica]]) hace exactamente esto con `@layer base/components/utilities`.

## Reset, normalize y modern-reset: la genealogía

| Generación | Ejemplo | Enfoque | Problema que tuvo |
|---|---|---|---|
| **Reset** (2007+) | Eric Meyer's Reset | `* { margin: 0; padding: 0 }`, borra todo | Destruye el diseño útil de la UA; mucho trabajo de reconstrucción |
| **Normalize** (2011+) | normalize.css (Nicolas Gallagher) | Corrige bugs e inconsistencias preservando lo útil | 400+ líneas que pocos entienden; nació antes de features modernas |
| **Modern reset** (2019+) | Andy Bell, modern-css-reset, sanitize | ~50 líneas con propiedades lógicas, `:focus-visible`, `prefers-reduced-motion` | Cada uno distinto; elegir es decisión de proyecto |
| **Preflight** | Tailwind CSS | Reset moderno + convenciones Tailwind (`border-0`, headings sin estilo) | Solo tiene sentido si usás Tailwind |

> [!tip] La pregunta no es "cuál es el mejor", sino "cuánto necesito"
> - Sitio liviano, sin framework → **normalize v8** o un **modern reset** corto.
> - Proyecto Tailwind → **Preflight** ya está; no cargues normalize encima.
> - Sistema de diseño con utilidades → modern reset mínimo dentro de `@layer reset`.

## Qué corrige normalize v8, regla por regla

normalize.css v8 (~400 líneas comentadas) es esencialmente esta lista de correcciones:

| Regla normalize | Qué corrige | Navegador/bug de referencia |
|---|---|---|
| `html { line-height: 1.15 }` | `line-height` base inconsistente | Firefox <61 y otros con valores heredados raros |
| `html { -webkit-text-size-adjust: 100% }` | Safari/iOS **aumenta texto automáticamente** al rotar | iOS Safari |
| `body { margin: 0 }` | `body` tiene margen UA de 8px | Todos los navegadores |
| `main { display: block }` | `main` (y HTML5) `display: inline` en navegadores viejos | IE ≤ 10, Safari ≤ 4 |
| `h1 { font-size: 2em; margin: .67em 0 }` | Márgenes de `h1` distintos entre motores | Safari ≤ 8, Firefox <... |
| `hr { box-sizing: content-box; height: 0; overflow: visible }` | `hr` se dimensionaba raro con box-sizing global | Firefox |
| `code, kbd, samp, pre { font-family: monospace, monospace; font-size: 1em }` | Tamaños/familias monospace inconsistentes | IE, Safari |
| `a { background-color: transparent }` | Fondo heredado de `background` en navegadores viejos | IE ≤ 9, Safari ≤ 4 |
| `abbr[title] { border-bottom: none; text-decoration: underline dotted }` | Tooltip nativo + subrayado duplicado | Todos |
| `b, strong { font-weight: bolder }` | `bolder` resuelve distinto según el padre | Firefox (700 vs 900) |
| `sub, sup { font-size: 75%; line-height: 0; position: relative; vertical-align: baseline }` | Desalineación de índices | Safari ≤ 5 |
| `img { border-style: none }` | Borde UA en imágenes con link | IE ≤ 9, Opera < 9.6 |
| `button, input, optgroup, select, textarea { font-family: inherit; font-size: 100%; line-height: 1.15; margin: 0 }` | Formularios **no heredan la tipografía** por defecto | Todos (cada uno distinto) |
| `button, select { text-transform: none }` | `text-transform: uppercase` forzado | Firefox, IE |
| `button, [type=button], [type=reset], [type=submit] { -webkit-appearance: button }` | Safari convierte botones sin type en "píldora" | WebKit |
| `fieldset { padding: .35em .75em .625em }` | Padding de fieldset distinto por motor | IE/Edge |
| `legend { box-sizing: border-box; color: inherit; display: table; max-width: 100%; padding: 0; white-space: normal }` | `legend` cortado/centrado raro | Edge, Firefox, Safari |
| `progress { vertical-align: baseline }` | `<progress>` desalineado | Firefox |
| `textarea { overflow: auto }` | `textarea` sin scroll propio | IE |
| `[type=checkbox], [type=radio] { box-sizing: border-box; padding: 0 }` | Tamaño calculado distinto | IE ≤ 9 |
| `[type=number]::-webkit-inner/outer-spin-button { height: auto }` | Spinners gigantes | Chrome/Opera antiguos |
| `[type=search] { -webkit-appearance: textfield; outline-offset: -2px }` | Búsqueda redondeada + outline cortado | WebKit/Blink |
| `::-webkit-file-upload-button { -webkit-appearance: button; font: inherit }` | Botón de archivo nativo sin heredar fuente | WebKit/Blink |
| `details { display: block }` | `<details>` "colapsado" en navegadores sin soporte | navegadores viejos |
| `summary { display: list-item }` | Marker del `<summary>` | navegadores viejos |
| `template { display: none }` | Contenido de `<template>` visible | navegadores viejos |
| `[hidden] { display: none }` | `display: list-item` de `[hidden]` en algunos UAs | Firefox |

> [!info] El patrón detrás de casi todas las reglas
> Casi todo normalize cabe en tres categorías: **(1)** hacer que los elementos HTML5 existan (`display: block`), **(2)** hacer que los formularios hereden tipografía y **(3)** neutralizar un puñado de bugs de motores concretos. Si entendés esas tres familias, ya entendés normalize.

## Instalación paso a paso

### Paso 1: descargarlo

Vamos a la página oficial de [normalize](https://necolas.github.io/normalize.css/):

![[Pasted image 20250124162622.png]]

Damos en *Download* y veremos el código fuente:

![[Pasted image 20250124162653.png]]

> [!tip] Desactivá el traductor automático
> Si tenés el traductor de Chrome activo, desactivalo en esta página: puede corromper los comentarios dentro del CSS al copiarlo.

Alternativas de instalación: npm (`npm i normalize.css` + import en tu bundler) o CDN (jsDelivr/unpkg) — para aprendizaje, copiar el archivo a mano está bien; para producción, npm/CDN con versión fija.

### Paso 2: guardarlo en nuestro proyecto

Copiamos todo el contenido y lo pegamos en un archivo llamado `normalize.css` dentro de nuestra carpeta de estilos.

### Paso 3: cargarlo antes que nada

Lo enlazamos en el HTML **como la primera hoja de estilos**: todo lo que venga después debe poder sobreescribirlo sin pelearse con la cascada (ver [[Especificidad]] para por qué el orden importa):

```html
<head>
  <link rel="stylesheet" href="css/normalize.css" />
  <link rel="stylesheet" href="css/estilos.css" />
</head>
```

![[Pasted image 20250124162743.png]]

> [!warning] Orden siempre fijo
> Si normalize va después de tu CSS, sus reglas (de especificidad baja, casi todas de elemento) pueden pisar tus estilos. Primero normalize, después lo tuyo.

## Modern resets: las alternativas actuales

| Opción | Tamaño aprox. | Qué aporta de extra |
|---|---|---|
| **normalize.css v8** | ~8 KB (~4 KB min) | Correcciones de bugs históricos; no toca diseño |
| **sanitize.css** | ~15 KB | Normalize + reset parcial de formularios cross-browser |
| **Modern CSS Reset** (Andy Bell, 2019) | ~25 líneas | `box-sizing`, margin 0 de body, `:focus-visible`, `prefers-reduced-motion`, listas sin marker |
| **minireset.css** | ~10 líneas | Lo mínimo: box-sizing, margin, media img, herencia de fuentes |
| **Preflight (Tailwind)** | integrado | `border-0` por defecto, headings sin estilo, links heredan color, `img { display: block }` — ver [[TailwindCSS - Sintaxis Básica]] |
| **Tu reset propio** | lo que decidas | Control total con propiedades lógicas modernas |

El modern reset de Andy Bell, para ver el estilo de las decisiones modernas:

```css
*, *::before, *::after { box-sizing: border-box; }   /* ver [[Box Model]] */
* { margin: 0; }
html { height: 100%; }
body { line-height: 1.5; -webkit-font-smoothing: antialiased; }
img, picture, video, canvas, svg { display: block; max-width: 100%; }
input, button, textarea, select { font: inherit; }
p, h1, h2, h3, h4, h5, h6 { overflow-wrap: break-word; }
p { text-wrap: pretty; }

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; scroll-behavior: auto !important; }
}
```

Detalles que valen la pena de cada línea:

- **`box-sizing: border-box` universal**: el cambio más influyente de la última década; sin él, `width: 100% + padding` desborda (ver [[Box Model]]).
- **`img { display: block }`**: elimina el "hueco fantasma" de línea base que los inline generan debajo de las imágenes.
- **`font: inherit` en controles**: arregla el problema de que los formularios no heredan tipografía.
- **`prefers-reduced-motion`**: respeto por accesibilidad desde el reset (ver [[Animaciones]] para el tema completo).
- **Sin `scroll-behavior` global forzado**: no imponer `smooth` a todos los usuarios.

## Cuándo un reset lastima

Un reset no es gratis. Casos donde conviene pensarlo dos veces:

1. **Sitios de contenido que dependen de la UA**: un blog de lectura donde `p`, `blockquote`, `ul` ya se leen bien con la UA puede necesitar solo ajustes puntuales, no un reset completo.
2. **Formularios con `appearance` nativa deseada**: Preflight/normalizaciones que borran bordes dejan botones "fantasma" si no reconstruís el estilo; iOS además tiene sus propios estilos de input (ver [[Formularios]]).
3. **Prototipos con utilidades**: con Tailwind u otro framework de utilidades, un reset extra duplica capa y conflicto.
4. **Cuando el reset es caja negra**: copiar 400 líneas que no entendés es peor que 20 líneas que sí. Si no podés explicar por qué está cada regla, preferí el reset corto.
5. **Heredar CSS viejo**: si el proyecto ya trae reset y encima agregás otro, los dobles `margin: 0` y `font: inherit` compiten por nada. Un solo reset por proyecto.

## Receta: reset propio moderno en una capa

```css
@layer reset;

@layer reset {
  /* 1. Caja */
  *, ::before, ::after { box-sizing: border-box; }

  /* 2. Flujo base */
  body { margin: 0; line-height: 1.5; }
  h1, h2, h3, h4, h5, h6 { font-size: inherit; font-weight: inherit; } /* tamaños se deciden en base */
  ul[role="list"] { list-style: none; padding: 0; }                   /* listas decorativas, no todas */

  /* 3. Medios */
  img, picture, video, canvas, svg { display: block; max-width: 100%; height: auto; }

  /* 4. Formularios */
  input, button, textarea, select { font: inherit; color: inherit; }

  /* 5. Enlaces decorativos */
  a { color: inherit; text-decoration: inherit; }

  /* 6. Accesibilidad */
  :focus-visible { outline: 2px solid currentColor; outline-offset: 2px; }
  @media (prefers-reduced-motion: reduce) {
    *, ::before, ::after { animation-duration: .01ms !important; transition-duration: .01ms !important; }
  }
}
```

Notas de la receta: `font-size: inherit` en headings es decisión de diseño (si no, usá `clamp()` para escalarlos — ver [[Medidas CSS]]); `ul[role="list"]` en vez de `ul` global porque las listas semánticas **sí** deben mostrar viñetas para lectores de pantalla (sección de accesibilidad); `currentColor` en `:focus-visible` funciona en cualquier tema sin duplicar colores.

## Errores comunes

1. **Cargar normalize después de los estilos propios**: sus reglas de elemento ganan por orden dentro del mismo origen. Síntoma: tus estilos "se pierden" misteriosamente. Fix: normalize primero, o capas con `@layer`.
2. **Doble reset**: normalize + Preflight de Tailwind, o dos resets copiados de tutoriales. Síntoma: formularios sin estilo útil, listas sin viñetas, y conflictos que nadie entiende. Fix: uno solo.
3. **Esperar que normalize arregle las fuentes**: no cambia `font-family` de `body` (solo la herencia en formularios). Cada proyecto define su tipografía — ver [[Fuentes Externas]].
4. **Olvidar `box-sizing`**: si usás normalize puro (que no lo incluye), `width: 100% + padding` desborda. Añadir la regla universal o pasar a modern reset.
5. **Borrar las viñetas de *todas* las listas**: `ul { list-style: none }` global destruye la percepción de lista para algunos lectores de pantalla. Limitar a listas decorativas y agregar `role="list"` si el HTML no puede cambiarse.
6. **Copiar el CSS a mano sin mirar la versión**: normalize v2/v3 son distintos de v8. Verificar versión y changelog; v8 eliminó reglas de soporte IE8/9 que en proyectos modernos sobran.

## Accesibilidad

- Un reset **no** debe borrar indicadores de foco: si tu reset incluye `outline: none` sin reemplazo, estás rompiendo la navegación por teclado. La receta de arriba usa `:focus-visible` como mínimo indispensable.
- `font-size: inherit` en formularios ayuda a la accesibilidad: los controles heredan el tamaño que el usuario configuró.
- `-webkit-text-size-adjust` (heredado de normalize) evita el zoom automático de iOS... pero ojo: `maximum-scale=1` o `user-scalable=no` en el viewport **sí** bloquean zoom y violan WCAG 2.2 — ver [[Responsive Web Design]].
- Verificar con el árbol de accesibilidad (DevTools → Accessibility) que el reset no escondió nada: `[hidden] { display: none }` es correcto, pero si tu CSS redefine `hidden` a `display: block` el contenido oculto se vuelve visible para todos.

## Rendimiento

- Un reset es CSS estático y minúsculo (~4 KB minificado normalize v8): se cachea para siempre, costo práctico nulo.
- Reglas universales (`* { ... }`) tienen mala fama de "lentas": con un solo `*` al inicio del selector el costo es irrelevante para los motores modernos; el problema sería `div * span * a` (selectores profundos — ver [[Selectores en CSS]]).
- Mantenelo **antes** de `@import` de fuentes o frameworks: no hay razón para que el reset esté bloqueado detrás de otras peticiones.

## Cheatsheet — lo que decide cada reset

| Decisión | normalize v8 | Modern reset | Preflight (Tailwind) | Efecto si lo omitís |
|---|---|---|---|---|
| `box-sizing: border-box` global | ❌ | ✅ | ✅ | paddings desbordan cajas fijas |
| `body { margin: 0 }` | ✅ | ✅ | ✅ | 8px de margen fantasma |
| Formularios heredan fuente | ✅ (`font-family/font-size: inherit`) | ✅ (`font: inherit`) | ✅ | UI del navegador ignora tu tipografía |
| Headings sin tamaño propio | ❌ | ✅ (`font-size: inherit`) | ✅ | tamaños UA fijos |
| `img { display: block }` | ❌ | ✅ | ✅ | hueco de línea base bajo imágenes |
| Listas sin marker | ❌ | parcial (`ul[role=list]`) | ✅ | viñetas visibles (puede ser correcto) |
| Links heredan color/decoración | ❌ | ✅ | ✅ | azul/morado UA |
| `:focus-visible` con outline | ❌ | ✅ | ❌ | foco invisible sin outline |
| `prefers-reduced-motion` | ❌ | ✅ | ❌ | animaciones sin respeto |
| Correcciones de bugs de motores | ✅ (su razón de ser) | ❌ | ❌ | bugs solo en un navegador |

## Relacionadas

- [[Html]] — dónde van los `<link>` de hojas de estilos
- [[Box Model]] — un normalize moderno también unifica `box-sizing`
- [[TailwindCSS - Sintaxis Básica]] — Preflight, el reset incluido en Tailwind y su sistema de capas
- [[Especificidad]] — por qué el orden de carga es tan estricto
- [[CSS Introducción]] — cascada, orígenes y capas en contexto
- [[Estilos para escribir CSS]] — cómo un reset convive con BEM y metodologías
- [[Formularios]] — qué estilos UA heredan los inputs y por qué importa
- [[Responsive Web Design]] — viewport y ajustes de texto en móvil relacionados con normalize
