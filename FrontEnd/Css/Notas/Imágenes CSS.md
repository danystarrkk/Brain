---
tags: 
Creado: 
Relacionado:
---
----------
# Imágenes en Css
Bueno desde css podemos agregar imágenes y en este caso vamos a ver como hacerlo como fondo. Para esto usamos:

```css
body {
	bacground-image: url(ruta de la imagen);
}

```

ya con esto podemos agregar la imagen con css, recordemos que la ruta comienza desde la carpeta que nos encontremos por lo que en ocasiones tenemos que retroceder directorios para ingresar a otras rutas.
Bueno tenemos mas propiedades como son:

```css
body {
	bacground-image: url(ruta de la imagen);
	background-repeat: no-repeat; -> por defecto si la imagen no cubre todo se repite, con esto evitamos esa repetición
	background-size: cover; -> Si se le quieta la repetición la imagen no cubre todo el espacio con esto lo corregimos
}

```