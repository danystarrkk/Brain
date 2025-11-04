---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Responsive Web Design

Esto no es mas que hacer que nuestra web sea adaptable a diferentes dispositivos como: móviles, tables, relojes, televisores, monitores y mas. Tenemos que entender que los usuario son los que eligen en que van a navegar por lo que nosotros no podemos designarles o muchos menos indicarles resoluciones o navegadores los cuales ellos tiene que usar. Aparte de esto al publicar los sitios web Google penaliza cuando un sitio web no es Responsive.

La solución para todo esto es aplicar lo que se conoce como _Responsive Web Design con Media Queries_.

Bueno todo lo explicado lo logramos con Media Queries que tiene un sintaxis de la siguiente manera:

![[Pasted image 20250124165031.png]]

tenemos el media y entre paréntesis nosotros definimos un condición, en este caso tenemos un min-width el cual nos permite definir un tamaño de pantalla por lo que en este caso es que cuando la pantalla tenga un mínimo de 768 px se ejecutara el media queries. Tenemos diferentes tamaños de pantalla y algunos conocidos son:

- 480px -> para un teléfono
- 768px -> este se usa para las Tablet.
- 1140px -> este es para una laptop y computadora de escritorio
- 1400px -> este para pantallas grandes.

Bueno en los media queries solo tenemos que reescribir lo que se modificara al cambiar el tamaño de la pantalla por lo que no es necesario reescribir todo. Vemos a continuación buenas practicas al usar los media queries:

- Es una buena practica usar los media queries debajo de la propiedad la cual se va a modificar para así poder identificarlos de mejor manera.