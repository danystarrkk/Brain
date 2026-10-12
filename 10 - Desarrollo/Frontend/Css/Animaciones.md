---
tags:
  - desarrollo
  - frontend
  - animaciones
tipo: concepto
estado: terminado
---

# Animaciones

CSS permite animar propiedades sin JavaScript. Una animación tiene dos piezas: los **`@keyframes`** (qué cambios ocurren en cada punto del tiempo) y la **propiedad `animation`** (cómo y cuándo se reproduce en el elemento). Existe además un segundo mecanismo más ligero —**transiciones**— para cambios de estado disparados por el usuario. Dominar ambos es dominar el tiempo en CSS: duración, curva de velocidad, repetición, retardo y estado final.

> [!info] Qué significa "animar" en el navegador
> Animar es **interpolar**: el navegador calcula los valores intermedios entre dos estados y repinta. Si la propiedad es de composición (`transform`, `opacity`) el trabajo ocurre en la GPU sin recalcular layout (60+ fps). Si es de layout (`width`, `top`, `margin`) cada frame fuerza un reflow completo y el resultado es un tironeo visible — ver "Rendimiento" más abajo.

## Los 3 pasos de una animación

### 1. Definir la animación en el elemento

```css
.animatedo {
  animation: animate 15s linear infinite;
}
```

La shorthand `animation` acepta hasta **8 sub-propiedades** (los tres primeros en orden fijo; después los tiempos pueden ir en cualquier orden):

| Propiedad | Valor del ejemplo | Qué controla | Valores posibles |
|---|---|---|---|
| `animation-name` | `animate` | Nombre de los `@keyframes` a usar | identificador o `none` |
| `animation-duration` | `15s` | Cuánto dura una vuelta completa | tiempo (`ms`/`s`) |
| `animation-timing-function` | `linear` | La curva de velocidad | ver tabla de curvas |
| `animation-iteration-count` | `infinite` | Repeticiones | número o `infinite` |
| `animation-delay` | _(omitido)_ | Espera antes de arrancar | tiempo (negativo = arranca en esa fracción) |
| `animation-direction` | _(normal)_ | Sentido de la reproducción | `normal`, `reverse`, `alternate`, `alternate-reverse` |
| `animation-fill-mode` | _(none)_ | Qué pasa antes/después de correr | `none`, `forwards`, `backwards`, `both` |
| `animation-play-state` | _(running)_ | Corre o pausa | `running`, `paused` |

```css
/* Forma expandida: más legible cuando se mezclan propiedades */
.expandida {
  animation-name: animate;
  animation-duration: 15s;
  animation-timing-function: linear;
  animation-iteration-count: infinite;
}
```

> [!warning] Sin `animation-duration` no hay animación
> La duración por defecto es **`0s`**: los keyframes se "ejecutan" instantáneamente y no ves nada. Es el error #1 de los principiantes — el síntoma es exactamente ese: "los keyframes no funcionan". Siempre escribí duración (y opcionalmente delay).

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

Reglas del formato:

- Los `%` van de `0%` a `100%`; `0%` y `100%` son obligatorios **explícitos o implícitos** (si el `0%` falta, el estado actual del elemento hace de inicial).
- `from`/`to` son sinónimos exactos de `0%`/`100%`.
- Varios `@keyframes` pueden compartir nombre separados por coma — útiles para *overrides* por media query.
- Si dos fotogramas se solapan en el tiempo, **gana el último declarado**.
- Las propiedades deben ser **animables** y tener el **mismo tipo de valor** en todos los puntos (no podés saltar de `10px` a `red`).
- Un keyframe puede declarar **sub-conjuntos** de propiedades: las que falten interpolan entre sus estados presentes/ausentes según `fill-mode`.

> [!tip] `from`/`to` como alternativa a `0%`/`100%`
> `@keyframes` admite `from { ... } to { ... }`, equivalente a `0%`/`100%`. Los `%` permiten pasos intermedios, así que son más versátiles.

### 3. Aplicarla al elemento

La regla `animation` del paso 1 activa los keyframes. Sin ella, los `@keyframes` existen pero nada se anima — y no hay forma de "activarlas globalmente"; la animación vive donde se declara la propiedad.

## Curvas de velocidad: `animation-timing-function`

La curva decide **cómo se reparte el tiempo** entre fotogramas — cambia la *sensación* sin cambiar ni un `%` de keyframes:

| Keyword | Curva | Sensación | Uso típico |
|---|---|---|---|
| `linear` | recta | constante, robótica | progreso, marquesinas, rotaciones |
| `ease` (defecto) | suave, más rápido al inicio | natural | la opción "segura" |
| `ease-in` | arranca lento, frena seco | alejarse, entradas | salidas de escena |
| `ease-out` | arranca rápido, frena suave | llegar, aparecer | entradas, hovers |
| `ease-in-out` | simétrica | movimiento contenido | desplazamientos en pantalla |
| `cubic-bezier(x1,y1,x2,y2)` | curva manual | control total | branding, easing de librería |
| `steps(n, pos)` | escalonada | efecto sprite/flip-book | contadores, pixel art |
| `linear(to right, …)` | interpola entre stops arbitrarios | rutas complejas | soporte aún limitado |

```css
/* cubic-bezier con salida elástica controlada */
.suave { animation-timing-function: cubic-bezier(0.22, 1, 0.36, 1); }

/* steps: 4 fotogramas como flip-book */
.pixel { animation-timing-function: steps(4, end); }
```

`steps(n, position)` acepta las posiciones `jump-start`, `jump-end` (defecto), `jump-none`, `jump-both`: definen si el primer/último fotograma incluye los extremos 0 y 1. Clásico patrón contador con 9 pasos: `steps(9)`.

## `animation-fill-mode`: el estado fuera de la animación

Por defecto, **antes del arranque** el elemento muestra su estado normal y **después de terminar** vuelve a él (la animación no "deja huella"). `fill-mode` cambia eso:

| Valor | Antes del arranque (durante el delay) | Después de terminar |
|---|---|---|
| `none` (defecto) | estado normal | vuelve al estado normal |
| `forwards` | estado normal | **queda en el último fotograma** |
| `backwards` | **muestra el primer fotograma** | estado normal |
| `both` | primer fotograma | último fotograma |

```css
.entrada {
  animation: subir 0.4s ease-out both;   /* espera invisible arriba, se queda arriba */
}
```

`both` es la opción por defecto recomendada para animaciones de entrada: sin `backwards`, el elemento aparece en su sitio final *durante el delay* (un parpadeo feo si el delay es visible); sin `forwards`, vuelve a saltar a su estado inicial al terminar.

## Animar con variables CSS

La duración puede calcularse con custom properties, útil cuando cada elemento tiene un retardo distinto desde un mismo CSS:

```css
.pregunta {
  --i: 1; /* cada elemento puede redefinirla: style="--i: 2" */
  animation: animate 15s linear infinite;
  animation-duration: calc(75s / var(--i));
}
```

Patrón canónico de escalonado (*stagger*) — un retardo distinto por hijo sin escribir una regla por hijo:

```css
.stagger > * {
  animation: aparecer 0.5s ease-out both;
  animation-delay: calc(var(--i, 0) * 80ms);
}
```
```html
<li style="--i: 0">uno</li>
<li style="--i: 1">dos</li>
<li style="--i: 2">tres</li>
```

> [!info] Por qué los keyframes no interpolan `var()` sin `@property`
> Las custom properties por defecto son *tokens*, no números: `var(--i)` dentro de un `%` de keyframes no interpola. Con `@property --i { syntax: "<number>" … }` el navegador las tipifica y **sí** las anima — desde ahí se pueden animar gradientes y counter por variable.

## Transiciones vs animaciones

| Aspecto | `transition` | `@keyframes` |
|---|---|---|
| Disparo | Requiere un cambio de estado (hover, clase, etc.) | Se reproducen solas al activarse |
| Complejidad | De un estado a otro | Infinitos pasos intermedios |
| Repetición | No (salvo que el estado oscile) | `infinite`, `alternate`, etc. |
| Control | binario: o hay cambio o no | play-state, direction, retardo, relleno |
| Costo | más liviana | requiere nombre + regla extra |

```css
.boton {
  transition: background-color 0.3s ease;
}
.boton:hover {
  background-color: #166534; /* la transición interpola el cambio */
}
```

### Las 5 propiedades de `transition`

| Propiedad | Valores | Defecto | Nota |
|---|---|---|---|
| `transition-property` | `none`, `all`, propiedad | `all` | acotar a propiedades concretas es más performante |
| `transition-duration` | tiempo | `0s` | **sin duración no hay transición** (mismo error que en animation) |
| `transition-timing-function` | igual que animation | `ease` | |
| `transition-delay` | tiempo | `0s` | negativo = arranca ya avanzada |
| `transition-behavior` | `normal`, `allow-discrete` | `normal` | permite transicionar `display: none` y custom properties |

La shorthand acepta **dos tiempos**: `transition: color 0.3s 0.1s` = duración `0.3s`, delay `0.1s` (el primero es siempre la duración).

```css
/* Transición completa: propiedades acotadas + retardo inverso en salida */
.tarjeta {
  transition: transform 0.25s ease, box-shadow 0.25s ease;
}
.tarjeta:hover {
  transform: translateY(-4px);
  box-shadow: 0 12px 24px rgb(0 0 0 / 0.15);
  transition-delay: 0s;   /* al salir del hover, reacciona al instante */
}
```

> [!tip] Retardo asimétrico: rápido al entrar, suave al salir
> El patrón de arriba es el estándar de UI: el hover "sube" con delay 0 y, al retirar el puntero, la transición de salida usa delay 0 y la de entrada la pequeña pausa. Se logra declarando `transition-delay: 0s` **solo** en el estado hover.

### Transiciones disparadas por clase (el patrón de la animación de entrada)

Como CSS no tiene "al terminar hacer X", la receta para animaciones one-shot es:

```css
.reveal {
  opacity: 0;
  transform: translateY(24px);
  transition: opacity 0.5s ease, transform 0.5s ease;
}
.reveal.visible {
  opacity: 1;
  transform: none;
}
```
```js
// JS agrega/quita la clase; idealmente con IntersectionObserver (ver Efectos de Scroll)
const io = new IntersectionObserver((entries) => {
  entries.forEach(e => e.target.classList.toggle('visible', e.isIntersecting));
});
document.querySelectorAll('.reveal').forEach(el => io.observe(el));
```

## Anatomía de una animación completa (recetas)

### 1. Spinner (rotación infinita con steps para efecto mecánico)

```css
.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #e5e7eb;
  border-top-color: #3498db;
  border-radius: 50%;
  animation: girar 0.8s linear infinite;
}
@keyframes girar { to { transform: rotate(360deg); } }
```

### 2. Skeleton loader (pulso sutil en shimmer)

```css
.skeleton {
  background: linear-gradient(90deg, #e5e7eb 25%, #f3f4f6 50%, #e5e7eb 75%);
  background-size: 200% 100%;
  animation: shimmer 1.4s linear infinite;
}
@keyframes shimmer { to { background-position: -200% 0; } }
```

### 3. Marquesina infinita sin saltos

```css
.ticker {
  display: flex;
  gap: 3rem;
  animation: ticker 20s linear infinite;
}
@keyframes ticker { to { transform: translateX(-50%); } }
/* El contenido está duplicado: al traducir el 50% el loop es perfecto */
```

### 4. Microinteracción "pop" al agregar producto

```css
.pop { animation: pop 0.35s cubic-bezier(0.34, 1.56, 0.64, 1); }
@keyframes pop {
  0%   { transform: scale(1); }
  40%  { transform: scale(1.25); }
  100% { transform: scale(1); }
}
```

### 5. Animación con `animation-play-state` para pausar (carrusel controlado por CSS)

```css
.carrusel:hover .track,
.carrusel:focus-within .track {
  animation-play-state: paused;  /* pausa al enfocar: clave para accesibilidad */
}
```

## Accesibilidad

> [!warning] Respeta `prefers-reduced-motion`
> Las animaciones intensas pueden marear a usuarios con sensibilidad al movimiento (vértigos, trastornos vestibulares). Detectalo con una media query y desactivá o reducí las animaciones:

```css
@media (prefers-reduced-motion: reduce) {
  .animatedo {
    animation: none;
    transition: none;
  }
  /* Alternativa más amable: respetar la petición pero con movimiento mínimo */
  .animatedo { animation-duration: 0.01ms; }
}
```

Otras reglas de accesibilidad para animaciones:

- **Toda animación infinita debe poder pausarse** (WCAG 2.2.2): hover/focus con `animation-play-state: paused`, o botón de control real. Las marquesinas de texto son el caso que más incumple esto.
- **No uses el movimiento como único canal de información**: si algo "se ilumina", que también cambie de ícono/texto.
- **Parpadeo** (WCAG 2.3.1): nada debe parpadear más de 3 veces por segundo — es peligroso para fotosensibilidad.
- El *stagger* largo con `opacity` de 0 puede dejar contenido invisible si el JS falla; la animación de entrada nunca debe ser la única razón de que algo sea visible.

## Rendimiento

> [!info] Animá `transform` y `opacity`
> `transform` y `opacity` se componen en la GPU sin repintar el layout, lo que da 60 fps constantes. Animar `top`, `left`, `width` o `margin` fuerza recálculos de layout en cada frame y produce tirones en móviles.

| Propiedad | Fase de render | Costo animarlo | Alternativa |
|---|---|---|---|
| `transform` | composición | **mínimo** (GPU) | — |
| `opacity` | composición | **mínimo** (GPU) | — |
| `width`, `height`, `top`, `left`, `margin` | layout | reflow cada frame | `transform: translate/scale` |
| `box-shadow` | pintado | repinta cada frame | capa con `transform` |
| `color`, `background-color` | pintado | aceptable en pocos elementos | animar capa opaca |
| `filter` | pintado/GPU mixto | aceptable, puede degradar | limitar a elementos chicos |

- **`will-change: transform, opacity`** avisa al navegador para que promueva el elemento a capa **antes** de animar. Usarlo con moderación: cada capa consume memoria; activarlo permanente en muchos elementos es contraproducente (ver también `Efectos de Scroll` para patrones ligados al scroll).
- **Transiciones de `filter: blur()`** son caras en pantallas grandes; preferí blur precalculado.
- Medí con la pestaña *Performance* de DevTools: frames rojos = pérdida de 60fps; mirá si el timeline muestra *Layout* o *Paint* en cada frame.

## Compatibilidad y novedades (2023–2026)

- **`@property`** + `transition-behavior: allow-discrete` + `@starting-style` permiten hoy animar `display: none`, custom properties tipificadas y el *primer render* de un elemento recién agregado al DOM. Soporte en todos los navegadores actuales.
- **`interpolate-size`** e `height: auto` animable (nativo) eliminan el viejo truco de animar hacia `auto` vía `grid-template-rows` o `max-height`.
- En el **Web Animations API** (JS) todo lo anterior tiene su espejo programático: `element.animate([{…},{…}], { duration: 300, fill: 'forwards' })` — útil cuando la animación depende de datos.

## Errores comunes

1. **Olvidar `animation-duration`** (o dejarla en `0s`): no se ve nada, sin errores en consola.
2. **Keyframes declarados pero sin `animation-name`** en ningún lado, o con el nombre mal escrito (distingue mayúsculas).
3. **Animar propiedades no interpolables**: `display`, `font-family` entre nombres distintos, `auto` como largo de `width`. Síntoma: salto seco en vez de interpolación.
4. **Mezclar unidades no convertibles** entre fotogramas: `50%` → `10px` no interpola en `width` sin calc; `red` → `#00ff00` sí (mismo tipo color).
5. **`forwards` olvidado** en entradas: al terminar, el elemento vuelve a su estado inicial y "desaparece".
6. **Transiciones que compiten**: si `transition` está en `:hover` en vez de en la clase base, **solo hay animación al entrar**, y la salida es un corte seco.
7. **Animar todo con `all`**: `transition: all 0.3s` anima también propiedades que no querés (y las que cambian por media queries) — acotá las propiedades.
8. **Loop que "salta"**: en marquesinas el salto final es porque no duplicaste el contenido o traduciste un % distinto al largo real del contenido.

## Cheatsheet — propiedades de animación

| Propiedad | Valores | Defecto |
|---|---|---|
| `animation-name` | identificador, `none` | `none` |
| `animation-duration` | `ms`, `s` | `0s` |
| `animation-timing-function` | `linear`, `ease`, `ease-in/out/in-out`, `cubic-bezier()`, `steps()`, `linear()` | `ease` |
| `animation-delay` | `ms`, `s` | `0s` |
| `animation-iteration-count` | número, `infinite` | `1` |
| `animation-direction` | `normal`, `reverse`, `alternate`, `alternate-reverse` | `normal` |
| `animation-fill-mode` | `none`, `forwards`, `backwards`, `both` | `none` |
| `animation-play-state` | `running`, `paused` | `running` |
| `animation-composition` | `replace`, `add`, `accumulate` | `replace` |
| `animation` (shorthand) | nombre duración función retardo conteo dirección relleno | `none 0s ease 0s 1 normal none` |

## Cheatsheet — propiedades de transición

| Propiedad | Valores | Defecto |
|---|---|---|
| `transition-property` | `none`, `all`, lista de propiedades | `all` |
| `transition-duration` | `ms`, `s` | `0s` |
| `transition-timing-function` | igual que animation | `ease` |
| `transition-delay` | `ms`, `s` | `0s` |
| `transition-behavior` | `normal`, `allow-discrete` | `normal` |
| `transition` (shorthand) | propiedad duración función retardo | `all 0s ease 0s` |

## Relacionadas

- [[Efectos de Scroll]] — efectos ligados al desplazamiento (scroll snap)
- [[Degradados en CSS]] — fondos que combinan bien con animaciones
- [[Colores CSS]] — valores de color para animar `background`
- [[Sombras CSS]] — animar sombras con cuidado (costo de pintado)
- [[Tipos de Posiciones]] — referencia útil al animar con `position`
- [[Especificidad]] — por qué una transición "no aplica": gana otra regla
