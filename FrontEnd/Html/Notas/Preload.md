---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[Brain/FrontEnd/Html/HTML|HTML]]"
---
----------
# Preload
El preload funciona en dos partes, la primera consiste en definir lo necesario y luego realizar la llamada de lo mismo de la siguiente forma:

```html
  <!-- Preload -->

  <link rel="preload" href="css/normalize.css" as="style">
  <link href="css/normalize.css" rel="stylesheet">

  <link rel="preload" type=""
    href="<https://fonts.googleapis.com/css2?family=Open+Sans:ital,wght@0,300..800;1,300..800&family=PT+Sans:wght@400;700&display=swap>"
    crossorigin="crossorigin" as="font">
  <link
    href="<https://fonts.googleapis.com/css2?family=Open+Sans:ital,wght@0,300..800;1,300..800&family=PT+Sans:wght@400;700&display=swap>"
    rel="stylesheet">

  <link rel="preload" href="css/style.css" as="style">
  <link href="css/style.css" rel="stylesheet">
```