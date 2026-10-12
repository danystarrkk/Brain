---
tags:
  - desarrollo
  - frontend
  - tailwindcss
tipo: concepto
estado: terminado
---

# TailwindCSS - Instalación y Configuración

**Tailwind CSS** es un framework CSS *utility-first*: en lugar de escribir clases propias, compones la interfaz usando utilidades predefinidas (`bg-red-500`, `p-4`, `text-xl`) directamente en el HTML (ver [[TailwindCSS - Sintaxis Básica]]). Esta nota es la referencia completa del setup con **Tailwind v4**: arquitectura, vías de instalación, configuración de tema CSS-first, detección de contenido y build de producción.

> [!info] Tailwind v4 es CSS-first
> A diferencia de la v3, en la v4 **no existe `tailwind.config.js` por defecto**: la configuración y personalizaciones se hacen desde el propio CSS (`@import "tailwindcss"`, `@theme`, `@config` para compatibilidad). También cambió el paquete del ejecutable: ahora es `@tailwindcss/cli`.

## Arquitectura v4: motor Oxide y configuración en CSS

| Capa | v3 | v4 (2024+) |
|---|---|---|
| Config | `tailwind.config.js` (JS) | **CSS-first**: `@theme` en tu CSS (JS opcional vía `@config`) |
| Motor | JS (JIT en Node) | **Oxide** (Rust + Lightning CSS): ~3× más rápido en builds grandes |
| JIT | Opt-in en 3.x, estándar en 3.2+ | Único modo: genera solo las utilidades usadas |
| Prefijos de paquete | `tailwindcss/core`, `/postcss`… | `tailwindcss` (core), `@tailwindcss/cli`, `@tailwindcss/vite`, `@tailwindcss/postcss` |
| Detección de contenido | `content: [...]` explícito | **Automática** desde la raíz del proyecto (con `.gitignore`) + `@source` |
| Preflight | Preflight configurable | Igual, con activación explícita vía `@import` layer |

El pipeline: al compilar, Tailwind **escanea el código en busca de nombres de utilidades candidatos** (tokens que parecen clases suyas), genera esas reglas con sus valores resueltos del tema, y descarta el resto. En v4 esto lo hace Oxide en Rust, con caché incremental.

## Instalación: matriz completa de vías

| Vía | Paquetes | Cuándo usarla |
|---|---|---|
| **CLI** | `tailwindcss @tailwindcss/cli` | Proyectos vanilla, aprendizaje, setups a medida |
| **Vite** | `tailwindcss @tailwindcss/vite` | React/Vue/Svelte con Vite (recomendado — ver [[ReactJS - Vite]]) |
| **PostCSS** | `tailwindcss @tailwindcss/postcss` | Pipelines PostCSS existentes, webpack, Astro |
| **Laravel** | `tailwindcss @tailwindcss/laravel` | Apps Laravel (Vite integration oficial) |
| **Browser** | `@tailwindcss/browser@4` | Prototipos por CDN — **no producción** |
| **Framework** | plugins de Next.js/Nuxt/SvelteKit/ Remix | Sus integraciones documentadas |

Requisito común: **Node.js completo** (npm/npx):

```bash
node -v   # verificar instalación
npm -v
```

### Ruta A — CLI (la directa, paso a paso)

**1. Iniciar el proyecto**:

```bash
npm init -y
```

El flag `-y` acepta los valores por defecto sin pasar por el asistente interactivo. Se genera `package.json` en la raíz.

![[Pasted image 20260603094619.png]]

![[Pasted image 20260603094740.png]]

**2. Instalar Tailwind**:

```bash
npm install tailwindcss @tailwindcss/cli
```

- `tailwindcss`: el motor (utilities, theme, preflight).
- `@tailwindcss/cli`: el ejecutable que escanea el código y genera el CSS.

**3. Estructura de archivos** — entrada CSS con el import único:

```css
/* src/input.css */
@import "tailwindcss";
```

Y el `index.html` apuntando al CSS **compilado** (`style.css`, todavía no existe):

```html
<!doctype html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link href="/src/style.css" rel="stylesheet">
</head>
<body>
  <h1 class="text-3xl font-bold underline">
    Hello world!
  </h1>
</body>
</html>
```

![[Pasted image 20260603095821.png]]

**4. Compilar**:

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/style.css
```

> [!tip] ¿Por qué no se aplican los estilos sin compilar?
> Tailwind no funciona en tiempo de ejecución: el CLI escanea los archivos buscando clases utilitarias (`text-3xl`, `font-bold`, `underline`) y escribe **solo esas** en el CSS de salida. Sin compilar, el archivo no existe.

**Modo desarrollo** (watch):

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/style.css --watch
```

### Ruta B — Vite (la habitual en proyectos React)

```bash
npm create vite@latest mi-app -- --template react
cd mi-app && npm install
npm install tailwindcss @tailwindcss/vite
```

```js
// vite.config.js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
});
```

```css
/* src/index.css — importado desde main.jsx */
@import "tailwindcss";
```

### Ruta C — PostCSS

```bash
npm install tailwindcss @tailwindcss/postcss
```

```js
// postcss.config.js
export default {
  plugins: { '@tailwindcss/postcss': {} },
};
```

> [!warning] No mezclar CLI y plugin a la vez
> Un solo integrador por proyecto: o CLI, o Vite plugin, o PostCSS. Los tres generan el mismo CSS final; tener dos duplica trabajo y genera confusión sobre cuál es la fuente de verdad.

## Anatomía de `@import "tailwindcss"`

La única línea del setup trae **tres capas**, en orden:

```css
@import "tailwindcss";
// equivalente a (v4, orden de capas explícito):
// @import "tailwindcss/theme.css"  layer(theme);   ← tokens @theme → custom properties
// @import "tailwindcss/preflight.css" layer(base); ← reset de estilos base
// @import "tailwindcss/utilities.css" layer(utilities); ← las utilidades activas
```

| Capa | Contenido |
|---|---|
| `theme` | Los valores del tema convertidos a **custom properties** (`--color-red-500`, `--spacing`, `--font-sans`…) |
| `base` (preflight) | El reset: box-sizing, margins en 0, herencia de tipografía, etc. |
| `utilities` | Solo las utilidades **detectadas en tu código**, con sus variantes |

Desglose opcional (cuando querés sin preflight o piezas separadas):

```css
@import "tailwindcss/theme.css" layer(theme);
@import "tailwindcss/utilities.css" layer(utilities);
/* sin preflight: simplemente no se importa */
```

## Configuración de tema: `@theme`

En v4 el tema se define **en CSS**, y cada token se convierte en una custom property utilizable:

```css
@import "tailwindcss";

@theme {
  /* Colores: crea utilities bg-*, text-*, border-* Y la var --color-* */
  --color-primario: #3498db;
  --color-primario-oscuro: #2980b9;
  --color-exito: #2ecc71;

  /* Tipografías */
  --font-display: "Inter", sans-serif;

  /* Escala de espaciado (define p-*, m-*, gap-*...) */
  --spacing: 0.25rem;   /* p-1 = 0.25rem, p-4 = 1rem, p-12 = 3rem */

  /* Radios, breakpoints, sombras… */
  --breakpoint-tableta: 48rem;    /* crea la variante tableta: */
  --radius-boton: 0.5rem;         /* rounded-boton */
}

/* Uso directo de los tokens generados */
.tema-oscuro {
  --color-fondo: #111;
}
```

| Tipo de token | Prefijo | Utility generada | Extra |
|---|---|---|---|
| Color | `--color-x` | `bg-x`, `text-x`, `border-x`, `fill-x`… | También var `--color-x` para CSS propio |
| Fuente | `--font-x` | `font-x` | |
| Espaciado | `--spacing` (escala base) | escala `p-*`/`m-*`/`gap-*` completa | Recalcula toda la escala |
| Radius | `--radius-x` | `rounded-x` | |
| Breakpoint | `--breakpoint-x` | variante responsive `x:` | |
| Sombra | `--shadow-x` | `shadow-x` | |
| Animación | `--animate-x` | `animate-x` | |

```css
/* Valores con lógica: símbolos de función */
@theme {
  --spacing: calc(var(--spacing-base) * 1px);  /* temas dinámicos */
  --color-fondo: var(--tema-fondo);             /* encadenar con custom props */
}
```

Para mantener compatibilidad con configs JS heredadas existe `@config "./tailwind.config.js";` (modo compatibilidad), pero la vía nativa es `@theme`.

## `@apply` y extracción de componentes

`@apply` permite usar utilidades **dentro de CSS propio**:

```css
.btn-primario {
  @apply bg-primario text-white px-4 py-2 rounded-boton font-medium
         hover:bg-primario-oscuro transition-colors;
}
```

| `@apply` bien | `@apply` mal (anti-patrón) |
|---|---|
| Clases de componente **estables** que se reutilizan tal cual en many places | "Ahorrar escribir" en el HTML (para eso están las utilities) |
| Normalizar un patrón visual (botón de formulario) | Encadenar 15 utilities = solo moviste el problema |
| Dentro de CSS modules / capas de componente | Usarlo en variantes condicionales complejas (mejor JS) |

> [!warning] `@apply` pierde el JIT
> Cada `@apply` es una regla fija en el CSS: no hereda la semántica de variantes de contexto. Además rompe la idea utility-first (el HTML deja de documentar el estilo). Regla: extráe **componentes** con `@apply` (o clases normales), no "atajos de sintaxis".

Alternativa sin `@apply` — componentes en JSX (React):

```jsx
// Patrón "cn": composición de utilities en JS (clsx / tailwind-merge)
const clases = 'bg-primario text-white px-4 py-2 hover:bg-primario-oscuro';
<button className={clases}>Enviar</button>
```

## Preflight: qué resetea

El preflight es el reset de Tailwind (heredado de modern-normalize):

- `box-sizing: border-box` universal.
- Margin/padding de `body`, `h1–h6`, `p`, `ul`, `blockquote`… a **0**.
- Herencia de `font-family`, `font-size`, `line-height`, `color` desde `html`.
- Reset de `button`, `input`, `select`, `textarea` (background transparent, herencia de tipografía).
- Imágenes `display: block; max-width: 100%; height: auto`.
- Listas sin `list-style`, headings sin tamaño propio.

| Reset | Filosofía | Peso |
|---|---|---|
| **Preflight (Tailwind)** | Base uniforme para componer utilities | Se desactiva con `@import` omitido o `all: revert` selectivo |
| [[Normalize]] | Preserva estilos útiles del agente de usuario | Chico |
| Reset agresivo (Eric Meyer) | Todo a 0 | Máximo control, más trabajo |

> [!warning] Preflight rompe estilos heredados
> Si agregás Tailwind a un proyecto existente, los headings pierden tamaño, las listas pierden viñetas y los inputs pierden apariencia: todo eso ahora es **explícito** (`text-2xl`, `list-disc`, `border`…). Es esperado, no un bug.

## Detección de contenido: cómo encuentra Tailwind tus clases

| Mecanismo | Comportamiento |
|---|---|
| **Detección automática (v4)** | Escanea el proyecto desde la raíz del proyecto (el directorio del CSS o cwd), **respetando `.gitignore`**, ignorando binarios y `node_modules` |
| **`@source`** | Paths explícitos cuando la detección automática no basta (contenido generado, fuera del árbol, heredado) |
| **`@source inline()`** | Fuerza utilities aunque no aparezcan literales (p. ej. clases construidas dinámicamente) |

```css
@import "tailwindcss";

/* Fuera del árbol normal */
@source "../lib/components";

/* Clases dinámicas: asegurar que existan en el CSS */
@source inline("bg-*-500 text-*-500");
```

> [!warning] Clases construidas dinámicamente
> `class="bg-${color}-500"` **no se detecta** (el scanner busca literales). Soluciones: mapa completo de clases (`{ rojo: 'bg-rojo-500', ... }`), `@source inline()`, o safelist vía config. Es el error #1 de Tailwind junto a rutas mal vistas.

## Modo oscuro: `media` vs `class`

En v4 se declara en CSS:

```css
@import "tailwindcss";

/* Estrategia 1: respeta el sistema */
@custom-variant dark (&:where(.dark, .dark *));

/* Estrategia 2: media query del sistema */
@custom-variant dark (&:where(@media (prefers-color-scheme: dark)));
```

| Estrategia | Selector | Cuándo |
|---|---|---|
| `media` | `@media (prefers-color-scheme: dark)` | Sin toggle en la UI; sigue al sistema |
| `class` (`.dark`) | `.dark &` | Botón de tema en la app — la más usada en apps reales |

```html
<div class="bg-white dark:bg-gray-900 text-black dark:text-white">
  <!-- claro por defecto; oscuro bajo .dark o el media query según variante -->
</div>
```

## Ecosistema de plugins

| Plugin | Rol |
|---|---|
| `@tailwindcss/typography` | `prose` — estilo de contenido largo (markdown, blogs) |
| `@tailwindcss/forms` | Reset coherente de formularios nativos |
| `@tailwindcss/aspect-ratio` | `aspect-video` y compañía |
| `@tailwindcss/container-queries` | Variantes por container (v4: ya en core, plugin legacy) |
| Plugins comunitarios | datepicker, headless-ui helpers, etc. |

En v4 los plugins son **paquetes CSS** que se importan:

```css
@import "tailwindcss";
@import "@tailwindcss/typography";
```

## Build de producción

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/style.css --minify
# o con el plugin de Vite: vite build ya lo incluye
```

- **Purgado JIT**: el CSS final contiene **solo** las utilidades usadas — un sitio pequeño sale con ~5–15 KB de CSS (gzip). No hay "purge config" como en v3: la detección es el mecanismo.
- **Minificación**: con `--minify` (Lightning CSS); Vite/PostCSS lo delegan al bundler.
- **Orden de capas**: utilities dentro de `@layer utilities` de CSS nativo — la cascada resuelve conflicts por orden de capa, no por orden de aparición (ver [[Especificidad]]).
- **Source maps**: CLI `--map`; integrados en el build del bundler.

## Tailwind v3 → v4: qué cambió

| Tema | v3 | v4 |
|---|---|---|
| Config | `tailwind.config.js` | `@theme` en CSS (`@config` para legacy) |
| Content | `content: ['...']` manual | Detección automática + `@source` |
| Dark mode | `darkMode: 'media' \| 'class'` | `@custom-variant dark (...)` |
| Variants | `variants` config | `@custom-variant` |
| Plugins JS | `plugin(({...}) => ...)` | Paquetes CSS `@import` |
| Paquete CLI | `tailwindcss` | `@tailwindcss/cli` |
| Prefijos globales | `prefix: 'tw-'` | `@variant` / `prefix` vía CSS |
| Engine | JS | Oxide (Rust) + Lightning CSS |
| Base color | Paleta extendida v3 | Paleta revisada (oklch), `--color-*` renombrados en parte |

Migración oficial: `npx @tailwindcss/upgrade` migra config y clases renombradas.

## Cuándo Tailwind es la herramienta equivocada

- **Contenido tipográfico largo sin sistema de diseño** — necesitás estilos base propios; `prose` ayuda pero no reemplaza.
- **Equipos 100% CSS** que no quieren clases en HTML — un CSS Modules o BEM convencional es más natural para ellos.
- **Diseño altamente dinámico/animado** (canvas, animaciones complejas) — el peso de Tailwind no aporta.
- **Emails HTML** — soporte de clases externas limitado; CSS inline dedicado.
- Dicho esto: Tailwind convive bien con Sass/CSS modules por secciones (Tailwind para utilities y layout, CSS propio para lo que las utilities no cubren — ver [[Qué es Sass]]).

## Errores comunes

1. **Sin estilos tras compilar** — falta la compilación (Tailwind no es runtime), o el `<link>` apunta al archivo equivocado.
2. **Utility inexistente** — typo (`text-big` no existe) o valor fuera de escala; el JIT solo genera lo que reconoce.
3. **Clases dinámicas no aparecen** — construir nombres con template strings; usar mapas o `@source inline`.
4. **Colisión de variantes** — `md:hover:` (en breakpoints) vs `hover:md:` (mal orden); variantes en orden: responsive → estado → utilidad.
5. **Preflight "rompe" el diseño existente** — esperado; ajustar con estilos explícitos o quitar preflight.
6. **Doble motor CLI+PostCSS** — duplica compilación; elegir uno.
7. **Subcarpeta de despliegue** — CSS OK pero assets 404: configurar `base` en Vite (ver [[ReactJS - Vite]]).

## Cheatsheet

| Tema | Referencia |
|---|---|
| Setup mínimo | `npm i tailwindcss @tailwindcss/cli` + `@import "tailwindcss";` |
| Compilar | `npx @tailwindcss/cli -i in.css -o out.css --watch` |
| Vite | `@tailwindcss/vite` plugin + `@import "tailwindcss"` |
| Tema | `@theme { --color-x: ...; --spacing: ...; }` |
| Componentes | `@apply utilities;` (con medida) |
| Preflight | Capa `base` del import; reset completo |
| Fuentes de contenido | Auto-detección + `@source "../ruta"` |
| Dark | `@custom-variant dark (&:where(.dark, .dark *));` |
| Plugins | `@import "@tailwindcss/typography";` |
| Producción | `--minify`; solo utilities usadas (JIT) |
| Migrar v3 | `npx @tailwindcss/upgrade` |

## Relacionadas

- [[TailwindCSS - Sintaxis Básica]] — cómo se escriben y combinan las utilidades
- [[CSS Introducción]] — cascada y fundamentos que Tailwind abstrae
- [[Especificidad]] — cómo resuelve conflictos por capas (`@layer`)
- [[Flexbox]] — las propiedades detrás de `flex-*`, `justify-*`, `items-*`
- [[CSS Grid]] — detrás de `grid-cols-*`, `gap-*`
- [[Responsive Web Design]] — el patrón mobile-first de las variantes `sm:`–`2xl:`
- [[Normalize]] — el ancestro de los resets que Tailwind modernizó
- [[ReactJS - Vite]] — integración del plugin en proyectos Vite/React
