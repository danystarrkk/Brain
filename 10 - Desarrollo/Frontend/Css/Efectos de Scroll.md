---
aliases: ["Efecto de Scroll", "Scroll snap", "scroll-snap"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Animaciones]]"
  - "[[Responsive Web Design]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/CSS_scroll_snap_module"
---

# Efectos de Scroll

El **scroll snap** (encaje del desplazamiento) permite que al bajar o deslizar, el navegador "encaje" automáticamente en puntos definidos — como las pantallas de una app móvil. Es CSS puro, sin JavaScript.

## Cómo funciona

Se definen **dos** piezas:

1. En el **contenedor** (el que scrollea): qué eje snap y con qué fuerza.
2. En cada **sección** (hijos): dónde se engancha.

### 1. El contenedor

```css
html {
  scroll-snap-type: y mandatory;
}
```

- `y` es el eje del snap (`x`, `y`, `proximity`...).
- `mandatory` **obliga** a que el scroll termine siempre en un punto de snap (ideal para landing pages tipo presentación).
- La alternativa es `proximity`: solo encaja si el punto está cerca — comportamiento más suave y menos "forzado".

> [!warning] `mandatory` en contenedores impredecibles
> Con contenido de altura variable (FAQs, listas), `mandatory` puede atrapar al usuario en una sección cuyo contenido no cabe en pantalla. Para scrolls de lectura, preferí `proximity`.

### 2. Cada sección

```css
.servicios,
.navegacion-principal {
  scroll-snap-align: center;
  scroll-snap-stop: always;
}
```

- `scroll-snap-align` dice **en qué parte** de la sección se encaja: `start` (borde superior), `center` (centro) o `end` (borde inferior).
- `scroll-snap-stop: always` impide que un gesto rápido **salte dos secciones** de una: el scroll se detiene en cada una.

```css
/* Variantes comunes: encajar al inicio de cada sección */
.seccion {
  scroll-snap-align: start;
}
```

## Scroll snap suave al navegar

Combinado con `scroll-behavior`, los enlaces internos también encajan animadamente:

```css
html {
  scroll-behavior: smooth;
}

nav a {
  scroll-snap-align: start;
}
```

## Alternativa: scroll con snap en un carrusel horizontal

El mismo mecanismo sirve para sliders horizontales sin JS:

```css
.carrusel {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  gap: 1rem;
}

.carrusel .card {
  flex: 0 0 80%;      /* tarjeta fija, visiblemente sobresale la siguiente */
  scroll-snap-align: center;
}
```

## Consideraciones de accesibilidad y UX

- **No elimines el scroll libre** en contenido informativo: `mandatory` en páginas normales frustra a usuarios con teclado o lector de pantalla.
- Respetá `prefers-reduced-motion`: para algunos usuarios el snap es desorientador — ver [[Animaciones]] para el patrón de media query.
- Probá en móvil: el snap se siente natural con touch, pero puede ser brusco con rueda de ratón.

## Relacionadas

- [[Animaciones]] — efectos de movimiento complementarios
- [[Responsive Web Design]] — secciones full-screen adaptables
- [[Tipos de Posiciones]] — contexto de desplazamiento de los elementos
