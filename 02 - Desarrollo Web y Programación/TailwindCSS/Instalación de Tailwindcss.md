-------------

Primero y mas importante necesitamos tener instalado o instalar nodejs completo esto para poder usar `npm`.


## Iniciar proyecto
Si ya tenemos instalado nodejs vamos a comenzar con el comando:

```bash
npm init
```

![[Pasted image 20260603094619.png]]

Nos permite iniciar un nuevo proyecto como vemos en la imagen vamos rellenado lo que nos vaya pidiendo pero el objetivo es que al terminar y darle enter al yes se cree un archivo llamado `package.json`:

![[Pasted image 20260603094740.png]]

## Instalación de tailwindcss

como vemos ya esta listo la base para proceder con al instalación de tailwindcss así que el siguiente comando es:

```bash
npm install tailwindcss @tailwindcss/cli
```

## Estructura de archivos

Una vez terminado lo siguiente es la estrucutra de carpetas que vamos a usar y sera la creación de`/src/input.css` y dentro de esta vamos a integrar lo siguiente:

```css
@import "tailwindcss";
```

![[Pasted image 20260603095821.png]]

como observamos ya tenemos lista la importación y antes de cualquier cosa creamos nuestro index para poder tener algo que manipular y se aria de la siguiente manera:

```html
<!doctype html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link href="/src/style.css" rel="stylesheet">
</head>
<body>
  <h1 class="text-3xl font-bold underline">
    Hello world!
  </h1>
</body>
</html>
```

bueno en este punto no vamos a ver ni tener absolutamente nada y no se aplican las cosas pero para que si lo hagan y esto sucede necesitamos 




