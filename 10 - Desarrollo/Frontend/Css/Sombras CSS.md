---
aliases: ["Box shadow", "Sombras en CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Colores CSS]]"
  - "[[Degradados en CSS]]"
  - "[[Box Model]]"
  - "[[Tipos de Posiciones]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/box-shadow"
---

# Sombras CSS

Las sombras dan **profundidad** a la interfaz: indican qué está elevado, qué es interactivo y qué está detrás. En CSS se aplican con `box-shadow` (a cajas) y `text-shadow` (a texto).

## `box-shadow`: anatomía de la sombra

Podemos aplicar sombras a cualquier caja — imágenes, botones, tarjetas:

```css
.button {
  box-shadow: 0.2rem 0.4rem 1rem rgba(0, 0, 0, 0.1);
  /*            X     Y     difuminado  color */
}
```

Los valores, en orden:

| Valor           | Significado                                              | Ejemplo            |
| --------------- | -------------------------------------------------------- | ------------------ |
| **X**           | Desplazamiento horizontal (negativo = a la izquierda)     | `0`                |
| **Y**           | Desplazamiento vertical (negativo = arriba)               | `0.4rem`           |
| **Difuminado**  | Radio de desenfoque (blur) — solo se puede agrandar      | `1rem`             |
| **Spread** (opcional) | Cuánto crece o encoge la sombra antes de difuminar | `0`                |
| **Color**       | Casi siempre con transparencia (`rgba` o `color-mix`)    | `rgba(0,0,0,.1)`   |
| **`inset`** (prefijo opcional) | Sombra **hacia adentro** (efecto hundido) | `inset 0 2px 4px …` |

> [!tip] Sombras sutiles, no llamativas
> La sombra moderna copia a la luz real: **poco desplazamiento, mucho desenfoque y mucha transparencia**. Ejemplo de escala de elevación tipo Material:
>
> ```css
> .elevacion-1 { box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06); }
> .elevacion-2 { box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08); }
> .elevacion-3 { box-shadow: 0 12px 32px rgba(0, 0, 0, 0.12); }
> ```

## Sombras múltiples

`box-shadow` acepta **varias sombras separadas por coma**, la primera pinta encima de las demás:

```css
.card {
  box-shadow:
    0 1px 2px rgba(0, 0, 0, 0.08),
    0 8px 24px rgba(0, 0, 0, 0.12);
}
```

Combinar una sombra dura y cercana con otra suave y amplia da un resultado mucho más natural que una sola.

## `text-shadow`

Misma idea, sin spread ni `inset`:

```css
h1 {
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.4);
}
```

## `box-shadow` vs `filter: drop-shadow`

| Propiedad               | Se dibuja sobre...                     | Característica                                   |
| ----------------------- | -------------------------------------- | ------------------------------------------------ |
| `box-shadow`            | El **rectángulo** de la caja           | Barato; ignora la forma real (p. ej. bordes redondeados con transparencia) |
| `filter: drop-shadow()` | La **silueta visible** del elemento    | Sigue la forma (PNG transparentes, `border-radius`); más costoso; sin spread |

```css
.foto-recortada {
  filter: drop-shadow(0 8px 16px rgba(0, 0, 0, 0.25));
}
```

> [!info] Las sombras no afectan el layout
> `box-shadow` **no ocupa espacio**: se dibuja fuera del flujo y puede sobrepasar los bordes vecinos. Para separar elementos seguí usando `gap`/márgenes — ver [[Box Model]].

## Generadores

Para experimentar con los valores existe [cssmatic](https://www.cssmatic.com/es/box-shadow) (también tiene generador de degradados — ver [[Degradados en CSS]]): solo copiá el código que genera y pegalo donde lo necesites. Útil para calibrar a ojo; después conviene normalizar los valores a una escala de sombras reutilizable.

## Relacionadas

- [[Colores CSS]] — `rgba`, `hsl` y `color-mix()` para elegir el color de la sombra
- [[Degradados en CSS]] — el otro generador de cssmatic
- [[Box Model]] — por qué la sombra no suma a las medidas del elemento
- [[Tipos de Posiciones]] — capas y `z-index`, donde las sombras se nota el orden
