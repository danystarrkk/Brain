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
fuente: "https://react.dev/learn"
---

# ReactJS - Fundamentos

**React** es una librería de JavaScript para construir interfaces basadas en **componentes**: piezas autónomas de UI que se componen entre sí. Esta nota cubre los conceptos esenciales: componentes, estado reactivo, propiedades y renderizado condicional/listas.

> [!info] Antes de empezar
> Se asume JavaScript moderno (arrow functions, desestructuración, módulos `import/export`). Ver [[Sintaxis Moderna]] y [[Funciones]] si necesitas repasar. Los proyectos se crean con [[ReactJS - Vite]].

## Componentes

Todo en React se construye mediante componentes. Un componente es **una función que retorna JSX**:

![[Pasted image 20260321214121.png]]

```jsx
const MyFirstComponent = () => {
  const saludo = 'Hola';
  return <div>{saludo}, este es mi primer componente</div>;
};

export default MyFirstComponent;
```

Puntos clave:

- El JSX se parece a HTML pero vive dentro de JavaScript: `{saludo}` interpola una expresión JS.
- `export default` hace al componente importable desde otros archivos.
- **Buena práctica**: cada archivo exporta un solo componente y el archivo se nombra con PascalCase (`MyFirstComponent.jsx`).

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

1. El valor actual del estado (el argumento, `0`, es el valor inicial).
2. Su **setter**: la función que actualiza el valor **y dispara el re-render** del componente.

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

Al llamar `setValor(valor + 1)`, React vuelve a ejecutar la función del componente con el nuevo valor y actualiza el DOM. El estado no se muta directamente (`valor = 5` no hace nada): **siempre se actualiza a través del setter**.

El estado puede ser de cualquier tipo de dato — números, strings, arreglos, objetos:

```jsx
const [nombre, setNombre] = useState('Daniel');
const [items, setItems] = useState([]);
```

> [!warning] No crees timers ni suscripciones en el cuerpo del render
> Un `setTimeout`/`setInterval` escrito directamente en el cuerpo del componente se re-crea en cada render, generando bucles de actualización. El lugar correcto para ese tipo de código es el hook `useEffect` — ver [[ReactJS - Efectos (useEffect)]].

## Propiedades (Props)

Las **props** permiten pasar datos de un componente padre a un hijo — son el canal de la **comunicación unidireccional** (los datos fluyen hacia abajo, nunca se modifican desde el hijo):

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

Como cualquier dato JS se puede enviar: strings, números, booleanos, arreglos, objetos e incluso funciones. Si el padre se re-renderiza con props nuevas, el hijo se re-renderiza con ellas — la actualización baja por todo el árbol de componentes.

## Inmutabilidad

React compara el estado anterior con el siguiente para saber si debe re-renderizar. Por eso el estado **nunca se muta directamente**: en lugar de modificar un arreglo u objeto existente, se crea uno nuevo con los cambios.

```jsx
// ❌ Mutación: React no detecta el cambio
setItems(items.push(nuevoItem));

// ✅ Nuevo arreglo: React re-renderiza
setItems([...items, nuevoItem]);
```

Esto se aplica especialmente con arreglos y objetos — ver [[Arreglos]] para las herramientas de copia/fusión (`spread`, `slice`, `map`).

## Renderizado condicional

Cuando un componente debe mostrar cosas distintas según una condición, hay tres vías:

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

Un componente que retorna `null` no renderiza **nada**.

### 2. Operador ternario `? :`

Útil cuando la condición está *dentro* del JSX:

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

Si el caso `else` ocupa varias líneas, se engloba en paréntesis o se extrae a una variable.

### 3. Operador lógico AND `&&`

Muestra algo **solo si la condición es verdadera** — la forma más corta:

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
> En JS, `0 && <p>hola</p>` retorna `0` (falsy), y React renderiza ese `0` en pantalla. Usá `&&` con booleanos o valores que nunca sean `0`/`''`, o convertí explícitamente: `{valor > 0 && <p>hola</p>}`.

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

### La propiedad `key`

React muestra una advertencia en consola si los elementos de la lista no tienen `key`:

![[Pasted image 20260322094521.png]]

`key` es un identificador **único** por elemento que le permite a React saber qué nodo cambió, se agregó o se eliminó sin re-renderizar la lista entera. Por eso **no sirve el índice del arreglo** ni un string genérico:

```jsx
// ❌ Problemas al reordenar/filtrar la lista
<li key={index}>

// ✅ Identificador estable del dato
<li key={`person-${person.id}`}>
```

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

## Relacionadas

- [[ReactJS - Efectos (useEffect)]] — timers, fetch y efectos secundarios
- [[ReactJS - Vite]] — crear y ejecutar proyectos React
- [[Funciones]] — arrow functions y scope
- [[Iteradores]] — `map()`, `filter()` y compañía
- [[Arreglos]] — manipulación de arreglos e inmutabilidad
- [[Document Object Model (DOM)]] — qué reemplaza React con el Virtual DOM
