---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
## Números

```jsx
// Definir numeros

const num1 = 20; // Esto es un numero, no llevan comillas
const num2 = 15;
const num3 = 18.5; // Esto es un numero
const num4 = -14; // Tambien podemos crear numeros negativos

console.log(num1);
console.log(num2);
console.log(num3);

const num2 = 15;

let resultado;

//suma
resultado = num1 + num2;

//Resta
resultado = num1 - num2;

//Multiplicación
resultado = num1 * num2;

// Divicion
resultado = num1 / num2;

//Modulo
resultado = num1 % num2; // nos caulcula l modulo o reciduo

// Uso del Objeto Math

console.log(Math.PI); // Nos da el valor de pi

console.log(Math.floor(2.6)); // Redondea hacia abajo
console.log(Math.ceil(2.4)); // Redondear hacia arriva

console.log(Math.round(2.5)); // Redondea tanto para arriba como para abajo

console.log(Math.sqrt(144)); // Nos permite obtener la raiz cuadrada

console.log(Math.abs(-500)); // Valor absoluta de un numero n

console.log(Math.pow(4, 2)); // Nosp permite elvar un valor a una potencia

console.log(Math.min(3, 2, 5, -5, 1, 9));// Nos permite obtener el valor minimo de un arreglo

console.log(Math.max(3, 2, 5, -5, 1, 9)); // Nos permite obtener el valor maximo de un arreglo

console.log(Math.random()); // nunca nos da numeros enteros

console.log(Math.random() * 20); // Esto ya nos da valores mayores a 1

// Valores enteros random en un rando de numeros en este caso del 0 30
console.log(Math.round(Math.random() * 30));

// Incrementos o Decrementos

let puntaje = 5;

puntaje++; // Nos permite incrementar en 1 el valor
++puntaje; // Tambien nos permite aumentar en 1 el valor, sirve en bucles y estructuras repetitivas.

// valores
puntaje += 2; // Nos permite incrementar en 2 el valor

console.log(puntaje);

let puntaje2 = 10;

puntaje2--; // Nos permite reducir el resultado
--puntaje2; // Nos permite reducri el puntaje en 1, sirve en bulces y estructuras repetitivas.

// valores 
puntaje -= 2; // Nos permite reducir el valor en 2;

console.log(puntaje2);

// Convertir Strings a Numeros

const num1 = "20";
const num2 = "20.2";
const num3 = "Uno";
const num4 = 20;

console.log(typeof num1); // Nos regresa el tipo de variable que es

console.log(Number.parseInt(num1)); // Nos permite convertir de un string a un numero entero
console.log(Number.parseFloat(num2)); // Nos permite convertir de un string a un numero flotante
console.log(Number.parseInt(num3)); // No podemos convertir las cadenas a numeros

// Revisar si un valor es un entero

console.log(Number.isInteger(num4));
console.log(Number.isInteger(Number.parseFloat(num2)));
```