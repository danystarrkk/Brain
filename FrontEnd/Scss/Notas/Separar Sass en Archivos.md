---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[SASS]]"
---
----------
# Separar sass en archivos

Nosotros siempre tenemos que tener un archivo principal el cual será `app.scss` por lo que los demás archivos que se creen llevaran un “_” al comienzo de su nombre de la siguiente manera:

![[Pasted image 20250125074749.png]]

ahora nosotros para poder usar variables o código de un archivo a otro tenemos que indicarle al archivo de donde sacamos ese código por lo tanto vamos a usar `@use` para indicarle de la siguiente manera:

```scss
@use 'variables' as v;
```

También podemos crear carpetas en las cuales podemos tener variables y características, de esta forma no llenaremos la capeta a lo loco de archivos y nos permite organizar correctamente el código, esto es simple vamos a tener dentro de cada carpeta un archivo llamado `_index.scss` el cual contendrá el código de todos los archivos dentro de la misma carpeta con ayuda de `@forward` de la siguiente manera:

```scss
@forward 'variables';
@forward 'normalize';
```

ya con esto tenemos que agregar ese índex dentro de nuestro archivo principal para que así si se compile, ya que si lo demos así nunca compilara el código, por lo tanto, vamos a usar la palabra reservada `@use`además de que solo especificamos la carpeta porque sass sabe que tiene que buscar el archivo `_index.scss` dentro de la capeta para obtener el contenido y compilarlo:

```scss
@use 'globales';
```

