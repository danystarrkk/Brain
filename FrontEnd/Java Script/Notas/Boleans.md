---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Boleans

```jsx
// Definir Booleans
const boolean1 = true;
const boolean2 = false;
const boolean3 = "true";
const boolean4 = new Boolean(true); // Es un objeto mas que un booleano

console.log(boolean1);
console.log(boolean2);

console.log(typeof boolean1);
console.log(typeof boolean2);

// Comparación entre Strings y booleans

console.log(boolean1 == boolean3); // Los trata como dos valores diferentes por eso es false
console.log(boolean1 === boolean3); // Obiamente seran diferentes (false)

console.log(boolean1 == boolean2); // (false)

console.log(boolean1 == "true"); // (false)

console.log(boolean1 == true); // (true)

const autenticado = true;

// La comprobación lo hace de forma implicita

if (autenticado) {
  console.log("Si puedes ver netflix");
} else {
  console.log("No puede verlo");
}

// Operador Ternario Uso correcto

console.log(autenticado ? "Si esta Autenticado" : "No esta autenticado");
```