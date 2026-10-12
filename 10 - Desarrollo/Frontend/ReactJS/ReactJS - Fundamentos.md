---
aliases: ["Esenciales", "Esenciales2", "React Fundamentos", "ReactJSX"]
tags:
  - dev/frontend/react
tipo: concepto
estado: terminado
creado: 2026-03-21
actualizado: 2026-10-11
relacionado:
  - "[[ReactJS - Efectos (useEffect)]]"
  - "[[ReactJS - Vite]]"
  - "[[Sintaxis Moderna]]"
  - "[[Funciones]]"
  - "[[Arreglos]]"
  - "[[Document Object Model (DOM)]]"
fuente: "https://react.dev/learn"
---

# ReactJS - Fundamentos

**React** es una librería de JavaScript para construir interfaces basadas en **componentes**: piezas autónomas de UI que se componen entre sí. Esta nota es la referencia completa del modelo: qué es un componente, cómo y cuándo React vuelve a pintar, cómo compila el JSX, qué son las `key` y por qué la reconciliación funciona como funciona.

> [!info] El paradigma en una frase
> React implementa **UI = f(estado)**: declarás qué debería verse para un estado dado, y React se encarga de calcular y aplicar el diff mínimo contra lo que ya hay en el [[Document Object Model (DOM)|DOM]]. Vos nunca editás el DOM a mano (como sí hacías con `document.getElementById` — ver [[Document Object Model (DOM)]]): describís el resultado y React lo sincroniza.

Se asume JavaScript moderno (arrow functions, desestructuración, módulos `import/export`). Ver [[Sintaxis Moderna]] y [[Funciones]] si necesitas repasar. Los proyectos se crean con [[ReactJS - Vite]].

## Componentes: funciones que devuelven UI

Un componente es **una función que retorna JSX**:

![[Pasted image 20260321214121.png]]

```jsx
const MyFirstComponent = () => {
  const saludo = 'Hola';
  return <div>{saludo}, este es mi primer componente</div>;
};

export default MyFirstComponent;
```

- El JSX se parece a HTML pero vive dentro de JavaScript: `{saludo}` interpola una expresión JS.
- `export default` hace al componente importable desde otros archivos.
- **Convención**: cada archivo exporta un solo componente, nombrado en PascalCase (`MyFirstComponent.jsx`). React usa esa misma convención para distinguir componentes de etiquetas HTML (`<div>` vs `<MyComponent>`).

Para usarlo en otro componente, se importa y se usa **como una etiqueta HTML**:

```jsx
import MyFirstComponent from './MyFirstComponent';

function App() {
  return (
    <>
      <MyFirstComponent />
    </>
  );
}

export default App;
```

> [!warning] El import debe coincidir con el nombre del archivo
> Si el archivo se llama `MyFirstComponent.jsx`, el path del import es `'./MyFirstComponent'` (sin extensión). Un typo acá (`./MyFifstComponent`) rompe la app con un error de módulo no encontrado.

### Componentes = funciones puras (el contrato mental)

El modelo mental correcto: **un componente debe comportarse como una función pura de sus props y su estado**. Dadas las mismas entradas, debe producir la misma salida, sin efectos visibles en el cuerpo del render:

```jsx
// ✅ Puro: calcula y retorna
function Saludo({ nombre }) {
  return <p>Hola {nombre}</p>;
}

// ❌ Impuro: muta el DOM directo en el cuerpo del render
function Saludo({ nombre }) {
  document.title = `Hola ${nombre}`;   // efecto secundario → va en useEffect
  return <p>Hola {nombre}</p>;
}
```

¿Por qué este contrato? Porque React puede re-ejecutar tu función **en cualquier momento y más de una vez** (StrictMode lo hace a propósito en desarrollo para destapar impurezas). Si tu componente tiene efectos secundarios en el cuerpo, esas repeticiones rompen cosas. Los efectos legítimos van en [[ReactJS - Efectos (useEffect)|useEffect]].

### Dos formas de definir componentes

| Forma | Ejemplo | Notas |
|---|---|---|
| Function declaration | `function Contador() { ... }` | Se "hoisted" (usable antes de declararse); la forma recomendada en la mayoría de casos |
| Arrow function asignada | `const Contador = () => { ... }` | Común; útil cuando el componente es el valor de un módulo |
| Class component (legado) | `class Contador extends React.Component` | Solo en código pre-hooks; usa `this.state`, `this.props`, `componentDidMount`… no aprenderlo salvo mantenimiento |

Los **class components** son historia: desde hooks (React 16.8, 2019) todo se escribe con function components.

## JSX: azúcar sintáctico sobre `createElement`

El navegador no entiende JSX. En tiempo de build, Babel (o el compilador React) lo transforma en llamadas a función:

```jsx
// Lo que escribís
const elemento = <h1 className="titulo">Hola {nombre}</h1>;

// Lo que el compilador genera (API clásica)
const elemento = React.createElement(
  'h1',
  { className: 'titulo' },
  'Hola ',
  nombre
);
```

Puntos clave de la traducción:

- `className` en vez de `class`: el resultado de `createElement` es un objeto JS, y `class` es palabra reservada del lenguaje.
- Los valores de las expresiones `{expr}` se convierten en **argumentos hijos**: React los trata como datos, no como código.
- Un objeto tipo `{"a": 1}` **no** se renderiza como JSON; React solo renderiza strings, números, arrays, booleans (`null`/`true`/`false` → nada) y elementos React.

En proyectos modernos el runtime es el **JSX Transform nuevo** (`react/jsx-runtime`): ya no hace falta `import React` en cada archivo — el compilador inyecta el `jsx()` automáticamente. Por eso los templates de [[ReactJS - Vite]] funcionan sin importar React.

```jsx
// Con el transform moderno, esto:
<div className="c" data-x={4}>hola</div>
// se compila (aprox.) a:
jsx('div', { className: 'c', 'data-x': 4, children: 'hola' })
```

> [!tip] JSX = expresiones, no statements
> Dentro de `{}` solo va una **expresión** (valor resultante): variables, ternarios, llamadas a funciones, `map()`. No podés poner `if`, `for` ni asignaciones. Si necesitás lógica, extraela a una variable o función antes del `return`.

## El ciclo de render: cuándo y cómo pinta React

"Re-render" no significa "reescribir el DOM". El ciclo tiene dos fases:

| Fase | Qué pasa | ¿Toca el DOM? |
|---|---|---|
| **Render** (ejecución del componente) | React ejecuta tu función con nuevas props/estado y obtiene un árbol de elementos (objetos JS) nuevo | **No** |
| **Reconciliation** (diff) | Compara el árbol nuevo contra el anterior, calcula el mínimo de cambios | **No** (es cálculo) |
| **Commit** | Aplica esos cambios mínimos al DOM real, en un solo lote sincrónico | **Sí** |

1. **Al montar**: React ejecuta el componente, construye el árbol y lo pinta por primera vez.
2. **Al actualizar** (cambio de estado con `setState`, cambio de props desde el padre, o re-render forzado): React vuelve a ejecutar la función con los nuevos valores, genera un árbol nuevo y lo reconcilia contra el anterior.
3. **Al desmontar**: React retira el nodo del DOM (y corre los cleanups de efectos — ver [[ReactJS - Efectos (useEffect)]]).

Dos consecuencias prácticas de este modelo:

- **El render no es "pintar"**: es *calcular qué debería verse*. Por eso escribir console.log en el cuerpo del componente muestra ejecuciones que no siempre coinciden con cambios visuales.
- **Renderizar es barato; commit es caro**: React optimiza agrupando (`batching`) los commits. Ver "Batching" más abajo.

### Reconciliación: el algoritmo de diff (por qué es O(n))

React compara árboles con una heurística de **igualdad por tipo y posición**:

1. **Tipos diferentes** → desmonta el árbol viejo completo y monta el nuevo (estado perdido). `<div>` → `<span>` destruye todo lo de adentro.
2. **Mismo tipo en la misma posición** → actualiza las props del nodo existente y desciende a los hijos. El estado del componente se preserva.
3. **Hijos**: aquí entra el juego de las `key` (sección más abajo).

| Situación | Qué hace React | Estado del componente |
|---|---|---|
| `<A/><B/>` → `<A/><C/>` | Reemplaza B por C | B desmontado, C montado con estado nuevo |
| `<Foo x=1/>` → `<Foo x=2/>` | Actualiza props de la misma instancia | **Conservado** |
| `<div><A/></div>` → `<section><A/></section>` | Desmonta todo el div, monta section | A pierde su estado |

> [!warning] Cambiar el tipo o la posición destruye estado
> Bug clásico: meter un componente dentro de un `div` condicional (o cambiarlo de lugar en el JSX) reinicia su estado porque React lo desmonta y vuelve a montar. Si necesitás conservar estado al re-envolver, extraé el wrapper a un componente que cambie de forma sin cambiar de tipo, o pasá el estado hacia arriba (*lifting state up*).

### Render y commit en el mismo "turno"

React **no** pinta medio render: cuando el estado cambia, ejecuta el render completo del componente y sus hijos hasta terminar, y recién ahí hace el commit. Nunca verás la pantalla en un estado intermedio.

## Estado y reactividad

La **reactividad** es lo que hace a React: cuando el usuario interactúa, el código reacciona y vuelve a mostrar la UI con los valores actualizados. Esto se logra con el **estado** (`state`), gestionado por el hook `useState`:

```jsx
import { useState } from 'react';

const Contador = () => {
  const [valor, setValor] = useState(0); // [valor actual, función para actualizarlo]

  return <div>Contador: {valor}</div>;
};
```

`useState` retorna un **arreglo de 2 posiciones**:

1. El valor actual del estado (el argumento, `0`, es el valor inicial — solo se usa en el **primer** render; renders siguientes ignoran ese argumento).
2. Su **setter**: la función que encola la actualización **y dispara el re-render** del componente.

### Actualizar el estado

```jsx
const Contador = () => {
  const [valor, setValor] = useState(0);

  return (
    <div>
      Contador: {valor}
      <button onClick={() => setValor(valor + 1)}>+1</button>
    </div>
  );
};
```

Al llamar `setValor(valor + 1)`, React vuelve a ejecutar la función del componente con el nuevo valor y actualiza el DOM. El estado **no** se muta directamente (`valor = 5` no hace nada, y `valor++` tampoco): **siempre se actualiza a través del setter**. El setter no modifica la variable — crea un nuevo estado y le pide a React un re-render.

El estado puede ser de cualquier tipo de dato — números, strings, arreglos, objetos:

```jsx
const [nombre, setNombre] = useState('Daniel');
const [items, setItems] = useState([]);
```

### Actualizaciones funcionales: leer el valor "más reciente"

Los setters aceptan una **función** como argumento, que recibe el estado más reciente y devuelve el siguiente. Es obligatoria cuando la nueva valor depende del anterior y pueden ocurrir varias llamadas en el mismo lote:

```jsx
// ❌ Con valor directo: ambas usan el MISMO `valor` del render actual → +1, no +2
<button onClick={() => { setValor(valor + 1); setValor(valor + 1); }}>+2</button>

// ✅ Función updater: React le pasa el estado más reciente en cadena
<button onClick={() => { setValor(v => v + 1); setValor(v => v + 1); }}>+2</button>
```

> [!tip] Regla práctica
> Si el nuevo valor se calcula **a partir del anterior**, usá la forma funcional (`setState(prev => ...)`). Es más segura, más legible en callbacks tardíos, y es el único modo correcto dentro de efectos y handlers asíncronos.

### Batching: React 18 agrupa actualizaciones (¡incluso asíncronas!)

Cuando ocurren varios `setState` "en el mismo evento" o en el mismo ciclo, React **no** renderiza por cada uno: agrupa (*batch*) todas las actualizaciones en **un solo re-render y un solo commit**. Ejemplo verificado:

```jsx
function Saludo() {
  const [texto, setTexto] = useState('');
  const [num, setNum] = useState(0);

  function manejarClick() {
    setTexto('hola');   // encolada
    setNum(1);          // encolada
    console.log(texto, num); // ← todavía 'hola vieja' y 0: el render aún no corrió
  }
  // React ejecuta UN render con ambas actualizaciones juntas
}
```

Desde **React 18**, el batching es **automático en todas partes**, incluidos timeouts, fetchs y handlers de eventos nativos:

| Escenario | React 17 | React 18+ |
|---|---|---|
| Varios `setState` en un evento de DOM | Agrupados | Agrupados |
| `setState` dentro de `setTimeout` | **Un render por cada uno** | Agrupados |
| `setState` después de un `await` | **Un render por cada uno** Agrupados |
| Promesas y `Promise.then` | Un render por llamada | Agrupados |

Si necesitás forzar un render intermedio (caso rarísimo, p. ej. mediciones), existe `flushSync`:

```jsx
import { flushSync } from 'react-dom';
flushSync(() => setA(1)); // commit inmediato de A antes de continuar
```

### Setters estables

La función del setter **tiene identidad estable entre renders** (no se re-crea). Por eso no necesita estar en arreglos de dependencias de efectos ni romper `memo`:

```jsx
const [n, setN] = useState(0);
console.log(setN === setN); // true, siempre
```

> [!warning] No crees timers ni suscripciones en el cuerpo del render
> Un `setTimeout`/`setInterval` escrito directamente en el cuerpo del componente se re-crea en cada render, generando bucles de actualización o fugas de memoria. El lugar correcto para ese tipo de código es el hook `useEffect` — ver [[ReactJS - Efectos (useEffect)]].

## Propiedades (Props)

Las **props** permiten pasar datos de un componente padre a un hijo — son el canal de la **comunicación unidireccional** (los datos fluyen hacia abajo; el hijo nunca las modifica):

```jsx
// Padre
import MyFirstComponent from './MyFirstComponent';

function App() {
  return <MyFirstComponent titulo="Eventos" cantidad={3} visible={true} />;
}
```

```jsx
// Hijo: reciben todas juntas como un objeto `props`
const MyFirstComponent = (props) => {
  return <h1>{props.titulo}</h1>;
};
```

O bien **desestructurándolas** (la forma más común):

```jsx
const MyFirstComponent = ({ titulo, cantidad }) => {
  return <h1>{titulo} ({cantidad})</h1>;
};
```

Como cualquier dato JS se puede enviar: strings, números, booleanos, arreglos, objetos e incluso funciones (los *callbacks*: el hijo invoca la función, el padre decide qué hace — el evento se "propaga hacia arriba" sin que el hijo sepa del estado):

```jsx
function Padre() {
  const [n, setN] = useState(0);
  return <Hijo onIncrementar={() => setN(n + 1)} />;
}
```

- **Props children**: todo lo que se escriba *dentro* de las etiquetas del componente llega en `props.children` (ver "Composición" más abajo).
- Si el padre se re-renderiza con props **nuevas (distintas)**, el hijo se re-renderiza; si las props son idénticas, puede no hacerlo si está memoizado (ver Rendimiento).

> [!warning] Las props son de solo lectura
> Modificar `props` dentro del hijo (`props.titulo = 'x'`) lanza error de React o produce bugs silenciosos. Si el hijo necesita "cambiar" un dato, el padre debe pasarle un callback.

## Inmutabilidad: por qué React necesita comparar por referencia

React compara el estado anterior con el siguiente para saber si debe re-renderizar. Por eso el estado **nunca se muta directamente**: en lugar de modificar un arreglo u objeto existente, se crea uno nuevo con los cambios.

```jsx
// ❌ Mutación: React no detecta el cambio (misma referencia)
setItems(items.push(nuevoItem));   // push además retorna la LONGITUD, no el arreglo
setItems(items);                   // misma referencia: no re-renderiza

// ✅ Nuevo arreglo: React re-renderiza
setItems([...items, nuevoItem]);
```

Verificado con node:

```js
const arr = [1, 2];
arr.push(3);        // → 3 (¡retorna el largo!)
const copia = [...arr, 4];  // arr intacto: [1,2,3], copia: [1,2,3,4]
```

Operaciones inmutables habituales (ver [[Arreglos]] para el detalle de cada una):

| Quiero... | Mutación (❌) | Inmutable (✅) |
|---|---|---|
| Agregar | `arr.push(x)` | `[...arr, x]` |
| Eliminar por índice | `arr.splice(i, 1)` | `arr.filter((_, k) => k !== i)` |
| Actualizar por índice | `arr[i] = x` | `arr.map((it, k) => k === i ? x : it)` |
| Ordenar | `arr.sort()` | `[...arr].sort()` (¡`sort` muta!) |
| Modificar objeto | `obj.x = 1` | `{ ...obj, x: 1 }` |

La regla rige también para props: nunca mutar un objeto recibido, crear uno nuevo con cambios.

## Renderizado condicional

Cuando un componente debe mostrar cosas distintas según una condición, hay varias vías — cada una con su nicho:

| Patrón | Sintaxis | Cuándo usarlo |
|---|---|---|
| `if` clásico | `if (c) return <A/>; return <B/>;` | Ramas completas, tempranas; el más legible para UIs muy distintas |
| Ternario | `c ? <A/> : <B/>` | Dos alternativas **dentro** del JSX |
| AND `&&` | `c && <A/>` | Mostrar algo **solo si** la condición es verdadera (sin else) |
| Variable | `const el = c ? <A/> : <B/>;` | Cuando la rama es grande y ensucia el JSX |
| Early return + hooks | hooks siempre arriba, `return` condicional abajo | Componentes con estado; respeta las reglas de hooks |

### 1. `if` clásico

```jsx
const Navbar = ({ cargando }) => {
  if (cargando) {
    return <div>Cargando....</div>;
  }

  return (
    <div>
      <p>Eventos</p>
      <input placeholder="Busca tu evento favorito" />
    </div>
  );
};
```

Un componente que retorna `null` no renderiza **nada** (React pone un comentario en su lugar). Retorna `false` tiene el mismo efecto.

### 2. Operador ternario `? :`

```jsx
function App() {
  const [valor, setValor] = useState(0);

  return (
    <>
      {valor <= 10 ? <div>Cargando....</div> : <Navbar />}
    </>
  );
}
```

Si una rama ocupa varias líneas, se engloba en paréntesis o se extrae a una variable.

### 3. Operador lógico AND `&&`

```jsx
function App() {
  const [valor, setValor] = useState(0);

  return (
    <>
      {valor <= 10 && <div>Cargando....</div>}
      <Navbar />
    </>
  );
}
```

> [!warning] Cuidado con `&&` y números
> En JS, `0 && <p>hola</p>` retorna `0` (falsy) y React **renderiza ese 0 en pantalla**. Verificado: `0 && 'hola'` → `0`. Usá `&&` con booleanos o convertí explícitamente: `{valor > 0 && <p>hola</p>}`. Es el bug visual más común de `&&`.

## Renderizado de listas

Los datos que vienen de una API o de un arreglo se renderizan con **`map()`** (o `filter()` para filtrar) — ver [[Iteradores]] para el detalle de estos métodos:

```jsx
const numbers = [1, 2, 3, 4, 5, 6, 7, 8];

function App() {
  // Opción 1: extraer a una variable
  const items = numbers.map((n) => <li key={`item-${n}`}>{n}</li>);

  return <ul>{items}</ul>;
}

// Opción 2: directo en el JSX (misma sintaxis)
function App() {
  return (
    <ul>
      {numbers.map((n) => (
        <li key={`item-${n}`}>{n}</li>
      ))}
    </ul>
  );
}
```

`map` devuelve un **nuevo arreglo** de elementos React; React los renderiza en orden. `forEach` no sirve (retorna `undefined`), y las listas necesitan un nodo raíz común o fragmento (ver Composition).

### La propiedad `key`

React muestra una advertencia en consola si los elementos de la lista no tienen `key`:

![[Pasted image 20260322094521.png]]

`key` es un identificador **único y estable** por elemento que le permite a React emparejar el nodo viejo con el nuevo **por identidad**, no por posición. Sin keys estables, React usa la posición y todo se recalcula mal.

### Keys con índice: por qué rompen (ejemplo trabajado)

```jsx
// ❌ key = índice
lista.map((item, i) => <Fila key={i} texto={item.texto} />);
```

Estado inicial: `[{texto:'Ana'}, {texto:'Beto'}, {texto:'Carla'}]` → keys `0,1,2`.
Ahora **eliminás el primero** (Ana). Quedan `[{texto:'Beto'}, {texto:'Carla'}]` con keys `0,1`.

¿Qué empareja React? **Posición con posición**:

| Posición | Key | Antes | Después | React hace |
|---|---|---|---|---|
| 0 | 0 | Ana | Beto | Actualiza el nodo (reusa instancia) |
| 1 | 1 | Beto | Carla | Actualiza el nodo |
| 2 | 2 | Carla | — | Elimina |

Si cada `Fila` tuviera estado interno (un input, un checkbox, animaciones), **ese estado se quedaría pegado a la posición**: el checkbox marcado en la fila 0 seguiría marcado, mostrando a Beto con el estado de Ana. Con `key={item.id}` los nodos se emparejan por identidad: desaparece la fila de Ana (y su estado) y Beto conserva la suya.

| Tipo de key | Estabilidad | Riesgo | Veredicto |
|---|---|---|---|
| `item.id` (del backend) | ✅ Estable | Ninguno | **La correcta** cuando existe id |
| `item.id` + prefijo | ✅ Estable | Ninguno | Útil para depurar (`user-12`) |
| Índice del map | ❌ Cambia con el orden | Estado pegado a posición; bugs al reordenar/filtrar/insertar | Solo si la lista **nunca** cambia de orden ni se filtra |

> [!warning] Nada de claves genéricas arbitrarias
> `key={Math.random()}` o `key={Date.now()}` son **peores** que el índice: cada render genera keys nuevas, así que React desmonta y remonta **toda la lista en cada render** (pérdida total de estado y rendimiento). La key debe ser un dato **del item**, estable en el tiempo.

### Listas de objetos

```jsx
const people = [
  { id: 1, name: 'Danna', age: 7 },
  { id: 2, name: 'Daniel', age: 22 },
  { id: 3, name: 'Dylan', age: 11 },
];

function App() {
  return (
    <ul>
      {people.map((person) => (
        <li key={`person-${person.id}`}>
          {person.name} ({person.age})
        </li>
      ))}
    </ul>
  );
}
```

Cuando no hay id: generá una key determinista a partir del contenido (p. ej. concatenar campos únicos) — nunca aleatoria.

## Composición: `children` y patrones de slots

React no tiene "huecos" como Vue/Angular: el mecanismo de slots es **`props.children`**, el JSX escrito entre las etiquetas del componente. Es la forma estándar de construir layouts genéricos:

```jsx
// Componente "contenedor" que no sabe qué renderiza, solo cómo lo enmarca
function Panel({ titulo, children }) {
  return (
    <section className="panel">
      <h2>{titulo}</h2>
      <div className="panel__body">{children}</div>
    </section>
  );
}

// Uso: el contenido vive en children
<Panel titulo="Resumen">
  <p>Cualquier JSX va aquí</p>
  <Boton onClick={guardar} />
</Panel>
```

Patrones de composición reales:

| Patrón | Forma | Ejemplo típico |
|---|---|---|
| **Children** | `props.children` | Layouts, wrappers, modales, tarjetas |
| **Slots con nombre** | Props que reciben JSX (`header`, `footer`) | `<Page header={<Nav/>} footer={<Footer/>}>` — un objeto de slots |
| **Render props** | Prop función que retorna JSX | `<Lista render={(item) => <Fila {...item}/>} />` |
| **Compound components** | Componentes hermanos que comparten contexto | `<Select><Select.Option/></Select>` (tabs, accordion) |

> [!tip] Composición vs props-datos
> Cuando un componente empieza a recibir 5+ props que solo sirven para "personalizar la forma", casi siempre conviene pasar **JSB en children** en vez de más props: el padre decide el contenido, el hijo decide el marco. Menos acoplamiento, menos API.

### Regla de oro: `children` no necesita keys

`props.children` escrito a mano es una estructura estática (React conoce los elementos y sus posiciones): no necesita `key`. Las keys solo son obligatorias en **listas generadas dinámicamente** con `map`/`filter`.

## Cuándo extraer un componente

Señales de que un bloque debe convertirse en componente propio:

1. **Se usa en varios lugares** (2+ instancias del mismo patrón visual).
2. **El JSX del render supera ~50–80 líneas** o tiene ramas condicionales anidadas: extraer da aire y reduce anidamiento.
3. **Una parte tiene lógica propia** (estado, efectos) que no afecta al resto: aislarla evita re-renders de todo el árbol.
4. **Se repite una combinación de estilos + comportamiento** (un botón con loading, una fila editable).

Y cuándo **no** extraer prematuremente: una pequeña duplicación es mejor que una abstracción con 6 flags. Extraé cuando el patrón ya existe y es estable, no cuando imaginás que existirá.

## Reglas de los hooks

Los hooks (`useState`, `useEffect`, `useMemo`…) tienen dos reglas inviolables:

1. **Solo se llaman en el nivel superior** del componente — nunca dentro de `if`, loops, ni funciones internas.
2. **Siempre en el mismo orden** en cada render — React los registra en un array posicional interno; un hook condicional desalinea todo el resto.

La razón es interna: React guarda el estado de los hooks en una **lista ligada posicional**. `if (cond) return null;` **antes** de los hooks haría que el render siguiente, sin condición, lea el estado equivocado. Por eso el patrón correcto es: hooks arriba, early returns abajo:

```jsx
function Perfil({ usuario }) {
  const [datos, setDatos] = useState(null);
  useEffect(() => { /* fetch */ }, [usuario]);

  if (!usuario) return null;   // early return DESPUÉS de todos los hooks ✅
  return <div>{datos?.nombre}</div>;
}
```

La semántica completa de `useEffect` — su ciclo, dependencias y cleanup — vive en [[ReactJS - Efectos (useEffect)]].

## Rendimiento: cuándo preocuparse (y cuándo no)

La regla inicial: **React ya es rápido; medí antes de optimizar**. En la práctica, los mecanismos que necesitás conocer:

| Herramienta | Qué hace | Cuándo |
|---|---|---|
| `React.memo` | Saltar el re-render del hijo si sus props son referencialmente iguales | Hijo costoso que recibe props estables |
| `useMemo` | Memoizar el **valor** calculado entre renders | Cálculos caros (sort/filter grande, objetos de deps) |
| `useCallback` | Memoizar la **identidad** de una función | Cuando la función se pasa a un hijo memoizado y cambia cada render |
| Claves estables | Evitar remontajes de listas | Siempre (ver `key`) |
| Estado local | Bajar el estado al componente que lo usa | Evita re-renders de padres completos |

Los tres primeros son detalles de implementación de [[ReactJS - Efectos (useEffect)]] o de notas de rendimiento; el punto de entrada mental es: *el re-render de un componente hijo no depende de si cambió algo arriba, sino de si cambió algo que **él** usa*.

```jsx
// Ejemplo mínimo de memo + useCallback
const Lista = React.memo(function Lista({ items, onSelect }) {
  return items.map((it) => <button key={it.id} onClick={() => onSelect(it.id)}>{it.nombre}</button>);
});

function App() {
  const [n, setN] = useState(0);
  const seleccionar = useCallback((id) => console.log(id), []); // identidad estable
  return <Lista items={datos} onSelect={seleccionar} />;        // no re-renderiza por n
}
```

## Límites del modelo: error boundaries

Si un componente lanza excepción durante render, **React 18 desmonta toda la app** (en versiones viejas, toda la raíz). El mecanismo de contención son los **error boundaries**: componentes de clase con `componentDidCatch` que muestran un fallback en lugar de romper el árbol:

```jsx
class ErrorBoundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) { return { error }; }
  componentDidCatch(error, info) { /* log a servicio de errores */ }
  render() {
    return this.state.error
      ? <p>Algo salió mal: {this.state.error.message}</p>
      : this.props.children;
  }
}
```

No existe hook equivalente todavía; patrón actual: wrapper `<ErrorBoundary>` alrededor de secciones sensibles (rutas, widgets).

## React DevTools

La extensión **React DevTools** (Chrome/Firefox) es el complemento del DOM inspector: muestra el **árbol de componentes** (no el DOM), props, estado (incluidos hooks) y permite **inspeccionar el render** (qué componentes re-renderizaron y por qué, con la pestaña "Profiler"). Indispensable para depurar re-renders y keys duplicadas. Se instala desde la tienda de extensiones del navegador; en dev apunta a la app de [[ReactJS - Vite]] en `localhost:5173`.

## Errores comunes

1. **Mutar estado directamente** (`items.push(x)` + `setItems(items)`) → no re-renderiza (misma referencia) o comportamiento erróneo. Copia siempre.
2. **`&&` con números**: `{count && <p>...}</p>` muestra `0` cuando count es 0. Comparar con `> 0` o usar ternario.
3. **Keys aleatorias o con índice** en listas mutables → estado "salta" de fila. Usar id estable.
4. **Hooks después de un `return` condicional** → React lanza *"Rendered more hooks than during the previous render"*. Early returns solo después de todos los hooks.
5. **Pensar que props son reactivas dentro de closures tardías**: un callback viejo (timer, evento) captura las props del render en que se creó. Ver stale closures en [[ReactJS - Efectos (useEffect)]].
6. **Cambiar el tipo/posición de un componente** en el JSX (envolverlo condicionalmente) → pierde su estado. Mismo tipo + misma posición preserva la instancia.
7. **Crear objetos/arreglos inline como props sin memo** → el hijo memoizado re-renderiza igual (nueva referencia cada render).

## Cheatsheet

| Concepto | API | Nota clave |
|---|---|---|
| Componente | `function C() { return <JSX/> }` | Función pura de props+estado; PascalCase |
| Estado | `const [v, setV] = useState(inicial)` | Setter encola actualización; inicial solo en 1er render |
| Actualización funcional | `setV(prev => f(prev))` | Obligatoria cuando el nuevo depende del anterior |
| Batch (React 18+) | Automático | Incluso en timeouts/promesas; un render por lote |
| Props | `<C dato={x} cb={fn} />` | Unidireccionales; children = JSX interno |
| Inmutabilidad | `[...arr, x]`, `{...obj, k: v}` | Nunca mutar estado ni props |
| Condicional | `if` / ternario / `&&` / variable | `&&` con 0 renderiza "0" |
| Lista | `arr.map(item => <F key={id} .../>)` | key = id estable, nunca índice/aleatoria |
| Composición | `props.children` | El "slot" de React; no necesita key |
| Hooks | Nivel superior, orden fijo | Early return solo después de los hooks |
| Memoria | `memo` / `useMemo` / `useCallback` | Medir antes de aplicar |

## Compatibilidad

- React 18/19 (2026): batching automático, `createRoot`, StrictMode con doble invocación en dev (ver [[ReactJS - Efectos (useEffect)]]).
- Class components y `ReactDOM.render` (legacy root) siguen compilando pero se consideran legado.
- El JSX runtime nuevo elimina el `import React` obligatorio; requiere Babel plugin o bundler moderno ([[ReactJS - Vite]] lo trae por defecto).

## Relacionadas

- [[ReactJS - Efectos (useEffect)]] — timers, fetch, suscripciones y el ciclo de efectos completo
- [[ReactJS - Vite]] — crear, ejecutar y construir proyectos React
- [[Funciones]] — arrow functions, closures y scope (por qué "capturan" el render)
- [[Iteradores]] — `map()`, `filter()` y compañía para listas
- [[Arreglos]] — manipulación inmutable de arreglos
- [[Objetos]] — spread y actualización inmutable de objetos
- [[Document Object Model (DOM)]] — qué reemplaza React con su árbol virtual
- [[Eventos]] — el manejo de eventos del navegador que React sintetiza
- [[Sintaxis Moderna]] — destructuring, spread y syntax sugar del JSX
