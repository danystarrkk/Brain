---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[CSS]]"
---
----------
# Selectores en css
Tenemos diferentes formas de escribir selectores en css vamos a ver cuales son y recomendaciones al usar cada uno:
## Selector de Elemento
De manera básica es cuando seleccionamos un elemento de acuerdo a su etiqueta:
![[Pasted image 20250124161224.png]]
Con lo que vemos en la imagen sucederá que todos los párrafos se modificaran a azul ya que es tan abstracto que tomara todas las etiquetas p, y sucede lo mismo con cualquier otra etiqueta.
## Selector de clase
De manera sencilla un selector de clase modifica a partir de la clase todos sus elementos o un elemento especifico:
![[Pasted image 20250124161246.png]]
Otro dato interesante de las clases es que están son re utilizables por lo que podemos usarlas las beses que queramos. Por ultimo cuando se usa clases como podemos ver comenzamos con un punto.
## Selectores de ID
De una forma muy simple es lo mismo que una clase pero no es re utilizable por lo que solo podremos tener una vez cada id:
![[Pasted image 20250124161315.png]]
Como podemos ver en este caso los selectores de id los vamos a comenzar con un #.

## Selectores de Atributos
En este caso se seleccionan los elementos basados en algún atributo que tengan:
![[Pasted image 20250124161348.png]]
Como vemos esto es algo muy extraño pero con el tiempo y la practica lo comprenderemos y usaremos

## Combinación de descendentes
Bueno esto no es mas que seleccionar como un ejemplo a una clase dentro de otra clase:
![[Pasted image 20250124161421.png]]
Lo que hace es buscar la clase de cliente y luego busca la clase de nombre y aplica los cambios.

## Todos los Hijos
Este tipo se selector lo que hace es modificar todos los hijos específicos de una clase, id o etiqueta:
![[Pasted image 20250124161442.png]]
Es sencillo en este caso dice que modificara a todos los párrafos que pertenezcan a la clase cliente y se puede modificar cualquier otra etiqueta