---
aliases:
  - Vim
  - nvim
  - Editor de texto Vim
tags:
  - linux/comandos
estado: 🟢 Terminado
---
# Vim: Comandos Esenciales para la Edición Rápida

Para comprender Vim, es esencial saber que es un editor de texto con el que trabajaremos en diversas tareas. Este editor permite la navegación sin necesidad de usar el ratón, lo que agiliza significativamente la modificación o creación de código.

A continuación, exploraremos varios comandos que nos permitirán realizar múltiples acciones.

| Comandos   | Descripción                                                                                                  |
| ---------- | ------------------------------------------------------------------------------------------------------------ |
| `i`          | Nos permite entrar en modo inserción (escritura) para introducir texto. El cursor se posiciona a la izquierda del carácter actual. |
| `a`          | Nos permite entrar en modo inserción (escritura) para introducir texto. El cursor se posiciona a la derecha del carácter actual.   |
| `h, j, k, l` | Permiten la navegación en modo normal: `h` (izquierda), `j` (abajo), `k` (arriba), `l` (derecha).                              |
| `jk`         | Nos permite salir del modo inserción y volver al modo normal.                                                                             |
| `0`          | Mueve el cursor al principio de la línea actual.                                                                        |
| `$`          | Mueve el cursor al final de la línea actual.                                                                        |
| `w`          | Mueve el cursor una palabra hacia la derecha.                                                                   |
| `d`          | Permite borrar texto. Su comportamiento depende de la combinación de teclas (ej. `dw` borra una palabra, `d$` borra hasta el final de la línea). |
| `dd`         | Borra la línea completa donde se encuentra el cursor.                                                                       |
| `v`          | Desde el modo normal, activa el modo visual, permitiendo seleccionar texto para operaciones como copiar, cortar o pegar.                                                          |
| `o`          | Crea una nueva línea debajo de la actual y entra en modo inserción.                                                              |
| `q + a`      | Inicia la grabación de una macro en el registro `a`. Presionar `q` nuevamente detiene la grabación. La macro se guarda en el registro especificado (en este caso, `a`).  |
| `@<valor>`   | Ejecuta la macro guardada en el registro `<valor>` (ej. `@a` ejecuta la macro guardada en `a`).                               |
| `/<valor>`   | Realiza una búsqueda hacia adelante del patrón `<valor>` en el modo normal. Presiona `n` para la siguiente coincidencia y `N` para la anterior.                               |
| `:`          | Activa la línea de comandos para ejecutar comandos de Vim. Ejemplo: `: %s/nologin/yeslogin/g` sustituye todas las ocurrencias de 'nologin' por 'yeslogin' en todo el archivo (`%` para todo el archivo, `g` para todas las ocurrencias en cada línea).         |

> [!tip]
> Para salir del modo inserción y volver al modo normal, la tecla `Esc` es el método estándar y más universalmente reconocido en Vim, además de la combinación `jk`.

> [!info]
> Las macros en Vim (`q + <registro>`) son extremadamente potentes para automatizar tareas repetitivas. Puedes grabar una secuencia de comandos y luego reproducirla múltiples veces, incluso en diferentes líneas o archivos, lo que mejora drásticamente la eficiencia en la edición de texto.

> [!info]
> El comando de sustitución (`:%s/patron/reemplazo/g`) utiliza expresiones regulares (regex) para buscar y reemplazar texto. El `%` indica que la operación se aplica a todo el archivo, y la `g` (global) asegura que todas las ocurrencias del patrón en cada línea sean reemplazadas, no solo la primera.

Lo anterior es solo una introducción a las capacidades de Vim/Neovim. Para una referencia más exhaustiva, consulte el siguiente enlace:
[Vim Cheat Sheet](https://vim.rtorr.com/)