---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Array Methods

```jsx
const meses = ['Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo', 'Junio', 'Julio'];

const carrito = [
  { nombre: 'Monitor 27 Pulgadas', precio: 500 },
  { nombre: 'Televisión', precio: 100 },
  { nombre: 'Tablet', precio: 200 },
  { nombre: 'Audifonos', precio: 300 },
  { nombre: 'Teclado', precio: 400 },
  { nombre: 'Celular', precio: 700 },
]

// Comprobar si un Valor existe dentro de un arreglo

// meses.forEach(mes => console.log(mes === "Enero" ? console.log("Enero si existe") : "Enero no existe")); // forma sin arraymethod

// el arraymethod .includes solo nos funciona con arreglos

const resultado = meses.includes("Enero"); // forma con array method
console.log(resultado);

// el arraymethod .some nos sirve en el uso de objetos
const existe = carrito.some(producto => producto.nombre === "Celular");
console.log(existe);

// tambien es funiconal en un arreglo
const existe2 = meses.some(mes => mes === "Mayo");
console.log(existe2);

const meses = ['Enero', 'Febrero', 'Marzo', 'Abril', 'Mayo', 'Junio', 'Julio'];

const carrito = [
  { nombre: 'Monitor 27 Pulgadas', precio: 500 },
  { nombre: 'Televisión', precio: 100 },
  { nombre: 'Tablet', precio: 200 },
  { nombre: 'Audifonos', precio: 300 },
  { nombre: 'Teclado', precio: 400 },
  { nombre: 'Celular', precio: 700 },
]

// Encontrar la posición de un array con ayuda de un bucle foreach
let indice = meses.forEach((mes, i) => {
  if (mes === "Mayo") {
    console.log(i);
  }
})

// Encontrar la posición de un array mediante un findindex
const indice2 = meses.findIndex(mes => mes === 'Abri'); // en el caso de no encontrarlo va a retornar un -1
console.log(indice2 > 0 ? `El indice de Abril es ${indice2}` : "Mes no encontrado"); // forma de comprobar si no existe.

const indice3 = carrito.findIndex(carrito => carrito.nombre == "Celular");
console.log(indice3 > 0 ? `El indice de Celular es ${indice3}` : "Valor no encontrado");

// Mediante un foreach
let total = 0;

carrito.forEach(producto => total += producto.precio);
console.log(total);

// Con un reduce
let resultado = carrito.reduce((vprevio, vactual) => vprevio + vactual, 0);// tenemos valoreprevio, valoractual, 0 es el valor con el que comience valorprevio
console.log(total);

// filter

let resultado;

resultado = carrito.filter(producto => producto.precio > 400); // nos crea un nuevo arreglo que cumpla con la condición que se requiere
console.log(resultado);

resultado = carrito.filter(producto => producto.precio < 600);
console.log(resultado);

// Puede ser util en un carrito de compras
resultado = carrito.filter(producto => producto.nombre != "Audifonos");
console.log(resultado);

// con foreach
let resultado = "";

carrito.forEach((producto, indice) => {
  if (producto.nombre == "Tablet") {
    resultado = carrito[indice];
  }
})
console.log(resultado);
// sintaxis del foreach reducida

carrito.forEach((producto, indice) => producto.nombre == "Celular" ? resultado = carrito[indice] : "");
console.log(resultado);

// con .find

const resultado2 = carrito.find(producto => producto.nombre === "Tablet");// nos permite buscar por un valor y asignarlo a una variable
console.log(resultado2);

// con foreach
let resultado = "";

carrito.forEach((producto, indice) => {
  if (producto.nombre == "Tablet") {
    resultado = carrito[indice];
  }
})
console.log(resultado);
// sintaxis del foreach reducida

carrito.forEach((producto, indice) => producto.nombre == "Celular" ? resultado = carrito[indice] : "");
console.log(resultado);

// con .find

const resultado2 = carrito.find(producto => producto.nombre === "Tablet");// nos permite buscar por un valor o el primer valor que cumpla la condición
// y asignarlo a una variable, pero solo a un valor no mas de UNO
console.log(resultado2);

// uso de .every
// el objetivo es determinar si todos los elementos cumplen la condición
const resultado = carrito.every(producto => producto.precio < 1000);
console.log(resultado);

// concat nos permite unir arreglos
const resultado = meses.concat(meses2, meses3, 'OtroMes');
console.log(resultado);

// spread operator
const resultado2 = [...meses, ...meses2, ...meses3, "OtroMes"];
// si pasmos un string como ..."OtroMes" se agregan cada letra como un elemento
console.log(resultado2);

// concat nos permite unir arreglos
const resultado = meses.concat(meses2, meses3, 'OtroMes');
console.log(resultado);

// spread operator
const resultado2 = [...meses, ...meses2, ...meses3, "OtroMes"];
// si pasmos un string como ..."OtroMes" se agregan cada letra como un elemento
console.log(resultado2);

// Mas con spread Operator
const meses2 = [...meses, 'Julio'];
console.log(meses2); // esto no modifica al arreglo origina y es la programación funcional

// En objetos

const producto = { nombre: 'Disco Duro', precio: 300 };
// Si usamos los 3 puntos en un objeto nos dara un error ya que no es operable
const carrito2 = [...carrito, producto];
console.log(carrito2);
```