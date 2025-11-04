---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Strings

```jsx
// Definir Strings
const producto = "Monitor de 20 pulgadas";
const producto2 = String("Monitor de 24 Pulgadas");
const producto3 = new String("Monitor de 27 pulgadas");

console.log(producto);
console.log(producto2);
console.log(producto3);

// Representar caracteres escamando caracteres

const productos = "\"Hola mundo entre comillas\"";
console.log(productos);

console.log(producto.length); // Nos permite saber el numero de letras de la variable.

// nos permite saver donde comienza la palabra y si no la encuentra se -1
console.log(producto.indexOf("pulgadas"));

// Nos permite verificar la existentcia de una palabra, devuelve un true o false

console.log(producto.includes("Tablet"));
console.log(producto.includes("Monitor"));
console.log(producto.includes("monitor"));

const precio = "30 USD";

// Podemos unir dos strings
console.log(producto.concat(precio));
console.log(producto.concat("Descuentos"));
console.log(producto + precio);
console.log(producto + "Con un precio de: " + precio);
console.log("El producto " + producto + " tiene un precio de: " + precio);
console.log("El producto", producto, "Tien un precio de ", precio);

// Tamplate String, la mas usada y recomendada
console.log(`${producto} tiene un precio de ${precio}`);

const producto = "          Monitor de 20 pulgadas               ";

// Eliminar espacios en blanco de un String
console.log(producto.trimStart()); // Eliminamos al comienzo del string.
console.log(producto.trimEnd()); // Elimina los espacios alfinal del string.

console.log(producto.trimStart().trimEnd()); // Elimina los espacios del inicio y final.

console.log(producto.trim()) // Elimina los espacios del inicio y final.

// remplazar
console.log(producto);
console.log(producto.replace("Pulgadas", "\""));
console.log(producto.replace("Monitor", "Monitor Curvo"));

// Cortar palabras
// Nos permite mostrar el texto que se encuentra en el rango definido

console.log(producto.slice(0, 10));
console.log(producto.slice(8)); // Con solo ese valor muestra todo apartir del mismo
console.log(producto.slice(2, 1)); // Si el primer valor es mayor al segundo no muestra nada

// Alternativa a .slice
console.log(producto.substring(0, 10)); // Funciona igual
console.log(producto.substring(2, 1)); // En este caso si el primer valor es mayor al segundo lo invierte

const usuario = "Juan";

console.log(usuario.substring(0, 1));
console.log(usuario.charAt(0)); // Nos imprime la letra en la posicion que le pasamos.

// Repetir una cadena de texto

const texto = " en Promoción".repeat(3); // si el valor no es entero se redondea

console.log(texto);
console.log(`${producto} ${texto}`);

// Podemos dividir un string

const actividad = "Estoy aprendiendo JavaScript Moderno";
console.log(actividad.split(" ")) // Crea un arreglo de palabras divididas por el espacio

const hobbies = "Leer, Caminar, Escuchar Musica, Programar, Arreglar";
console.log(hobbies.split(","));

console.log(producto.toUpperCase()); // Nos permite transformar todo a mayusculas

console.log(producto.toLowerCase()); // Nos permite transformar todo a minusculas

const precio = 300;
console.log(precio.toString()); // Nos permite convertir una variable numerica a String
```