---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Fuentes externas

Lo primero que tenemos que saber es como cambiar la fuente, esto es sencillo y lo podemos hacer con:

```css
html {
	font-family: fuente
}

```

Ya con esto podemos cambiar a fuentes ya predeterminadas y que todos los navegadores siempre tienen.

Bueno para agregar o añadir fuentes externas tenemos que sacarlas de algún lado, en este caso vamos a sacarlas de [browse fonts](https://fonts.google.com/). Dentro de esa pagina vamos a conseguir varias fuentes, veamos como se agrega una para comprender:
![[Pasted image 20250124162423.png]]

escogemos cualquier fuente en este caso la de roboto para el ejemplo:
![[Pasted image 20250124162505.png]]

Bueno en esta barra nos da lo esencial por lo que primero tendremos que agregar la etiqueta link con la fuente a nuestro html y luego agregar como nos indica mas abajo a nuestro css y listo.
Como un tip en el caso de usar un mismo tipo de fuente en toda la web vamos a agregarlo al body así modificándose en todo el documento.