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
  - "[[Compilar Sass]]"
  - "[[Node JS]]"
fuente: "https://sass-lang.com/documentation/at-rules/use/"
---

# Separar Sass en Archivos

Un proyecto Sass real se divide en **módulos**: archivos pequeños con una responsabilidad, ensamblados por el sistema de módulos `@use`/`@forward`.

![[Pasted image 20250125074749.png]]

## Partials: archivos con `_`

Los archivos parciales llevan **guion bajo al inicio** (`_variables.scss`, `_mixins.scss`). Esa convención le dice al compilador: **no los compiles como CSS independiente** — solo existen cuando otro archivo los importa.

```
src/scss/
├── abstracts/
│   ├── _variables.scss
│   ├── _mixins.scss
│   └── _index.scss
├── base/
│   ├── _reset.scss
│   ├── _tipografia.scss
│   └── _index.scss
├── components/
│   ├── _botones.scss
│   └── _cards.scss
└── app.scss          ← único archivo sin "_": el punto de entrada
```

## `@use`: cargar un módulo

```scss
// app.scss
@use 'abstracts/variables' as v;
@use 'abstracts/mixins' as m;

body {
  color: v.$color-texto;          // namespace + $ = variable del módulo
  font-family: v.$fuente-principal;
}
```

`as v` crea un **namespace**: el código del módulo no contamina el scope global (a diferencia del viejo `@import`). También se puede hacer `@use '...' as *` para traer todo sin namespace — cómodo en tests, peligroso en proyectos grandes.

Reglas importantes:

- `@use` va **siempre arriba** del archivo (antes que cualquier regla de estilo).
- Un módulo se carga **una sola vez** por archivo; múltiples `@use` del mismo módulo se resuelven a la misma instancia.

## `_index.scss` y `@forward`: carpetas como módulos

Cuando una carpeta crece, el archivo **`_index.scss`** la expone como un único módulo re-exportando sus archivos con `@forward`:

```scss
// abstracts/_index.scss
@forward 'variables';
@forward 'mixins';
@forward 'functions';
```

Y en el punto de entrada se carga **solo la carpeta** — Sass sabe que debe buscar `_index.scss` dentro:

```scss
// app.scss
@use 'abstracts';      // trae variables, mixins y functions

body {
  color: variables.$color-texto;   // namespace = nombre del archivo
}
```

> [!warning] El índice no se compila solo
> Un `_index.scss` jamás compila por sí mismo: **siempre debe consumirse con `@use` desde el archivo principal** (`app.scss`) o desde otro módulo. Si no aparecen tus estilos, casi siempre es porque falta ese `@use` en la entrada.

### `@forward` con control de exposición

```scss
@forward 'variables' show $color-texto, $fuente;   // solo esto es público
@forward 'mixins' hide $util-interna;              // todo menos esto
@forward 'variables' as tema-*;                    // renombra: $tema-color-texto
```

## Migración desde `@import` (deprecado)

Desde **Dart Sass 1.80.0**, `@import` está deprecado y desaparecerá en Sass 3.0. Si heredás un proyecto con `@import`, la traducción es directa:

```scss
// ❌ Antes (scope global, orden frágil, sin namespaces)
@import 'variables';
@import 'mixins';

// ✅ Después
@use 'abstracts/variables' as v;
@use 'abstracts/mixins' as m;
```

Diferencias clave al migrar:

1. Cada archivo que use variables/mixins debe declarar su **propio `@use`** (ya no hay scope global).
2. Los selectores anidados dentro de un módulo **no se anidan más** con el consumidor (el `@import` anidaba; `@use` no).
3. Las funciones globales deprecadas (`darken()`, `map-get()`) pasan a sus módulos (`@use 'sass:color'` → `color.adjust()`).

## Estructura recomendada

| Carpeta | Contenido |
|---|---|
| `abstracts/` | Variables, mixins, funciones — **cero CSS emitido** |
| `base/` | Reset/normalize, tipografía global, elementos elementales |
| `components/` | Botones, cards, navbar — unidades reutilizables |
| `layout/` | Grid del sitio, header, footer, sidebar |
| `pages/` | Estilos exclusivos de páginas concretas (si los hay) |
| `app.scss` | Punto de entrada: `@use` de todo en orden de cascada |

El orden de los `@use` en la entrada **es** el orden de la cascada: base antes que components, components antes que pages.

## Relacionadas

- [[Qué es Sass]] — qué es el preprocesador y cómo se compila
- [[Sintaxis de Sass]] — variables, mapas y funciones que viven en estos módulos
- [[Mixins]] — los mixins típicamente viven en `abstracts/_mixins.scss`
- [[Compilar Sass]] — compilación del proyecto con Gulp
- [[Node JS]] — el runtime que provee la CLI `sass`
