---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
---
----------
# Box Model Css

Bueno lo primero que tenemos que comprender es que en css todo es una caja, pero el como sea esa caja depende de 4 cosas:

- El tamaño del contenido
- Tamaño de relleno(padding)
- Tamaño del borde
- El margen.

Bueno veamos un ejemplo:

![[Pasted image 20250124165421.png]]

Bueno en ocasiones o en la gran mayoría de ocasiones nos especifican los tamaños de los elementos por lo que si nos dijeran que se requiere un total de 200 px de espacio tendríamos que reajustar lo que es el width, el padding y el border para que todo nos de los 200 px, una solución para esto es la siguiente.

Agracias aun desarrollador de Google Chrome que publico lo [siguiente](https://www.paulirish.com/2012/box-sizing-border-box-ftw/) podemos evitar este problema de la siguiente manera:

```css
html {
box-sizing: border-box;
}
*, *:before, *:after {
	box-sizing: inherit;
}

```

Comprendamos el código, le estamos aplicando la propiedad de border-box a todo el documento, es por eso el uso de dentro de la etiqueta html, luego tenemos estos \* que son un selector universal ósea se aplica en todo por lo que no es necesario pasar el parámetro dentro de este a los demás selectores, luego vemos que tiene comas esto solo es o se usa cuando queremos aplicara las mismas propiedades a diferentes selectores y lo hacemos con las comas, luego tenemos ese dos puntos after y before los cuales son seudo elementos que no existen como tal también se les aplicara la configuración.