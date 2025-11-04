---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Estilos para Escribir Código Css
Tenemos diferentes formas de escribir código css que son:

- BEM
- Utility First
- Módulo Si el proyecto si es grande podemos llegar a usar los 3.
## Bloques, Elementos, Modificadores (BEM)
Bem es una metodología para crear código reutilizable y ordenado en css, para utilizar esto tenemos que seguir convecciones de BEM, gracias a esto se evita la colisión de nombres.
## Reglas de BEM
### Bloques
Son un contenedor, si una sección por si sola es significativa y no requiere de otras secciones para sus apariencia (CSS) deberá ir en un bloque.

![[Pasted image 20250124164325.png]]
### Elementos
Los elementos en BEM son parte de un bloque, depende del bloque y no es por si solo significativo. Tiene la característica de que utilizan el nombre del bloque y después el doble guion bajo.

![[Pasted image 20250124164347.png]]
### Modificadores
Un bloque o Elemento tendrá una variante?. Se utiliza un modificador que es una _bandera_ que avisa que es el elemento tendrá un diseño diferente.

![[Pasted image 20250124164425.png]]
### Utility First
De forma básica creamos clases con una sola propiedad que describe que es lo que aria:

![[Pasted image 20250124164451.png]]

Bueno esta es una forma que tiene mucho código pero para ciertas paginas es muy útil.
### Módulos
Esta forma es cuando definimos el contenido principal y luego seleccionamos los elementos dentro de esto:

![[Pasted image 20250124164515.png]]