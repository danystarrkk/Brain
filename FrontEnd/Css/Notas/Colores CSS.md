---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Colores Css
Bueno tenemos que saber que los colores los podemos representar de varias formas en css y seria de la siguiente forma:
## Nombre
Por nombre es sencillo, seria de la siguiente manera:
![[Pasted image 20250124161916.png]]
Esta forma no es recomendable ya que son muy sencillos.
## Hexadecimal
Esta es una forma sencilla de representar los diferentes colores, podemos representarlos de 3 0 6 dígitos:
![[Pasted image 20250124162011.png]]

Otros formatos son:
- RGB
- HLS

## Paleta de colores
Bueno para crear un paleta de colores vamos a crear un selector, para esto tiene que comenzar con dos puntos:
```css
:root {

}

```

Con esto ya tenemos nuestro selector listo con el nombre de root. Ya con esto podemos crear nuestros custom-properties, para esto cuando creamos custom-properties estos comienzan con doble guion de la siguiente manera:
```css
:root {
	--blanco: #ffffff
}

```

Como vemos así se definir custom-properties y ya con esto podemos definir una paleta de colores.