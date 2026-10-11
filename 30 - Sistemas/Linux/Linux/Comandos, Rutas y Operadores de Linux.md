---
aliases:
  - Comandos Linux
  - Rutas Linux
  - Operadores Linux
  - Bash Comandos
tags:
  - linux/comandos
  - scripting/bash
estado: 🟢 Terminado
---
# Comandos, Rutas y Operadores de Linux

A todos estos comandos, rutas y operadores podemos incluir los  y, además, para más información sobre las rutas, la .

## Comandos Esenciales de Linux

| Comando | Descripción |
|---|---|
| `whoami` | Muestra el nombre de usuario efectivo actual. |
| `id` | Muestra la identidad del usuario (UID, GID y grupos a los que pertenece). |
| `sudo su` | Permite cambiar al usuario `root` (o a otro usuario) con privilegios de superusuario, si el usuario actual está en el grupo `sudoers`. |
| `sudo <comando>` | Ejecuta un comando específico con privilegios de superusuario. |
|> [!info] Archivo `sudoers`
|> El archivo `/etc/sudoers` (editado con `visudo`) define qué usuarios o grupos pueden ejecutar qué comandos como `root` o como otros usuarios, y bajo qué condiciones. Es crucial para la gestión de privilegios en sistemas Linux.
|
| `which` | Localiza la ruta absoluta de un comando ejecutable. |
| `echo "texto"` | Imprime una cadena de texto o variables en la salida estándar. |
| `echo $<variable>` | Muestra el contenido de una variable de entorno del sistema (ej. `PATH`, `SHELL`). |
| `grep "Patrón"` | Filtra líneas de texto que coinciden con un patrón. Parámetros comunes: `-n` (número de línea), `-v` (invertir coincidencia), `-r` (recursivo), `-E` (expresiones regulares extendidas), `-c` (contar coincidencias), `-l` (listar archivos con coincidencias). |
|> [!tip] Expresiones Regulares con `grep`
|> `grep` es extremadamente potente cuando se combina con expresiones regulares (regex). El parámetro `-E` (o `egrep`) permite usar expresiones regulares extendidas, facilitando patrones más complejos para la búsqueda de texto.
|
| `cd` | Cambia el directorio de trabajo actual. Combinaciones: `..` (directorio padre), `~` (directorio personal), `-` (directorio anterior), `nombre_directorio`. |
| `ls` | Lista el contenido de un directorio. Parámetros comunes: `-a` (todos los archivos, incluyendo ocultos), `-l` (formato largo), `-lh` (formato largo legible para humanos), `-lah` (todos, largo, legible). |
| `whereis <binario>` | Localiza los archivos binarios, de código fuente y de manual de un comando. |
| `find <archivos o directorios>` | Busca archivos y directorios en una jerarquía de directorios. |
| `whatis <comando>` | Muestra una descripción concisa de un comando de manual. |
| `touch <nombre>` | Crea un archivo vacío o actualiza la marca de tiempo de un archivo existente. |
| `nvim, nano` | Editores de texto basados en terminal que permiten crear y editar archivos. |
| `mkdir <nombre>` | Crea un nuevo directorio. |
| `rmdir <nombre>` | Elimina un directorio vacío. |
| `rm <archivo o directorio>` | Elimina archivos o directorios. Para directorios no vacíos, se usa con `-r` (recursivo) y `-f` (forzar). |
|> [!warning] Precaución con `rm -rf`
|> El comando `rm -rf` es muy destructivo. `-r` (recursivo) elimina directorios y su contenido, y `-f` (forzar) ignora archivos inexistentes y no pide confirmación. Un error tipográfico puede llevar a la pérdida irrecuperable de datos críticos del sistema. ¡Úsalo con extrema precaución!
|
| `mv <origen> <destino>` | Mueve o renombra archivos y directorios. |
| `disown` | Desvincula un proceso de la terminal que lo inició, permitiendo que continúe ejecutándose en segundo plano incluso después de cerrar la sesión. |
|> [!info] `disown` vs `nohup`
|> Mientras `disown` desvincula un proceso ya iniciado de la terminal, `nohup` se usa para iniciar un comando de manera que no se vea afectado por el cierre de la terminal. Ambos permiten que los procesos continúen ejecutándose en segundo plano, pero `nohup` también redirige la salida a `nohup.out` por defecto.
|
| `pwd` | Muestra el directorio de trabajo actual (Print Working Directory). |
| `useradd` | Crea una nueva cuenta de usuario. Opciones: `-s` (especificar shell de inicio), `-d` (especificar directorio personal). |
| `chgrp` | Cambia el grupo propietario de un archivo o directorio. |
| `chown` | Cambia el propietario de un archivo o directorio. También permite cambiar el propietario y el grupo simultáneamente con `chown usuario:grupo <archivo/directorio>`. |
| `groupadd` | Crea un nuevo grupo en el sistema. |
| `xargs` | Construye y ejecuta líneas de comandos a partir de la entrada estándar. |
|> [!tip] Potencial de `xargs`
|> `xargs` es fundamental para construir pipelines complejos. Permite tomar la salida de un comando (generalmente una lista de elementos) y usarla como argumentos para otro comando. Por ejemplo, `find . -name "*.log" | xargs rm` eliminaría todos los archivos `.log` encontrados.
|
| `file` | Determina el tipo de un archivo. |
| `sort` | Ordena líneas de archivos de texto. |
| `uniq` | Reporta o filtra líneas repetidas en un archivo. El parámetro `-u` muestra solo las líneas únicas. |
| `strings` | Imprime las secuencias de caracteres imprimibles de un archivo (útil para binarios). |
| `base64` | Codifica o decodifica datos en Base64. `-d` para decodificar. |
| `xxd -ps` | Crea un volcado hexadecimal de un archivo. `-ps` para formato 'plain hexdump'. `-r` para revertir (convertir de hexadecimal a binario). |
| `tee` | Lee la entrada estándar y la escribe tanto en la salida estándar como en uno o más archivos. |
| `sponge` | Lee toda la entrada antes de escribirla en la salida, útil para redirigir la salida de un comando al mismo archivo que se está leyendo. |
|> [!tip] `sponge` para Redirección Segura
|> A diferencia de `>` (redirección estándar), que trunca el archivo de destino antes de que el comando lea su contenido, `sponge` (del paquete `moreutils`) lee toda la entrada antes de escribirla. Esto es crucial cuando se quiere modificar un archivo "in-place" con comandos como `grep` o `sed`, evitando la pérdida de datos. Ejemplo: `grep -v "patron" archivo.txt | sponge archivo.txt`.
|
| `diff` | Compara dos archivos línea por línea y muestra las diferencias. |
| `more` | Muestra el contenido de un archivo pantalla por pantalla. |

## Rutas Básicas del Sistema de Archivos

| Ruta | Descripción |
|---|---|
| `/etc/group` | Contiene información sobre los grupos de usuarios del sistema. |
|> [!info] Formato de `/etc/group`
|> Similar a `/etc/passwd`, cada línea en `/etc/group` describe un grupo. Los campos son: `nombre_grupo:x:GID:miembros`. La 'x' indica que las contraseñas de grupo (si existen) están en `/etc/gshadow`. Los `miembros` son una lista de usuarios adicionales que pertenecen a ese grupo.
|
| `/etc/shells` | Lista los shells válidos disponibles en el sistema. |
| `/etc/passwd` | Almacena información sobre las cuentas de usuario locales, incluyendo nombre de usuario, UID, GID, directorio personal y shell por defecto. |
|> [!info] Formato de `/etc/passwd`
|> Cada línea en `/etc/passwd` representa una cuenta de usuario y está dividida por dos puntos (`:`). Los campos son: `nombre_usuario:x:UID:GID:comentario:directorio_personal:shell`. La 'x' indica que la contraseña está almacenada en `/etc/shadow` por seguridad.
|

## Control de Flujo y Operadores

### Operadores de Control de Flujo

| Operador | Descripción |
|---|---|
| `comando1; comando2` | Ejecuta `comando1` y luego `comando2`, independientemente del éxito del primero. |
| `comando1 && comando2` | Ejecuta `comando2` solo si `comando1` se ejecuta con éxito (código de salida 0). |
| `comando1 || comando2` | Ejecuta `comando2` solo si `comando1` falla (código de salida distinto de 0). |
