---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
---
----------
# Tipos de Posiciones
Tenemos dos tipos de posiciones en que son:

- Relativas
- Absolutas
- fixed
## Fixed
Esto es fixed es un atributo el cual le podemos pasar al la posición de un elemento dentro de nuestro css con el objetivo de que este se mantenga en ese punto sin importar que, lo asignamos de la siguiente manera:

```jsx
/*Si lo asignamos a un h1*/
h1 {
	position: fixed;
}
```

Esto es muy útil en le caso de usar botones flotantes en la pantalla mediante los cuales nosotros podremos colocar en un punto y estos sin importar el scroll van a mantenerse en ese punto de la pantalla.
## Relativas
La posición relativa es aquella que nos permitirá el uso de forma normal de los atributos left, right, top, bottom.

```jsx
/*volvemos a realizarlo en un h1*/
h1 {
	position: relative; /*Desbloquemos para el uso de los demas atributos*/
	/*Para los siguientes atributos tambien podemos usar porcentajes meditante los cuales podemos hacer todo mas sencillo*/
	left: 10rem;
	top: 2rem;
	bottom: 5rem;
	right: 2rem;
}
```

Es muy útil para ubicar cosas en la pantalla.
## Absolutas
La posición absoluta nos permite ubicar elementos en nuestra web a partir de elemento padre del elemento a modificar, un requisito es que este elemento padre este posicionado, es decir tenga activo el uso de position, caso contrario las posiciones se tomaran a partir del body.