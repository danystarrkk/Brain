---
aliases: ["Background position", "Fondos en CSS", "Propiedades de background"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Degradados en CSS]]"
  - "[[Imágenes CSS]]"
  - "[[Formatos Avif y Webp]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/background-position"
---

# Background

Las propiedades de `background` controlan el fondo de un elemento: color, imagen, degradados, repetición, tamaño y posición. Este nota se centra en **`background-position`**, la propiedad que ubica la imagen de fondo dentro del elemento.

## Ubicar la imagen: `background-position`

`background-position` recibe **1, 2 o 4 valores** y define desde qué punto se coloca la imagen de fondo:

```css
background-position: top;                              /* 1 valor: alineación */
background-position: top 50px;                        /* 2 valores: eje X e Y */
background-position: bottom 50px right 100px;         /* 4 valores: edge offset edge offset */
```

| Forma     | Ejemplo                              | Significado                                              |
| --------- | ------------------------------------ | -------------------------------------------------------- |
| 1 valor   | `center`, `top`, `left`              | Posición en ese eje; el otro queda centrado              |
| 2 valores | `top 50px` o `50% 50%`               | Eje Y y eje X (en ese orden para keywords mixtas)        |
| 4 valores | `bottom 50px right 100px`            | Borde vertical + desplazamiento, borde horizontal + desplazamiento |

> [!info] El punto de referencia se mueve hacia el lado contrario
> El valor que indicás es el **punto de anclaje**: `top` dice "pegá la parte superior de la imagen al borde superior del elemento", y los píxeles o porcentajes posteriores empujan la imagen **hacia el lado opuesto**. `right 100px` mueve la imagen 100px hacia la izquierda desde el borde derecho. Cualquier unidad sirve: `px`, `%`, `rem`, `vw`, `vh`, etc.

### Porcentajes: el comportamiento no obvio

Con porcentajes, `background-position: 100% 0%` equivale a `top right`, y el 50% central es `center`. La fórmula real es:

```
posición = (contenedor - imagen) × porcentaje
```

Un 20% alinea el 20% de la imagen con el 20% del contenedor — por eso `50%` centra exactamente.

## El resto de propiedades de background

```css
.hero {
  background-color: #111827;
  background-image: url("hero.avif");
  background-repeat: no-repeat;
  background-size: cover;
  background-position: center 30%;
  background-attachment: fixed;
}
```

| Propiedad               | Para qué sirve                                              |
| ----------------------- | ----------------------------------------------------------- |
| `background-color`      | Color de fondo (por debajo de la imagen)                    |
| `background-image`      | Imagen o degradado (ver [[Degradados en CSS]])              |
| `background-repeat`     | `repeat`, `no-repeat`, `repeat-x`, `repeat-y`               |
| `background-size`       | `cover`, `contain`, o valores con unidad (`100% auto`)      |
| `background-position`   | Ubicación de la imagen                                      |
| `background-attachment` | `scroll` (normal) o `fixed` (efecto parallax clásico)       |
| `background` (shorthand)| Todas juntas en una línea                                   |

```css
/* shorthand equivalente a todas las anteriores */
.hero {
  background: #111827 url("hero.avif") no-repeat fixed center 30% / cover;
}
```

> [!warning] Cuidado con `background` shorthand
> La shorthand reinicia **todas** las sub-propiedades que no se mencionen (por ejemplo, `background-repeat` vuelve a `repeat`). Si vas a usar shorthand, declara todo lo que necesites en esa misma línea.

> [!tip] `background-attachment: fixed` en móvil
> En iOS y muchos móviles `fixed` no funciona (se comporta como `scroll`) porque rompe el rendimiento del scroll. Usá parallax con cuidado o con JavaScript controlado.

## Formatos de imagen recomendados

Para fondos pesados, usá formatos modernos como AVIF o WebP en el `background-image` — ver [[Formatos Avif y Webp]] y [[Imágenes CSS]] para la estrategia completa con imágenes.

## Relacionadas

- [[Degradados en CSS]] — `linear-gradient()` y `radial-gradient()` como fondo
- [[Imágenes CSS]] — etiqueta `img` vs fondo CSS
- [[Formatos Avif y Webp]] — formatos comprimidos para fondos
- [[Animaciones]] — combinar fondos con animación
