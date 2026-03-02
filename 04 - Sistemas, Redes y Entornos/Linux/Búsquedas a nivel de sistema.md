---
aliases:
  - Búsquedas de sistema
  - Comandos de búsqueda
  - Filtrado de archivos
tags:
  - linux/comandos
  - scripting/bash
estado: 🟢 Terminado
---
# Búsquedas a nivel de sistema

Muchas veces vamos a querer filtrar o buscar archivos con ciertas características descriptivas, como permisos especiales y más. Para realizar este tipo de búsquedas, uno de los mejores comandos es `find`. Su uso es sencillo y presentaremos algunos ejemplos explicados para una mejor comprensión. Para ello, utilizaremos conceptos de  y  que nos permitirán filtrar y obtener archivos con .

> [!info] El comando `find`
> `find` es una herramienta fundamental en sistemas Unix/Linux para localizar archivos y directorios basándose en una amplia gama de criterios (nombre, tipo, tamaño, fecha de modificación, permisos, propietario, etc.). Es extremadamente potente y flexible, permitiendo ejecutar acciones sobre los resultados encontrados.

## Ejemplos:

```bash
find / -name passwd 2>/dev/null
```

Este comando busca, desde la raíz (`/`), todos los archivos que contengan el nombre `passwd`. Además, al no poder acceder a ciertas rutas, se redirige la salida de error estándar (`stderr`) a `/dev/null` para evitar que los mensajes de error se muestren en pantalla.

> [!tip] Redirección de errores
> La redirección `2>/dev/null` es una práctica común para suprimir mensajes de error (salida de error estándar, descriptor de archivo 2) que no son relevantes para el resultado deseado, especialmente en scripts o cuando se buscan archivos en directorios con permisos restringidos.

```bash
find / -name passwd 2>/dev/null | xargs ls -l
```

Este comando es similar al anterior, pero utiliza `xargs` para pasar cada resultado de `find` como argumento al comando `ls -l`, mostrando así los detalles (permisos, propietario, tamaño, fecha) de los archivos encontrados.

> [!info] Uso de `xargs`
> `xargs` es una utilidad que lee elementos de la entrada estándar, los construye en líneas de comandos y luego ejecuta esos comandos. Es particularmente útil cuando la salida de un comando (como `find`) genera una lista de elementos que deben ser procesados por otro comando que espera sus argumentos en la línea de comandos.

```bash
find / -group stark -type d 2>/dev/null 
```

Este comando busca todos los elementos que pertenezcan al grupo `stark` y sean de tipo directorio (`-type d`). Si se desean buscar archivos regulares, se usaría la letra `f` (`-type f`).

> [!tip] Opciones de `-type` en `find`
> Las opciones comunes para `-type` incluyen:
> *   `f`: Archivo regular
> *   `d`: Directorio
> *   `l`: Enlace simbólico
> *   `b`: Archivo de bloque
> *   `c`: Archivo de carácter
> *   `p`: Tubería con nombre (FIFO)
> *   `s`: Socket

```bash
find / -user root -writable 2>/dev/null
```

Este comando busca archivos y directorios que pertenezcan al usuario `root` y que tengan permisos de escritura (`-writable`).

> [!info] Opciones de permisos en `find`
> Además de `-writable`, `find` ofrece:
> *   `-readable`: Busca archivos a los que el usuario actual tiene permiso de lectura.
> *   `-executable`: Busca archivos a los que el usuario actual tiene permiso de ejecución.
> *   `-perm <modo>`: Busca archivos con permisos exactos o mínimos especificados en formato octal (ej. `-perm 644` o `-perm /u=w,g=r`).

```bash
find / -name dex\* 2>/dev/null 
```

Este comando busca archivos cuyo nombre comience con `dex` seguido de cualquier secuencia de caracteres (`*`). La barra invertida (`\]`) escapa el asterisco para que sea interpretado literalmente por `find` y no por el shell.

> [!warning] Escapado de comodines
> Cuando se usan comodines (`*`, `?`, `[]`) con la opción `-name` de `find`, es crucial escaparlos con una barra invertida (`\]`) o encerrar el patrón entre comillas simples (`'patron*'`) para evitar que el shell los expanda antes de que `find` los procese.

```bash
find / -name \*ex\* 2>/dev/null
```

En este caso, se buscan archivos cuyo nombre contenga la secuencia `ex` en cualquier posición. Los asteriscos escapados (`\*`) actúan como comodines para cualquier secuencia de caracteres al principio y al final del patrón.

> [!info] Patrones de búsqueda en `find`
> La opción `-name` de `find` soporta patrones de shell (globbing) como `*` (cero o más caracteres), `?` (un solo carácter) y `[]` (rango de caracteres). Para patrones más complejos, se puede usar `-regex` con expresiones regulares.

Es importante tener en cuenta que `find` es un comando extremadamente potente y versátil. Siempre es recomendable consultar su manual (`man find`) o recursos adicionales para explorar todas sus capacidades, como el siguiente enlace: [Cómo usar los comandos find y locate en Linux](https://www.hostinger.es/tutoriales/como-usar-comando-find-locate-en-linux/)

## Expresiones regulares

> [!info] Expresiones Regulares (Regex)
> Las expresiones regulares son secuencias de caracteres que forman un patrón de búsqueda. Son herramientas poderosas para la manipulación de texto, permitiendo buscar, reemplazar y extraer cadenas de texto que coinciden con patrones complejos. Comandos como `grep`, `sed`, y `awk` hacen un uso extensivo de ellas.

### Grep

| Parámetro | Descripción                                                              |
| :-------- | :----------------------------------------------------------------------- |
| `-r`      | Realiza una búsqueda recursiva en directorios.                           |
| `\w`      | Coincide con caracteres alfanuméricos (letras, números y guiones bajos). |
| `-v`      | Invierte la coincidencia, mostrando las líneas que NO coinciden con el patrón. |
| `-vE`     | Combina `-v` (invertir coincidencia) con `-E` (usar expresiones regulares extendidas), permitiendo excluir líneas que coincidan con múltiples patrones. |
| `-n`      | Muestra el número de línea de cada coincidencia.                         |
| `-i`      | Ignora la distinción entre mayúsculas y minúsculas al buscar.            |

> [!tip] `grep` y expresiones regulares
> `grep` es una herramienta esencial para buscar patrones de texto en archivos. Por defecto, usa expresiones regulares básicas (BRE). Con la opción `-E` (o `egrep`), se pueden usar expresiones regulares extendidas (ERE), que ofrecen más funcionalidades como `+`, `?`, `|`, y `()`. 

### Tail and head

| Parámetro     | Descripción                                                              |
| :------------ | :----------------------------------------------------------------------- |
| `-n <numero>` | Muestra las últimas `<numero>` líneas del archivo (para `tail`) o las primeras `<numero>` líneas (para `head`). |

> [!info] `tail -f` para monitoreo
> La opción `-f` (follow) de `tail` es muy útil para monitorear archivos de registro en tiempo real. Muestra las últimas líneas del archivo y luego espera y muestra cualquier nueva línea que se añada al archivo.

### Cut

| Parámetro    | Descripción                                                      |
| :----------- | :--------------------------------------------------------------- |
| `-d “valor”` | Define el carácter delimitador a usar (ej. `-d ':'`).           |
| `-f <numero>`| Selecciona el campo o los campos especificados por `<numero>`. |

> [!tip] Uso de `cut`
> `cut` es ideal para extraer columnas o campos específicos de un archivo de texto o de la salida de un comando, basándose en delimitadores o posiciones de caracteres. Es muy eficiente para procesar datos tabulares.

### Awk

| Parámetro          | Descripción                                                              |
| :----------------- | :----------------------------------------------------------------------- |
| `‘{print $<valor>}`| Imprime el campo especificado por `<valor>` (ej. `$1` para el primer campo). |
| `FS=”<valor>”`     | Define el separador de campos de entrada (Field Separator).              |
| `‘NF{print $NF}’`  | Imprime el último campo de cada línea (`NF` es el número total de campos). |
| `NR==<numero>`     | Procesa solo la línea con el número de registro (`NR`) especificado.     |

> [!info] `awk` para procesamiento de texto avanzado
> `awk` es un lenguaje de programación de propósito especial diseñado para el procesamiento de texto basado en patrones. Es extremadamente potente para analizar y manipular datos estructurados, permitiendo realizar operaciones complejas como filtrado, formateo y cálculos.

### Tr

| Parámetro        | Descripción                                                              |
| :--------------- | :----------------------------------------------------------------------- |
| `‘algo’ ‘cambio’`| Sustituye los caracteres del primer conjunto por los caracteres correspondientes del segundo conjunto. |

> [!tip] `tr` para manipulación de caracteres
> `tr` (translate or delete characters) es una utilidad de línea de comandos que se utiliza para traducir o eliminar caracteres de la entrada estándar y escribir el resultado en la salida estándar. Es útil para cambiar mayúsculas a minúsculas, eliminar caracteres específicos o comprimir secuencias de caracteres repetidos.

### Find

| Parámetro   | Descripción                                                              |
| :---------- | :----------------------------------------------------------------------- |
| `-type`     | Busca por tipo de archivo (ej. `f` para archivo regular, `d` para directorio, `l` para enlace simbólico, `b` para bloque, `c` para carácter). |
| `-name`     | Busca archivos por nombre, aceptando comodines.                          |
| `-readable` | Busca archivos a los que el usuario actual tiene permiso de lectura.     |
| `-writable` | Busca archivos a los que el usuario actual tiene permiso de escritura.   |
| `-executable`| Busca archivos a los que el usuario actual tiene permiso de ejecución.   |
| `-size`     | Busca archivos por tamaño. Se puede especificar la unidad (ej. `c` para bytes, `k` para KB, `M` para MB, `G` para GB). |
| `-user`     | Busca archivos que pertenecen al usuario especificado.                   |
| `-group`    | Busca archivos que pertenecen al grupo especificado.                     |

> [!warning] Consideraciones de rendimiento con `find`
> Al usar `find` en el directorio raíz (`/`), especialmente sin opciones de filtrado específicas, puede consumir muchos recursos del sistema y tardar mucho tiempo, ya que debe recorrer todo el sistema de archivos. Es recomendable limitar el ámbito de búsqueda siempre que sea posible.

### Sed

| Parámetro                  | Descripción                                                              |
| :------------------------- | :----------------------------------------------------------------------- |
| `‘s/palabra/palabranueva/’`| Sustituye la primera ocurrencia de `palabra` por `palabranueva` en cada línea. |
| `‘s/palabra/palabranueva/g’`| Sustituye TODAS las ocurrencias de `palabra` por `palabranueva` en cada línea (`g` de global). |

> [!info] `sed` como editor de flujo
> `sed` (stream editor) es un potente editor de texto no interactivo que procesa texto línea por línea. Es ideal para transformaciones de texto automatizadas, como buscar y reemplazar, insertar o eliminar líneas, y filtrar contenido. Es una herramienta fundamental en scripting de shell.