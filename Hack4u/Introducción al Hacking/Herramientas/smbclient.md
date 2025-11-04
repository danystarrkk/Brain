----
**smbclient** es un cliente de línea de comandos para sistemas SMB/CIFS que permite acceder a recursos compartidos de red, transferir archivos, y administrar servicios SMB como si fuera un sistema de archivos local.

## Tabla de Opciones Principales de smbclient

### Opciones de Conexión

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-L`|Listar recursos del servidor|`smbclient -L //servidor`|
|`-U`|Especificar usuario|`smbclient -U usuario%password`|
|`-W`|Especificar workgroup|`smbclient -W WORKGROUP`|
|`-I`|Especificar IP del servidor|`smbclient -I 192.168.1.100`|
|`-p`|Especificar puerto|`smbclient -p 445`|
|`-N`|Conexión sin password|`smbclient -N`|
|`-A`|Archivo de autenticación|`smbclient -A authfile`|
|`-k`|Usar kerberos|`smbclient -k`|

### Opciones de Comportamiento

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-c`|Ejecutar comando y salir|`smbclient -c "ls"`|
|`-t`|Timeout de conexión|`smbclient -t 30`|
|`-d`|Nivel debug|`smbclient -d 3`|
|`-s`|Archivo de configuración|`smbclient -s smb.conf`|
|`-D`|Directorio inicial|`smbclient -D /share`|
|`-T`|Comandos de tar|`smbclient -T`|

## Comandos Internos de smbclient

Una vez conectado a un share, smbclient ofrece estos comandos internos:

### Comandos de Navegación

|Comando|Descripción|Ejemplo|
|---|---|---|
|`ls`|Listar archivos|`ls`|
|`cd`|Cambiar directorio|`cd directorio`|
|`pwd`|Mostrar directorio actual|`pwd`|
|`mkdir`|Crear directorio|`mkdir nuevo_dir`|
|`rmdir`|Eliminar directorio|`rmdir viejo_dir`|

### Comandos de Archivos

|Comando|Descripción|Ejemplo|
|---|---|---|
|`get`|Descargar archivo|`get archivo.txt`|
|`put`|Subir archivo|`put archivo.txt`|
|`mget`|Descargar múltiples|`mget *.txt`|
|`mput`|Subir múltiples|`mput *.txt`|
|`rename`|Renombrar|`rename viejo nuevo`|
|`rm`|Eliminar archivo|`rm archivo.txt`|

### Comandos de Información

|Comando|Descripción|Ejemplo|
|---|---|---|
|`du`|Uso de disco|`du`|
|`df`|Espacio libre|`df`|
|`allinfo`|Info de archivo|`allinfo archivo.txt`|
|`setmode`|Cambiar permisos|`setmode archivo +r`|

### Comandos Avanzados

|Comando|Descripción|Ejemplo|
|---|---|---|
|`tar`|Backup con tar|`tar c backup.tar *`|
|`blocksize`|Tamaño de bloque|`blocksize 1024`|
|`printmode`|Modo impresión|`printmode text`|
|`queue`|Cola de impresión|`queue`|