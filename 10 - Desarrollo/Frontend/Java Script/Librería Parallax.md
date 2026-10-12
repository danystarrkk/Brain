---
aliases: ["Parallax", "parallax.js", "Efecto parallax", "Efecto de profundidad"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Librería GSAP]]"
  - "[[Html]]"
  - "[[Archivos SVG]]"
  - "[[Animaciones]]"
fuente: "https://github.com/wagerfield/parallax"
---

# Librería Parallax

El **parallax** es un efecto en el que las capas de una escena se mueven a **velocidades diferentes** según el scroll, el mouse o el dispositivo. El resultado es una ligera sensación de **profundidad** (un sutil estilo 3D) que revela partes de la escena que con un plano fijo no se percibirían.

Para lograrlo se acomodan los elementos en varias **capas** que se mueven de forma independiente (en sentido vertical u horizontal), generalmente controladas por el desplazamiento del usuario por la página, aunque también pueden moverse de forma automática (por tiempo o mouse).

> [!info] ¿Cuándo usarlo?
> Es ideal cuando se busca **inmersión**: contar una historia, promocionar un juego o producto, o dar carácter a una landing. Con moderación enriquece; en exceso dificulta la lectura y, en el peor de los casos, marea.

Implementarlo a mano es largo (hay que calcular offsets en cada evento de scroll/mousemove), por eso usamos **[parallax.js](https://github.com/wagerfield/parallax)**, que recrea el efecto de forma mucho más rápida y sencilla. Más abajo está la versión a mano, las alternativas modernas y la matemática de fondo — porque usar una librería sin entender qué hace es copiar, no programar.

## La matemática del efecto

La intuición viene de la naturaleza: al mirar el paisaje desde un auto en movimiento, los árboles cercanos "pasan volando" y las montañas lejanas casi no se mueven. El cerebro interpreta **diferencias de velocidad = profundidad**.

En una página, la fórmula es la misma para cualquier capa:

```
desplazamiento_de_la_capa = factor_de_profundidad × offset_del_origen_del_efecto
```

- El **offset** es el "input": cuánto se movió el mouse respecto del centro de la escena, o cuánto scrolleó la página respecto de un punto de referencia.
- El **factor de profundidad** decide la velocidad relativa:

| Capa | Factor | Comportamiento |
|---|---|---|
| Fondo lejano (cielo, montañas) | `0.05`–`0.2` | Casi fijo |
| Medio (árboles, edificios) | `0.3`–`0.6` | Se mueve algo |
| Primer plano (personaje, texto) | `0.8`–`1.2` | Sigue casi al 100% del input |

Ejemplo concreto (parallax de scroll): si la página scrolleó 400px, una capa con factor `0.2` viaja `400 × 0.2 = 80px`, y una con `0.8` viaja `320px`. La diferencia de 240px entre ambas es lo que el ojo lee como "están a distintas distancias".

```js
// Núcleo del parallax de scroll, a mano (3 líneas de matemática)
const factor = 0.3;                                  // 0 = fijo, 1 = igual que el scroll
const offset = window.scrollY;                       // input
capa.style.transform = `translateY(${-offset * factor}px)`;
```

Para parallax de **mouse**, el input es la distancia del puntero al centro de la escena, normalizada a [-1, 1]:

```js
const rect = escena.getBoundingClientRect();
const x = (e.clientX - rect.left) / rect.width - 0.5;   // -0.5 … +0.5
const y = (e.clientY - rect.top) / rect.height - 0.5;
capa.style.transform = `translate(${x * 40}px, ${y * 40}px)`;  // 40px = amplitud
```

> [!info] El parallax de "héroes" en portales de noticias
> El clásico *parallax de fondo* (la foto de la nota se mueve más lento que el texto) es este mismo cálculo aplicado al scroll de la página — y hoy tiene implementación nativa en CSS (ver abajo).

## Enfoque 1: CSS puro (el más barato)

Antes de tocar JS, hay dos técnicas nativas:

### `background-attachment: fixed`

```css
.hero {
  min-height: 60vh;
  background: url('fondo.jpg') center / cover no-repeat fixed;
}
```

El fondo queda "atado a la ventana" y el contenido scrollea por delante: el efecto aparece solo. Cero JS, cero CPU.

> [!warning] `fixed` en móvil
> iOS Safari (y Android desde 2020) **ignoran** `background-attachment: fixed` en dispositivos táctiles — el fondo se ve mal o no scrollea. Alternativa táctil: `background-attachment: scroll` con `background-position-y` animado, o el enfoque `position: sticky` de abajo.

### `position: sticky` (parallax de dos capas, 2026)

```css
.fondo { position: sticky; top: 0; height: 100vh; }
.contenido { position: relative; }   /* scrollea por encima del fondo */
```

El fondo se "queda pegado" mientras el contenido pasa — es la base de casi todos los "efectos de scroll" modernos, con hardware acelerado y sin una línea de JS. Complemento en [[Efectos de Scroll]].

### Transiciones en clases

Para parallax discreto (la capa se reposiciona al entrar/sección), basta con una transición de `transform` — ver [[Animaciones]]:

```css
.capa { transform: translateY(0); transition: transform 0.6s ease-out; }
.seccion-visible .capa { transform: translateY(-30px); }
```

## Enfoque 2: JavaScript a mano

El patrón correcto tiene **tres piezas**: (1) guardar el input en un listener barato, (2) hacer los cálculos solo en el siguiente cuadro con `requestAnimationFrame`, (3) escribir **solo `transform`/`opacity`** (nunca `top` — ver [[Librería GSAP]] y [[Animaciones]]).

```js
// 1) Estado: el listener solo guarda, no calcula
let scrollY = 0;
let pendiente = false;

const capas = [
  { el: document.querySelector('.cielo'), factor: 0.1 },
  { el: document.querySelector('.arboles'), factor: 0.4 },
  { el: document.querySelector('.personaje'), factor: 0.9 },
].filter((c) => c.el);   // proteger contra selectores sin match (ver [[Acceder a Html]])

// 2) requestAnimationFrame: el cálculo corre 1× por cuadro como máximo
function pintar() {
  pendiente = false;
  for (const { el, factor } of capas) {
    el.style.transform = `translate3d(0, ${-scrollY * factor}px, 0)`;
  }
}

// 3) Listener liviano: marca "hay trabajo" y deja que el navegador lo programe
window.addEventListener('scroll', () => {
  scrollY = window.scrollY;
  if (!pendiente) {
    pendiente = true;
    requestAnimationFrame(pintar);
  }
}, { passive: true });   // passive: el listener no bloquea el scroll
```

Por qué así y no `scroll → calcular → pintar` directo: el evento `scroll` puede dispararse **varias veces por cuadro** (el navegador empuja eventos más rápido que el refresco). Sin `requestAnimationFrame`, el handler corre 2-3× por frame y la página se siente pesada. El patrón *bandera + rAF* garantiza **como máximo un pintado por cuadro**.

| Mecanismo | Para qué |
|---|---|
| `requestAnimationFrame` | Sincronizar con el refresco del monitor (60 fps); no calcula si la pestaña está oculta |
| `{ passive: true }` | Le al navegador "prometo no hacer `preventDefault`" → el scroll nunca se bloquea |
| `translate3d(...)` | Fuerza capa de composición GPU (misma razón del `force3D` de GSAP) |
| Throttle (tiempo) | Variante por tiempo (`máx. 1 vez cada 16ms`); rAF suele ser mejor porque se sincroniza con el frame real |
| Debounce | Para `resize`/`input`, no para scroll (espera a que paren los eventos) |

> [!warning] El anti-patrón clásico
> `window.addEventListener('scroll', () => { element.style.top = ... })` — recalcula en **cada** evento, escribe una propiedad de **layout** (reflow por frame) y no respeta el cuadro. Es la receta exacta del jank.

## Enfoque 3: parallax.js (la librería)

**parallax.js** es una librería ligera (~2 KB) que implementa el parallax de **mouse + device orientation** con física de *lerp* (suavizado: la capa *persigue* al objetivo, no lo copia de golpe).

### Instalación

No siempre hace falta instalar nada: podemos incluirlo mediante un `<script>` en nuestro [[Html|HTML]]:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/parallax/3.1.0/parallax.min.js"></script>
```

O instalarlo por npm cuando trabajemos con un proyecto y bundler:

```bash
npm i -s parallax-js
```

### Uso

1. En el HTML creamos el contenedor de las capas y le asignamos un `id`, por ejemplo `id="scene"`:

```html
<div id="scene">
  <img data-depth="0.2" src="fondo.png" alt="" />
  <img data-depth="1.0" src="personaje.png" alt="" />
</div>
```

2. A cada elemento interno le agregamos el atributo `data-depth`, que controla la **profundidad** (y por tanto cuánto se moverá respecto al puntero/scroll): valores bajos (`0.1`–`0.3`) para fondos lejanos, valores altos (`1`+) para primeros planos — la tabla de factores de la sección de matemática aplica tal cual.

3. Finalmente inicializamos la instancia apuntando al elemento contenedor:

```js
const scene = document.getElementById('scene');
const parallaxInstance = new Parallax(scene);
```

> [!warning] Error típico de sintaxis
> La línea correcta lleva el signo `=`: `const scene = document.getElementById('scene');`. Sin él, JS lanza `SyntaxError: Unexpected identifier`. También verifica que el `id` del HTML y el del `getElementById` coincidan **exactamente**.

> [!tip] Instancia por ID o por elemento
> `new Parallax()` acepta tanto el elemento (`document.getElementById('scene')`) como el selector como string (`'#scene'`). Al destruir una instancia (por ejemplo al desmontar un componente) usa `parallaxInstance.destroy()` para liberar los listeners de eventos.

### Opciones del constructor

```js
const p = new Parallax(scene, {
  relativeInput: true,   // coords. del mouse relativas a la escena (no a la ventana)
  hoverOnly: true,       // solo reacciona al hover (ideal en touch: evita el parallax con dedo)
  scalarX: 10,           // amplitud: px máx. de movimiento en X
  scalarY: 10,           // amplitud en Y
  frictionX: 0.1,        // inercia (0-1): menor = llega más lento y suave
  frictionY: 0.1,
  limitX: false,         // limitar desplazamiento al tamaño de la capa
  originX: 0.5,          // punto de reposo del mouse (centro)
  invertX: false,        // invertir dirección (fondo "huye" del cursor)
  inputElement: null,    // elemento que aporta las coords (default: la ventana)
});
```

La fricción es el detalle que distingue a la librería del enfoque a mano: en vez de posicionar la capa exactamente donde dice el cálculo, cada cuadro se acerca un 10% (`friction: 0.1`) al objetivo — resultando en el **arrastre elástico** característico.

### API de la instancia

| Método | Efecto |
|---|---|
| `p.destroy()` | Elimina listeners — **obligatorio** al desmontar |
| `p.disable()` / `p.enable()` | Pausar/reanudar sin destruir |
| `p.setDepth(x, y)` | Cambiar profundidades en vivo |
| `p.consistentSizes(bool)` | Normalizar tamaños de capas (v3) |

> [!warning] Datos decorativos en las capas
> Si las capas son `<img>`, usá `alt=""` (son decoración — ver accesibilidad abajo y [[Archivos SVG]] como formato ideal de capa vectorial). Si el texto del hero es información, ese texto **no** es capa animada.

## Alternativas modernas

| Alternativa | Cuándo elegirla |
|---|---|
| **CSS `background-attachment: fixed`** | Parallax de fondo simple en desktop; aceptar el fallback móvil |
| **CSS `position: sticky`** | El patrón "fondo fijo + contenido encima" — gratis y estable |
| **CSS scroll-driven animations** (`animation-timeline: view()`) | El futuro nativo: la animación avanza con el scroll, sin JS. A 2026, soporte Chromium pleno y el resto progresando — verificar antes de producirlo |
| **GSAP ScrollTrigger** (ver [[Librería GSAP]]) | Coreografía seria: scrub, pin, timelines sincronizadas con scroll. El estándar de facto profesional |
| **parallax.js** | Escenas interactivas con mouse/dispositivo y física de fricción, en HTML plano |
| **WebGL (Three.js, parallax shaders)** | Escenas 3D reales; otra liga de complejidad |

```js
// El mismo parallax de scroll con GSAP: sin estado, sin rAF, con scrub
gsap.to('.arboles', {
  yPercent: -30,
  ease: 'none',
  scrollTrigger: { trigger: '.hero', start: 'top top', end: 'bottom top', scrub: true },
});
```

## Accesibilidad y salud: el costo real del parallax

> [!warning] `prefers-reduced-motion` es obligatorio
> El parallax **induce mareo real**: para personas con trastornos vestibulares (migrañas, vértigo, mareo por movimiento) las capas desincronizadas pueden causar náuseas. Todos los navegadores grandes exponen la preferencia *"reducir movimiento"*. Toda implementación debe apagarse:

```js
const sinMovimiento = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
if (!sinMovimiento) {
  const p = new Parallax(scene);
}
```

```css
@media (prefers-reduced-motion: reduce) {
  .capa { transform: none !important; transition: none !important; }
}
```

- **Contenido antes que efecto**: el texto de un hero con parallax debe ser legible con las capas congeladas; el efecto es capa extra, nunca la portadora del significado.
- **Móvil = parallax apagado por defecto**: en pantallas táctiles el mouse no existe, el scroll es el dedo (impredecible) y el jank aparece mucho antes. La mayoría de los sitios serios sirve parallax solo en `hover: hover` y viewports grandes:

```js
const apto = window.matchMedia('(hover: hover) and (min-width: 768px) and (prefers-reduced-motion: no-preference)');
function montar() { /* new Parallax(scene) */ }
function desmontar() { /* p.destroy() */ }
apto.addEventListener('change', (e) => (e.matches ? montar() : desmontar()));
```

- **Rendimiento**: toda capa animada en scroll es un candidato a *compositor*; si una capa tiene `box-shadow` gigante, filtros pesados o `<video>`, el jank está garantizado. Capas livianas (PNG/SVG — ver [[Formatos Avif y Webp]] y [[Archivos SVG]]) y solo `transform`.

## Recetas completas

**1. Hero de landing con parallax de scroll, a mano y accesible:**

```html
<section class="hero">
  <div class="hero__fondo"><img src="montana.avif" alt="" /></div>
  <h1>Título del producto</h1>
</section>
```

```css
.hero { position: relative; min-height: 80vh; overflow: hidden; display: grid; place-items: center; }
.hero__fondo { position: absolute; inset: -10% 0; z-index: -1; }
.hero__fondo img { width: 100%; height: 100%; object-fit: cover; }
@media (prefers-reduced-motion: reduce) { .hero__fondo img { transform: none !important; } }
```

```js
const img = document.querySelector('.hero__fondo img');
const sinMov = matchMedia('(prefers-reduced-motion: reduce)').matches;
if (!sinMov) {
  let pend = false;
  addEventListener('scroll', () => {
    if (pend) return;
    pend = true;
    requestAnimationFrame(() => {
      pend = false;
      img.style.transform = `translate3d(0, ${scrollY * 0.25}px, 0)`;
    });
  }, { passive: true });
}
```

**2. Escena con mouse (parallax.js) y salida limpia:**

```js
let escena;
const apto = matchMedia('(hover: hover) and (prefers-reduced-motion: no-preference)');
function montar() {
  if (escena || !apto.matches) return;
  escena = new Parallax(document.getElementById('scene'), { scalarX: 12, scalarY: 12, frictionX: 0.08, frictionY: 0.08 });
}
function desmontar() {
  escena?.destroy();
  escena = null;
}
montar();
apto.addEventListener('change', (e) => (e.matches ? montar() : desmontar()));
```

**3. Lista de capas con profundidades uniformes** — declararlas en el HTML con `data-depth` y crear la instancia con delegation es todo el código que hace falta; para SVG vectorial como capa, ver [[Archivos SVG]].

## Errores comunes

1. **Parallax en móvil sin alternativa** → jank garantizado y usuarios mareados. Apagar con media queries.
2. **Olvidar `prefers-reduced-motion`** → violación directa de accesibilidad (WCAG 2.3.3).
3. **Escribir `top`/`margin` en el listener de scroll** → reflow por evento. Solo `transform`.
4. **No usar `requestAnimationFrame`** → los cálculos corren más veces que cuadros hay.
5. **`getElementById` con el `id` equivocado** → devuelve `null` y `new Parallax(null)` lanza error; verificar coincidencia exacta.
6. **No llamar `destroy()`** → listeners huérfanos siguen moviendo elementos ya fuera del DOM (típico al desmontar componentes).
7. **Texto informativo como capa animada** → inaccesible y frágil; texto quieto, decoración en parallax.
8. **Demasiadas capas con fricción alta** → cuadros perdidos; menos capas y `transform` puro.

## Cheatsheet

| Concepto | Regla |
|---|---|
| Fórmula | `desplazamiento = profundidad × offset` |
| Profundidad fondo | `0.05`–`0.2` |
| Profundidad primer plano | `0.8`–`1.2` |
| Input de scroll | `window.scrollY` |
| Input de mouse | distancia al centro, normalizada −0.5…0.5 |
| Propiedad a animar | Solo `transform` (y `opacity`) |
| Throttle moderno | `requestAnimationFrame` + bandera |
| Listener | `{ passive: true }` |
| CSS barato | `background-attachment: fixed` / `position: sticky` |
| Librería | parallax.js: `data-depth` + `new Parallax(scene)` |
| Gravedad de la escena | `frictionX/Y: 0.08`–`0.15` |
| Limpieza | `instance.destroy()` |
| Accesibilidad | Respetar `prefers-reduced-motion`; off en táctil |
| Profesional | GSAP ScrollTrigger `scrub` (ver [[Librería GSAP]]) |

## Relacionadas

- [[Librería GSAP]] — la alternativa profesional con ScrollTrigger/scrub
- [[Animaciones]] — transiciones CSS y cuándo el CSS puro alcanza
- [[Efectos de Scroll]] — sticky y patrones de scroll en CSS
- [[Html]] — estructura de la escena con capas
- [[Archivos SVG]] — buenas candidatas para las capas del parallax (peso cero al escalar)
- [[Formatos Avif y Webp]] — formatos ligeros para las capas fotográficas
- [[Eventos]] — `scroll`, `mousemove` y el modelo de listeners
- [[Acceder a Html]] — selección de capas y protección contra `null`
