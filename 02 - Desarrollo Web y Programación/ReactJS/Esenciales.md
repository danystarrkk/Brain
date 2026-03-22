---
tags:
  - programacion/react
aliases:
  - ReactJSX
estado: 🟢 Terminado
---
## Componentes

Todo en React se construye mediante componentes, su estructura usual es la siguiente:

![[Pasted image 20260321214121.png]]

Como observamos es simple y el archivo principal sera el de `app.jsx`, como se dijo se trabaja por componentes por lo que creamos uno nuevo que ya vemos en la captura llamado `MyFirstComponent.jsx` donde su contenido no son mas que que funciones flecha o funciones tradicionales que retornan código de JSX simple:

```jsx
const MyFirstComponent = () => {

const hola = 'Hola';
return <div>{hola}, este es mi primer componente</div>;
};

export default MyFirstComponent;
```

Como podemos observar no es nada del otro mundo pero tiene su ciencia recordando que que tenemos a comparación de JavaScript una sintaxis algo diferente. Una buena practica es retornar solo un componente

Ahora para a este nuevo componente importarlo no es mas que hacer lo siguiente:
```jsx
import MyFirstComponent from './MyFifstComponent';
function App() {
const [count, setCount] = useState(0);
return (
<>
<MyFirstComponent />
</>
);

}
export default App;
```

Esa es la sintaxis de nuestro archivo principal y como vemos los componentes los importamos en realidad como etiquetas.

## Reactividad

La reactividad es una de las capacidades que hace de react una de las mejores librerías con las que se puede trabajar y es que sus componentes serán reactivos debido a que cuando un usuario comience a interactuar con los componente del mismo el código reaccione a este y muestre al usuario lo que necesite.

Entonces es la habilidad de poder llamar a los componentes nuevamente con valores nuevos o solicitados dependiendo de la interacción o solicitud de los usuarios.

Entonces un ejemplo de como podemos crear estos componentes reactivos es de la siguiente manera:

```jsx
import { useState } from 'react'; // importación para funciones reactivas 

const MyFirstComponent = () => {

const [value, setValue] = useState(0); // el como usar dicha función

const hola = 'holaf';

return <div>{hola}, este es mi primer componente {value}</div>;

};

export default MyFirstComponent;
```

podemos observar la importación que sera el pan de cada día cuando trabajemos con react.

Observamos su importación no es nada del otro mundo, pero al implementarla lo que hace es retornar un arreglo de 2 posiciones donde primero obtenemos su valor y en la siguiente posición su `setter` para actualizarse, ahora el valor que enviamos a la función en este caso el `0` es por decirlo de alguna forma su valor por defecto y sera lo que se presente cuando enviamos a `value`.

Ahora como hacemos para un nuevo valor si queremos volver al componente reactivo, vamos a hacerlo simple con un `tiemeout`:

```jsx
import { useState } from 'react'; // importación para funciones reactivas 

const MyFirstComponent = () => {

const [value, setValue] = useState(0); // el como usar dicha función

const hola = 'holaf';

setTimeout(() => {
	setValue(value + 1);
}, 3000)

return <div>{hola}, este es mi primer componente {value}</div>;

};

export default MyFirstComponent;
```

como podemos observar estamos alterando un valor de forma continua y la lo que hace la web es render de forma continua con el objetivo de mover en este caso el contador sin ningún problema, ahora si lo hacemos con Strings no es ningun problema:

```jsx
import { useState } from 'react'; // importación para funciones reactivas 

const MyFirstComponent = () => {

const [value, setValue] = useState(0); // el como usar dicha función

const [hola, setHola] = useState("hola");

setTimeout(() => {
	setValue(adios);
}, 3000)

return <div>{hola}, este es mi primer componente {value}</div>;

};

export default MyFirstComponent;
```

Como podemos observar podemos pasarle la mayoría de tipos de datos incluso objetos y no tendremos problemas.

## Inmutabilidad

Bueno el principio de inmutabilidad y el como lo aplica react es básicamente crear a partir de un objeto sus clases y atributos otro pero con ciertos componentes diferentes sin manipular el objeto original y es que en JavaScript esto se debe comprobar de forma muy detallada pero en react viene implementado por detrás sin que nosotros implementemos mucho código.

## Propiedades (Props)

Nosotros en react podemos implementar `ClassName`e `id` los cuales podemos enviar como propiedades a los componentes de la siguiente manera:

```jsx
import MyFirstComponent from './MyFirstComponent';

function App() {
const [count, setCount] = useState(0);

return (
<>
<MyFirstComponent propOne="1" propTwo={2} propThree={{}} />
</>
);
}

export default App;
```

como observamos a nuestro componente `MyFirstComponent` le enviamos 3 propiedades y el como las recibimos es sencillo de igual forma:

```jsx
import { useState } from 'react'; // importación para funciones reactivas

const MyFirstComponent = (props) => {
const [value, setValue] = useState(1); // el como usar dicha función]
const hola = 'holaf';
  

console.log(props);// imprimos y es un objeto

setTimeout(() => {
setValue(value + 1);
}, 3000);


return (

<div>
{hola}, este es mi primer componente {value}
</div>
);};

  
export default MyFirstComponent;
```

Como observamos podemos recibirlos a todos como un objeto y listo o podríamos hacerlo por separado también:

```jsx
const MyFirstComponent = ({prop1, prop2 ,prop3}) => {

}
```

Como vemos es sencillo y podemos enviar cualquier tipo de datos y el componente las actualiza.

Pero como es que las actualiza y que sucede si el componente principal se actualiza?.

Bueno el como lo actualiza es sencillo y ya vimos es mediante la reactividad, pero ya hablando de que pasa si se actualiza el componente principal lo hace toda su descendencia también por lo tanto no es complicado de entender.