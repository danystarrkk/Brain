---
tags: 
Creado: 
Relacionado:
---
----------
# Degradados con Css

Bueno para hacer degradados con css vamos a hacerlos como si fuera una imagen y le pasaremos algunas propiedades de la siguiente manera:

```css
background-image: linear-gradient()
```

Dentro de gradiente se definen 3 cosas:

1. la dirección, si el degradado va de abajo Asia arriba(to top) o de arriba asía abajo(to bottom).
2. Primer color este es el que se encuentre el principio y se define (#ffffff 0%)
3. Segundo color este es el que se encuentre al final y se define (#212121 100%)

Un ejemplo de esto seria el siguiente:

```css
background-image: linear-gradient(to top, var(--negro) 0%, var(--blanco) 100%);
```