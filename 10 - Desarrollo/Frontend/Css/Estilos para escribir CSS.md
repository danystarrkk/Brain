---
aliases: ["Estilos para Escribir Código Css", "Metodologías CSS", "Estilos CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Especificidad]]"
  - "[[TailwindCSS - Sintaxis Básica]]"
  - "[[Sintaxis de Sass]]"
fuente: "https://getbem.com/"
---

# Estilos para escribir CSS

A medida que crece un proyecto, el CSS se vuelve difícil de mantener: selectores que se pisan, nombres repetidos y specificity descontrolada. Existen **metodologías** para escribir estilos ordenados y reutilizables. Las tres grandes:

- **BEM** — convención de nombres por bloques.
- **Utility First** — clases utilitarias atómicas.
- **Módulos (CSS Modules / scopes)** — aislar estilos por componente.

Si el proyecto es grande, es habitual **combinar las tres**: utilidades para ajustes, módulos para aislamiento y convenciones BEM dentro de los componentes.

## BEM: Bloques, Elementos, Modificadores

BEM es una metodología para escribir CSS reutilizable y ordenado. Su aporte central son las **convenciones de nombres**, que evitan las colisiones entre estilos de partes distintas de la app y mantienen la especificidad plana (todas las clases = 0–1–0, ver [[Especificidad]]).

### Bloques

Un bloque es un contenedor con significado propio: si una sección **se entiende por sí sola** y no depende de otras para su apariencia, es un bloque.

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
> BEM prohíbe `.card__titulo__icono`: un elemento pertenece **directo** a su bloque. Si necesitás anidar, creá un bloque nuevo (`.icono`) reutilizable dentro de varios elementos.

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
```

En HTML se combinan las clases:

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

## Utility First

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

- **Contra**: mucho CSS generado (aunque con purging de no usados, como hace Tailwind, el CSS final es mínimo) y ruido en el HTML.
- **A favor**: diseño 100% consistente (espaciados y colores de un sistema) sin specificity wars — todas las clases empatan y el orden del CSS decide limpiamente.

El exponente moderno de esta filosofía es [[TailwindCSS - Sintaxis Básica]].

## Módulos

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

En frameworks modernos este concepto evolucionó a los **CSS Modules** (`styles.module.css` importado en el componente, con nombres de clase hasheados por build) y a CSS scoping con Shadow DOM: el objetivo es el mismo — que los estilos de una parte **no fugen** a otra.

> [!tip] Qué elegir
> - Proyectos pequeños/médicos: BEM con especificidad plana.
> - Prototipos y equipos que ya piensan en utilidades: Tailwind.
> - SPAs con componentes (React, Vue): CSS Modules o styled-components, con utilidades puntuales.
> Lo que **no** funciona en ninguno: selectores largos anidados con `!important` de por medio — ver [[Especificidad]].

## Relacionadas

- [[Especificidad]] — el problema que estas metodologías buscan controlar
- [[TailwindCSS - Sintaxis Básica]] — utility-first en la práctica
- [[Sintaxis de Sass]] — anidación y organización de archivos
- [[Selectores en CSS]] — herramientas de selección disponibles
