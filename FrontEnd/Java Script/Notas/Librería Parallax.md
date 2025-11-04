---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Parallax
Parallax es un efecto donde el fondo va a estar en un movimiento diferente a nuestro contenido, donde el resultado será un ligero efecto de profundidad para lograr observar partes que antes no se lograban ver provocando un ligero estilo de 3D.

Para lograr este tipo de efecto tenemos que acomodar los elementos en varias capas las cuales se moverán de forma independiente en sentido vertical u horizontal donde suelen ser controladas por el usuario al desplazarse por la web o puede hacerlo de forma automática.

Es bueno usar este efecto en casos donde se requiera una inmersión total del usuario a una experiencia atrapante y divertida como contar una historia, promocionar un juego, etc.

Este efecto es algo largo de implementar de forma manual y para no tener que hacerlo se creo un proyecto llamado [Paralax.js](https://github.com/wagerfield/parallax?tab=readme-ov-file#11-installation) el cual nos permitirá recrear este tipo de efecto de una forma mucho mas efectiva, rápida y fácil.

## Instalación
Puede que sea en algunas ocasiones ser necesaria la instalación de este recurso, nosotros usamos otra alternativa la cual nos permite implementarlo mediante el llamado a un script en nuestro **HTML** de la siguiente forma:

```HTML
<script src="https://cdnjs.cloudflare.com/ajax/libs/parallax/3.1.0/parallax.min.js"></script>
```

En el caso de querer instalarlo podemos hacerlo por medio de **npm** de la siguiente forma:

```bash
npm i -s parallax-js
```

## Uso
 1. Lo primero que debemos hacer es en nuestro **HTML** crear los elementos que queremos que se muevan y asignarles un `id="scene`.
 2. Luego tenemos que agregar a nuestros elementos una clase la cual nos permite ajustar la profundidad de los elementos y es  `data-depth="0.2"`. Como se explica vamos a controlar la profundidad de los elementos.
 3. Por ultimo tenemos que ejecutar el script y para esto vamos a usar:
```js
var scene document.getElementById('scene');
var parallaxInstance = new Parallax(scene);
```
