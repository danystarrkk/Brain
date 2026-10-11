---
aliases: ["@keyframes", "Animaciones CSS", "Crea animaciones"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Efectos de Scroll]]"
  - "[[Degradados en CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/@keyframes"
---

# Animaciones

CSS permite animar propiedades sin JavaScript. Una animación tiene dos piezas: los **`@keyframes`** (qué cambios ocurren en cada punto del tiempo) y la **propiedad `animation`** (cómo y cuándo se reproduce en el elemento).

## Los 3 pasos de una animación

### 1. Definir la animación en el elemento

```css
.animatedo {
  animation: animate 15s linear infinite;
}
```

La shorthand `animation` acepta hasta 6 valores (los tres primeros en orden fijo):

| Propiedad              | Valor del ejemplo | Qué controla                                     |
| ---------------------- | ----------------- | ------------------------------------------------ |
| `animation-name`       | `animate`         | Nombre de los `@keyframes` a usar                |
| `animation-duration`   | `15s`             | Cuánto dura una vuelta completa                  |
| `animation-timing-function` | `linear`     | La curva de velocidad (lineal, acelerada, etc.)  |
| `animation-iteration-count` | `infinite`  | Repeticiones: un número o para siempre           |
| `animation-delay`      | _(omitido)_       | Espera antes de arrancar                         |
| `animation-direction`  | _(normal)_        | `normal`, `reverse`, `alternate`                 |

```css
/* Forma expandida: más legible cuando se mezclan propiedades */
.expandida {
  animation-name: animate;
  animation-duration: 15s;
  animation-timing-function: linear;
  animation-iteration-count: infinite;
}
```

### 2. Definir los fotogramas con `@keyframes`

Los `@keyframes` describen en qué momento (en `%` de la duración total) ocurre cada estado. El navegador "interpola" los valores intermedios:

```css
@keyframes animate {
  0% {
    /* estado inicial: abajo de la pantalla y sin escala */
    transform: translateY(100vh) scale(0);
  }

  100% {
    /* estado final: un poco sobre el borde superior y al doble de tamaño */
    transform: translateY(-10vh) scale(2);
  }
}
```

Se pueden usar más de dos puntos para curvas más ricas:

```css
@keyframes latido {
  0%, 100% { transform: scale(1); }   /* dos puntos en el mismo estado */
  50%      { transform: scale(1.15); }
}
```

### 3. Aplicarla al elemento

La regla `animation` del paso 1 activa los keyframes. Sin ella, los `@keyframes` existen pero nada se anima.

> [!tip] `from`/`to` como alternativa a `0%`/`100%`
> `@keyframes` admite `from { ... } to { ... }`, equivalente a `0%`/`100%`. Los `%` permiten pasos intermedios, así que son más versátiles.

## Animar con variables CSS

La duración puede calcularse con custom properties, útil cuando cada elemento tiene un retardo distinto desde un mismo CSS:

```css
.pregunta {
  --i: 1; /* cada elemento puede redefinirla: style="--i: 2" */
  animation: animate 15s linear infinite;
  animation-duration: calc(75s / var(--i));
}
```

> [!warning] No todas las propiedades son animables
> Solo se animan propiedades **numéricas o interpolables** (`transform`, `opacity`, `color`, `margin`...). Propiedades discretas como `display` o `background-image` no interpolan (aunque `display` ya admite animarse de forma limitada con `@starting-style` y `transition-behavior` en navegadores modernos). Para mostrar/ocultar, la alternativa clásica es animar `opacity` + `transform`.

## Transiciones vs animaciones

| Aspecto             | `transition`                       | `@keyframes`                       |
| ------------------- | ---------------------------------- | ---------------------------------- |
| Disparo             | Requiere un cambio de estado (hover, clase, etc.) | Se reproducen solas al activarse |
| Complejidad         | De un estado a otro                | Infinitos pasos intermedios        |
| Repetición          | No (salvo que el estado oscile)    | `infinite`, `alternate`, etc.      |

```css
.boton {
  transition: background-color 0.3s ease;
}
.boton:hover {
  background-color: #166534; /* la transición interpola el cambio */
}
```

## Rendimiento

> [!info] Animá `transform` y `opacity`
> `transform` y `opacity` se componen en la GPU sin repintar el layout, lo que da 60 fps constantes. Animar `top`, `left`, `width` o `margin` fuerza recálculos de layout en cada frame y produce tirones en móviles.

> [!warning] Respeta `prefers-reduced-motion`
> Las animaciones intensas pueden marear a usuarios con sensibilidad al movimiento. Detectalo con una media query y desactivá o reducí las animaciones:

```css
@media (prefers-reduced-motion: reduce) {
  .animatedo {
    animation: none;
  }
}
```

## Relacionadas

- [[Efectos de Scroll]] — efectos ligados al desplazamiento (scroll snap)
- [[Degradados en CSS]] — fondos que combinan bien con animaciones
- [[Colores CSS]] — valores de color para animar `background`
- [[Tipos de Posiciones]] — referencia útil al animar con `position`
