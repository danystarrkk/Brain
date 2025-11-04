---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[SASS]]"
---
----------
# Qué son los mixins?
Los mixins nos permiten escribir código el cual puede ser reutilizado en nuestra hoja de estilos, de esta forma la cantidad de clases o código repetido puede ser considerablemente menor, por lo que el código html será semánticamente mejor.

## Como creamos mixins?
La sintaxis para la creación de mixins es bastante sencilla:

```scss
@mixin nombre-mixin
```

Para utilizarlos mixins vamos a usarlos de la siguiente manera:

```scss
@include nombre-mixin
```

Los mixins pueden tomar parámetros como las funciones pero no son funciones ya que nosotros en sass también tenemos funciones
Un ejemplo es para la creación de media queries de la siguiente forma:

```scss
@use 'variables' as v;

@mixin mobile {
  @media (min-width: v.$mobile) {
    @content;
  }
}

@mixin tablet {
  @media (min-width: v.$tablet) {
    @content;
  }
}

@mixin desktop {
  @media (min-width: v.$desktop) {
    @content;
  }
}

@mixin desktopXl {
  @media (min-width: v.$desktopXl) {
    @content;
  }
}
```

## Mixins inteligentes
Nosotros podemos crear mixins inteligentes a los cuales podemos pasar parámetros, esto es muy útil cuando trabajamos con display grid, en el cual podemos modificar ciertos aspectos y crear un mixin muy reutilizable:

```scss
// Grid

@mixin grid($columns) {
  display: grid;
  grid-template-columns: repeat($columns, 1fr);
  gap: 5rem;
}
```

También podemos pasarle mas de un parámetro y dejar valores a usarse por default en caso de no pasarle nada de la siguiente manera:

```scss
@mixin grid($columns: 1, $gap: 5rem) {
  display: grid;
  grid-template-columns: repeat($columns, 1fr);
  gap: $gap;
}
```

