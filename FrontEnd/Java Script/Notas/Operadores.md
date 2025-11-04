---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Operadores

```jsx
const num1 = 20;
const num2 = "20";
const num3 = 30;

// Operador Mayor que....

console.log(num1 > num3);
console.log(num3 > num1);

// Operador menor que..

console.log(num1 < num3);
console.log(num3 < num1);

// igual que..

console.log(num1 == num3);
console.log(num1 == num2); // Comparador NO Estricto solo se fija en el valor

// Comparador Estricto

console.log(num1 === num2); // al tener 3 signos iguales se fija en el valor y en el tipo de dato
console.log(num1 == parseInt(num2)); // como el typo y el valor es igual funciona

// Diferente que...

const password1 = "admin";
const password2 = "Admin";

console.log(password1 != password2);
console.log(num1 != num2); // Es un comparador No Estricto solo se fija en el valor

// Diferente que Estricto..

console.log(num1 !== num2);

let numero;

console.log(numero);

console.log(typeof numero); // El valor es no definido, es por esto que se vuelve undefined

let numero2 = null;

console.log(numero2);

console.log(typeof numero2); // Sera un objeto por definición del lenguaje ya que es JavaScript los null son objetos

console.log(numero == numero2);// Esta comparación no es valida y puede devolver valores extraños, es preferible usar
// comparadores estrictos.

console.log(numero === numero2);
```