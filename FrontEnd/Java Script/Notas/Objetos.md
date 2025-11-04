---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Objetos en JavaScript

```jsx
const nombre = "Monitor de 20 pulgadas";
const precio = 30;
const disponible = true;

// Un objeto agrupa todo en una sola variable

// Object Literal
const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true
}

console.log(producto);

console.log(producto);
console.log(producto.nombre); // Accedemos al valor de nombre (Sintaxis de punto)

console.log(producto['precio']); // Esta es otra forma de acceder al valor

// Agregar nuevas propiedades al objeto
producto.imagen = "imagen.jpg"; // Agregamos un nuevo atributo al objeto

// Eliminar diponible
delete producto.disponible; // Nos permite borrar atributos del objeto

console.log(producto);

console.log(producto.nombre);

// Asignando el valor de un atrivuto a una variabla
// const nombre = producto.nombre;

// Destructuring de Objetos
const { nombre } = producto; // creamos la variable nombre y le asignamos el valor que  tiene dentor del objeto producto

// Multiples valores con Destructuring
const { precio, disponible } = producto; // de igual forma extrae y crea la variable aparitr del objeto

console.log(nombre);
console.log(precio);
console.log(disponible);

const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true,
  informacion: {
    medidas: {
      peso: "1kg",
      medida: "1m"
    },
    fabricacion: {
      pais: "china"
    }
  }
}

console.log(producto);
console.log(producto.informacion);
console.log(producto.informacion.medidas);
console.log(producto.informacion.fabricacion.pais);

// Destructuring a Objetos Anidados

const { nombre, informacion, informacion: { fabricacion: { pais } } } = producto;

console.log(nombre);
console.log(informacion);
console.log(pais);

// Las propiedades de un objeto si se pueden eliminar o modificar, sin importar si este es defino con const

producto.disponible = false;

console.log(producto.disponible);

------

"use strict"; // hace que el código de JS sea mas estricto

const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true
}

Object.freeze(producto); // Esto hace que el objeto no se pueda modificar, eliminar o agregar

console.log(Object.isFrozen(producto)); // Nos permite verificar si un objeto esta conjelado

----

"use strict"; // hace que el código de JS sea mas estricto

const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true
}

Object.seal(producto); // Nos permite modificar los atributos del objeto, mas NO podemos agregar o eliminar atributos

producto.disponible = false;

console.log(producto);
console.log(Object.isSealed(producto)); // Nos permite ver si un objeto esta sellado.

-------

const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true
}

const medidas = {
  peso: "1kg",
  medida: "1m"
}

// Unir o combinar dos objetos
const resultado = Object.assign(producto, medidas);
console.log(resultado);

// Spread Operator o Rest Operator Para realizar una copia
const resultado2 = { ...producto, ...medidas };
console.log(resultado2);

------

// Object Literal
const producto = {
  nombre: "Monitor de 20 pulgadas",
  precio: 300,
  disponible: true
}

// Object Constructor para la optimización de la creación de objetos multiples

function Producto(nombre, precio) {
  this.nombre = nombre;
  this.precio = precio;
  this.disponible = true;
}

const producto2 = new Producto("Monitor de 24 Pulgadas", 500);

console.log(producto2);

const product3 = new Producto("Audifonos Logitech", 100);

console.log(product3);
```