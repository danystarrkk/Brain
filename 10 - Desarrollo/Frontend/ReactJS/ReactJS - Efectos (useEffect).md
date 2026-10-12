---
aliases: ["useEffect", "Efectos React", "Hooks useEffect"]
tags:
  - dev/frontend/react
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[ReactJS - Fundamentos]]"
  - "[[ReactJS - Vite]]"
  - "[[Funciones]]"
  - "[[Qué es una API]]"
fuente: "https://react.dev/learn/synchronizing-with-effects"
---

# ReactJS - Efectos (useEffect)

`useEffect` es el hook para **sincronizar React con sistemas externos**: timers, peticiones a APIs, suscripciones al navegador, conexión a WebSockets, medición del DOM. Ejecuta código **fuera del render** y su semántica exacta (cuándo corre, cuándo limpia, qué ve cada closure) es la base para evitar los bugs más comunes de React.

> [!warning] ¿Qué NO es un efecto?
> No es un "callback post-render" genérico para lógica de negocio. Si un valor se puede **calcular** a partir de props/estado, va en el cuerpo del componente (o `useMemo`), no en un effect. Los efectos son para lo que React **no puede** calcular: llamar a otro sistema. Ver "Efectos derivados" al final.

## Anatomía

```jsx
import { useState, useEffect } from 'react';

function Contador() {
  const [valor, setValor] = useState(0);

  useEffect(() => {
    // Este bloque se ejecuta DESPUÉS del commit en el DOM
    const timer = setTimeout(() => {
      setValor((v) => v + 1);
    }, 3000);

    // Cleanup: se ejecuta antes del siguiente effect (y al desmontar)
    return () => clearTimeout(timer);
  }, []); // [] = dependencias: solo se ejecuta al montar

  return <div>Contador: {valor}</div>;
}
```

| Parte | Cuándo corre | Propósito |
|---|---|---|
| **Primer argumento** (el effect) | Después de que React pinta en el DOM (asíncrono, tras el commit) | Configurar la conexión al sistema externo |
| **Return** (cleanup) | **Antes** de correr el effect otra vez y **al desmontar** | Desconectar / cancelar / limpiar lo que se configuró |
| **Segundo argumento** (deps) | Define **cuándo** se re-evalúa | React compara deps con Object.is entre renders |

> [!info] Mental model: "conectar/desconectar"
> Cada effect es un par **setup + cleanup**. React garantiza que nunca hay dos setups vivos a la vez: antes del segundo setup, corrió el cleanup anterior. El cleanup recibe **exactamente los mismos valores** que su setup (misma closure), por eso `clearTimeout(timer)` funciona.

## Ciclo de vida exacto: orden verificado

```
Montaje:
  1. Render del componente (React calcula el árbol)
  2. Commit: React aplica cambios al DOM
  3. (asíncrono, después del paint) → EFFECT: setup()

Cambio de dependencias:
  1. Nuevo render
  2. React compara deps con Object.is
  3. Si cambiaron: CLEANUP del effect anterior (con la closure vieja)
  4. Commit del nuevo DOM
  5. EFFECT: setup() nuevo (con la closure nueva)

Desmontaje:
  1. CLEANUP del último effect vivo
  2. Nodo retirado del DOM
```

Demostración del orden con logs:

```jsx
function Ejemplo({ id }) {
  useEffect(() => {
    console.log(`setup  id=${id}`);
    return () => console.log(`cleanup id=${id}`);
  }, [id]);

  return <p>{id}</p>;
}
// Montaje:        setup  id=1
// id → 2:         cleanup id=1   ← la closure VIEJA, con id=1
//                 setup  id=2
// Desmontaje:     cleanup id=2
```

> [!warning] La app invisible tras el cleanup
> Después del cleanup y antes del nuevo setup, el "sistema externo" está desconectado. Nunca pongas lógica asincrónica en el cleanup que deba correr *junto con* el render — el cleanup solo deshace.

### Varios efectos: el orden de ejecución

Efectos **distintos** (distintos `useEffect`) corren **en el orden en que se declararon**, arriba→abajo, en cada ciclo — y sus cleanups también corren en orden de declaración (así lo fijó React 17; en versiones anteriores los cleanups de unmount corrían en orden errático):

```jsx
useEffect(() => { console.log('A'); return () => console.log('fin A'); }, []);
useEffect(() => { console.log('B'); return () => console.log('fin B'); }, []);
// mount:   A → B
// unmount: fin A → fin B   (mismo orden de declaración)
```

> [!tip] Si un efecto depende del resultado de otro, están mal segmentados
> Dos efectos que se comunican entre sí son señal de que deberían ser uno solo, o de que la lógica compartida no es un efecto sino estado/derivados.

## El arreglo de dependencias

| Dependencias | Cuándo corre el efecto |
|---|---|
| `[]` | Solo al montar el componente (una vez); cleanup al desmontar |
| `[valor]` | Al montar y **cada vez que `valor` cambia** (por identidad) |
| Sin arreglo (undefined) | En **cada** render, después de cada pintado (casi siempre un error) |
| Sin return | Setup sin cleanup — válido si no hay nada que limpiar |

```jsx
useEffect(() => {
  document.title = `Contador: ${valor}`;
}, [valor]); // reacciona a cambios de `valor`
```

### Mecánica de comparación: igualdad referencial

React compara cada elemento del arreglo con el del render anterior usando **`Object.is`** — igualdad **estricta por referencia**, no por contenido:

```js
Object.is(NaN, NaN)  // true  (especialidad de Object.is)
Object.is(0, -0)     // false
{} === {}            // false — objetos literales NUNCA son "iguales"
[1] === [1]          // false — ídem arreglos
fn === fn            // true si es la MISMA función
```

Consecuencia práctica — **los objetos y arreglos literales como deps rompen el effect**:

```jsx
// ❌ Crea un objeto NUEVO en cada render → el effect corre SIEMPRE
useEffect(() => { fetchDatos(filtros); }, [{ pagina: 1 }]);

// ✅ Pasar valores primitivos (ellos sí tienen identidad estable)
useEffect(() => { fetchDatos(pagina); }, [pagina]);

// ✅ O memoizar la identidad del objeto
const filtros = useMemo(() => ({ pagina: 1 }), []);
useEffect(() => { fetchDatos(filtros); }, [filtros]);
```

Por eso el linter (`eslint-plugin-react-hooks`) insiste en **dependencias completas**: toda variable externa usada dentro del effect debe estar en el arreglo — omitirla congela su valor en la closure del primer render (ver "Stale closures").

## Stale closures: el bug profundo de React

Una **closure vieja** (*stale closure*) es una función que "recuerda" los valores del scope del render en que fue creada. Toda función creada durante un render captura ese render. Si sobrevive (timer, event handler guardado, effect con deps incompletas), ve valores viejos **para siempre**.

```jsx
function Contador() {
  const [n, setN] = useState(0);

  useEffect(() => {
    const t = setInterval(() => {
      console.log(n);      // ❌ SIEMPRE imprime el n del render en que corrió el setup
      setN(n + 1);         // ❌ suma siempre al mismo n → se queda en 1
    }, 1000);
    return () => clearInterval(t);
  }, []); // la closure de n quedó congelada en 0

  return <p>{n}</p>;
}
```

Las tres curas, de menos a más:

| Fix | Cuándo usarlo | Cómo |
|---|---|---|
| **Update funcional** | El cálculo depende del estado anterior | `setN(v => v + 1)` — el setter siempre ve el valor fresco |
| **Deps completas** | El effect reacciona a un valor externo | Agregar el valor al arreglo (o rearmar el effect entero) |
| **Ref** | El valor debe ser "fresquísimo" sin rearmar el effect | `const nRef = useRef(n); nRef.current = n;` y leer `nRef.current` dentro |

```jsx
// ✅ Fix 1: updater — el timer vive, el valor es fresco
setInterval(() => setN(v => v + 1), 1000);

// ✅ Fix 3: ref como espejo mutable del valor actual
const nRef = useRef(0);
useEffect(() => { nRef.current = n; }, [n]);      // espejo
useEffect(() => {
  const t = setInterval(() => console.log(nRef.current), 1000);
  return () => clearInterval(t);
}, []);                                            // lee siempre el último
```

> [!info] Por qué existe useRef para esto
> `useRef` es un contenedor mutable **compartido entre renders** (misma referencia, contenido editable sin re-render). Es el escape hatch legal para leer el "ahora" desde closures viejas. Leer `.current` **durante** el render para derivar valores es anti-patrón (ver useRef como espejo en la doc oficial).

## Reglas de oro

- **Efectos lo mínimo posible**: todo lo que pueda calcularse en cada render va en el cuerpo del componente, no en un effect (ver "Efectos derivados").
- **Siempre limpiar** lo que tenga cleanup (`setTimeout`, `setInterval`, `addEventListener`, suscripciones, `AbortController`).
- **Deps completas**: si el effect usa una variable del scope, agrégala al arreglo — omitirla genera closures viejas y bugs difíciles de cazar.
- **Efectos asincrónicos entre sí no garantizan orden**: dos fetchs disparados en dos effects pueden responder en cualquier orden — por eso cada patrón asincrónico necesita su propia guarda (siguiente sección).
- **No sincronices estado con props/estado** en un effect (ver anti-patrón de derivados abajo).

## Patrón: obtener datos de una API

El problema real del fetch en un effect: **condición de carrera**. Si el usuario cambia el filtro rápido, la respuesta del request viejo puede llegar *después* de la nueva y sobrescribir los datos correctos. Dos defensas equivalentes:

### Opción A — bandera `activo` (clásica)

```jsx
function Usuarios({ busqueda }) {
  const [usuarios, setUsuarios] = useState([]);
  const [cargando, setCargando] = useState(true);

  useEffect(() => {
    let activo = true; // bandera para ignorar respuestas tardías

    fetch(`/api/usuarios?q=${encodeURIComponent(busqueda)}`)
      .then((res) => res.json())
      .then((datos) => {
        if (activo) {            // si ya hubo cleanup, se descarta
          setUsuarios(datos);
          setCargando(false);
        }
      })
      .catch(() => { if (activo) setCargando(false); });

    return () => { activo = false; };
  }, [busqueda]); // re-fetch cuando cambia el filtro

  if (cargando) return <p>Cargando....</p>;
  return <ul>{usuarios.map((u) => <li key={u.id}>{u.nombre}</li>)}</ul>;
}
```

### Opción B — `AbortController` (recomendada: cancela de verdad)

```jsx
useEffect(() => {
  const control = new AbortController();

  fetch(`/api/usuarios?q=${busqueda}`, { signal: control.signal })
    .then((res) => res.json())
    .then(setUsuarios)
    .catch((err) => {
      if (err.name !== 'AbortError') setError(err); // el aborto es esperado, no es error
    });

  return () => control.abort(); // cancela la conexión, no solo ignora la respuesta
}, [busqueda]);
```

La bandera evita el setState tardío; el `AbortController` además **libera la conexión** — en móvil con límite de peticiones, es la opción correcta. El renderizado condicional con `cargando` sigue el patrón de [[ReactJS - Fundamentos]], y el listado usa `key` única por elemento.

> [!tip] Fetch asincrónico → patrón "async inside, sync outside"
> No se puede marcar el callback de useEffect como `async` (retornaría una Promise y React perdería el cleanup). Patrón seguro: función interna `async` invocada y capturada:

```jsx
useEffect(() => {
  let activo = true;
  (async () => {
    const res = await fetch(url);
    const datos = await res.json();
    if (activo) setDatos(datos);
  })();
  return () => { activo = false; };
}, [url]);
```

### Y en producción: librerías de fetching

El boilerplate anterior (caché, reintentos, deduplicación, re-fetch on focus, estados de error) es tan repetido que existen librerías especializadas — **TanStack Query** (React Query), SWR, RTK Query. Manejan caché y condiciones de carrera fuera del effect:

```jsx
const { data, isLoading } = useQuery({
  queryKey: ['usuarios', busqueda],
  queryFn: ({ signal }) => fetch(`/api/usuarios?q=${busqueda}`, { signal }).then((r) => r.json()),
});
```

Regla: **fetch manual en useEffect** para aprendizaje y casos únicos; **librería de query** cuando la app depende de datos remotos con caché.

## Patrón: suscripciones

El effect es el lugar natural de las conexiones con **cualquier API con cleanup simétrico**:

```jsx
function useOnline() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const on = () => setOnline(true);
    const off = () => setOnline(false);
    window.addEventListener('online', on);
    window.addEventListener('offline', off);
    return () => {
      window.removeEventListener('online', on);
      window.removeEventListener('offline', off);
    };
  }, []);

  return online;
}
```

Mismo esquema para `resize`/`scroll` (con `passive: true`), WebSocket, `matchMedia`, geolocalización, Web Locks… — siempre: **suscribir en el setup, desuscribir en el cleanup**, con la misma referencia de handler (por eso se declara adentro del effect, no fuera).

## `useEffect` vs `useLayoutEffect`

| | `useEffect` | `useLayoutEffect` |
|---|---|---|
| Cuándo corre | **Después** del paint (asíncrono) | **Antes** del paint, después del commit (síncrono) |
| Causa parpadeo visible | Puede (si mide/muta el DOM) | No — corre antes de que el usuario vea |
| Uso correcto | 95% de los casos: fetch, subscriptions, timers | Medir el DOM y corregir antes del paint; animaciones con GSAP/Lenis; tooltips |
| En SSR | No advierte | **Advierte** (*does not work on the server*) — usar `useEffect` o dynamic import |

> [!warning] Regla de oro
> Si no sabés por qué necesitás `useLayoutEffect`, usá `useEffect`. Especialízalo solo cuando hayas visto el parpadeo/flash de layout que querés evitar.

## StrictMode y el doble efecto (React 18, solo desarrollo)

Con `<StrictMode>` activo (los templates de [[ReactJS - Vite]] lo traen), en **desarrollo** React invoca cada componente **dos veces** y ejecuta **setup → cleanup → setup** en el mount, a propósito:

```
render, render (segunda vez para detectar impurezas)
setup, cleanup, setup     ← en DEV únicamente
```

Objetivo: destapar efectos que **no sobreviven a un cleanup correcto** (fugas de memoria, suscripciones duplicadas). Efectos bien escritos (setup/cleanup simétricos) son idempotentes: montar, desmontar y volver a montar no deja residuo. **No** se desactiva el StrictMode "para arreglar" esto: se arregla el effect. En producción el doble mount no ocurre.

Síntoma clásico: el efecto dispara **dos fetches** en desarrollo — con `AbortController` el primero se cancela y no hay problema.

## Efectos derivados: el anti-patrón

Sincronizar estado que es **calculable** produce renders extra, bugs de consistencia y código redundante:

```jsx
// ❌ Anti-patrón: "derived state" en effect + estado duplicado
function Total({ precio, cantidad }) {
  const [total, setTotal] = useState(0);
  useEffect(() => { setTotal(precio * cantidad); }, [precio, cantidad]); // render extra
  return <p>{total}</p>;
}

// ✅ Calcular en el render (sin estado ni effect)
function Total({ precio, cantidad }) {
  const total = precio * cantidad; // se recalcula solo cuando el componente renderiza
  return <p>{total}</p>;
}

// ✅ Solo si el cálculo es caro, memoizar
const total = useMemo(() => precio * cantidad, [precio, cantidad]);
```

| Situación | Solución |
|---|---|
| Valor calculado de props/estado | Calcular en el render |
| Cálculo caro | `useMemo` |
| Resetear estado cuando cambian props | Llamar al setter **durante el render** (`setX(props.x)` cuando `props.x !== prevX`) o usar `key` |
| Sincronizar con sistema externo | **Effect** — el único caso legítimo |

La pregunta de control: *"si React pudiera re-ejecutar mi componente en cualquier momento, ¿importa?"* — si la respuesta es "solo quiero el valor derivado", no es un effect.

## Errores comunes (tabla de síntomas)

| Síntoma | Causa raíz | Fix |
|---|---|---|
| Bucle infinito de renders | Effect llama `setState` con un **objeto/arreglo nuevo** (`setObj({a})`) y las deps incluyen ese estado — nueva referencia cada vez | Pasar valores primitivos como deps; memoizar la identidad (`useMemo`) |
| Bucle infinito | Dos effects que se alimentan mutuamente (`setA` en effect de B, `setB` en effect de A) | Unificar o recalcular en render (derivados) |
| Estado viejo dentro de timer/fetch | Stale closure: deps incompletas o `[]` con valor mutable | Updater funcional, deps completas, o `useRef` |
| Doble fetch en desarrollo | StrictMode double-invoke | `AbortController` o bandera `activo`; es esperado |
| Respuesta vieja pisa a nueva | Condición de carrera al cambiar filtros rápido | `AbortController` / bandera activa (ver fetch pattern) |
| "Can't perform a React state update on an unmounted component" | `setState` post-desmontaje sin cleanup | Cleanup con abort/bandera |
| Límite de ejecución excedido (React DevTools) | Actualización en loop infinito | Revisar filas 1–4; casi siempre deps con objetos nuevos |
| Efecto se pierde al desmontar/montar rápido | Setup async que resuelve tarde | Guarda `activo` + cleanup temprano |

## Relacionadas

- [[ReactJS - Fundamentos]] — componentes, estado, props y el ciclo de render
- [[ReactJS - Vite]] — entorno de desarrollo (StrictMode, HMR)
- [[Funciones]] — closures y scope (por qué las dependencias importan)
- [[Qué es una API]] — el contexto de `fetch` y las peticiones HTTP
- [[Document Object Model (DOM)]] — el "sistema externo" que React toca vía efectos

## Cheatsheet

| Concepto | API | Regla |
|---|---|---|
| Efecto | `useEffect(() => { setup; return cleanup; }, [deps])` | Solo sincronización con sistemas externos |
| Mount | `[]` como deps | Setup una vez; cleanup al desmontar |
| Update reactivo | `[valor]` | `Object.is` por referencia — primitivos o memo |
| Sin deps | `useEffect(fn)` | Corre en cada render (casi siempre bug) |
| Fetch seguro | `AbortController` en cleanup | Cancela request viejo, evita race |
| Bandera activa | `let activo = true` | Alternativa ligera al abort |
| Timer seguro | updater `setN(v => v+1)` | Escapa a stale closures |
| Valor fresco en closure | `useRef` espejo | `.current` sin re-render |
| Antes del paint | `useLayoutEffect` | Solo medición/corrección visual |
| Dev doble ejecución | StrictMode | Arreglar el effect, no desactivar |
