---
tags: 
Creado: 
Relacionado:
---
----------
# Archivos WebP y Avif
Nosotros en nuestras imágenes podemos crear o servir imágenes webp en nuestras paginas web, no es del todo soportado el formato por lo que vamos a a ver como cargarlas y como en el caso de no se soportado podremos cargar las imágenes jpg

```html
<picture>
	<source loading="lazy" srcset="img/blog1.avif" type="image/avif">
	<source loading="lazy" srcset="img/blog1.webp" type="image/webp">
    <img loading="lazy" src="img/blog1.jpg" alt="Imagen blog">
</picture>
```

En el caso de tener múltiples imágenes las cuales podamos servir en diferentes tamaños de pantalla vamos a usar el siguiente código:

```html
    <picture>
    <!--tamaños de pantalla 1920w, 1280w y 640w-->
    <source
       sizes="1920w, 1280w, 640w" 
       srcset="img/imagen.avif 1920w, 
           img/imagen-1280.avif 1280w, 
           img/imagen-640.avif 640w" 
       type="image/avif">
       <!--Con srcset defimos la imagen para cierto tamaño de pantalla-->
    <source
       sizes="1920w, 1280w, 640w" 
       srcset="img/imagen.webp 1920w, 
           img/imagen-1280.webp 1280w, 
           img/imagen-640.webp 640w" 
       type="image/webp">
    <source
       sizes="1920w, 1280w, 640w" 
       srcset="img/imagen.jpg 1920w, 
           img/imagen-1280.jpg 1280w, 
           img/imagen-640.jpg 640w" 
       type="image/jpeg">
    <img loading="lazy" decoding="async" src="img/imagen.jpg" lazyalt="imagen" width="500" height="300">
    </picture>
```

