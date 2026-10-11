---
aliases: ["Functions", "Funciones en JavaScript"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Estructuras de Control]]"
  - "[[Iteradores]]"
  - "[[Sintaxis Moderna]]"
  - "[[Arreglos]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Functions"
---

# Funciones

Una **función** es un bloque de código con nombre (o asignado a una variable) que se ejecuta **cuando se le llama**, no cuando se define. Sirven para dividir el programa en piezas pequeñas, **reutilizables** y con una sola responsabilidad.

```js
function sumar() {
  console.log(2 + 2);
}

sumar(); // llamada a la función: aquí se ejecuta
```

## Declaración vs expresión

Hay dos formas de definir una función, y se comportan distinto frente al **hoisting** (el "izado" que hace JS al preparar el scope):

```js
// Function Declaration: puede llamarse ANTES de definirse
sumar(); // funciona

function sumar() {
  console.log(2 + 2);
}

// Function Expression: guardada en una variable, NO puede usarse antes
sumar2(); // ReferenceError: no se puede usar antes de su inicialización

const sumar2 = function () {
  console.log(3 + 3);
};
```

| Estilo | ¿Hoisting? | Cuándo usarlo |
| ------ | ---------- | ------------- |
| `function nombre() {}` | Sí (la función completa) | funciones nombradas, métodos, helpers |
| `const nombre = function () {}` | No (solo la variable, en TDZ) | callbacks, asignaciones dinámicas |
| `const nombre = () => {}` | No | la forma preferida en código moderno |

> [!info] Funciones vs métodos
> `parseInt("20")` es una **función** global: se llama suelta. `"20".toString()` es un **método**: pertenece a un objeto y se llama con punto. Misma idea, distinta forma de invocación.

## Parámetros y argumentos

Los **parámetros** son los nombres de la entrada; los **argumentos**, los valores concretos que se pasan al llamar:

```js
function sumar(a, b) {
  console.log(b + a);
}

sumar(2, 4);          // 6
sumar(4534523, 23454325); // reutilizable con otros valores
```

Ejemplo práctico con `prompt` y `split`:

```js
function saludar(nombre, apellido) {
  console.log(`Hola ${nombre} ${apellido}, es un gusto saludarte`);
}

const nombreCompleto = prompt('Digita tu nombre y tu apellido');
const partes = nombreCompleto.split(' '); // separa por espacio -> arreglo

saludar(partes[0], partes[1]);
```

### Parámetros por defecto

Evitan errores cuando falta una entrada: si el argumento no llega (o es `undefined`), se usa el valor por defecto:

```js
function saludar(nombre = 'Invitado', apellido = '') {
  console.log(`Hola ${nombre} ${apellido}, es un gusto saludarte`);
}

saludar('Daniel');        // "Hola Daniel , es un gusto saludarte"
saludar();                // "Hola Invitado , es un gusto saludarte"
saludar('Daniel', 'Pérez');
```

## Comunicación entre funciones

Una función puede llamar a otras: así se arma el flujo del programa desde un punto de entrada:

```js
iniciarApp();

function iniciarApp() {
  console.log('Iniciando APP');
  segundaFuncion();
}

function segundaFuncion() {
  console.log('Hola desde la segunda función');
  login('Daniel');
}

function login(nombre) {
  console.log('Autenticando usuario.....');
  console.log(`Usuario ${nombre} correctamente autenticado`);
}
```

## Retorno de valores

`return` termina la función y entrega un resultado que quien la llamó puede guardar o usar:

```js
function sumar(a, b) {
  return a + b; // sin return, la función retornaría undefined
}

const resultado = sumar(2, 3);
console.log(resultado); // 5
```

Ejemplo combinando funciones con estado compartido:

```js
let total = 0;

function agregarCarrito(precio) {
  return (total += precio); // suma al total global y lo retorna
}

function calcularImpuesto(total) {
  return total * 1.15; // 15% de impuesto
}

agregarCarrito(300);
agregarCarrito(100);
agregarCarrito(700);

const totalPagar = calcularImpuesto(total);

console.log(total); // 1100
console.log(`El total a pagar es de: ${totalPagar}`); // 1265
```

> [!warning] Una sola sentencia con `return`
> En una función con llaves, `return` debe estar en la misma línea que el valor o en la siguiente: si hay una llave de cierre intermedia, el `return` ya no es alcanzable.

## Funciones dentro de objetos (métodos)

Cuando una función vive en una propiedad de un objeto, se llama **método**:

```js
const reproductor = {
  reproducir: function (id) {
    console.log(`Reproducir canción con el id ${id}`);
  },

  pausar() {
    console.log('Reproductor en pausa...');
  },

  agregarPlaylist(list) {
    console.log(`Agregando la playlist ${list}`);
  },
};

reproductor.reproducir(20);
reproductor.pausar();
reproductor.agregarPlaylist('Bachata');

// También se pueden agregar métodos después, como si fueran propiedades
reproductor.reproducirPlaylist = function (list) {
  console.log(`Reproducir la playlist ${list}`);
};

reproductor.borrar = (id) => {
  console.log(`Borrar canción con el id ${id}`);
};

reproductor.borrar(20);
reproductor.reproducirPlaylist('Vallenato');
```

Ver [[Objetos]] para el resto de operaciones con objetos.

## Arrow functions

La sintaxis moderna (ES6) más usada, especialmente como callbacks. Son más cortas y resuelven el problema del `this`:

```js
// Equivalentes: function expression vs arrow function
const aprendiendo = function () {
  console.log('Aprendiendo JavaScript');
};

const aprendiendo2 = () => console.log('Aprendiendo JavaScript');

// Con un paréntesis opcional si hay UNO solo
const aprendiendo3 = (tecnologia) => `Aprendiendo ${tecnologia}`;
console.log(aprendiendo3('JavaScript'));

const aprendiendo4 = tecnologia => `Aprendiendo ${tecnologia}`; // sin paréntesis

// Con dos o más parámetros SÍ se necesitan paréntesis
const aprendiendo5 = (tecnologia, tecnologia2) =>
  `Estamos aprendiendo ${tecnologia} y ${tecnologia2}`;
console.log(aprendiendo5('JavaScript', 'NextJS'));
```

> [!tip] Return implícito
> Si el cuerpo es **una sola expresión**, las llaves y el `return` son opcionales: `const doble = (n) => n * 2;`. Con varias líneas se usan llaves y `return` explícito.

## Arrow functions con `map` y `forEach`

Aquí las arrow functions brillan: los callbacks de [[Arreglos|los arreglos]] quedan en una sola línea — ver [[Iteradores]] y [[Array Methods]]:

```js
const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audífonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
];

// map: transforma y retorna un arreglo nuevo (return implícito)
const mensajes = carrito.map(
  (producto) => `${producto.nombre} con un precio de ${producto.precio}`
);

// forEach: ejecuta por cada elemento (no retorna nada)
carrito.forEach((producto) =>
  console.log(`${producto.nombre} tiene un precio de ${producto.precio}`)
);

console.log(mensajes);
```

## Relacionadas

- [[Sintaxis Moderna]] — donde viven las arrow functions y el resto del JS actual
- [[Estructuras de Control]] — las condiciones que las funciones suelen evaluar
- [[Arreglos]] — callbacks con `map`, `forEach` y compañía
- [[Array Methods]] — `filter`, `reduce`... todo recibe funciones
- [[Objetos]] — métodos: funciones dentro de objetos
- [[ReactJS - Fundamentos]] — en React, los componentes son funciones
