------
Bueno vamos a tener que comprender es que vim es un editor de texto con el cual vamos a trabajar para muchas cosas, este editor de texto podemos movernos sin necesidad de presionar el teclado, es así que es mucho mas rápido realizar modificación o crear código en este editor.

Vamos a ver varios comandos con los cuales podemos hacer muchas cosas.

| Comandos   | Descripción                                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------------------ |
| i          | Nos permite entrar en modo escritura para nosotros poder obvio escribir, el cursor lo pone por la izquierda. |
| a          | Nos permite entrar en modo escritura para nosotros poder obvio escribir, el cursor lo pone por la derecha.   |
| h, j, k, l | Nosotros podemos movernos siempre con esas letras cuando estamos en modo normal                              |
| jk         | Nos permite pasar a modo normal.                                                                             |
| 0          | Nos movemos al principio de la linea.                                                                        |
| $          | Nos movemos al final de la linea.                                                                            |
| w          | Nos movemos entre palabras para la derecha                                                                   |
| d          | Nos permite borrar todo lo que nosotros seleccionemos o dependiendo de la combinación de teclas              |
| dd         | Borramos directamente una linea.                                                                             |
| v          | Esto desde el modo normal nos lleva al modo visual.                                                          |
| o          | Nos crea una nueva linea                                                                                     |
| q + a      | Nos permite comenzar a grabar un macro, ósea una acción, con q dejamos de grabar y el macro se guarda en a.  |
| @<valor>   | con esto podemos usar los macros, el valor es la letra que usamos para guardar                               |
| /<valor>   | Con esto podemos hacer una búsqueda solo en el modo normal.                                                  |
| :          | Podemos crear expresiones como: :%s/nologin/yeslogin/g. → sustituimos todo lo de nologin a yeslogin.         |

Lo anterior es solo un poco de lo que podemos hacer con nvim, pero si queremos mas podemos ver el siguiente link:
[Vim Cheat Sheet](https://vim.rtorr.com/)