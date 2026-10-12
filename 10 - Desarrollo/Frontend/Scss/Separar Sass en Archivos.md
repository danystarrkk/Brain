---
aliases: ["Partials Sass", "Módulos Sass", "Separar Sass"]
tags:
  - dev/frontend/scss
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Qué es Sass]]"
  - "[[Mixins]]"
  - "[[Sintaxis de Sass]]"
  - "[[Compilar Sass]]"
  - "[[Node JS]]"
fuente: "https://sass-lang.com/documentation/at-rules/use/"
---

# Separar Sass en Archivos

Un proyecto Sass real se divide en **módulos**: archivos pequeños con una responsabilidad, ensamblados por el sistema de módulos **`@use`/`@forward`**. Esta nota cubre la mecánica completa: partials, namespaces, configuración, índices y la migración desde el `@import` deprecado.

![[Pasted image 20250125074749.png]]

## Partials: mecánica del guion bajo

Los archivos **parciales** llevan `_` al inicio (`_variables.scss`, `_mixins.scss`). Esa convención le dice al compilador: **no generes CSS de este archivo por sí mismo** — solo existe cuando otro módulo lo carga. Sin `_`, la CLI compila el archivo a un `.css` independiente.

```bash
sass scss:dist
# scss/_variables.scss   → (nada; es partial)
# scss/app.scss          → dist/app.css
# scss/paginas/acerca.scss → dist/paginas/acerca.css   ← lo que evitamos
```

Reglas de resolución de paths en `@use`/`@forward`:

| Escrito | Significa |
|---|---|
| `@use 'variables'` | Mismo directorio → `_variables.scss` (o `variables.scss`) |
| `@use 'abstracts/variables'` | Relativo al archivo actual |
| `@use 'abstracts'` | Carpeta → busca `abstracts/_index.scss` |
| `@use 'sass:math'` | Módulo **incorporado** del lenguaje (los de `sass:*`) |

> [!warning] Partials no son "globals con otra cara"
> Cada módulo `@use` tiene su **propio scope**: las variables de `_variables.scss` no existen en quien lo carga salvo que accedan por namespace. El encapsulado es real — a diferencia del viejo `@import`.

## `@use`: cargar un módulo

```scss
// app.scss
@use 'abstracts/variables' as v;
@use 'abstracts/mixins' as m;

body {
  color: v.$color-texto;          // namespace + $ = variable del módulo
  font-family: v.$fuente-principal;
  @include m.sombra-suave;        // namespace + mixin
}
```

### Namespaces

| Forma | Resultado | Cuándo |
|---|---|---|
| `@use 'ruta'` | Namespace = **último segmento del path** (`variables.$x`, `mixins.$y`) | Por defecto |
| `@use 'ruta' as v` | Namespace aliasado (`v.$x`) | Abreviar paths largos |
| `@use 'ruta' as *` | **Sin namespace**: todo queda en el scope del módulo | Mínimos casos; peligroso en proyectos grandes (colisiones, opacidad) |
| `@use 'ruta' as nombre-huevo` | Namespace con guiones (`nombre-huevo.$x`) | Nombres compuestos |

> [!warning] `as *` anula el encapsulado
> Trae todas las variables/mixins al scope sin prefijo: dos módulos con `$color` definido colisionan silenciosamente (gana el último cargado). Regla de equipo: namespaces siempre, salvo en archivos de prueba o wrappers minúsculos.

### Reglas de carga

1. **`@use` va arriba** del archivo — antes que cualquier regla de estilo (incluso comentarios con `/*! */` puede conflictuar; siempre, arriba).
2. **Un módulo se carga una sola vez por "grafo de compilación"**: todo el mundo que lo `@use` recibe la **misma instancia** (singleton). Los efectos laterales (como un `@debug` o la existencia de sus reglas CSS) no se duplican.
3. Los `@use` **no heredan entre sí**: si `app.scss` usa `variables` y `mixins.scss` también, el namespace en `app.scss` es el que `app.scss` declaró — cada archivo pide lo suyo.
4. Las **reglas CSS** de un módulo (no solo funciones/mixins) se emiten **una vez**, en el primer lugar donde se carga el módulo en el grafo.

### Configuración con `with`

Los partials configurables exponen variables con `!default` y el consumidor las sobreescribe **al cargar**:

```scss
// _variables.scss
$color-primario: #3498db !default;
$spacing-base: 1rem !default;
$breakpoints: (tablet: 768px, desktop: 1024px) !default;

// _mixins.scss — consume la config vía su propio @use
@use 'variables' as v;
@mixin desde($nombre) { /* usa v.$breakpoints */ }

// app.scss — el consumidor configura ANTES de instanciar
@use 'abstracts/variables' with (
  $color-primario: #e74c3c,
  $breakpoints: (tablet: 800px, desktop: 1200px),
);
@use 'abstracts/mixins' as m;   // ← misma instancia ya configurada

body { color: variables.$color-primario; }
```

| Paso | Detalle |
|---|---|
| En el módulo | Variables con `!default` (sinónimo de "configurable") |
| En el consumidor | `@use ... with ($var: valor, ...)` solo puede configurar variables **`!default`** |
| Consistencia | Como el módulo es singleton, **toda** la app ve la config del primer `with` — si dos archivos configuran distinto, el segundo recibe warning/error |

> [!info] Cascada mental del sistema de módulos
> El orden de los `@use` importa **solo para las reglas CSS** que emiten los módulos: el CSS se escribe en el orden en que los módulos se cargan por primera vez (resolviendo dependencias primero). La resolución completa es un **grafo acíclico de módulos** cargado en post-order — el "cascada waterfall" es el orden de carga, no texto que se copia (a diferencia del `@import`).

## `@forward`: re-exportar y componer APIs

`@forward` hace que **este archivo re-exponga** lo de otro módulo: quien carga el wrapper ve la API del módulo original. Es la base de los **índices** (`_index.scss`) y de las APIs de librerías.

```scss
// abstracts/_index.scss
@forward 'variables';
@forward 'mixins';
@forward 'functions';

// app.scss — con una sola carga obtiene TODO lo de abstracts
@use 'abstracts';

body { color: variables.$color-texto; }  // namespace = nombre del archivo re-exportado
```

### Control de exposición

```scss
@forward 'variables' show $color-primario, $spacing-base;  // solo esto es público
@forward 'functions' hide $helper-interna;                 // todo menos esto
@forward 'variables' as tema-*;                            // prefijo: tema-$color-primario
```

| Directiva | Efecto |
|---|---|
| (sin opciones) | Re-exporta todo |
| `show $a, $b` | Whitelist: solo las listadas (resto invisible al consumidor) |
| `hide $a` | Blacklist: todo salvo las listadas |
| `as prefijo-*` | Renombra variables/mixins/functions con prefijo (evita colisiones entre módulos) |

### Configuración a través de `with` (config passthrough)

Un módulo intermedio puede **pasar la config** al módulo que re-exporta, permitiendo configurar "la carpeta entera" desde un solo punto:

```scss
// abstracts/_index.scss
@forward 'variables';
@forward 'mixins';   // mixins consume variables internamente

// app.scss — configura variables al cargar el índice
@use 'abstracts' with ($color-primario: #e74c3c);
```

Funciona porque el índice re-exporta `variables`, que tiene los `!default`: el `with` del índice se aplica a esas variables, y como `mixins` (que hace `@use 'variables'`) recibe la **misma instancia singleton**, todo el sistema queda configurado de una sola vez.

> [!tip] `@forward` vs `@use` — cuándo cada uno
> `@forward` **no** trae nada al scope del archivo que lo escribe (solo re-exporta). `@use` sí trae el namespace para usarlo. Un `_index.scss` solo hace `@forward`; un componente que necesita los mixins hace `@use 'abstracts'`.

### Carga circular

Si `a.scss` hace `@use 'b'` y `b.scss` hace `@use 'a'`, Dart Sass detecta el ciclo y falla con `This file is already being loaded` (no cuelga). La solución es estructural: extraer lo compartido a un tercer módulo (`_shared.scss`) que ambos usen. Los índices `_index.scss` bien diseñados (una dirección: app → índice → leaves) evitan el problema por diseño.

## `_index.scss`: la carpeta como módulo

Cuando una carpeta crece, su `_index.scss` la expone como **un único módulo** re-exportando sus archivos:

```scss
// abstracts/_index.scss
@forward 'variables';
@forward 'mixins';
@forward 'functions';

// base/_index.scss
@forward 'reset';
@forward 'tipografia';

// components/_index.scss
@forward 'botones';
@forward 'cards';
```

Y en el punto de entrada se carga **solo la carpeta** — Sass busca `_index.scss` dentro:

```scss
// app.scss
@use 'abstracts';
@use 'base';
@use 'components';
@use 'layout';

body {
  color: variables.$color-texto;   // vía abstracts/_index
  @include mixins.sombra-suave;
}
```

> [!warning] El índice no se compila solo
> Un `_index.scss` jamás produce CSS por sí mismo: **siempre debe consumirse con `@use`** desde el archivo principal (`app.scss`) o desde otro módulo. Si no aparecen tus estilos, casi siempre es porque falta ese `@use` en la entrada.

## Estructura canónica de proyecto

```
src/scss/
├── abstracts/          ← cero CSS emitido: solo tokens y herramientas
│   ├── _variables.scss   (configurables con !default)
│   ├── _mixins.scss
│   ├── _functions.scss
│   └── _index.scss
├── base/               ← reset, tipografía global, elementos elementales
│   ├── _reset.scss
│   ├── _tipografia.scss
│   └── _index.scss
├── components/         ← unidades reutilizables
│   ├── _botones.scss
│   ├── _cards.scss
│   └── _index.scss
├── layout/             ← header, footer, grid del sitio
│   ├── _header.scss
│   ├── _grid.scss
│   └── _index.scss
├── pages/              ← estilos exclusivos de páginas concretas (si los hay)
│   └── _inicio.scss
└── app.scss            ← único archivo sin "_": el punto de entrada
```

| Carpeta | Contenido | ¿Emite CSS? |
|---|---|---|
| `abstracts/` | Variables, mixins, funciones | ❌ nunca |
| `base/` | Reset, tipografía, elementales | ✅ |
| `components/` | Botones, cards, navbar | ✅ por uso |
| `layout/` | Header, footer, grid del sitio | ✅ |
| `pages/` | Estilos de páginas específicas | ✅ selectivo |
| `app.scss` | `@use` de todo **en orden de cascada** | El CSS final |

```scss
// app.scss — el orden DE LOS @use ES EL ORDEN DE LA CASCADA
@use 'abstracts';        // primero: tokens (no emite, pero configura)
@use 'base';             // reset antes que componentes
@use 'layout';
@use 'components';       // overrides de componentes sobre layout
@use 'pages';            // último: páginas ganan
```

> [!tip] Regla de orden
> Base antes que components, components antes que pages — **el orden de carga de los `@use` determina el orden de las reglas en el CSS final**, y por tanto quién gana la cascada en especificidades iguales (ver [[Especificidad]]).

### Patrones de organización para equipos

- **Config por carpeta**: cada carpeta configurable expone su config en el `_index` con `@forward ... with` — los equipos personalizan secciones sin tocar sus internals.
- **`show/hide` en los índices**: exponer solo la API pública (`show $tokens, $mixins de uso general`) evita que alguien use helpers internas.
- **Una responsabilidad por archivo**: `_botones.scss` no incluye cards. Archivos > 200 líneas son candidatos a dividirse.
- **`pages/` selectivo**: si `pages/` se carga con `@use 'pages'` en `app.scss`, todas las páginas viajan en el CSS global. Alternativa: un entry SCSS por página (`paginas/inicio.scss` con sus propios `@use`) — más trabajo de build, menos CSS.

## Migración desde `@import` (deprecado, eliminado en Sass 3.0)

Desde **Dart Sass 1.80.0**, `@import` está deprecado. La traducción es directa, pero tiene **cambios de comportamiento** que rompen código si no se conocen:

| Antes (`@import`) | Después (`@use`/`@forward`) | Cambio de comportamiento |
|---|---|---|
| `@import 'variables'` | `@use 'abstracts/variables' as v;` | Namespace: `v.$x` en vez de `$x` global |
| Varios imports repetidos | Un `@use` (singleton) | No se duplica; carga una vez |
| Scope global compartido | Scope por módulo | **Cada archivo que use variables debe declarar su `@use`** |
| `@import` anidado (dentro de un selector) heredaba el contexto | `@use` solo a nivel superior | `@import` dentro de selector anidaba selectores con el contexto; `@use` no existe ahí (error) |
| `darken()`, `map-get()` globales | `color.adjust`, `map.get` vía `@use 'sass:color'` | Funciones globales deprecadas |
| Orden = texto copiado | Orden = grafo de módulos (post-order) | Reglas CSS se emiten en el primer punto de carga del módulo |

**Pasos de migración**:

1. Reemplazar cada `@import` por `@use` (alias `as *` para primer paso de compatibilidad, si el archivo es grande).
2. Arriba de cada archivo, agregar los `@use` de los módulos que usa (`abstracts` y sus dependencias).
3. Prefijar accesos con namespace (`variables.$color`) o quitar `as *` gradualmente.
4. Reemplazar funciones globales por módulos (`@use 'sass:color'`, `sass:map`, `sass:math`).
5. Convertir imports de "colecciones" en `_index.scss` + `@forward`; configurar con `with`.
6. Verificar salida: compilar con `--style=expanded` y **diff del CSS** antes/después — debe ser idéntico salvo orden de reglas justificado.

```scss
// ❌ Antes (scope global, orden frágil, sin namespaces)
@import 'variables';
@import 'mixins';

// ✅ Después
@use 'abstracts/variables' as v;
@use 'abstracts/mixins' as m;
```

## Módulos incorporados (`sass:*`) y el sistema de módulos

`@use 'sass:math'`, `sass:color`, `sass:map`, `sass:list`, `sass:string`, `sass:selector`, `sass:meta` (catálogo completo en [[Sintaxis de Sass]]) son módulos como cualquier otro: se cargan con `@use` arriba y se usan con namespace (`math.div`, `color.adjust`). Conviven con tus módulos de proyecto en el mismo archivo sin problema:

```scss
@use 'sass:math';
@use 'sass:map';
@use 'abstracts' with ($spacing-base: 1.25rem);

.ancho-col { width: math.div(100%, 3); }
```

## Errores comunes

1. **`@use` después de reglas** → `This at-rule must be written before any other rules`. Mover todos los `@use` al inicio del archivo.
2. **Undefined variable en el consumidor** → olvidar el namespace (`$color` en vez de `variables.$color`), o esperar scope global que `@import` daba y `@use` no.
3. **`This module was loaded previously` / configuración duplicada** → dos `@use ... with` del mismo módulo con configs distintas; unificar la config en un solo punto (el índice o la entrada).
4. **CSS "perdido"** → el módulo nunca fue `@use`d desde la entrada; los `_index.scss` no emiten nada solos.
5. **Error de extensión de selector con @extend** → `@extend` de un placeholder que vive en otro contexto de media query (ver [[Mixins]]).
6. **Colisiones con `as *`** → dos `$color` definidos; namespaces explícitos.
7. **Carga circular** → `@use` mutuo entre archivos; extraer lo común a un tercer módulo.

## Cheatsheet

| Concepto | Sintaxis |
|---|---|
| Partial | `_nombre.scss` — no compila solo |
| Cargar | `@use 'ruta' as ns;` → `ns.$var`, `ns.@mixin`… |
| Sin namespace | `@use 'ruta' as *` (con cuidado) |
| Configurar | `@use 'ruta' with ($var: valor)` (solo vars `!default`) |
| Re-exportar | `@forward 'ruta'` |
| Exponer selectivo | `@forward 'ruta' show $a, $b` / `hide $c` |
| Prefijar API | `@forward 'ruta' as tema-*` |
| Índice de carpeta | `_index.scss` + `@use 'carpeta'` |
| Singleton | Un módulo = una instancia por grafo de compilación |
| Orden cascada | Orden de los `@use` en `app.scss` |
| Módulos built-in | `@use 'sass:math'` (y color, map, list, string, selector, meta) |

## Relacionadas

- [[Qué es Sass]] — qué es el preprocesador y el roadmap de `@import`
- [[Sintaxis de Sass]] — variables, maps y funciones que viven en estos módulos
- [[Mixins]] — los mixins típicamente viven en `abstracts/_mixins.scss`
- [[Especificidad]] — por qué el orden de carga importa para la cascada final
- [[Compilar Sass]] — compilación del proyecto con Gulp
- [[Node JS]] — el runtime que provee la CLI `sass`
