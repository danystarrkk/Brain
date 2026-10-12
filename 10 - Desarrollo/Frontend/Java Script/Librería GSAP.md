---
aliases: ["GSAP", "GreenSock", "GreenSock Animation Platform", "ScrollTrigger"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-27
actualizado: 2026-10-11
relacionado:
  - "[[Librería Parallax]]"
  - "[[Java Script]]"
  - "[[Eventos]]"
  - "[[Animaciones]]"
fuente: "https://gsap.com/docs/v3/"
---

# Librería GSAP

**GSAP** (GreenSock Animation Platform) es una biblioteca de JavaScript para crear **animaciones de alto rendimiento** que funcionan en **todos** los navegadores principales. Anima CSS, SVG, canvas, objetos genéricos, React, Vue, WebGL, colores, cadenas, rutas de movimiento… todo lo que JavaScript pueda tocar.

GSAP resuelve innumerables inconsistencias del navegador; tus animaciones **simplemente funcionan**. En esencia, GSAP es un **manejador de propiedades de alta velocidad**: recibe un objeto de destino y una de valores objetivo, y actualiza esas propiedades cuadros por segundo (FPS) con extrema precisión y sin drift temporal. Consulta [por qué GSAP](https://gsap.com/why/) para ver qué lo hace tan especial.

> [!info] ¿Librería o framework?
> GSAP es una **librería**: solo aporta animación y no impone estructura al proyecto. Se usa igual en un HTML plano que dentro de [[ReactJS - Fundamentos|React]]. Llama al código vos (lo contrario de un framework, ver [[Qué es una API]]).

## Por qué GSAP y no CSS puro

| | **CSS (`transition`/`@keyframes`)** | **GSAP** |
|---|---|---|
| Arranque | Ideal para hover/estados simples | Cualquier cosa |
| Secuenciar pasos | Limitado (delays, animation-fill) | `timeline` con posiciones exactas |
| Orquestar muchos elementos | Muy verboso | `stagger`, timelines anidadas |
| Animar números/objetos/JS | No | Sí (`gsap.to(objeto, ...)`) |
| Control en tiempo real | No (pausar "a la mitad" es imposible) | `seek()`, `progress()`, `pause()` |
| Sincronizar con scroll | No nativo | **ScrollTrigger** |
| Curvas ricas | Básicas (cubic-bezier) | Catálogos de easing + propios |
| Física (rebote, elasticidad) | Muy difícil | `elastic`, `bounce`, `back` nativos |

> [!tip] ¿CSS me alcanza?
> Si la animación es **reaccional** (hover, focus, un estado que entra/sale con una clase), CSS es más simple y gratis — ver [[Animaciones]]. Si necesitás **coreografía** (varios pasos coordinados, scroll, precisión de tiempo, control programático), ahí entra GSAP. La regla: CSS para estados; GSAP para narrativa.

## Instalación (CDN)

Para usar GSAP sin instalar nada, agregamos el script principal en el HTML:

```html
<script
  src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"
  integrity="sha512-7eHRwcbYkK4d9g/6tD/mhkf++eoTHwpNM9woBxtPUBWm67zeAfFC+HrdoE2GanKeocly/VxeLvIqwvCdk7qScg=="
  crossorigin="anonymous"
  referrerpolicy="no-referrer"
></script>
```

Con esto queda disponible el objeto global `gsap`. También se puede instalar por npm (`npm i gsap`) para usarlo con módulos y bundlers:

```js
import { gsap } from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);
```

> [!tip] El orden importa
> El script de GSAP debe cargarse **antes** de nuestro código que usa `gsap`. Ver [[Java Script]] para `defer` y dónde colocar los `<script>`.

## Conceptos nucleares: tween, timeline, selección

| Concepto | Qué es |
|---|---|
| **Tween** | La unidad: anima propiedades de un objetivo de estado A a estado B durante un tiempo |
| **Timeline** | Contenedor ordenado de tweens; el tiempo de cada hijo se calcula respecto de la posición donde se inserta |
| **Target** | Lo que se anima: selector string, elemento DOM, objeto JS o NodeList |
| **Vars** | El objeto de variables: qué propiedades, cuánto dura, con qué easing, callbacks |
| **Vars de tween** | `x`, `y`, `rotation`, `scale`, `opacity`, `duration`, `ease`, `delay`, `stagger`, `onComplete`… |
| **Plugin** | Módulo que extiende las propiedades animables (ScrollTrigger, MotionPath…) |

GSAP acepta como target: `'.clase'`, `'#id'`, `elemento`, `[el1, el2]`, `document.body` — y un mismo tween puede apuntar a **muchos** elementos a la vez.

## Animaciones básicas: `to`, `from`, `fromTo`

El corazón de GSAP son tres métodos: `gsap.to`, `gsap.from` y `gsap.fromTo`. Todos animan un elemento HTML desde/CSS hacia nuevos valores durante un tiempo determinado:

```html
<style>
  .caja {
    width: 100px;
    height: 100px;
    background: dodgerblue;
  }
</style>

<div class="caja"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
<script>
  // to: parte del estado actual y va hacia los valores indicados
  gsap.to('.caja', { x: 300, rotation: 360, duration: 1.5, ease: 'power2.out' });

  // from: parte de los valores indicados y vuelve al estado actual
  // gsap.from('.caja', { opacity: 0, duration: 1 });

  // fromTo: define explícitamente estado inicial y final
  // gsap.fromTo('.caja', { x: -100 }, { x: 100, duration: 1 });
</script>
```

Cuándo usar cada uno:

| Método | Semántica | Riesgo | Cuándo |
|---|---|---|---|
| `to` | "De donde esté ahora, andá a…" | El valor inicial depende del CSS actual | Lo más usado |
| `from` | "Vení de X al estado actual" | *Flash of unanimated content*: el elemento se ve en su estado final hasta que GSAP arranca | Entradas (fade-in) |
| `fromTo` | "De X a Y, sin ambigüedad" | Ninguno — los dos extremos son explícitos | Cuando el inicial debe ser exacto |

### Variables comunes de un tween (vars)

| Var | Para qué sirve | Nota |
|---------| -------------- |---|
| `x`, `y` | Traslación horizontal/vertical (mejor que `left`/`top`) | Van por `transform` → GPU |
| `rotation`, `rotationX/Y` | Rotación en grados (3D disponible) | |
| `scale`, `scaleX/Y` | Escala | |
| `opacity` | Transparencia | Compositor, barato |
| `duration` | Duración en **segundos** (no ms) | `1.5` = 1.5s |
| `ease` | Curva de aceleración | Ver catálogo abajo |
| `delay` | Retraso antes de iniciar | | |
| `stagger` | Retardo escalonado entre varios elementos | Número, o `{ each, from, amount }` |
| `paused`, `repeat`, `yoyo`, `repeatDelay` | Control de ciclo | `repeat: -1` = infinito; `yoyo` va y vuelve |
| `onStart`, `onUpdate`, `onComplete` | Callbacks del ciclo de vida | `this` = el tween |
| `overwrite` | ¿Matar tweens previos del mismo target? | `'auto'` resuelve conflictos |
| `immediateRender` | Renderizar el estado inicial al crear | Clave en `from`/`fromTo` |
| `modifiers`, `snap` | Transformar valores en vuelo | Avanzado |

Para animar **varias propiedades a la vez** se pasa un solo objeto con todas ellas (como en el ejemplo), y para animar varios elementos se pasa una lista de selectores; GSAP aplica `stagger` automáticamente cuando se lo pide:

```js
// Todos los items, con 0.08s de retardo escalonado desde el primero
gsap.from('.item', { y: 30, opacity: 0, duration: 0.6, stagger: 0.08 });

// Variantes potentes del stagger
gsap.to('.item', { opacity: 1, stagger: { each: 0.1, from: 'center', amount: 0.5 } });
// from: 'start' | 'center' | 'edges' | 'random' | índice | nombre de elemento
```

### `gsap.set` y utilidades de control

- `gsap.set(target, vars)` = un tween de duración 0: posiciona sin animar (ej. `gsap.set('.modal', { autoAlpha: 0 })`).
- `const tw = gsap.to(...)` devuelve el tween: `tw.pause()`, `tw.resume()`, `tw.reverse()`, `tw.progress(0.5)`, `tw.seek(1.2)`, `tw.timeScale(2)` (velocidad), `tw.kill()` (matar), `tw.restart()`.
- `gsap.killTweensOf('.caja')` mata los tweens de un target — indispensable antes de re-animar en eventos repetidos.
- `gsap.utils.toArray('.item')` → NodeList convertida a arreglo real; `gsap.utils.clamp(0, 100, v)`, `gsap.utils.mapRange(...)`, `gsap.utils.snap(5)`.

```js
// Seleccionar con contexto (útil para no pisar otros componentes)
const items = gsap.utils.toArray('.card');         // arreglo ordenado del DOM
const primero = document.querySelector('.card');   // selección cruda del DOM (ver [[Acceder a Html]])
```

## Secuenciación con `gsap.timeline`

Para encadenar animaciones sin anidar callbacks, se usa una **línea de tiempo**: cada `.to()` se agrega al final de la anterior, y los `defaults` se heredan por todos los hijos:

```js
const tl = gsap.timeline({ defaults: { duration: 0.8, ease: 'power2.out' } });

tl.to('.hero', { opacity: 1, y: 0 })   // paso 1: 0 → 0.8s
  .to('.titulo', { opacity: 1, scale: 1 }, '-=0.3') // empieza 0.3s antes de que termine el anterior
  .to('.caja', { x: 300, rotation: 360, duration: 1.5 })
  .add('fin')                            // etiqueta (label)
  .to('.footer', { opacity: 1 }, 'fin+=0.2');  // 0.2s DESPUÉS de la etiqueta 'fin'
```

### Posiciones: el parámetro que hace magia

El segundo argumento de cada `.to()` es la **posición** — *cuándo* se inserta en la línea:

| Sintaxis | Significado | Ejemplo |
|---|---|---|
| *(omitido)* | Justo después del final del tween anterior | secuencia estricta |
| `1.5` | En el segundo **1.5 absoluto** de la timeline | sincronizar con un evento externo |
| `'+=0.5'` | 0.5s **después** del final de lo anterior | pausa entre pasos |
| `'-=0.3'` | 0.3s **antes** del final de lo anterior | solape / crossfade |
| `'<` | Al **inicio** del tween anterior | paralelo exacto |
| `'<+=0.2'` | 0.2s después del inicio del anterior | paralelo con retardo |
| `'nombre'` | En la **etiqueta** `nombre` | anclar a un punto nombrado |
| `'fin+=0.2'` | 0.2s después de la etiqueta `fin` | offsets sobre etiquetas |
| `[0, 1]` | Insertar entre los tweens en índice 0 y 1 | avanzado |

> [!tip] Etiquetas (*labels*) para coreografía grande
> `tl.add('ataque')` marca un punto nombrado; después cualquier tween puede anclarse a `'ataque'` o `'ataque+=0.5'`. Reordenar la animación se vuelve mover etiquetas, no recalcular números.

- `tl.restart()`, `tl.pause()`, `tl.seek(2)`, `tl.progress(0.5)`, `tl.timeScale(1.5)`, `tl.repeat(-1)` funcionan igual que en un tween (la timeline **es** un tween especial).
- `tl.to('.a', {...}, 0)` — posición `0` = "al principio": útil para metadatos/overlays que deben arrancar con todo.
- Timelines se anidan: `tl.add(otraTimeline, 1)` — módulos de animación reutilizables.

## Easing: la curva define la personalidad

`ease` controla **cómo** se reparte el progreso en el tiempo (no la duración). Elegir mal es la diferencia entre "profesional" y "flash de Windows 95".

| Categoría | Valores | Carácter |
|---|---|---|
| Lineal | `none` / `power1.in`≈ | Sin aceleración — solo para sliders/valores técnicos |
| Suave | `power2.out`, `power3.out` | Llegada elegante; el 90% de los UI fades |
| Entrada | `power2.in` | Acelera al salir — entradas dramáticas |
| Entrada y salida | `power1.inOut`, `power2.inOut` | Transiciones de sección |
| Rebote | `bounce.out` | Rebota al llegar (física de pelota) |
| Elástico | `elastic.out(1, 0.3)` | Resorte; parámetros: amplitud, periodo |
| Overshoot | `back.out(1.7)` | Se pasa un poco y vuelve — botones, checks |
| Exponencial | `expo.out` | Casi quieto y de golpe llega |
| Circulares | `circ.inOut` | Curva circular suave |

La familia `powerN` = `power1` (cuadrática), `power2` (cúbica), `power3`, `power4` — cuanto mayor N, más pronunciada la curva. Cada una tiene `.in`, `.out`, `.inOut`. Nombres personalizados: `CustomEase.create('custom', 'M0,0 C0.4,0 0.2,1 1,1')`.

```js
gsap.to('.modal', { scale: 1, duration: 0.6, ease: 'back.out(1.7)' });   // aparece con personalidad
gsap.to('.caida', { y: 300, duration: 1.2, ease: 'bounce.out' });         // rebota al llegar
gsap.to('.suave', { opacity: 0, duration: 0.4, ease: 'power2.out' });     // fade clásico
```

> [!info] Regla de oro del easing
> Las **cosas que llegan** usan `.out` (frenan al llegar). Las **cosas que se van** usan `.in` (arrancan despacio). Las **transiciones continuas** usan `.inOut`.

## ScrollTrigger: animaciones ligadas al scroll

Para agregar animaciones ligadas al **scroll** agregamos el plugin ScrollTrigger:

```html
<script
  src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/ScrollTrigger.min.js"
  integrity="sha512-onMTRKJBKz8M1TnqqDuGBlowlH0ohFzMXYRNebz+yOcc5TQr/zAKsthzhuv0hiyUKEiQEQXEynnXCvNTOk50dg=="
  crossorigin="anonymous"
  referrerpolicy="no-referrer"
></script>
```

Lo primero es **registrar el plugin** (una sola vez) y luego asignar la animación al elemento/clase que vamos a alterar:

```js
gsap.registerPlugin(ScrollTrigger);

// La animación se dispara cuando la sección entra en pantalla
gsap.from('.seccion', {
  scrollTrigger: {
    trigger: '.seccion',   // elemento que dispara
    start: 'top 80%',      // dispara cuando su tope llegue al 80% del viewport
    toggleActions: 'play none none reverse', // reproduce al entrar, revierte al salir
  },
  y: 100,
  opacity: 0,
  duration: 1,
});
```

La clase `.seccion` debe existir en el HTML:

```html
<section class="seccion">
  <h2>Contenido que aparece al hacer scroll</h2>
</section>
```

### Opciones de `scrollTrigger`

| Opción | Valores | Función |
|---|---|---|
| `trigger` | elemento/selector | Qué elemento observa |
| `start` | `'top 80%'`, `'top center'`, `'top bottom'`, `'top 100px'` | Cuándo empieza: *borde del trigger* + *posición del viewport* |
| `end` | `'bottom top'`, `'+=500'`, `'bottom 20%'` | Cuándo termina (defecto: `bottom top`) |
| `toggleActions` | `'onEnter onLeave onEnterBack onLeaveBack'` | Qué hacer en cada cruce: `play pause resume reverse restart reset complete none` |
| `scrub` | `true`, `false`, número (segundos) | **Liga el progreso al scroll**: la animación avanza/retrocede con el dedo. El número suaviza con retardo |
| `pin` | `true` / elemento / selector | **Fija** el trigger mientras dura el tramo (efecto "pantalla") |
| `pinSpacing` | `true`/`false` | ¿Deja espacio para que el scroll continúe tras el pin? (casi siempre `true`) |
| `markers` | `true` | Muestra visualmente start/end/inicio del scrub — solo en desarrollo |
| `once` | `true` | Se activa una sola vez y se desconecta |
| `anticipatePin` | `1` | Anticipa el pin para evitar saltos |
| `refreshPriority` | número | Orden al recalcular (múltiples triggers) |

```js
// Scrub: la animación es 1:1 con el scroll (efecto "despliegue")
gsap.to('.progreso', {
  scaleX: 1,
  scrollTrigger: { trigger: '.hero', start: 'top top', end: 'bottom top', scrub: true },
});

// Pin: la sección se queda fija mientras el contenido avanza
gsap.to('.panel-2', {
  xPercent: -100,
  ease: 'none',
  scrollTrigger: {
    trigger: '.contenedor',
    pin: true,
    scrub: 1,
    end: '+=1000',   // 1000px de scroll "consumidos" por el pin
  },
});
```

> [!warning] No uses ScrollTrigger sin registrar el plugin
> Si olvidas `gsap.registerPlugin(ScrollTrigger)` o cargas el plugin después de usarlo, GSAP lanza `ScrollTrigger is not defined` o ignora la animación silenciosamente.

> [!warning] `scrub` vs `toggleActions`
> Son filosofías distintas: `toggleActions` **dispara** animaciones (one-shot: entran y ya). `scrub` las **escribe** según la posición del scroll (el usuario controla el tiempo). Mezclarlos sin entender produce el bug típico: "la animación se queda a medias porque el usuario no scrolleó hasta el final".

Tras cambiar el layout (cargar imágenes, togglear contenido): `ScrollTrigger.refresh()` recalcula todas las posiciones; en apps dinámicas se llama dentro de `window.addEventListener('load', ...)` o desde los loaders de datos.

## Integración con el DOM y ciclos de vida

- GSAP es un **script clásico de DOM**: accede a los elementos vía selectores, igual que en [[Document Object Model (DOM)]] / [[Acceder a Html]]. Los selectores son strings de CSS normales (`.clase`, `#id`, `'li:nth-child(2)'`).
- **Disparar con interacción**: los tweens se crean/despausan desde handlers de [[Eventos]] — `boton.addEventListener('click', () => tl.restart())`.
- **React / frameworks**: GSAP toca el DOM directamente, lo que choca con el DOM virtual. Reglas: animá con `useEffect` + refs (`ref.current` como target, no selectores globales), y **limpiá** con `return () => { ctx.revert(); }` usando `gsap.context`:

```js
useEffect(() => {
  const ctx = gsap.context(() => {
    gsap.from(ref.current, { opacity: 0, y: 20, duration: 0.6 });
  }, ref);        // scope: los selectores de adentro buscan dentro de ref
  return () => ctx.revert();   // desmonta limpio: mata tweens y restaura estilos
}, []);
```

Para patrones de animación en React con GSAP y Vite, ver [[ReactJS - Fundamentos]] y [[ReactJS - Vite]].

## Rendimiento

- Anima **`transform`** (`x`, `y`, `scale`, `rotation`) y `opacity`: el navegador las acelera con la GPU, sin repintar el layout. GSAP lo detecta solo.
- Evita animar `top`, `left`, `width` o `height`: provocan **reflows** en cada frame. Para mover cajas de layout, animá `margin`/`transform` y, si hace falta, aceptá el costo con conocimiento (ver [[Animaciones]]).
- **`force3D`**: por defecto GSAP fuerza `translate3d` en transformaciones para que el elemento viva en una capa de composición propia (GPU). `'auto'` (defecto) lo decide según conveniencia; `force3D: true` lo obliga, `false` ahorra memoria en animaciones 2D simples.
- `will-change: transform` solo en elementos que se animan con frecuencia, no en toda la página.
- **Matar lo que no corre**: `ScrollTrigger` con `once: true` para entradas one-shot; `mm.add()` (matchMedia) para crear triggers solo en ciertos viewports y destruirlos al salir:

```js
// Solo en desktop; al cambiar de breakpoint se revierte todo
const mm = gsap.matchMedia();
mm.add('(min-width: 768px)', () => {
  gsap.to('.sidebar', { x: 0, scrollTrigger: { trigger: '.main', scrub: true } });
});   // al salir del media query: context.revert() automático
```

- Un `scrub` sobre muchísimos elementos en móvil da *jank*: reducir el alcance o usar CSS puro en pantallas chicas (la misma lección que en [[Librería Parallax]]).

## Errores comunes

1. **`gsap` is not defined** — el script de GSAP no cargó antes de usarse, o el código corre con el script en `head` sin `defer`. Ver [[Java Script]].
2. **`ScrollTrigger is not defined`** — no se cargó/registró el plugin (arriba).
3. **Selector sin elementos**: `gsap.to('.no-existe', ...)` no lanza error, **no hace nada**. Si la animación "no corre", verificá `document.querySelector('.no-existe')` en la consola (ver [[La Consola]]).
4. **`from` con FOUC**: el `from` deja el elemento visible hasta que el script corre → parpadeo. Solución: `immediateRender: true` (por defecto en `from`) o estado inicial en CSS.
5. **Duración en ms por costumbre CSS**: `duration: 800` en GSAP son **800 segundos**. GSAP usa segundos con decimales (`0.8`).
6. **Animar `width`/`top`** en vez de `x`/`scale` — reflow por frame.
7. **Olvidar `refresh()`** tras cambios de layout: los `start`/`end` quedan apuntando a posiciones viejas.
8. **Dos tweens peleando por la misma propiedad** en eventos repetidos → usar `overwrite: 'auto'` o `gsap.killTweensOf()` antes.

## Recetas

**Reveal por secciones al hacer scroll:**

```js
gsap.utils.toArray('.bloque').forEach((bloque) => {
  gsap.from(bloque, {
    scrollTrigger: { trigger: bloque, start: 'top 85%', once: true },
    y: 40, opacity: 0, duration: 0.7, ease: 'power2.out',
  });
});
```

**Contador animado (números, ver [[Números]]):**

```js
const contador = { valor: 0 };
gsap.to(contador, {
  valor: 1000, duration: 2, ease: 'power1.out',
  onUpdate: () => { salida.textContent = Math.round(contador.valor); },
});
```

**Menú modal con timeline reversible:**

```js
const tl = gsap.timeline({ paused: true });
tl.to('.overlay', { opacity: 0.5, duration: 0.3 })
  .fromTo('.modal', { y: 40, opacity: 0, scale: 0.95 }, { y: 0, opacity: 1, scale: 1, duration: 0.4, ease: 'back.out(1.4)' }, '-=0.1');
abrir.addEventListener('click', () => tl.play());
cerrar.addEventListener('click', () => tl.reverse());
```

## Recursos

- Docs oficiales: [gsap.com/docs/v3](https://gsap.com/docs/v3/) — referencia completa de todos los métodos y plugins.
- [ScrollTrigger docs](https://gsap.com/docs/v3/Plugins/ScrollTrigger/) y demos en CodePen del sitio oficial.
- **GSAP es gratuito para el 99% de usos** (incluyeBonus club para efectos premium): revisar la licencia de GreenSock antes de uso comercial corporativo.

## Relacionadas

- [[Librería Parallax]] — efecto de profundidad con capas al hacer scroll (y su versión a mano)
- [[Animaciones]] — cuándo CSS puro alcanza (transiciones y keyframes)
- [[Java Script]] — cómo cargar los scripts de la librería
- [[Eventos]] — para disparar animaciones con interacción del usuario
- [[Document Object Model (DOM)]] — los elementos que GSAP selecciona y muta
- [[Acceder a Html]] — selección de elementos sin librerías
- [[ReactJS - Fundamentos]] — GSAP también funciona en componentes React (con `useEffect` y contexto)
