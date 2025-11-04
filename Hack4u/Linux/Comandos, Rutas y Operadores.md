-------
A todos estos comandos Rutas y Operadores podemos incluir los [[Comandos Útiles]] y ademas para mas información sobre las rutas la [[Estructura de Directorios]]

|Comandos|Descripción|
|---|---|
|whoami|Permite ver el usuario actual en uso.|
|id|Nos permite ver los grupos asignados del usuario en uso.|
|sudo su|Nos permite acceder al usuario root (solo si estamos en el grupo sudo).|
|sudo <comando>|Nos permite ejecutar comandos como root.|
|which|Nos permite ver las rutas absolutas de los binarios.|
|echo “lo que sea”|Nos permite imprimir en la consola todo lo que este en paréntesis.|
|echo $<variable>|Tenemos diferentes variables en el sistema como son PATH, SHELL, etc. Nos permite ver el contenido|
|grep “Palabra”|Nos permite filtrar por diferentes palabras y tiene deferentes parámetros como: -n, -v, -r, -E, -c, -l|
|cd|Es un comando que nos permite movernos entre directorios y podemos combinarlo con: .. , ~, -, nombre del directorio, etc.|
|ls|Nos permite listar archivos y directorios y podemos combinarlo con: -a, -l, -lh, -lah.|
|whereis <binario>|Nos permite buscar todo lo relacionado con el binario.|
|find <archivos o directorios>|Nos permite buscar por archivos o directorios|
|whatis <comando>|Nos permite obtener un descripción pequeña del comando|
|touch <nombre>|Nos permite crear un archivo sin abrirlo.|
|nvim, nano|Nos permite crear un archivo y abrirlo al tiempo|
|mkdir <nombre>|crear un nuevo directorio|
|rmdir <nombre>|Eliminar directorio vacío|
|rm <archivo o directorio>|Nos permite eliminar archivos, y para eliminar directorios podemos combinarlo con: -rf.|
|mv <archivo> <ruta a donde mover>|Nos permite mover archivos|
|disown|Nos permite independizar procesos.|
|pwd|Nos permite ver la ruta absoluta en la que nos encontramos.|
|useradd|nos permite crear un usuario en el sistema, con: -s asignarle un shell y con -d una ruta para el usuario|
|chgrp|Nos permite asignar a una carpeta o archivos asignarle un nuevo usuario|
|chown|Nos permite modificar a un archivo o carpeta el usuario, pero también podemos asignar tanto el usuario y grupo de la siguiente forma: chown user:group <archivo o carpeta>|
|groupadd|Nos permite crear nuevo grupos al sistema|
|xargs|nos permite ejecutar comandos sobre el resultado de otros|
|file|Nos permite identificar el tipo de archivo que es.|
|sort|Nos permite realizar un ordenamiento a las cadenas|
|uniq|Nos permite listar las cadenas únicas siempre y cuando pongamos el parámetro -u|
|strings|Nos permite en el caso de ser binarios o archivos extraños ver las cadenas legibles.|
|base64|Nos permite codificar texto en base 64 y con -d podemos decodificar.|
|xxd -ps|Nos permite pasar información a hexadecimal y podemos hacer lo inverso con el parámetro -r.|
|tee|Nos permite crear un archivo con el contenido que se imprima en pantalla.|
|sponge|Nos permite redirigir data para el mismo archivo el cual estamos viendo.|
|diff|Nos permite ver las diferencias entre dos archivos|
|more||
|||
|||
|||

## Rutas Básicas

|Rutas|Descripción|
|---|---|
|/etc/group|En la ruta encontramos todos los grupos existentes en el sistema.|
|/etc/shells|Nos permite ver los tipos de shells instaladas en el sistema.|
|/etc/passwd|Nos permite ver todos los usuario, rutas de los usuarios e identificadores.|
|||
|||
|||

## Control de Flujo
### Operadores

|operador|Descripción|
|---|---|
|comando1; comando2|Esto nos ejecuta los comandos al tiempo.|
|comando1 && comando2|Esto nos ejecuta comando2 solo si comando1 es exitoso.|
|comando1||
