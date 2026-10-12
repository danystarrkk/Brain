---
tags:
  - desarrollo
  - frontend
  - tailwindcss
tipo: concepto
estado: terminado
---

# TailwindCSS - Sintaxis Básica

Referencia completa de **cómo se escriben** las utilidades de Tailwind: anatomía de las clases, catálogo de variantes, escalas de valores, mapeo a CSS nativo y patrones responsive. El setup previo vive en [[TailwindCSS - Instalación y Configuración]].

## Filosofía utility-first (y de dónde viene)

La idea no es nueva: las "utilities" (clases de una sola propiedad) existen desde años 2000 (oocss, inuitcss, Basscss). Tailwind las sistematizó (2017, Adam Wathan) con tres decisiones que definieron el estilo:

1. **API completa y consistente**: una utility para cada propiedad/valor útil del CSS moderno — no un subconjunto estilizado.
2. **Diseñar en el HTML**: las decisiones visuales (espaciado, color, tipografía) se toman donde se compone la interfaz, sin saltar entre archivos.
3. **JIT real**: el compilador genera **solo las clases usadas** — el "framework gigante" es un mito: el CSS final de un sitio típico ronda los 5–15 KB gzip.

> [!info] Utility-first ≠ sin diseño
> Componer clases no es "estilo inline con más pasos": las escalas del `@theme` **fuerzan consistencia** (`p-4` = 1rem en todo el sistema, nunca 13px). La restricción del sistema ES el diseño coherente. Ver tokens en [[TailwindCSS - Instalación y Configuración#Configuración de tema: @theme]].

## Anatomía de una clase

Patrón completo: **`[variante:]propiedad-valor`**, con modificadores opcionales:

```
md:hover:bg-red-500/70
│  │     │     │   │
│  │     │     │   └─ modificador de valor: opacidad 70%
│  │     │     └───── valor de la escala (500)
│  │     └─────────── utility (background-color)
│  └───────────────── variante de estado (hover)
└──────────────────── variante responsive (md, ≥768px)
```

| Parte | Forma | Ejemplos |
|---|---|---|
| Utility base | `propiedad-valor` | `p-4`, `text-lg`, `bg-blue-500` |
| Variants | `estado:` / `sm:` / `dark:` … (encadenables) | `hover:`, `focus:`, `md:`, `group-hover:`, `aria-checked:` |
| Valor arbitrario | `[valor]` con guion bajo = espacio | `w-[13px]`, `grid-cols-[1fr_200px]`, `top-[17px]` |
| Modificador opacidad | `valor/nn` o `[color]/[nn]` | `bg-red-500/50`, `text-white/80` |
| Negativo | prefijo `-` | `-mt-4`, `-translate-x-2` |
| Importante | sufijo `!` | `bg-red-500!` (Tailwind v4; en v3 `!bg-red-500`) |

### Valores arbitrarios `[...]`

Cualquier valor CSS que no esté en la escala, entre corchetes (guion bajo = espacio):

```html
<div class="w-[13px] h-[calc(100vh-4rem)] grid-cols-[200px_1fr_auto]
            text-[#1da1f2] leading-[3.5] tracking-[0.2em]
            data-[active=true]:bg-blue-500">
  ...
</div>
```

También funciones completas: `shadow-[0_8px_30px_rgb(0,0,0,0.12)]`, `@media` arbitrario: `[@media(min-width:900px)]:p-8`.

> [!warning] Los arbitrarios rompen la consistencia de escala
> `p-[13px]` es escape hatch, no práctica: cada valor único es un token de diseño no documentado. Usá `[...]` para lo genuinamente único (medidas de diseño externa, cálculos), y para todo lo demás la escala.

### Negativos y `!important`

```html
<div class="-mt-4 -translate-y-1/2">   <!-- solo para propiedades donde tiene sentido -->
<div class="bg-red-500! text-white!">  <!-- v4: sufijo ; v3: prefijo !bg-red-500 -->
```

Negativos válidos: márgenes, offsets de grid/posición, translate, z-index negativo. `!important` es último recurso (p. ej. ganarle a un plugin); su uso extensivo señala problemas de orden de capas (ver [[Especificidad]]).

## Catálogo de variantes

### Responsive: mobile-first

| Variante | Ancho mínimo | Nota |
|---|---|---|
| `sm` | 640px | |
| `md` | 768px | |
| `lg` | 1024px | |
| `xl` | 1280px | |
| `2xl` | 1536px | |
| `min-*` / `max-*` | arbitrarios | `min-[900px]:`, `max-lg:` |

Todas son `min-width` (mobile-first, igual que [[Responsive Web Design]]). Se **apilan**: una utility escrita en `md:` aplica desde 768px en adelante, y puede sobreescribirse en `lg:`.

### Estados de interacción y foco

| Variante | Selector CSS | Uso |
|---|---|---|
| `hover:` | `&:hover` | Pasar el mouse |
| `focus:` | `&:focus` | Foco visible (accesibilidad: no eliminarlo) |
| `focus-visible:` | `&:focus-visible` | Foco solo por teclado — preferido a `focus:` |
| `focus-within:` | `&:focus-within` | Contenedor cuando un hijo tiene foco |
| `active:` | `&:active` | Botón presionado |
| `visited:` | `&:visited` | Enlaces visitados |
| `target:` | `&:target` | Ancla activa |
| `disabled:` | `&:disabled` | Deshabilitados |
| `enabled:` / `checked:` / `required:` / `invalid:` | pseudo-classes homónimas | Formularios |
| `first:` `last:` `odd:` `even:` `nth-*:` | `:first-child`… | Listas |
| `group-hover:` | `.group:hover &` | Estado del padre |
| `peer-checked:` | `.peer:checked ~ &` | Estado de hermano previo |
| `peer-focus:` / `peer-disabled:` | análogos | Sibling state |

```html
<!-- Patrón group: el contenedor marca .group, los hijos reaccionan -->
<div class="group">
  <img class="opacity-70 group-hover:opacity-100 transition-opacity">
  <button class="hidden group-hover:block">Ver</button>
</div>

<!-- Patrón peer: el checkbox controla al label hermano -->
<input type="checkbox" class="peer sr-only" id="tos">
<label for="tos" class="text-gray-400 peer-checked:text-black peer-focus-visible:underline">
  Acepto los términos
</label>
```

### Sistema, aria y datos

| Variante | Selector | Ejemplo |
|---|---|---|
| `dark:` | según estrategia configurada | `dark:bg-gray-900` (ver setup) |
| `aria-*` | `[aria-...]` | `aria-checked:opacity-100`, `aria-[sort=ascending]:` |
| `data-*` | `[data-...]` | `data-[active=true]:bg-blue-500`, `data-[state=open]:` |
| `supports-[...]` | `@supports` | `supports-[backdrop-filter]:bg-white/50` |
| `motion-safe:` / `motion-reduce:` | preferencias de movimiento | Accesibilidad vestibular |
| `print:` | `@media print` | Hoja de estilo de impresión |
| `rtl:` / `ltr:` | direccionalidad | i18n |
| `[selector]` arbitrario | cualquiera | `[@media(min-width:900px)]:p-8` |

**Orden de variantes**: de afuera hacia adentro del contexto: `md:hover:bg-...` (responsive → estado → utility). El orden inverso falla silenciosamente.

## Sistema de espaciado

Escala derivada de `--spacing` (0.25rem = 4px por defecto): **`{utility}-{n}` = n × escala**.

```html
p-0 → 0        p-1 → 0.25rem    p-2 → 0.5rem    p-4 → 1rem
p-8 → 2rem     p-12 → 3rem      p-px → 1px      p-[13px] → arbitrario
```

| Utility | Propiedad | Ejemplos |
|---|---|---|
| `p-*` | `padding` | `p-4` |
| `px-*` / `py-*` | eje X / Y | `px-6 py-2` |
| `pt-*` `pr-*` `pb-*` `pl-*` | por lado | `pl-10` |
| `m-*`, `mx-*`, `-mt-*`… | `margin` (+ negativos) | `-mx-4` |
| `gap-*`, `gap-x-*`, `gap-y-*` | `gap` | `gap-4` |
| `space-x-*` | `> * + *` margin | legacy; preferir `gap` |

> [!tip] `gap` sobre `space-x`
> `space-x-4` aplica margin a `* + *`; con wrap y reverse se comporta raro. `gap-4` en flex/grid es más predecible (ver [[Flexbox]]).

## Sistema de colores y opacidad

Formato: **`[propiedad]-[color]-[escala]`** — paleta completa por defecto (slate, gray, red, amber, emerald, blue, sky, indigo, violet, purple, fuchsia, pink, rose) con escalas 50–950.

```html
bg-blue-500          <!-- azul medio -->
bg-blue-500/70       <!-- 70% de opacidad (color-mix en CSS) -->
text-emerald-600
border-gray-200
ring-indigo-500      <!-- ring: el "outline" de foco de Tailwind -->
from-sky-400 via-blue-500 to-indigo-600   <!-- gradientes -->
```

| Utility de color | Propiedades |
|---|---|
| `bg-*` | `background-color` (+ `bg-gradient-to-*`, `bg-cover`…) |
| `text-*` | `color` (o `font-size` si es escala de tamaño — desambigúa el compilador por valor) |
| `border-*` `divide-*` | color de borde / divisores |
| `ring-*` | `box-shadow` de anillo de foco |
| `placeholder-*` | color de placeholder |
| `from-/via-/to-*` | `background-image: linear-gradient(...)` |
| Colores custom | tokens `--color-x` en `@theme` (ver setup) |

Opacidad: `/nn` de 0–100, o arbitraria `/[13%]`, o con variables `bg-red-500/[var(--mi-opacidad)]`.

> [!info] Escalas 50–950
> 50 = tinte claro (backgrounds), 500 = "color puro", 900–950 = fondos oscuros/texto. Los usos reales se concentran en 50–200 (superficies), 500–600 (acciones), 700–900 (texto). En v4 los colores son **oklch** (gama más amplia).

## Tipografía

```html
<h1 class="text-4xl font-bold tracking-tight">Título</h1>
<p class="text-base font-sans leading-relaxed text-gray-700">Cuerpo</p>
<span class="text-sm font-mono uppercase">mono</span>
```

| Grupo | Utilities | CSS |
|---|---|---|
| Tamaño | `text-xs` `sm` `base` `lg` `xl` `2xl`…`9xl` | `font-size` + `line-height` acoplado |
| Peso | `font-thin`…`font-black` (100–900) | `font-weight` |
| Familia | `font-sans` `font-serif` `font-mono` (+ custom) | `font-family` |
| Espaciado | `tracking-tight/widest` | `letter-spacing` |
| Interlínea | `leading-none/snug/relaxed/loose` | `line-height` |
| Alineación | `text-left/center/right/justify` | `text-align` |
| Decoración | `underline line-through no-underline` | `text-decoration` |
| Transform | `uppercase lowercase capitalize normal-case` | `text-transform` |
| Wrapping | `whitespace-nowrap break-words text-ellipsis` | `white-space`, `overflow-wrap` |

## Mapeo utility → CSS nativo (layout)

Las utilities de layout son **mapeos 1:1 de [[Flexbox]] y [[CSS Grid]]** — aprender Tailwind es aprender el CSS detrás:

| Utility | CSS | Referencia |
|---|---|---|
| `flex` / `inline-flex` | `display: flex` | [[Flexbox]] |
| `flex-row` / `flex-col` | `flex-direction` | [[Flexbox]] |
| `flex-1` / `flex-auto` / `flex-none` | `flex: 1 1 0%` / `1 1 auto` / `0 0 auto` | [[Flexbox]] |
| `flex-wrap` | `flex-wrap: wrap` | [[Flexbox]] |
| `justify-start/center/end/between/around/evenly` | `justify-content` | [[Flexbox]] |
| `items-start/center/end/baseline/stretch` | `align-items` | [[Flexbox]] |
| `self-*` | `align-self` | [[Flexbox]] |
| `content-*` | `align-content` | [[Flexbox]] |
| `order-1` | `order: 1` | [[Flexbox]] |
| `grid` / `inline-grid` | `display: grid` | [[CSS Grid]] |
| `grid-cols-3` | `grid-template-columns: repeat(3, minmax(0,1fr))` | [[CSS Grid]] |
| `col-span-2` | `grid-column: span 2` | [[CSS Grid]] |
| `grid-rows-*` / `row-span-*` | filas | [[CSS Grid]] |
| `auto-rows-min` | `grid-auto-rows` | [[CSS Grid]] |
| `block` `inline` `hidden` `table`… | `display` | [[Tipos de Display]] |
| `relative` `absolute` `fixed` `sticky` | `position` | [[Tipos de Posiciones]] |
| `top-4` `left-1/2` `inset-0` | offsets | [[Tipos de Posiciones]] |
| `overflow-auto` `truncate` | overflow | |
| `z-10` | `z-index` | |
| `shadow-lg` | `box-shadow` | [[Sombras CSS]] |
| `rounded-lg` | `border-radius` | |
| `border` `border-2` | `border-width` | |

> [!tip] Las utilities resuelven el "problema de los valores intermedios"
> `grid-cols-3` genera `repeat(3, minmax(0, 1fr))` — no `repeat(3, 1fr)`: el `minmax(0,1fr)` evita el desborde de contenido típico de grid. Tailwind codificó años de fixes de CSS real en sus defaults (como `min-width: 0` implícito en flex — ver [[Flexbox]]).

## Patrones responsive (recetas mobile-first)

```html
<!-- 1. Stack → fila -->
<div class="flex flex-col gap-4 md:flex-row">

<!-- 2. Grilla de 1 → 2 → 3 columnas -->
<div class="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">

<!-- 3. Sidebar visible solo en escritorio -->
<aside class="hidden lg:block lg:w-64">…</aside>

<!-- 4. Tipografía fluida por breakpoint -->
<h1 class="text-3xl md:text-5xl lg:text-6xl font-extrabold">…</h1>

<!-- 5. Padding de página responsivo -->
<main class="mx-auto max-w-5xl px-4 sm:px-6 lg:px-8">…</main>

<!-- 6. Nav móvil con menú (estado con peer/group) -->
<button class="lg:hidden" aria-expanded="false">☰</button>
```

Regla de lectura: **primero lo móvil** (clases base), luego las variantes `sm/md/lg` sobreescriben hacia arriba — el mismo orden mental de [[Responsive Web Design]].

## Cuando la lista de clases se complica → extracción

Señales de que una lista de utilities necesita estructura:

1. **> 6–8 utilities** en un mismo elemento, repetidas en varios lugares.
2. La **misma combinación** aparece en 3+ componentes.
3. El JSX/HTML deja de ser legible ("ruido de clases").

Opciones, en orden de preferencia:

| Técnica | Cuándo | Cómo |
|---|---|---|
| **Constante JS/TS** | Misma lista en un archivo | `const CARD = 'rounded-xl border p-4 shadow-sm'` |
| **Componente** | Patrón con lógica/props | `<Card variant="primario">` encapsula las clases |
| **`@apply` en CSS** | Clase de componente estable | Ver [[TailwindCSS - Instalación y Configuración#apply y extracción de componentes]] |
| **Plugin** | Patrón de equipo completo | Paquete CSS propio |

No extraigas **demasiado** pronto: la legibilidad de las utilities inline es el punto del enfoque; extraer cuando el patrón ya es estable.

## Rendimiento y tooling

- **JIT / Oxide**: solo se generan las clases usadas, con caché incremental — el tiempo de build apenas crece con el número de clases en el proyecto (ver arquitectura en el setup).
- **CDN/browser caveat**: `@tailwindcss/browser` compila **en el navegador** en cada carga (sin caché de build): prototipos y demos, jamás producción.
- **Orden determinista**: el orden de utilities en el CSS final no es el del HTML sino el del orden de propiedades del compilador; por eso se resuelven conflictos sin `!important` — y por eso "mover clases de lugar" nunca cambia la cascada (ver [[Especificidad]]).
- **Tailwind IntelliSense** (extensión VS Code): autocompletado, preview de valores, detección de clases inválidas en el buffer. Imprescindible; configurar `tailwindCSS.includeLanguages` para templating engines.
- Otras herramientas: `tailwind-merge` (deduplicar clases conflictivas en JS), `clsx` (clases condicionales), prettier-plugin-tailwindcss (ordena las clases automáticamente).

```js
// clsx + composición condicional
import clsx from 'clsx';
<button className={clsx('px-4 py-2 rounded', activo ? 'bg-blue-600' : 'bg-gray-200')}>
```

## Errores comunes

1. **Clase dinámica inexistente en el CSS** — `text-${size}` no compila; mapa de clases o `@source inline` (ver setup).
2. **`md:hover:` vs `hover:md:`** — orden de variantes incorrecto; responsive primero.
3. **Valores arbitrarios por todos lados** — pierde consistencia; volver a la escala.
4. **`text-red` sin escala** — la utility requiere el número (`text-red-500`); colores custom necesitan token `--color-*`.
5. **Asumir orden de aparición en el HTML** — el CSS tiene orden propio; dos utilities conflictivas se resuelven por orden de propiedades, no por posición en `class`.
6. **`!important` masivo** — señal de conflicto estructural; revisar capas y variantes.
7. **Olvidar `group`/`peer` en el padre** — `group-hover:` sin la clase `group` en el contenedor no hace nada.

## Cheatsheet

| Tema | Patrón |
|---|---|
| Clase completa | `[variante:]prop-valor[/opacidad][!]` |
| Negativo | `-mt-4` |
| Arbitrario | `w-[13px]`, `grid-cols-[1fr_200px]` |
| Responsive | `sm:` `md:` `lg:` `xl:` `2xl:` (min-width) |
| Estado | `hover:` `focus-visible:` `active:` `disabled:` |
| Padre/hijo | `group` + `group-hover:` · `peer` + `peer-checked:` |
| Datos/aria | `data-[active=true]:` `aria-checked:` |
| Espaciado | `p-{n}` = n × 0.25rem · `gap-*` |
| Color + opacidad | `bg-blue-500/70` |
| Layout | `flex`, `grid-cols-*`, `justify-*`, `items-*` |
| Extracción | constante JS → componente → `@apply` |
| IDE | Tailwind IntelliSense + prettier plugin |

## Relacionadas

- [[TailwindCSS - Instalación y Configuración]] — setup, `@theme`, preflight y build
- [[Especificidad]] — por qué el orden de utilities no depende del HTML
- [[Flexbox]] — el CSS real detrás de `flex-*`, `justify-*`, `items-*`
- [[CSS Grid]] — detrás de `grid-cols-*`, `col-span-*`, `gap-*`
- [[Responsive Web Design]] — el patrón mobile-first de las variantes
- [[Tipos de Display]] — `hidden`, `block`, `flex` como utilities de display
- [[Tipos de Posiciones]] — `relative`, `absolute`, `sticky` + offsets
- [[Sombras CSS]] — las utilidades `shadow-*` por dentro
- [[Qué es Sass]] — la filosofía alternativa (preprocesador) con la que Tailwind convive
