---
tags:
  - programacion/react
aliases:
  - esenciales2-react
estado: 🟢 Terminado
---
# Renderizado condicional

Esto es muy utilizado cuando tenemos componentes que siguen cierta lógica, es decir queremos que suceda algo cuando se cumplan ciertas condiciones por ejemplo si falla la imagen en su carga queremos que se cargue una imagen genérica sobre esta, eso seria un ejemplo de como lo podemos condicionar, nosotros podemos hacer mediante `if`, `&&` y los operadores `? :`.

## Devolución de nada `null`

Cuando un componente regresa `null` no se va a colocar o a renderizar absolutamente nada. Un ejemplo sencillo y controlado seria el siguiente:

```jsx
const Navbar = () => {

// Condicional
  if (true) {
    return <div>Cargando....</div>
  }

  return (

    <div>
      <p>Eventos</p>
      <input placeholder="Busca tu evento favorito" />
    </div>

  )

}

export default Navbar;
```

Esto permite que como siempre es verdadero en este caso retorna el valor de `Cargando....` ese componente, veamos otro ejemplo para poder tener una condicional mucho mas interactiva, en este caso de la siguiente menara:

```jsx
import { useState } from 'react'
import Navbar from './components/Navbar'
import './App.css'

function App() {

  const [value, setValue] = useState(0);

  setTimeout(() => {

    setValue(value + 1);

  }, 3000);

  console.log(value);

  if (value <= 10) {
    return <div>Cargando....</div>
  }

  return (
    <>
      <Navbar />
    </>
  )
}

export default App

```

Como podemos observar estamos utilizando ahora si el render condicional podemos aplicarlo a la carga de una API a espera de un evento o diferentes cosas y elementos.

## Operadores ternarios 

Nosotros podemos usar operadores ternarios para disminuir la sintaxis del código, podemos ver un ejemplo de la siguiente manera:

```jsx
import { useState } from 'react'
import Navbar from './components/Navbar'
import './App.css'

function App() {

  const [value, setValue] = useState(0);

  setTimeout(() => {

    setValue(value + 1);

  }, 3000);

  console.log(value);

  return (
    <>
      {value <= 10 ? <div>Cargando....</div> : (
        <Navbar />
      )}
    </>
  )
}

export default App
```

Como podemos observar en este caso utilizamos operadores ternarios para decidir que se va a cargar dependiendo de alguna condición, en este caso tenemos la misma condición pero dentro de nuestro html, como vemos yo use los parenticis `()` para englobar el `else`, esto se usa mucho mas cuando tenemos mas de una linea o bloques de código completos, en este caso podíamos no usarlo y dejar todo en una misma linea.

```jsx
  return (
    <>
      {value <= 10 ? <div>Cargando....</div> : <Navbar />}
    </>
  )
```

## Operador lógico AND `&&` 

Este operador es mucho mas corto, su uso es:

```jsx
{name} {isBoolean && 'hola'}
```

donde `isBoolean` tiene que a fuerzas ser un operador booleano por lo tanto tenemos que tener esto muy encuenta.

Un ejemplo mas practico aplicado a nuestro código puede ser:

```jsx
  return (
    <>
      {value <= 10 && <div>Cargando....</div>}
      <Navbar />
    </>
  )

```

como podemos observar su uso es simple peor este a diferencia de los otros se mantiene con el resto de código, por lo tanto este sera mucho mas usado cuando queramos realizar interacciones con el usuario como hovers o clics sera mucho mas útil.

# Renderizado de listas

Cuando tenemos o trabajamos con información que viene de una API o tenemos dicha información dentro de un arreglo tenemos que iterar cada una de estas listas, para iterar estos elementos vamos a usar `map()` o `filter()` según sena el caso.

Un ejemplo de este proceso seria el siguiente:

 ```jsx
 // lista de iterables
const arrayOfNumbers = [1, 2, 3, 4, 5, 6, 7, 8];

function App() {
  // iteramos sobre los valores
  const items = arrayOfNumbers.map((item) => <li>{item}</li>);

  return (
    <>
      <Navbar />
      // se imprimen los elementos
      <ul>{items}</ul>
    </>
  )
}

 ```

como observamos esto es mucho mas claro y como podemos ver no es nada complicado.

Ahora si nosotros queremos hacerlo directamente en el HTML se puede y quedaría:
```jsx
  return (
    <>
      <Navbar />

      // se imprimen los elementos
      {arrayOfNumbers.map((item) => (
        <li>{item}</li>
      ))}
    </>
  )

```

como podemos observar es la misa sintaxis prácticamente.

Ahora tenemos algo en la consola un aviso que es el siguiente:

![[Pasted image 20260322094521.png]]

Este nos habla de un atributo key que tenemos que asignarle a cada valor de la lista y esto lo tenemos que hacer de la siguiente manera:

```jsx
const items = arrayOfNumbers.map((item) => <li key={`array-number-item-${item}`}>{item}</li>);
```

como podemos observar le estamos asignando un `key` y algo importante de los `key` esta en que cada elemento tiene que tener uno único por lo que no podemos usar algo simplemente genérico es por eso que especificamos y concatenamos el valor de `item` como lo vemos en el ejemplo y con esto solucionamos el problema.

Ahora que sucede si tenemos objetos, seria de una forma similar:

```jsx
// lista de iterables
const arrayOfPeople = [
  {
    "id": 1,
    "name": "Danna",
    "age": 7,
  },
  {
    "id": 2,
    "name": "Daniel",
    "age": 22,
  },
  {
    "id": 3,
    "name": "Dylan",
    "age": 11,
  },
  {
    "id": 4,
    "name": "Dome",
    "age": 11,
  },
]

const peapleList = arrayOfPeople.map((person) => (<li key={`array-peaple-item-${person.id}`}>{person.name}</li>));


function App() {

  return (
    <>
      <Navbar />
      // se imprimen los elementos
      <ul>{peapleList}</ul>
    </>
  )
}

export default App

```

Como vemos no es nada del diferente de lo que ya tenemos y comprendemos.



