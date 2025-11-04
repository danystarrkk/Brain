---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Iteradores

```jsx
// for loop
// Esta compuesto por 3 partes, <iniciar la variable> <Condicion a cumplirse> <incremento o decremento>

// bucle de 0 a 10
for (let i = 0; i <= 10; i++) {
  console.log(i);
}

// Pregunta de trabajo: Numeros pares y numeros impares

for (let i = 0; i <= 10; i++) {
  if (i == 0) {
    console.log(`El ${i} no es Par ni Impar`);
  } else if (i % 2 == 0) {
    console.log(`El ${i} es un numero Par`);
  } else {
    console.log(`El ${i} es un numero Impar`);
  }
}

// recorriendo un arreglo

const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audifonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
  { nombre: 'Mouse', precio: 20 },
  { nombre: 'Silla Gamer', precio: 140 },
  { nombre: 'Mouse Pad', precio: 10 }
]

for (let i = 0; i < carrito.length; i++) {
  console.log(`El ${carrito[i].nombre} tiene un precio de ${carrito[i].precio}`);
}

// Haciendo uso de Break

for (let i = 0; i < 10; i++) {
  console.log(`Es el numero: ${i}`)
  if (i === 5) {
    break;
  }
}

// Haciendo uso del continue

for (let i = 0; i < 10; i++) {
  if (i === 5) {
    console.log('cinco');
    continue;
  }
  console.log(`Es el numero: ${i}`)
}

// ejemplo con carrito

const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audifonos', precio: 50 },
  { nombre: 'Teclado', precio: 100, descuento: true },
  { nombre: 'Mouse', precio: 20 },
  { nombre: 'Silla Gamer', precio: 140 },
  { nombre: 'Mouse Pad', precio: 10 }
]

for (let i = 0; i < carrito.length; i++) {
  if (carrito[i].descuento) {
    console.log(`El ${carrito[i].nombre} esta en descuento`);
    continue;
  }

  console.log(carrito[i].nombre);

}

// reto fizz bus numeros del 1 al 100
// Todos los multiplos de 3 tiene que imprimir la palabra fizz
// Todos los multiplos de 5 tienen que imprimir la palabra bus
// Todos los multiplos de 5 y 3 tiene que imprimir la palbra fizz bus

for (let i = 1; i <= 100; i++) {
  if (i % 3 == 0 && i % 5 == 0) {
    console.log(`${i}: Fizz Bus`);
    continue;
  } else if (i % 3 == 0) {
    console.log(`${i}: FIZZ`);
    continue;
  } else if (i % 5 == 0) {
    console.log(`${i}: Bus`);
    continue;
  }
}
// bucle while, este es un iterador el cual se ejecuta mientras se cumpla una condicion

let i = 0; // iniciamos la variable

while (i < 10) {
  console.log(i)
  i++ // aumentamos la variable para que el bucle no sea infinito
}

// bucle do while, a diferencia de el while que primero comprueba la condicion el do while primero ejecuta y luego comprueba la condicion

let i = 10; // iniciando valor

do {
  console.log(i);
  i++;
} while (i < 10) // condicion

// bucle for each, este nos permite recorrer areglos

const pendientes = ['Tareas', 'Ejercicio', 'Programacion', 'Trabajo'];

pendientes.forEach((pendiente, indice) => console.log(`${pendiente} con indice ${indice}`)); // El for each es un arrow function, esta es su estructura 
//podemos obtener el indice de cada valor dentro del array padando un segundo valor a la función

// for of nos permite recorrer el arreglo de forma mucho mas sencilla
const pendientes = ['Tareas', 'Ejercicio', 'Programacion', 'Trabajo'];

const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audifonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
  { nombre: 'Mouse', precio: 20 },
  { nombre: 'Silla Gamer', precio: 140 },
  { nombre: 'Mouse Pad', precio: 10 }
]

// definimos una variable para que luego a esta se reasignen valores

for (let pendiente of pendientes) {
  console.log(pendiente);
}

for (let producto of carrito) {
  console.log(producto);
}

// El for in itera sobre indices en el caso de arreglos
const pendientes = ['Tareas', 'Ejercicio', 'Programacion', 'Trabajo'];

for (let pendiente in pendientes) {
  console.log(pendiente);
}

// for in itera sobre la definición sobre los metodos (modelo, motor, year)
const automovil = { modelo: "Toyota", motor: "Chevrolet", year: 5 }

for (let propiedad in automovil) {
  console.log(propiedad);
}

// Si queremos iterar sobre los valores del objeto podemos hacer

for (let propiedad in automovil) {
  console.log(automovil[propiedad]);
}

// Si queremos iterar sobre sobre el valor y la clave del objeto usamos for of de la siguiente forma:
for (let [clave, valor] of Object.entries(automovil)) {
  console.log(`${clave}:${valor}`);
}

```