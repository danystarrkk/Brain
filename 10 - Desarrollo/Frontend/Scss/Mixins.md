---
aliases: ["SCSS Mixins", "@mixin"]
tags:
  - dev/frontend/scss
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Sintaxis de Sass]]"
  - "[[Separar Sass en Archivos]]"
  - "[[Responsive Web Design]]"
fuente: "https://sass-lang.com/documentation/at-rules/mixin/"
---

# Mixins

Los **mixins** permiten definir estilos **reutilizables** que se insertan donde se necesiten con `@include`. Reducen la repetición en el CSS y, como beneficio colateral, dejan el HTML más semántico (menos clases utilitarias de apoyo).

> [!info] Mixin ≠ Función
> Un **mixin** emite **declaraciones CSS** (`display: grid`, media queries…). Una **función** devuelve un **valor** (`2rem`, `#3498db`) y no genera CSS por sí misma. Ver [[Sintaxis de Sass#Funciones]] para funciones.

## Definición e inclusión

```scss
// Definición
@mixin titulo-seccion {
  font-size: 2rem;
  font-weight: bold;
  margin-bottom: 1rem;
}

// Uso
.seccion h2 {
  @include titulo-seccion;
}
```

## Mixins con parámetros

```scss
@mixin grid($columnas, $gap: 5rem) {   // $gap tiene valor por defecto
  display: grid;
  grid-template-columns: repeat($columnas, 1fr);
  gap: $gap;
}

.layout-3 {
  @include grid(3);            // gap: 5rem (default)
}

.layout-2 {
  @include grid(2, 2rem);      // gap: 2rem
}
```

Los parámetros con valor por defecto hacen al mixin **progresivo**: funciona con 1 argumento y escala cuando necesitás control extra. Ver [[CSS Grid]] para el contexto de las propiedades usadas.

## `@content`: bloques de contenido

`@content` permite pasarle **CSS al mixin desde el `@include`** — es lo que hace posible encapsular media queries reutilizables:

```scss
@use 'variables' as v;   // v.$mobile, v.$tablet, v.$desktop

@mixin mobile {
  @media (min-width: v.$mobile) { @content; }
}

@mixin tablet {
  @media (min-width: v.$tablet) { @content; }
}

@mixin desktop {
  @media (min-width: v.$desktop) { @content; }
}

// Uso: el bloque {} se inyecta dentro del @media
.card {
  width: 100%;

  @include tablet {
    width: 50%;
  }

  @include desktop {
    width: 25%;
  }
}
```

> [!tip] Patrón responsivo mobile-first
> Con `min-width`, los estilos base aplican siempre y **suben** de tamaño. Es el enfoque recomendado de [[Responsive Web Design]] — el orden de los mixins en el CSS importa: de menor a mayor breakpoint.

### `@content` con argumentos

Desde Dart Sass 1.38.0, `@content` puede recibir valores que se usan dentro del bloque como si fueran variables locales:

```scss
@mixin hover-en-foco {
  @content(if($estado, $estado, hover));
}

// Uso avanzado: el bloque recibe el valor pasado
```

En la práctica el 95% de los casos no lo necesita; `@content` simple cubre casi todo.

## Patrones útiles

```scss
// Centrado absoluto
@mixin centrar-absoluto {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

// Solo lectores de pantalla (accesibilidad)
@mixin sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

// Truncar texto con puntos suspensivos
@mixin truncar($lineas: 1) {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: $lineas;
  overflow: hidden;
}
```

## Mixin vs placeholder (`@extend`)

| | Mixin `@include` | Placeholder `%` + `@extend` |
|---|---|---|
| Duplica declaraciones en el CSS | Sí (una por cada `@include`) | No — agrupa selectores |
| Acepta parámetros | ✅ | ❌ |
| Funciona dentro de `@media` | ✅ | ⚠️ problemático |
| Ideal para | Estilos parametrizados, media queries | Clases base compartidas sin parámetros |

```scss
%boton-base {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.btn-primario { @extend %boton-base; background: #3498db; }
.btn-peligro  { @extend %boton-base; background: #e74c3c; }
```

## Buenas prácticas

- **Una responsabilidad por mixin**: si hace 8 cosas no relacionadas, dividió.
- **Convención de nombres**: kebab-case (`card-hover`, `flex-centrado`).
- **No generes mixins que solo envuelven una propiedad** (`@mixin color($c) { color: $c; }` — eso es ruido; usá una variable).
- **Guardá los de media queries en un partial propio** (`_breakpoints.scss`) con las variables — ver [[Separar Sass en Archivos]].
- El abuso de mixins compila a CSS repetido: si el resultado final se dispara, revisá si un placeholder o una custom property servirían.

## Relacionadas

- [[Sintaxis de Sass]] — funciones, placeholders y el lenguaje base
- [[Separar Sass en Archivos]] — dónde viven los mixins en un proyecto real
- [[Responsive Web Design]] — patrones mobile-first con `@content`
- [[CSS Grid]] — los mixins de grid construidos sobre estas bases
