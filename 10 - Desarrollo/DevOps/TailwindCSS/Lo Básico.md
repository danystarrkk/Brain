----------------

# Uso mediante el script

Nosotros podemos usar tailwind agregando fácilmente desde una cabecera para poder usarlo sin necesidad de instalar nada esto no es optimo pero es practico para cosas básicas y sencillas ya que esta forma de usarlo cuenta con varias limitaciones al no ser su versión completa por decirlo de alguna forma.

```html
<!doctype html>
<html>

<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
</head>

<body>
  <h1>
    Hello world!
  </h1>
</body>

</html>

```

# Sintaxis básica

Vamos a ver las diferentes forma de escribir propiedades que seria

 - **Propiedad - Valor - Intensidad **

Todo lo vamos a hacer mediante clase un ejemplo seria el siguiente esto dentro de una etiqueta `h1`:

```html
  <h1 class="bg-red-500 text-white p-4">
```

Veamos otros ejemplos:

```html
  <div class="bg-orange-500">CSS</div>
  <div class="text-violet-400">CSS 2 </div>

  <p class="font-sans">
    Lorem, ipsum dolor sit amet consectetur adipisicing elit. Vel id fugiat rerum quo veritatis inventore
    deserunt, cumque saepe dicta quam velit reprehenderit consequuntur tempore ad quasi itaque nam ea doloribus.
  </p>
  <br>
  <p class="font-serif">
    Lorem ipsum dolor sit amet, consectetur adipisicing elit. Ratione, voluptatum? Dolores ratione eveniet quibusdam
    esse eaque sit magni modi! Odio possimus laboriosam distinctio ipsum optio aperiam sapiente corrupti porro officiis!
  </p>
```

