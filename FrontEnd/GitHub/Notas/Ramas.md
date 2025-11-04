---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[GITHUB]]"
---
----------
# Ramas
Las ramas son unas especies de caminos los cuales nosotros podemos seguir conforme avanzamos en nuestro repositorio. Con las ramas podemos hacer diferentes cosas como:

- **Merge - Uniones**: Nosotros podemos unir ramas y cuando lo hacemos podemos terminar en 3 posibles escenarios:
    - Fas-Forward: Esto sucede cuanto no tenemos ningún cambio en la rama principal y los cambios pueden integrarse a la rama principal, es decir todos los commits pasaran a formar parte de la rama principal.
    - Uniones Automáticas: Esto sucede cuando git detecta que en la rama principal se origino algún cambio el cual la rama secundaria desconoce pero al no haber modificado líneas iguales, no se generan conflictos y se unen con normalidad.
    - Union Manual: Esto sucede cuando git no puede resolver de forma automática un conflicto porque las mismas líneas de un mismo archivo se modificaron y para resolverlo se tendrá que hacer de forma manual.