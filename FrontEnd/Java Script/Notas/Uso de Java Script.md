---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# ¿Qué podemos hacer con JavaScript?
Con JavaScript es posible hacer una gran cantidad de cosas como las siguientes:

- Recolectar Datos:

```jsx
const nombre = prompt("Cual es tu nombre: ");
```

![[Pasted image 20250125075758.png]]

Podemos recolectar información y se almacenará dentro de la variable nombré.

- Seleccionar y modificar elementos en html:

```jsx
const nombre = prompt("Cual es tu nombre: ");
document.querySelector(".contenido").innerHTML = `${nombre} esta aprendiendo JavaScript Moderno`;
```

El objetivo es igual obtener información y luego vamos a insertar esta información con otros datos en el html sin modificar el archivo html. 

![[Pasted image 20250125075817.png]]