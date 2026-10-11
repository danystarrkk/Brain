---
aliases: ["Degradados Css", "Gradientes CSS", "linear-gradient"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Colores CSS]]"
  - "[[Background]]"
  - "[[Animaciones]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/gradient/linear-gradient"
---

# Degradados en CSS

Un **degradado** (gradiente) es una transición suave entre dos o más colores. En CSS no es una imagen: se genera con la función `linear-gradient()` (y variantes) y se usa donde se espera un `background-image`.

## `linear-gradient()`

```css
background-image: linear-gradient();
```

Dentro del gradiente se definen tres cosas:

1. **La dirección**: `to top` (de abajo hacia arriba), `to bottom` (por defecto, de arriba hacia abajo), `to right`, o un ángulo (`45deg`, `135deg`...).
2. **El color inicial**: el que está en el punto de arranque, por ejemplo `#ffffff 0%`.
3. **El color final**: el que está en el punto final, por ejemplo `#212121 100%`.

Un ejemplo completo usando las custom properties de [[Colores CSS]]:

```css
:root {
  --negro: #212121;
  --blanco: #ffffff;
}

.hero {
  background-image: linear-gradient(to top, var(--negro) 0%, var(--blanco) 100%);
}
```

### Paradas de color (stops)

Los porcentajes después de cada color son **paradas**: dicen en qué punto de la transición empieza o termina ese color. Se pueden usar más de dos colores y ajustar el ritmo del degradado:

```css
/* 3 colores: el amarillo ocupa el 10% central */
.bandera {
  background: linear-gradient(to right, #c60b1e 0% 45%, #ffc400 45% 55%, #c60b1e 55% 100%);
}
```

- Colores pegados (`#000 0%, #fff 100%`) = transición repentina en la mitad.
- Separados = transición gradual. Dos colores sin porcentaje reparten el espacio de forma uniforme.

## Otros tipos de gradiente

| Función                          | Forma                                    |
| -------------------------------- | ---------------------------------------- |
| `linear-gradient()`              | Transición en línea recta                |
| `radial-gradient()`              | Desde un punto hacia afuera (círculo/elipse) |
| `conic-gradient()`               | Rotación alrededor de un punto (reloj)   |

```css
.bola {
  background: radial-gradient(circle at 30% 30%, #fff, #3498db 70%);
}

.reloj {
  background: conic-gradient(from 0deg, #e74c3c, #f1c40f, #2ecc71, #e74c3c);
}
```

## Múltiples capas

`background-image` acepta **varias** funciones separadas por comas; la primera capa queda encima de las demás:

```css
.overlay {
  background-image:
    linear-gradient(to top, rgb(0 0 0 / 0.7), transparent),
    url("hero.avif");
  background-size: cover;
  background-position: center;
}
```

Este es el patrón clásico del "overlay" de texto sobre foto — ver [[Imágenes CSS]] y [[Formatos Avif y Webp]].

> [!tip] Degradados animables
> Los valores de gradiente interpolables pueden animarse con `@property` registrando el componente como `<color>` — ver [[Animaciones]]. Alternativa simple y universal: animar `background-position` de un gradiente grande.

> [!warning] Compatibilidad y rendimiento
> Los tres tipos de gradiente son compatibles con todos los navegadores modernos. Evitá degenerados con decenas de paradas en elementos grandes: son costosos de repintar. En textos, verificá el contraste con el color de fondo resultante.

## Relacionadas

- [[Colores CSS]] — formatos de color usados en los degradados
- [[Background]] — cómo encaja `background-image` con el resto de propiedades de fondo
- [[Sombras CSS]] — otra forma de dar profundidad a los elementos
- [[Animaciones]] — dar vida a los degradados
