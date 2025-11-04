---
tags:
  - FrontEnd
Creado: 2024-01-24
Relacionado:
  - "[[Brain/FrontEnd/Gulp/GULP|GULP]]"
---
----------
# Instalar gulp
Para instalar gulp vamos a usar el comando:

```json
npm i -D gulp
```

Con gulp vamos a automatizar todo y para poder configurar esta automatización vamos a crear un archivo llamado `gulpfile.js`.
Dentro de este archivo vamos a crear múltiples funciones en Java Script para automatizar diferentes procesos.

## Primera tarea
vamos a crear una función simple en nuestro gulpfile.js así que lo primero es crear dicha función en este caso un hola mundo:

```jsx
function hola() {
  console.log("Hola de gulp");
}
```

Lo siguiente es agregar un exports con el objetivo de que nuestra función sea ejecutable desde el Script del Package.json:

```jsx
export function hola(done) {
  console.log("Hola de gulp");
  done();
}
```

Algo a tener en cuenta es que se debe cambiar el tipo dentro del Package.js de commonjs  a module para que admita la sintaxis nueva de JavaScript.