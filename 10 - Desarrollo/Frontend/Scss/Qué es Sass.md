---
aliases: ["SASS", "SCSS", "Syntactically Awesome Style Sheets"]
tags:
  - dev/frontend/scss
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Sintaxis de Sass]]"
  - "[[Mixins]]"
  - "[[Separar Sass en Archivos]]"
  - "[[Compilar Sass]]"
fuente: "https://sass-lang.com/documentation/"
---

# ¿Qué es Sass?

**SASS** son las siglas de ***S**yntatically **A**wesome **S**tyle**s**heets*: es un **preprocesador de CSS** — un programa que toma archivos con sintaxis extendida (variables, anidación, mixins) y los **compila** a CSS estándar que el navegador entiende.

> [!info] Preprocesador = CSS con superpoderes compilados
> Los navegadores no entienden Sass. Un compilador traduce `.scss` → `.css` antes de que el navegador vea nada. Por eso Sass no es "mejor CSS", es una **capa de productividad** sobre CSS que produce CSS puro.

## Historia y estado actual

- Creado por **Hampton Catlin** (2006), originalmente en Ruby.
- Hoy el implementación oficial y activa es **Dart Sass** (escrita en Dart) — es la que usa la CLI oficial `sass`.
- Los archivos tienen dos sintaxis: **`.scss`** (con llaves, compatible con CSS) y **`.sass`** (indentación, sin llaves — la sintaxis original, hoy en desuso).

## Ventajas

- **Orden y estructura**: proyectos de CSS grandes se mantienen modularizados (ver [[Separar Sass en Archivos]]).
- **Variables**: colores, espaciados y tipografías centralizados — cambiar un token actualiza todo el proyecto.
- **Anidación**: la estructura del CSS refleja la del HTML.
- **Mixins y funciones**: código reutilizable con parámetros (ver [[Mixins]]).
- **Módulos**: `@use`/`@forward` permiten dividir el proyecto en archivos con namespaces explícitos.
- **Compatible**: todo CSS válido es SCSS válido — la migración es incremental.

## Desventajas

- **Otra sintaxis** que aprender.
- **No es nativo**: requiere compilación (CLI o integración con el build — ver [[Compilar Sass]]).
- **La anidación abusa fácil**:CSS excesivamente anidado genera selectores frágiles y específicos (ver [[Especificidad]]) y más código del necesario.

## Compilación

Dart Sass se instala vía npm e invoca directo:

```bash
npm install -g sass          # instalar una vez
sass src/app.scss dist/app.css   # compilar una vez
sass --watch src:dist            # modo desarrollo (watch)
```

En proyectos Node/Vite suele usarse la integración del build en lugar de la CLI global — ver [[Compilar Sass]].

> [!warning] @import está deprecado
> Desde **Dart Sass 1.80.0** (octubre 2024), `@import` está oficialmente deprecado y desaparecerá en Sass 3.0. El sistema de módulos **`@use` y `@forward`** es la forma correcta desde 2019 — ver [[Separar Sass en Archivos]]. También están deprecadas las funciones globales (`darken()`, `lighten()`, `divide()`): usá los módulos oficiales `@sass:math`, `@sass:color`, etc.

## Sass vs CSS moderno vs Tailwind

| Necesidad | CSS moderno | Sass | Tailwind |
|---|---|---|---|
| Variables | `--color: red` (custom properties) | `$color: red` (en compile-time) | `@theme` / tokens |
| Anidación | Nativa (2023+) | Nativa | Clases utilitarias |
| Lógica (loops, condicionales) | `@container style()` (limitado) | `@if`, `@each`, `@for` | No aplica |
| Reutilización de estilos | Custom properties + clases | Mixins, placeholders | Utilidades + `@apply` |

Sass y Tailwind no se excluyen: Tailwind v4 se integra con pipelines Sass/PostCSS y muchos proyectos usan Sass para lo que CSS todavía no resuelve (matemática de unidades, lógica de diseño). Ver [[TailwindCSS - Sintaxis Básica]].

## Relacionadas

- [[Sintaxis de Sass]] — variables, anidación y el lenguaje completo
- [[Mixins]] — reutilización con parámetros y `@content`
- [[Separar Sass en Archivos]] — módulos `@use`/`@forward` y partials
- [[Compilar Sass]] — compilación con Gulp
- [[CSS Introducción]] — la base sobre la que Sass extiende
- [[Especificidad]] — por qué la anidación profunda es peligrosa
