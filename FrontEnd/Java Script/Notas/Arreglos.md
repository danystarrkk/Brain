---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Arreglos
Los arreglos nos permiten agrupar elementos del mismo tipo

```jsx
// Formas de definir arreglos
const numeros = [10, 20, 30]; // La forma mas comun y mas usada

const meses = new Array('Enero', 'Febrero', 'Marzo');

console.log(numeros);
console.log(meses);

// En los arreglos podemos tener todo tipo de datos, ademas de que estos pueden tener 1 a n valores

const deTodo = ["Hola", 10, true, "Si", numeros];
console.log(deTodo);

const numeros = [10, 20, 30];

console.log(numeros); // Imprime de forma normal
console.table(numeros); // Imprime en forma de tabla

// Imprimiendo por el indice de un arreglo

console.log(numeros[2]);
console.log(numeros[0]);
console.log(numeros[20]); // Cuando el indice no existe retorna valor No Definido

const meses = ["Enero", "Febrero", "Marzo", "Mayo", "Junio"];

console.table(meses);

// Cuando mide un arreglo

console.log(meses.length);

// recorrer un arrego

for (let i = 0; i < meses.length; i++) {
  console.log(meses[i])
}

const meses = ["Enero", "Febrero", "Marzo", "Mayo", "Junio"];

// reasignando un valor en una posicion

meses[0] = "Nuevo Mes"; // Los arreglos si se pueden modificar aunque sean declarados como constantes
meses[7] = "Ultimo Mes"; // No nos genera posiciones entre la definida, tantosolo locrea

console.log(meses);

const meses = ["Enero", "Febrero", "Marzo", "Abril", "Mayo"];

// Metodos de los arreglos

meses.push("Junio"); // Nos permite agregar valores al final de un arreglo

console.log(meses);

// La forma inperativa es cuando trabajamos directamente con la variable
const carrito = [];

function Producto(nombre, precio) {
  this.nombre = nombre;
  this.precio = precio;
}

const producto1 = new Producto("Celular", 500);
const producto2 = new Producto("Monitor", 110);
const producto3 = new Producto("Teclado", 50)
const producto4 = new Producto("Mouse", 20);
const producto5 = new Producto("Laptop", 900);

carrito.push(producto1); // ingresamos el objeto al final del arreglo, funciona con cualquier tipo de dato
carrito.push(producto2);
carrito.push(producto4);
carrito.push(producto5)
carrito.unshift(producto3); // ingresamos el objeto al comienzo del arreglo, funciona con cualquier tipo de dato

// Eliminar ultimo elemento de un arreglo
carrito.pop();

// Eliminar el primer elmento de un arreglo
carrito.shift();

// Eliminar cualquier elemento mediante su posición y valor de elementos a eleminar
carrito.splice(1, 1);

console.table(carrito);

// La forma declarativa es aquella que nos permite hacer nuestro código mas corto y mucho mas funcional

let resultado;

resultado = [...carrito, producto4]; // forma de agregar valores al final de una lista de forma declarativa
resultado = [producto5, ...resultado]; // forma de agregar valores al comienzo de una lista de forma declarativa

// Destructuring con arrelgos

const numeros = [10, 20, 30, 40, 50];

// En este caso nosotors podemos darle un nombre al valor

const [segundo, tercero] = numeros;

console.log(segundo);
console.log(tercero);

// Destructuring a valor exactos

const [primero, , , cuarto] = numeros;
console.log(primero);
console.log(cuarto);

const [, , ...lista] = numeros; // creamos una lista apartri del tercer elemento

console.log(lista);

// Metodos para Arreglos

const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audifonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
  { nombre: 'Mouse', precio: 20 },
  { nombre: 'Silla Gamer', precio: 140 },
  { nombre: 'Mouse Pad', precio: 10 }
]

// Rercorriendo mediante un for por las precios
for (let i = 0; i < carrito.length; i++) {
  console.log(`${carrito[i].nombre} tiene un precio de ${carrito[i].precio}`)
}

// Recoriendo el metodo foreach
carrito.forEach(function (producto) {
  console.log(`${producto.nombre} tiene un precio de: ${producto.precio}`)
});

// Metodos para Arreglos

const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audifonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
  { nombre: 'Mouse', precio: 20 },
  { nombre: 'Silla Gamer', precio: 140 },
  { nombre: 'Mouse Pad', precio: 10 }
]

// Rercorriendo mediante un for por las precios
for (let i = 0; i < carrito.length; i++) {
  console.log(`${carrito[i].nombre} tiene un precio de ${carrito[i].precio}`)
}

// Recoriendo el metodo foreach
carrito.forEach(function (producto) {
  console.log(`${producto.nombre} tiene un precio de: ${producto.precio}`)
});

// Metodo Map nos permite iterar sobre un arreglo y a la vez este nos permitira crear unno nuevo
const nuevoArreglo = carrito.map(function (producto) {
  return `${producto.nombre} tiene un precio de ${producto.precio}`;
})

console.log(nuevoArreglo);

```