---
tags:
  - desarrollo
  - frontend
  - eventos
tipo: concepto
estado: terminado
---

# Eventos

Un **evento** es cualquier notificación que el navegador lanza cuando algo pasa: un click, una tecla, un envío de formulario, el scroll... **Escuchar eventos** con `addEventListener` es lo que hace a una página *interactiva*: el código corre **reaccionando** al usuario, no en línea recta.

```js
console.log(1);

document.addEventListener('DOMContentLoaded', () => {
  console.log(2); // se ejecuta cuando el HTML terminó de cargarse
});

console.log(3);
// Orden en consola: 1, 3, 2 — el código normal no espera a los eventos
```

> [!info] Modelo mental: el programa espera, no se detiene
> JavaScript es monohilo pero el navegador no: los eventos se encolan en la *event queue* y el **event loop** los despacha al hilo principal cuando la pila de llamadas queda vacía. Por eso el `2` sale al final: el listener se registra ya, pero su ejecución espera su turno. Esta misma mecánica explica por qué un `while` infinito congela la página: el hilo principal nunca queda libre para despachar eventos.

## El flujo de eventos: captura, destino y burbujeo

Cuando ocurre un evento (p. ej. click en un `<span>` dentro de `<div.card>` dentro de `<body>`), el navegador lo recorre en **tres fases**:

```
        ┌─────────────────────────────────────────────┐
        │ FASE DE CAPTURA (capture): document → ... → │
        │ el padre más cercano del destino            │
        ▼                                             │
   document                                          │
     html                                            │
      body         FASE OBJETIVO:                    │
        div.card   el evento se dispara EN el         │
          span ◀── elemento donde ocurrió (target)    │
        div.card                                     │
      body         FASE DE BURBUJEO                  │
     html          (bubble): sube hasta document ▼   │
   document                                          │
        └─────────────────────────────────────────────┘
```

| Fase | Dirección | Quién la escucha |
| ---- | --------- | ---------------- |
| **Captura** | De `document` hacia abajo hasta el padre del destino | Listeners con `capture: true` (poco usada) |
| **Objetivo** | El propio `target` | Listeners del elemento (sin importar capture) |
| **Burbujeo** | Del padre hacia arriba hasta `document` | Casi todos los listeners (por defecto) |

Verificado en el ejemplo de "bubbling" más abajo: los listeners del padre se ejecutan **después** del hijo porque el evento *sube*. La captura existe para interceptar eventos **antes** de que lleguen al destino (p. ej. un único listener de `click` en `document` con `capture: true` que funcione aunque los hijos llamen `stopPropagation`).

> [!tip] Por qué la burbujeo es el 90% del diseño de eventos
> Al subir por el árbol, un **padre puede atender a sus hijos** sin registrar un listener por cada uno: la *delegación* (ver más abajo). Todos los mecanismos de frameworks — y el patrón de synthetic events de React — descansan en este bubbling.

## `addEventListener`: firma y opciones

```js
elemento.addEventListener(tipo, handler, opciones);
```

| Opción | Tipo | Efecto | Ejemplo de uso |
| ------ | ---- | ------ | -------------- |
| `capture` | boolean | El listener corre en **captura** en vez de en burbujeo | Interceptar antes que los hijos |
| `once` | boolean | Se auto-elimina tras el **primer** disparo | Modales de bienvenida, notificaciones |
| `passive` | boolean | Promete **no llamar** `preventDefault()`: el navegador scrollea sin esperar | `touchstart`, `wheel`, `scroll` |
| `signal` | `AbortSignal` | Se elimina todos los listeners asociados al abortar el controller | Limpieza ordenada (React, SPA) |

```js
// once: se ejecuta una sola vez
boton.addEventListener('click', () => alert('¡Bienvenido!'), { once: true });

// passive: scroll fluido — verificá que no uses preventDefault ahí
window.addEventListener('touchstart', onToque, { passive: true });

// capture: escuchar en la bajada (fase de captura)
document.addEventListener('click', (e) => console.log('captura:', e.target), true);

// signal: eliminación en bloque con AbortController
const ctrl = new AbortController();
boton.addEventListener('click', handler, { signal: ctrl.signal });
otroBoton.addEventListener('click', handler2, { signal: ctrl.signal });
ctrl.abort(); // elimina AMBOS listeners de golpe
```

> [!warning] `passive` y `preventDefault` son incompatibles
> Con `{ passive: true }` jurás que no vas a llamar `preventDefault()`; si lo hacés, el navegador emite un warning y la prevención se ignora. El modo *passive* existe porque el navegador **no puede esperar** a tu JS para decidir si scrollea: con listeners normales en `touchstart`/`wheel` debe posponer el scroll hasta ejecutarte (y si tu handler tarda, la página se siente lerda). Páginas de noticias y apps con scroll custom usan `passive` para eliminar ese cuello de botella.

### `removeEventListener`: las trampas

```js
// ❌ ESTO NO FUNCIONA: dos funciones distintas, aunque el código sea idéntico
boton.addEventListener('click', () => console.log('x'));
boton.removeEventListener('click', () => console.log('x'));

// ✅ La MISMA referencia de función
function alClick() { console.log('x'); }
boton.addEventListener('click', alClick);
boton.removeEventListener('click', alClick);
```

Las reglas para que la eliminación funcione:

1. **Misma referencia de función** — cada arrow function literal es un objeto nuevo; por eso "recrear" el handler no sirve.
2. **Mismo `type`** — `'Click'` y `'click'` son listeners distintos.
3. **Mismo `capture`** — remover sin la bandera que se agregó (o viceversa) no elimina nada. Las banderas `once`/`passive` no afectan la identidad.

> [!tip] `AbortController` como patrón de limpieza moderno
> En lugar de coleccionar `removeEventListener` sueltos, agrupa los listeners de un componente bajo un `AbortController` y aborta todo al desmontar. Es el patrón que usan las librerías modernas y evita el olvido número uno de las fugas de memoria. En React, el `useEffect` cleanup cumple el mismo rol — ver [[ReactJS - Efectos (useEffect)]].

## El objeto evento

El handler recibe un objeto `Event` (o una subclase) con el contexto completo:

| Propiedad / método | Qué da |
| ------------------ | ------ |
| `type` | Nombre del evento (`"click"`, `"input"`) |
| `target` | Elemento que **originó** el evento |
| `currentTarget` | Elemento cuyo listener se está ejecutando (el que lo registró) |
| `eventPhase` | 1 captura / 2 destino / 3 burbujeo |
| `bubbles` | ¿Este tipo de evento burbujea? |
| `cancelable` | ¿Admite `preventDefault()`? |
| `preventDefault()` | Evita la **acción por defecto** del navegador |
| `stopPropagation()` | Corta el recorrido: no llega a otros elementos |
| `stopImmediatePropagation()` | Corta el recorrido **y** los demás listeners del mismo elemento |
| `defaultPrevented` | ¿Ya se llamó preventDefault? |
| `timeStamp`, `isTrusted` | Cuándo ocurrió; si fue disparado por el usuario (`isTrusted: false` = generado con código) |

> [!info] `target` vs `currentTarget` — la distinción que salva bugs
> Con **delegación**, un click dentro del `<span>` del título hace que el listener del `.card` reciba `target = <span>` pero `currentTarget = <div.card>`. `target` es el origen (para identificar *qué* se clickeó); `currentTarget` es *dónde escuchas*. En arrow functions anidadas, `this` no apunta al elemento — `evt.currentTarget` es tu amigo (ver el ejemplo del botón más abajo).

### `preventDefault` vs `stopPropagation` vs `stopImmediatePropagation`

| Método | Corta | Ejemplo de efecto |
| ------ | ----- | ----------------- |
| `preventDefault()` | La **acción por defecto** (no la propagación) | En `submit`: no recarga; en `keydown`: no escribe la tecla |
| `stopPropagation()` | La **propagación** a otros elementos (no la acción) | El padre no se entera del click |
| `stopImmediatePropagation()` | Propagación + **otros listeners del mismo elemento y fase** | Dos handlers en el mismo botón: el segundo no corre |

> [!warning] `stopPropagation` con cuidado
> Detener la propagación en muchos niveles puede "ahogar" listeners legítimos del padre (p. ej. un analytics global en `document`). Prefiere diseñar el flujo y usar `stopPropagation` solo donde hace falta.

## Eventos de mouse

```js
const nav = document.querySelector('.navegacion');

nav.addEventListener('click', () => console.log('click en nav'));

nav.addEventListener('mouseenter', () => {
  console.log('entrando en la navegación');
  nav.style.backgroundColor = 'transparent';
});

nav.addEventListener('mouseout', () => {
  console.log('saliendo de la navegación');
  nav.style.backgroundColor = 'white';
});

nav.addEventListener('mousedown', () => console.log('Has presionado'));  // botón abajo
nav.addEventListener('mouseup', () => console.log('Has soltado el click')); // botón arriba
nav.addEventListener('dblclick', () => console.log('Doble click'));
```

| Evento | Cuándo dispara | ¿Burbujea? |
| ------ | -------------- | ---------- |
| `click` | presionar **y** soltar sobre el elemento | Sí |
| `dblclick` | doble click | Sí |
| `mousedown` / `mouseup` | solo la parte de presionar / soltar | Sí |
| `mouseenter` / `mouseleave` | el cursor entra o sale del elemento (**sin** dispararse al cruzar hijos) | **No** |
| `mouseover` / `mouseout` | igual, pero **sí** burbujea y se dispara al cruzar hijos | Sí |

Coordenadas del cursor (en el objeto evento): `clientX/Y` (respecto a la ventana), `pageX/Y` (respecto al documento completo, incluye scroll), `offsetX/Y` (respecto al elemento destino). `which`/`button` identifican qué botón del ratón.

## Eventos de teclado

```js
const busqueda = document.querySelector('.busqueda');

busqueda.addEventListener('keydown', () => console.log('Tecla presionada')); // al pulsar
busqueda.addEventListener('keyup', () => console.log('Tecla soltada'));       // al soltar
busqueda.addEventListener('blur', () => console.log('Salió del campo'));      // muy usado en validaciones

busqueda.addEventListener('copy', () => console.log('Copiar'));
busqueda.addEventListener('paste', () => console.log('Pegar'));
busqueda.addEventListener('cut', () => console.log('Cortar'));
```

### `key` vs `code`: la distinción importante

| Propiedad | ¿Qué mide? | Ejemplo con Shift+1 en teclado español | Ejemplo con `A` |
| --------- | ---------- | -------------------------------------- | --------------- |
| `event.key` | La **letra/función resultante** (depende del layout y Shift) | `key: "!"` | `key: "a"` o `"A"` |
| `event.code` | La **posición física** de la tecla | `code: "Digit1"` | `code: "KeyA"` |

Regla de oro: **juegos y atajos por posición** usan `code` (`ArrowUp`, `Space`, `KeyW`); **entrada de texto y comparaciones lingüísticas** usan `key` (`Enter`, `Escape`, `"ñ"`). `event.repeat` indica que la tecla está manteniéndose; las combinaciones se leen por `evt.ctrlKey`, `evt.shiftKey`, `evt.altKey`, `evt.metaKey` (Cmd/Ctrl).

### Tabulación y flechas

- El orden de **Tab** lo define el orden del DOM más `tabindex`: los elementos enfocables por defecto son `a[href]`, `button`, `input`, `select`, `textarea`; `tabindex="0"` añade al orden natural, `tabindex="-1"` saca de la tabulación pero permite foco programático, `tabindex > 0` crea orden artificial (evitar: rompe la lectura con lector de pantalla).
- Las **flechas** no tienen acción por defecto en la mayoría de controles (salvo `input` y componentes nativos como `select`). Manejarlas es patrón de widgets personalizados (menús, tabs):

```js
widget.addEventListener('keydown', (evt) => {
  if (evt.key === 'ArrowDown') { moverFocoAlSiguiente(); evt.preventDefault(); }
  if (evt.key === 'Enter') activarSeleccionado();
});
```

## Eventos de puntero: la unificación de ratón/touch/pen

Los *Pointer Events* unifican `mouse*` y `touch*` en una sola familia:

| Evento unificado | Reemplaza |
| ---------------- | --------- |
| `pointerdown` / `pointerup` | `mousedown/up` + `touchstart/end` |
| `pointermove` | `mousemove` + `touchmove` |
| `pointerover` / `pointerout` | `mouseover/out` + `touchstart` |
| `pointerenter` / `pointerleave` | `mouseenter/leave` + equivalente touch |
| `pointercancel` | pérdida del contacto (llamada, gesto del sistema) |

`evt.pointerType` distingue `"mouse"`, `"touch"` o `"pen"`; `evt.pointerId` identifica cada dedo (para multi-touch). Para interfaces que deben funcionar en móvil y escritorio, `pointer*` + `touch-action: none` en CSS reemplaza a los dos juegos por completo.

## Eventos de formulario

```js
const formulario = document.querySelector('#formulario');

formulario.addEventListener('submit', validarFormulario);

function validarFormulario(evt) {
  evt.preventDefault(); // detiene el envío tradicional (recarga de página)
  console.log(evt.target.method); // GET o POST
  console.log(evt.target.action); // a dónde se enviaría

  // en este punto puedes validar los campos o consumir una API
}
```

> [!warning] `preventDefault()` es casi obligatorio en `submit`
> Sin él, el navegador recarga la página y pierdes todo el trabajo de validación. Solo omítelo si de verdad quieres el envío nativo del formulario.

`input` vs `change` — el par más confundido:

| Evento | Se dispara en | `change` en `<input type="text">` |
| ------ | ------------- | -------------------------------- |
| `input` | **Cada** cambio de valor (teclear, pegar, borrar) | — |
| `change` | Al **confirmar**: al perder foco (texto) o al seleccionar (checkbox/radio/select) | Solo una vez, al salir del campo |

### El evento `input`: el más útil

El evento `input` se dispara **cada vez que el valor cambia** (teclear o borrar), así que cubre `keydown` y `keyup` a la vez:

```js
busqueda.addEventListener('input', (evt) => {
  console.log(evt.type);        // "input": el tipo de evento
  console.log(evt.target);       // el ELEMENTO que disparó el evento
  console.log(evt.target.value); // lo que el usuario lleva escrito

  if (evt.target.value === '') {
    console.log('El campo quedó vacío');
  }
});
```

`evt.target` es la clave para saber **qué elemento originó** el evento — ver "Delegación" más abajo. Otros útiles: `focus`/`blur` (con `focusin`/`focusout` si necesitás burbujeo — `focus` no burbujea) y `reset` del formulario. Validación nativa: `evt.target.checkValidity()` y el evento `invalid` por campo (ver [[Formularios]]).

## Scroll

```js
window.addEventListener('scroll', () => {
  const scrollPX = window.scrollY; // píxeles scrolleados en vertical

  const premium = document.querySelector('.premium');
  const ubicacion = premium.getBoundingClientRect();

  if (ubicacion.top < 784) {
    console.log('El elemento ya está visible');
  } else {
    console.log('Aún no, da más scroll');
  }
});
```

`getBoundingClientRect().top` es la distancia entre el borde superior del elemento y el borde superior de la ventana: cuando se vuelve negativa, el elemento entró en pantalla.

> [!tip] Rendimiento en `scroll`
> El evento `scroll` dispara a alta frecuencia. En proyectos grandes, limita su ejecución (*throttle*: ejecutar a intervalos fijos; *debounce*: ejecutar tras un silencio) o usa `IntersectionObserver`, que avisa cuando un elemento entra en pantalla sin recalcular en cada frame y **sin que el scroll dispare JS**. Ejecuta el handler con `{ passive: true }` si nunca llamas `preventDefault`.

## Delegación de eventos

En vez de poner un listener en cada hijo, se pone **uno solo en el padre** y se identifica el origen con `evt.target`. Es más eficiente y, sobre todo, **funciona con elementos que se crean dinámicamente**:

```js
const cardDiv = document.querySelector('.card');

cardDiv.addEventListener('click', (evt) => {
  console.log(evt.target);
  console.log(evt.target.classList);

  if (evt.target.classList.contains('titulo')) console.log('click en título');
  if (evt.target.classList.contains('precio')) console.log('click en precio');
  if (evt.target.classList.contains('categoria')) console.log('click en categoría');
});
```

Y la versión moderna con `closest`, que sobrevive a clicks en hijos *dentro* del hijo (p. ej. un `<strong>` dentro del título):

```js
cardDiv.addEventListener('click', (evt) => {
  const titulo = evt.target.closest('.titulo');
  const categoria = evt.target.closest('.categoria');
  if (titulo) console.log('click en título:', titulo.textContent);
  if (categoria) console.log('click en categoría:', categoria.textContent);
});
```

Beneficios de la delegación:

| Beneficio | Por qué |
| --------- | ------- |
| 1 listener vs N | Menos memoria y menos registros; escala con listas de miles de filas |
| Funciona con nodos nuevos | Los hijos creados después **ya** son atendidos — sin re-registrar |
| Lógica centralizada | Un solo lugar decide qué hacer con cada destino |

> [!warning] `closest` incluye al propio elemento
> `evt.target.closest('.x')` también matchea si el propio `target` tiene la clase. En la práctica es lo que quieres. Además, si `target` fue eliminado del DOM, `closest` puede fallar — validá existencia en listas mutantes.

## Eventos personalizados (`CustomEvent`)

Los eventos no son exclusivos del navegador: puedes **definir los tuyos** para comunicar capas de la aplicación (componente → componente, módulo → módulo):

```js
// emitir
const evt = new CustomEvent('producto:agregado', {
  detail: { id: 42, nombre: 'Monitor' },   // payload arbitrario
  bubbles: true,                            // ¿sube por el árbol?
  cancelable: true,                         // ¿admite preventDefault()?
});
elemento.dispatchEvent(evt);

// escuchar
elemento.addEventListener('producto:agregado', (e) => {
  console.log(e.detail.id);   // 42 — el payload vive en detail
});
```

| Campo | Significado |
| ----- | ----------- |
| `detail` | Cualquier dato (objeto, string...). En el listener se lee por `e.detail` |
| `bubbles` | Con `true` el evento sube — permite escuchar en ancestros o en `document` |
| `cancelable` | Con `true`, quien escucha puede llamar `e.preventDefault()` y el emisor puede chequear `e.defaultPrevented` |

Patrón de desacople: el emisor no sabe quién escucha (patrón observable). Usado por widgets legacy y por frameworks (el synthetic system de React es internamente un sistema de eventos propio).

> [!warning] CustomEvent no cruza orígenes (iframe, Web Worker)
> El evento vive en el mismo "mundo" (realm) JS. Para cruzar documentos usá `postMessage` — fuera del alcance de esta nota.

## Ciclo de vida de la página: `DOMContentLoaded` vs `load`

| Evento | Cuándo | Para qué |
| ------ | ------ | -------- |
| `DOMContentLoaded` (`DOMContentLoaded` en `document`) | El HTML fue **parseado**; el DOM existe (los `<script>` sin `defer` y los estilos críticos ya corrieron) | Seleccionar nodos, arrancar la app |
| `load` (en `window`) | El HTML **y todos** los recursos pesados (imágenes, CSS, fuentes) terminaron | Medir tiempos, precargar, lighthouse-style |
| `readystatechange` | Estados intermedios de `document.readyState` | Legacy |

```js
// patrón robusto: correr ya si el DOM está listo, o esperar si no
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', arrancar);
} else {
  arrancar(); // el DOM ya existe: el listener nunca dispararía
}
```

En [[Java Script]] se ve la relación con `defer`/`async` en el `<script>`: con `defer`, el script corre justo antes de `DOMContentLoaded`.

## Handlers como propiedades vs `addEventListener`

```js
// asignando handlers a elementos creados
const parrafo = document.createElement('p');
parrafo.textContent = '$800 por persona';
parrafo.classList.add('precio');

// Asignar la función SIN paréntesis: si escribes nuevaFuncion(), se ejecuta
// en el momento de asignar y el listener nunca se registra.
parrafo.onclick = nuevaFuncion;

// Para pasar argumentos, se envuelve en una función anónima/arrow
parrafo.onclick = () => nuevaFuncion(1);

function nuevaFuncion(id) {
  console.log(`nueva función con el id ${id}`);
}
```

> [!tip] `onclick` vs `addEventListener`
> `elemento.onclick = fn` solo admite **un** handler por evento (la última asignación pisa a la anterior; `onclick = null` limpia). `addEventListener` admite varios y permite opciones (`once`, `capture`, `signal`). Prefiere `addEventListener`; el `onclick` se ve sobre todo en HTML inline (`onclick="fn()"`), que mezcla marcado con lógica y solo admite handlers globales.

Ver [[Acceder a Html]] para `createElement`/`append` y el resto de manipulación del DOM.

## Memoria: fugas por listeners

Un listener mantiene **vivas** las referencias de su closure. La fuga clásica SPA: crear vistas con listeners y eliminar el nodo **sin** quitar los handlers — el nodo sale del DOM pero la referencia del listener lo mantiene en memoria:

```js
// ❌ FUGA: el nodo se va, el listener (y su closure) siguen vivos
function montarVista() {
  const btn = document.createElement('button');
  btn.addEventListener('click', () => procesar(datosGrandes)); // closure retiene datos
  document.body.append(btn);
}
function desmontar() { btn.remove(); } // ¡insuficiente!

// ✅ LIMPIO con removeEventListener (misma referencia)
function montar() { btn.addEventListener('click', onClick); }
function desmontar() { btn.removeEventListener('click', onClick); btn.remove(); }

// ✅ LIMPIO con AbortController (varios listeners de golpe)
function montar() {
  const ctrl = new AbortController();
  const { signal } = ctrl;
  btn.addEventListener('click', onClick, { signal });
  window.addEventListener('resize', onResize, { signal });
  return () => ctrl.abort(); // cleanup de una sola pieza
}
```

Señales de fuga: memoria que crece al navegar entre vistas y nunca baja. Diagnóstico: DevTools → pestaña **Memory** → tomar snapshot antes/después de navegar y comparar retenciones de `HTMLButtonElement`. En React, todo lo que se registra en `useEffect` debe limpiarse en su return — ver [[ReactJS - Efectos (useEffect)]].

## Errores comunes

1. **Ejecutar el handler en el momento de asignar**: `btn.onclick = mostrarMenu()` (con paréntesis) lo llama ya y asigna su resultado (`undefined`). Sin paréntesis se registra.
2. **`removeEventListener` con otra arrow function**: cada literal es un objeto distinto; nunca matchea. Extrae la función con nombre.
3. **Esperar `submit` en un `<button>` sin `type`**: dentro de un form, `<button>` es **submit por defecto** — un botón "Cancelar" dispara el envío. `type="button"` lo desactiva (ver [[Formularios]]).
4. **`focus`/`blur` que no burbujea**: para delegar validaciones usa `focusin`/`focusout`.
5. **`preventDefault` en evento no cancelable**: `click` y `submit` lo son; `load` o `error` de recursos, no — no tiene efecto.
6. **Confundir `target` con `currentTarget` en delegación**: `target` puede ser un hijo profundo; con `closest` lo resolvés.
7. **Listener infinito en `scroll` con `querySelector` dentro**: cada disparo recorre el DOM y fuerza layout. Cacheá el elemento y throttleá (o usa `IntersectionObserver`).
8. **Olvidar el cleanup**: registrar listeners cada vez que un componente se monta sin quitarlos al desmontar ⇒ duplicados (`dos alerts por click`) y fugas de memoria.

## Cheatsheet — catálogo rápido

| Tema | API clave |
| ---- | --------- |
| Registrar | `el.addEventListener(type, fn, { capture, once, passive, signal })` |
| Quitar | `el.removeEventListener(type, fn, { capture })` — misma referencia |
| Limpiar en bloque | `new AbortController()` + `signal` |
| Acción por defecto | `evt.preventDefault()` |
| Cortar recorrido | `evt.stopPropagation()` / `stopImmediatePropagation()` |
| Origen vs receptor | `evt.target` / `evt.currentTarget` |
| Payload | `evt.detail` (solo `CustomEvent`) |
| Ratón | `click`, `dblclick`, `mousedown/up`, `mouseenter/leave`, `mouseover/out` |
| Puntero (unificado) | `pointerdown/up/move/enter/leave`, `evt.pointerType` |
| Teclado | `keydown/keyup`, `evt.key` vs `evt.code`, modifiers |
| Formulario | `submit` (¡`preventDefault`!), `input`, `change`, `focusin/out`, `reset` |
| Ciclo de vida | `DOMContentLoaded` (DOM listo) / `load` (todo cargado) |
| Scroll | `scroll` (+`passive`), `IntersectionObserver` |
| Custom | `new CustomEvent(type, { detail, bubbles, cancelable })` |

## Relacionadas

- [[Document Object Model (DOM)]] — el árbol por el que viajan los eventos
- [[Acceder a Html]] — crear los elementos a los que se les agregan listeners
- [[Funciones]] — callbacks: la función que recibe `addEventListener`
- [[Formularios]] — validación nativa, `submit` y tipos de input
- [[ReactJS - Fundamentos]] — eventos sintéticos y delegación en React
- [[ReactJS - Efectos (useEffect)]] — registro/limpieza de listeners en componentes
- [[Qué es una API]] — qué ocurre cuando el evento dispara un `fetch`
