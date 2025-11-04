---
tags: 
Creado: 2025-05-24
Relacionado:
  - "[[CSS]]"
---
----------
# Crear archivo para Css

Bueno lo primero que tenemos que hacer es crear una carpeta contenedora de css, esta carpeta levara el nombre de css

![[Pasted image 20250124160634.png]]

ya dentro de esta carpeta crearemos lo que es nuestro archivo de estilos el cual llevara el nombre de _styles.css_ :

![[Pasted image 20250124160740.png]]

ya con esto podemos comenzar a escribir código css, recordemos que para que los cambios que realicemos en nuestro archivo css se apliquen en nuestra web vamos a tener que enlazarlo al archivo de html, esto lo hacemos de la siguiente manera:

- Dentro de nuestro head vamos a agregar:

```html
<link href="css/style.css" rel="stylesheet">

```

ya con esto se enlazara y aplicaran los estilos dispuestos en nuestra web.

Una forma de optimizar esto es agregar también:

```html
<link rel="preload" href="css/styles.css" as="style">
<link href="css/style.css" rel="stylesheet">

```

Ya con esto logramos optimizar y realizar la carga de la hoja de estilos mucho mas rápido.