---
tags: 
Creado: 
Relacionado:
---
----------
# Etiquetas en HTML
En html tenemos varias etiquetas las cuales podemos usar y vamos a separarlas un poco por categorías

## Textos
Tenemos diferentes etiquetas las cuales cumplen propósitos específicos al momento de ingresar texto, algunas son:
- `<p>` : Es muy común y utilizada para todos los párrafos.
- `<h1>,<h2>,…,<h6>` : Estos son los llamados headings, que son etiquetas que van del 1 al 6, mientras más alto es el número del heading menor es su importancia en el contenido. Los headings cuentan unas reglas:
	- Solo podemos tener un h1 por pagina.
	-  Podemos usar mas de un h2 por paginas y con ayuda de estos separaremos por secciones.
- `<span>` : Esta es una etiqueta se usa para modificar elementos dentro de otros elementos.

## Imágenes
Conforme se crea un proyecto web vamos a ver que las imágenes son importantes para el diseño, algunas de las etiquetas que usamos para añadir imágenes a la web son:

- <img>`<img>`: es la etiqueta mas usada para agregar imágenes.
	-  Para esto tenemos que usar lago llamado atributos en html para poder agregar imágenes en html un ejemplo es:

```html
<img src="imagen.jpg" alt="Texto Alternativo" width=200 height=300/>
```

- `<figure>` : soporta multiples imágenes.

## Imágenes
Algunos sitios como los de marketing solo requieren una página para funcionar, pero cuando los proyectos aumenten de tamaño vamos a necesitar mas de una pagina por lo que se tendrá que agregar enlaces, y las etiquetas las podemos hacer con:

- `<a>` : Mediante esta etiqueta vamos a poder agregar nuestros enlaces:

    - Al igual que en las imágenes vamos a hacer uso de los atributos para poder hacer uso de los mismo, un ejemplo es:

```html
<a href="URL o archivo">
podemos colocar contenido como texto o imagenes con sus repectivas etiquetas
</a>
```

## Estructura
En el caso de estructurar nuestra web vamos a tener las siguiente etiquetas y cuando usarlas:

- `<header>` : Parte superior de un sitio web o un elemento.
- `<footer>` : Parte inferior de un sitio web o un elemento.
- `<nav>` : Grupo de enlaces o navegación
- `<main>` : Contenido principal del archivo html, solo podemos usar uno por archivo.
- `<section>` : Cuando el primer hijo es un Heading e introduce una nueva sección.
- `<aside>` : El contenido que acompaña al principal
- `<div>` : Cuando ninguno de los anteriores se puede utilizar.

## Listas
Tenemos dos formas de crear listas en html, uno son las listas ordenadas y otras los no ordenadas.

- `<ul>` : Nos permite crear un listado no ordenas.
- `<ol>` : Nos permite crear un listado ordenado de elementos.
- `<li>` : Nos permite crear los elementos de una listas

Ejemplo:

```html
<ul>
	<li>Esta es</li>
	<li>Una</li>
	<li>Lista</li>
	<li>No ordenada</li>
</ul>

<ol>
	<li>Esta es</li>
	<li>Una</li>
	<li>Lista</li>
	<li>Ordenada</li>
</ol>
```

## Comentarios
En html también podemos usar comentarios, tenemos que comendar lo justo y lo necesario y para poder hacer esto usamos:

- `<!--Texto del comentario-->` : Nos permite poner comentarios.