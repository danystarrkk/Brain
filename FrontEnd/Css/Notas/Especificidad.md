---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Especificidad

Bueno esto es algo extraño pero lo vamos a entender. La especificidad es cuando nosotros describimos lo máximo posible al elemento a modificar, es decir que mientras mas especifico mas detallado sea la forma en la que usamos un selector, los cambios no van a ser modificados por selectores mas genéricos, vamos con un ejemplo:

```css
h1.titulo span {
color: green;
}

.titulo span {
color: red;
}

```

Como vemos en la primera parte decimos que el span de h1 que pertenezca a la clase titulo va a ser de color verde, luego le decimos que la clase titulo que contenga span será de color rojo, de manera lógica al decir que es estilo en cascada se aplicaría el ultimo modificable pero no es del todo cierto ya que el segundo al ser mas genérico que el primero no puede modificar, por lo que el span de h1 permanecerá de color verde.