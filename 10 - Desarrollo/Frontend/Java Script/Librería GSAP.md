---
aliases: ["GSAP", "GreenSock", "GreenSock Animation Platform"]
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
fuente: "https://gsap.com/docs/v3/"
---

# Librería GSAP

**GSAP** (GreenSock Animation Platform) es una biblioteca de JavaScript para crear **animaciones de alto rendimiento** que funcionan en **todos** los navegadores principales. Anima CSS, SVG, canvas, React, Vue, WebGL, colores, cadenas, rutas de movimiento, objetos genéricos… ¡todo lo que JavaScript pueda tocar!

GSAP resuelve innumerables inconsistencias del navegador; tus animaciones **simplemente funcionan**. En esencia, GSAP es un **manejador de propiedades de alta velocidad**: actualiza valores a lo largo del tiempo con extrema precisión. Consulta [por qué GSAP](https://gsap.com/why/) para ver qué lo hace tan especial.

> [!info] ¿Librería o framework?
> GSAP es una **librería**: solo aporta animación y no impone estructura al proyecto. Se usa igual en un HTML plano que dentro de [[ReactJS - Fundamentos|React]].

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

Con esto queda disponible el objeto global `gsap`. También se puede instalar por npm (`npm i gsap`) para usarlo con módulos y bundlers.

> [!tip] El orden importa
> El script de GSAP debe cargarse **antes** de nuestro código que usa `gsap`. Ver [[Java Script]] para `defer` y dónde colocar los `<script>`.

## Animaciones básicas

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

Propiedades clave:

| Propiedad | Para qué sirve |
| --------- | -------------- |
| `x`, `y` | Traslación horizontal/vertical (mejor que `left`/`top`) |
| `rotation`, `scale` | Rotación en grados y escala |
| `opacity` | Transparencia |
| `duration` | Duración en segundos |
| `ease` | Curva de aceleración (`power2.out`, `elastic.out(1, 0.3)`…) |
| `delay` | Retraso antes de iniciar |
| `stagger` | Retardo escalonado entre varios elementos |
| `onComplete` | Callback al terminar la animación |

Para animar **varias propiedades a la vez** se pasa un solo objeto con todas ellas (como en el ejemplo), y para animar varios elementos se pasa una lista de selectores GSAP aplica `stagger` automáticamente.

## Secuenciación con `gsap.timeline`

Para encadenar animaciones sin anidar callbacks, se usa una **línea de tiempo**:

```js
const tl = gsap.timeline({ defaults: { duration: 0.8, ease: 'power2.out' } });

tl.to('.hero', { opacity: 1, y: 0 })   // paso 1
  .to('.titulo', { opacity: 1, scale: 1 }, '-=0.3') // empieza 0.3s antes
  .to('.caja', { x: 300, rotation: 360, duration: 1.5 });
```

> [!tip] Posiciones en la timeline
> `'-=0.3'` hace que el paso empiece 0.3 s **antes** de que termine el anterior (solape); `'+=0.5'` añade una pausa de 0.5 s. Esto permite coreografías precisas sin `setTimeout`.

## ScrollTrigger

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

> [!warning] No uses ScrollTrigger sin registrar el plugin
> Si olvidas `gsap.registerPlugin(ScrollTrigger)` o cargas el plugin después de usarlo, GSAP lanza `ScrollTrigger is not defined` o ignora la animación silenciosamente.

## Consejos de rendimiento

- Anima **`transform`** (`x`, `y`, `scale`, `rotation`) y `opacity`: el navegador las acelera con la GPU, sin repintar el layout.
- Evita animar `top`, `left`, `width` o `height`: provocan **reflows** en cada frame.
- Usa `will-change: transform` solo en elementos que se animan con frecuencia, no en toda la página.

## Relacionadas

- [[Librería Parallax]] — efecto de profundidad con capas al hacer scroll
- [[Java Script]] — cómo cargar los scripts de la librería
- [[Eventos]] — para disparar animaciones con interacción del usuario
- [[ReactJS - Fundamentos]] — GSAP también funciona en componentes React
