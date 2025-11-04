---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Efecto de Scroll
Bueno para añadir diferentes efectos tenemos que definir dentro del html lo siguiente:

```css
html {
	scroll-snap-type: y mandatory;
}

```

aparte de mandatorio tenemos muchos pero como ejemplo lo usaremos, ya con esto tenemos que definir lo siguiente en cada parte de nuestra web:

```css
.servicios,
.navegacion-principal {
    scroll-snap-align: center;
    scroll-snap-stop: always;
}

```

bueno con esto definimos que al bajar se centre y pare en las secciones que se especifica.