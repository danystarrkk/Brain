---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Funciones
Las funciones de forma general nos permite dividir nuestro código en fragmentos de código más pequeños con el objetivo de hacer el código reutilizable y funcional.

```jsx
// Declaración de Función ( Function Declaration )
function sumar() {
  console.log(2 + 2);
}

sumar(); // Llamada a la función

// Expresión de Función (Function Expression)
const sumar2 = function () {
  console.log(3 + 3);
}

sumar2();

// Declaración de Función ( Function Declaration )

sumar(); // Si podemos llamar a la función antes de crearla
// Esto se deve a que la definimos como función y en la etapa de creación JS registra todas las funciones
function sumar() {
  console.log(2 + 2);
}

// Expresión de Función (Function Expression)
sumar2(); // No podemos llamar a la función antes de crearla
//Esto se deve a que como la definimos como una variable y con el uso de const no aplicaria el hosting probocando que la variable quede como no registrada
const sumar2 = function () {
  console.log(3 + 3);
}

alert("Hubo un error...");

prompt("Cual es tu edad: ");

console.log(parseInt("20"));

const num2 = 20;
const num = "20";

console.log(parseInt(num2)); // esto es una función

console.log(num.toString()); // esto es un metodo

// Los dos son parecidos pero al momento de llamarolos se los usa de forma distinta.

function sumar(a, b) { // Creamos una función que lleva parametros
  console.log(b + a);
}

sumar(2, 4); // Esta es la forma correcta de enviarle valores o argumentos a nuestra función
sumar(4534523, 23454325);

// Ejemplo muchos mas practico

function saludar(nombre, apellido) {
  console.log(`Hola ${nombre} ${apellido} es un Gusto Saludarte`);
}

nombreCompleto = prompt("Digita tu primer nombre y tu primer apellido");
nombreApellido = nombreCompleto.split(" ");

saludar(nombreApellido[0], nombreApellido[1]);

// Parametros por Default
// Esto lo usamos para tener parametros donde no podria tener y evitar fallos
function saludar(nombre = "Null", apellido = "Null") {
  console.log(`Hola ${nombre} ${apellido} es un Gusto Saludarte`);
}

nombreCompleto = prompt("Digita tu primer nombre y tu primer apellido");
nombreApellido = nombreCompleto.split(" ");

saludar(nombreApellido[0], nombreApellido[1]);
// Comunicación entre Funciones

iniciarApp();

function iniciarApp() {
  console.log("Iniciando APP");
  segundaFuncion();
}

function segundaFuncion() {
  console.log("Hola desde la segunda función");
  login("Daniel");
}

function login(nombre) {
  console.log("Authenticando Usuario.....");
  console.log(`Usuario ${nombre} Correctamente autenticado`);
}

// funciones que retornan valores

function sumar(a, b) {
  return a + b; // Esto nos permite retornar el valor por lo que tenemos que tener una variable la cual almacene el resultado
}

console.log(sumar(2, 3));

// Ejemplo

let total = 0;

function agregarCarrito(precio) {
  return total += precio;
}

function calcularImpuesto(total) {
  return total * 1.15;
}

agregarCarrito(300);
agregarCarrito(100);
agregarCarrito(700);

const totalPagar = calcularImpuesto(total);

console.log(total);
console.log(`El total a pagar es de: ${totalPagar}`);

// Agregando funciones en un objeto
const reproductor = {
  reproducir: function (id) {
    console.log(`Reproducir cancion con el id ${id}`);
  },

  pausar: function () {
    console.log(`Reproductor pausar...`);
  },

  agregandoPlayList(list) {
    console.log(`Agregando la Playlist ${list}`)
  }

}

reproductor.reproducir(20);
reproductor.pausar();

// Agregando una función a mi objeto de forma externa

reproductor.reproducirPlayList = function (list) {
  console.log(`Reproducir la Playlist ${list}`);
}

reproductor.borrar = function (id) {
  console.log(`Borrar cancion con el ${id}`);
}

reproductor.borrar(20);
reproductor.agregandoPlayList("Bachata");
reproductor.reproducirPlayList("Vallenato");

// funcion normal
**const aprendiendo = function () {
  console.log(`Aprendiendo JavaScript`);
}

// Funcion en arrow function

const aprendiendo2 = () => `Aprendiendo JavaScript`;

// Pasando parametros a las arrow function

const aprendiendo = (tecnologia) => `Aprendiendo ${tecnologia}`;

console.log(aprendiendo("JavaScript"));

// Cuando pasamos parametros a una arrow function los parametros son opcionales

const aprendiendo2 = tecnologia => `Aprendiendo ${tecnologia}`;
console.log(aprendiendo2("JavaScript"));

// Cuando pasamos mas de un parametro a una arrow function si es necesario el parentesis

const aprendiendo3 = (tecnologia, tecnologia2) => `Estamos aprendiendo ${tecnologia} y ${tecnologia2}`
console.log(aprendiendo3("JavaScript", "NextJS"));

// Usanso arrow function en forEach y Map

const carrito = [
  { nombre: 'Monitor', precio: 300 },
  { nombre: 'Audifonos', precio: 50 },
  { nombre: 'Teclado', precio: 100 },
  { nombre: 'Mouse', precio: 20 },
  { nombre: 'Silla Gamer', precio: 140 },
  { nombre: 'Mouse Pad', precio: 10 }
]

// Map, cuando tenemos una linea el return se da por implicito

const nuevoArreglo = carrito.map(producto => `${producto.nombre} con un precio de ${producto.precio}`);

// ForEach
carrito.forEach(producto => console.log(`${producto.nombre} tiene un precio de ${producto.precio}`));

console.log(nuevoArreglo);**

```