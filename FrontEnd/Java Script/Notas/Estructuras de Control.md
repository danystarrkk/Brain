---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Estructuras de Control

```jsx
const puntaje = 1000;

if (puntaje == 1001) {
  console.log(`El puntaje si es igual`);
} else {
  console.log(`El puntaje no es igual`)
}

// Estructura diferente sin los parentecis

if (puntaje == 1000)
  console.log(`El puntaje si es igual`);
else
  console.log(`El puntaje no es igual`)
  
//  Comparador No Estricto Diferente
const puntaje = 1000;

if (puntaje != 2000) {
  console.log(`El puntaje si es diferente`);
}

// Comparador Estricto Diferente

if (puntaje !== "1000") {
  console.log(`El puntaje si es diferente`);
}

// Comparador NO Estricto

if (puntaje == "1000") {
  console.log(`EL puntaje si es igual`);
} else {
  console.log(`No es Igual`);
}

// Comparador Estricto iguales

if (puntaje === "1000") {
  console.log(`EL puntaje si es igual`);
} else {
  console.log(`No es Igual`);
}

const dinero = 300;
const totalApagar = 300;

// Mayor que

if (dinero > totalApagar) {
  console.log(`Si podemos pagar`);
} else {
  console.log(`Fondos Insuficientes`);
}

// Menor que
if (dinero < totalApagar) {
  console.log(`Fondos Insuficientes`);
} else {
  console.log(`Si podemos pagar`);
}

// Mayor o Igual que

if (dinero >= totalApagar) {
  console.log(`Si podemos pagar`);
} else {
  console.log(`Fondos Insuficientes`);
}

// Mayor o Igual que

if (dinero <= totalApagar) {
  console.log(`Fondos Insuficientes`);
} else {
  console.log(`Si podemos pagar`);
}

// Elseif

const dinero = 300;
const tarjeta = true;
const totalApagar = 300;

if (dinero > totalApagar) {
  console.log(`Si podemos pagar`);
} else if (tarjeta) {
  console.log(`Con la tarjeta si puede pagar`);
} else {
  console.log(`Fondos Insuficientes`);
}

// Switch
const metodoPago = "efectivo";

switch (metodoPago) {
  case "efectivo":
    pagar(); // Podemos pasar funciones tambien
    console.log(`Pagaste con ${metodoPago}`);
    break;
  case "cheque":
    console.log(`Pagaste con ${metodoPago}`);
    break;
  case "tarjeta":
    console.log(`Pagaste con ${metodoPago}`);
    break;
  default:
    console.log(`Aún no es soportado el metodo de pago`);
    break;
}

function pagar() {
  console.log(`Pagando....`);
}

// operador and &&

const usuario = true;
const puedePagar = false;

// El !<valor> es una forma de indicar negación en el caso de un booleano

if (usuario && puedePagar) {
  console.log(`Es usuario y si puede pagar`);
} else if (!usuario && !puedePagar) {
  console.log(`No tienes fondos y no Eres usuario`);
} else if (!usuario) {
  console.log(`No estas registrado`);
} else if (!puedePagar) {
  console.log(`No tienes fondos suficioentes`);
}

// Operador OR revisa que se cumpla uno u otra, no son necesarias las dos

const efectivo = 300;
const credito = 1000;
const disponible = efectivo + credito;
const totalPagar = 600;

if (efectivo > totalPagar || credito > totalPagar) {
  console.log('Si podemos pagar');
} else {
  console.log("No podemos pagar");
}

const autenticado = true;

if (autenticado) {
  console.log("El usuario esta autenticado");
}

const puntaje = 400;

if (puntaje >= 300 && puntaje < 400) {
  console.log("Buen puntaje... Felicidades");
} else if (puntaje >= 400 && puntaje < 500) {
  console.log('Excelente');
}

// Operador Ternario

const autenticado = true;
const puedePagar = true;
// El ? es como el if y los : son como el else
console.log(autenticado ? "Si esta autenticado" : "no esta autenticado");

// Dos condiciones en un ternario
console.log(autenticado && puedePagar ? "Si puede Pagar" : "No puede pagar")

// anidaciones de if o if anidado

const efectivo = 300;
const credito = 400;
const disponible = efectivo + credito;
const totalPagar = 700;

if (autenticado) {
  if (puedePagar) {
    if (disponible) {
      console.log(`Si puede pagar`);
    }
  }
}

// operador ternario anidado
console.log(autenticado ? puedePagar ? disponible ? "Si puede pagar, desde el tenario anidado" : " No esta disponible" : "No puede pagar" : "No esta autenticado");

```