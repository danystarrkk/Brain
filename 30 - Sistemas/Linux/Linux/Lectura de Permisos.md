---
aliases:
  - Permisos Linux
  - chmod
  - chattr
  - lsattr
  - SUID
  - SGID
  - Sticky Bit
  - Capabilities
tags:
  - linux/seguridad
  - linux/comandos
estado: 🟢 Terminado
---

# Lectura de Permisos

Cuando nos referimos a la lectura de permisos, nos referimos a identificar si podemos o no escribir, leer o ejecutar archivos o carpetas, así como a identificar el usuario propietario y el grupo al que pertenece. Para ello, utilizaremos algunos  y también .

Para poder ver los permisos de un archivo, haremos uso de:

```bash
ls -l
```

![[Pasted image 20240608214316.png]]

Identificaremos los permisos en la sección resaltada en rojo.
En nuestro caso, lo que tenemos es:
![[Pasted image 20240608214350.png]]
Primero, dividiremos esto en tres pares:
![[Pasted image 20240608214417.png]]
Observamos tres pares de tres caracteres, cada uno con letras que significan lo siguiente:

- `r` → leer (read)
- `w` → escribir (write)
- `x` → ejecutar (execute)

> [!info] El primer carácter de los permisos
> El primer carácter de la cadena de permisos (`-rwxrwxrwx`) indica el tipo de archivo:
> - `-`: Archivo regular
> - `d`: Directorio
> - `l`: Enlace simbólico (symlink)
> - `c`: Archivo de dispositivo de caracteres (ej. `/dev/tty`)
> - `b`: Archivo de dispositivo de bloque (ej. `/dev/sda`)
> - `p`: Tubería con nombre (named pipe o FIFO)
> - `s`: Socket de dominio Unix

En el caso de directorios, tanto `w` como `x` requieren atención especial. Si un directorio tiene el permiso `w` (escritura), significa que se pueden crear, eliminar o renombrar archivos *dentro* de él. El permiso `x` (ejecución) es lo que nos permite ingresar (atravesar) al directorio. Para archivos, su significado es más intuitivo.

> [!tip] Permisos `w` y `x` en directorios
> - El permiso `w` (escritura) en un directorio permite crear, eliminar y renombrar archivos *dentro* de ese directorio, no modificar el directorio en sí.
> - El permiso `x` (ejecución) en un directorio es esencial para poder *acceder* a su contenido, listarlo o atravesarlo. Sin `x`, no se puede entrar al directorio, incluso si se tienen permisos de lectura (`r`).

Ahora, centrémonos en nuestras tres secciones principales, que se dividen en:

- Propietario (User)
- Grupo (Group)
- Otros (Others)

![[Pasted image 20240608214458.png]]
El propietario y el grupo se identifican en la siguiente sección:
![[Pasted image 20240608214521.png]]
Con esto, ya comprendemos cómo identificar el grupo y el usuario. Es importante tener en cuenta que el usuario propietario no siempre pertenece al grupo principal del archivo.

# Asignación de Permisos

Para cambiar los permisos de archivos y directorios, utilizaremos el comando `chmod` con notación simbólica:

```bash
chmod <u,g,o,a> <+,-,=> <r,w,x,X,s,t> <archivo o carpeta>

# Ejemplo

chmod u+x file # Asignamos al usuario propietario el permiso de ejecución.
chmod o-w file # Quitamos el permiso de escritura para otros.

# Para asignar o quitar varios permisos simultáneamente:
chmod u-r,o+x,g+x file # Quitamos lectura al usuario, asignamos ejecución a otros y a los grupos.

# También se pueden combinar permisos para cada categoría:
chmod u-rx,o+xw,g+xr file
```

Con esto, comprendemos cómo modificar los permisos de un archivo de forma sencilla. Es fundamental saber que para modificar los permisos de usuario, se debe ser el propietario del archivo. Para modificar los permisos de grupo, se debe pertenecer al grupo asignado o ser el propietario. Para modificar los permisos de otros, se debe ser el propietario o tener privilegios de `root`.

> [!info] Notación simbólica vs. Octal en `chmod`
> - La **notación simbólica** (`u+x`, `g-w`) es más intuitiva para cambios específicos e incrementales, ya que permite añadir o quitar permisos sin afectar los demás.
> - La **notación octal** (`755`, `644`) es más concisa para establecer un conjunto completo de permisos de una sola vez, pero requiere calcular el valor numérico deseado. Ambas son igualmente válidas y se usan según la preferencia o el contexto.

## Asignación de Permisos Octales

Esta es otra forma de asignar permisos. Convertiremos los permisos a un formato binario (ceros y unos), donde `1` indica un permiso concedido y `0` un permiso denegado:
![[Pasted image 20240608214543.png]]
Ahora, asignamos índices a cada posición, comenzando desde `0` de derecha a izquierda:
![[Pasted image 20240608214603.png]]
Finalmente, consideramos solo las posiciones con un `1` y elevamos `2` al índice correspondiente:
![[Pasted image 20240608214620.png]]
Por último, sumamos los valores resultantes para obtener el número octal para esa categoría de permisos:
![[Pasted image 20240608214639.png]]
Ya con esto claro, hacemos lo demás y debería quedarnos de la siguiente forma:
![[Pasted image 20240608214700.png]]
Esta es la forma larga. Otra forma es asignarle un valor a cada permiso y luego sumarlo. Los valores quedan de la siguiente manera:

- `r` (read) → 4
- `w` (write) → 2
- `x` (execute) → 1

Con estos valores, solo debemos comprender que si el permiso está asignado, lo sumamos, y si no, no lo hacemos. Con esto es mucho más fácil sacar el valor.

Para asignar los permisos, lo hacemos de la siguiente manera:

```bash
chmod 755 <nombre_del_archivo> # El '755' se reemplaza por el número octal deseado para los permisos.
```

> [!tip] Valores octales comunes para `chmod`
> - `777`: `rwxrwxrwx` (Lectura, escritura y ejecución para todos. **¡Peligroso!** Evitar en producción.)
> - `755`: `rwxr-xr-x` (Propietario: rwx; Grupo y Otros: rx. Común para directorios y ejecutables.)
> - `644`: `rw-r--r--` (Propietario: rw; Grupo y Otros: r. Común para archivos de texto.)
> - `600`: `rw-------` (Solo propietario: rw. Privado.)
> - `700`: `rwx------` (Solo propietario: rwx. Privado y ejecutable.)

# Permisos Especiales – Sticky Bit

El `Sticky Bit` es un tipo de permiso especial que se puede asignar a directorios. Su función principal es evitar que los usuarios (excepto el propietario del archivo o el `root`) puedan eliminar o renombrar archivos dentro de ese directorio, incluso si tienen permisos de escritura sobre el directorio. Esto es común en directorios como `/tmp`.

Para asignar este permiso, se utiliza:

```bash
chmod +t <nombre_del_directorio>
```

Otra forma de asignarlo es utilizando la notación octal, añadiendo un `1` al principio de los permisos existentes:

```bash
chmod 1755 <nombre_del_directorio> # No es necesario cambiar los otros permisos del directorio; se pueden mantener los mismos, pero con el '1' delante para asignar el Sticky Bit.
```

> [!info] Uso común del Sticky Bit
> El `Sticky Bit` es fundamental en directorios compartidos como `/tmp`. Sin él, cualquier usuario con permisos de escritura en `/tmp` podría eliminar o modificar archivos de otros usuarios, comprometiendo la privacidad y la integridad del sistema. Con el `Sticky Bit`, solo el propietario del archivo o `root` puede eliminarlo.

# Control de Atributos de Ficheros en Linux – Chattr y Lsattr

Los atributos de ficheros son metadatos adicionales que controlan el comportamiento del sistema de archivos para un archivo o directorio específico. Se asignan con `chattr` y se listan con `lsattr`.

La forma de asignar estos atributos es la siguiente:

```bash
chattr +i -V <nombre_del_archivo_o_carpeta> # Asigna el atributo 'inmutable'. Un archivo con este atributo no puede ser modificado, eliminado, renombrado ni enlazado, incluso por el usuario root. El flag '-V' muestra la versión del archivo.

# Atributos comunes:
+i -> inmutable (immutable)
+a -> solo añadir (append only)
+c -> no comprimir automáticamente (no compress)
+u -> guardar datos en caso de borrado (undeletable)
+s -> borrar de forma segura (secure deletion - llena bloques con ceros)
+S -> actualizaciones síncronas (synchronous updates)
```

Para listar los atributos de un archivo o directorio, se utiliza:

```bash
lsattr <nombre_del_archivo_o_directorio>
```

> [!warning] Implicaciones de seguridad de `chattr +i`
> El atributo `+i` (inmutable) es extremadamente potente. Un archivo con este atributo no puede ser modificado, eliminado o renombrado, incluso por el usuario `root`. Esto puede ser útil para proteger archivos críticos del sistema, pero también puede ser utilizado por atacantes para dificultar la limpieza de un sistema comprometido o para asegurar la persistencia de malware. Siempre se debe usar con precaución.

---

# Permisos Especiales – SUID y SGID

## Permisos SUID (Set User ID)

Los permisos SUID son un tipo de permiso especial que permite que un binario o archivo ejecutable se ejecute con los privilegios del usuario propietario del archivo, en lugar de los privilegios del usuario que lo está ejecutando. Esto es crucial para comandos como `passwd`.

Para asignar este permiso, se utiliza:

```bash
chmod u+s <binario_o_archivo> # Asigna el permiso SUID al usuario propietario.

# Otra forma de asignarlo es con la notación octal:
chmod 4755 <binario_o_archivo> # Se antepone un '4' a los permisos octales existentes.
```

## Permisos SGID (Set Group ID)

Este tipo de permiso tiene dos funciones principales:

- **En un archivo ejecutable**: Permite que cualquier usuario que ejecute el archivo lo haga con los privilegios del grupo propietario del archivo, en lugar de los privilegios del grupo principal del usuario que lo ejecuta.
- **En un directorio**: Cualquier archivo o subdirectorio creado dentro de este directorio heredará automáticamente el grupo propietario del directorio padre, en lugar del grupo principal del usuario que lo crea.

Para asignar este permiso, se utiliza:

```bash
chmod g+s <binario_o_archivo_o_directorio> # Asigna el permiso SGID al grupo.

# Otra forma de asignarlo es con la notación octal:
chmod 2755 <binario_o_archivo_o_directorio> # Se antepone un '2' a los permisos octales existentes.
```

> [!warning] SUID y SGID como vectores de escalada de privilegios
> Los permisos SUID y SGID son características poderosas, pero también son vectores comunes para la escalada de privilegios si no se configuran correctamente. Un binario SUID/SGID mal configurado o vulnerable puede permitir a un atacante ejecutar código con los privilegios del propietario o grupo del archivo, lo que a menudo significa obtener privilegios de `root`. Es crucial auditar regularmente los binarios con estos permisos en busca de posibles vulnerabilidades.

# Privilegios Especiales - Capabilities

Las `Capabilities` (capacidades) son un mecanismo de seguridad de Linux que permite dividir los privilegios del usuario `root` en unidades más pequeñas y discretas. Esto significa que un programa puede realizar ciertas operaciones privilegiadas sin necesidad de ejecutarse con todos los privilegios de `root`. Por ejemplo, un programa puede tener la capacidad de enlazar a puertos bajos (`CAP_NET_BIND_SERVICE`) sin tener todas las capacidades de `root`.

Para listar las capacidades asignadas a un binario, se utiliza la herramienta `getcap`:

```bash
getcap <archivo>
```

Para asignar capacidades a un binario, se utiliza `setcap`:

```bash
setcap <capacidad> <archivo>
# Ejemplo: setcap 'cap_net_bind_service=+ep' /usr/bin/nginx
```

Para listar las capacidades, como se mencionó, se usa `getcap`.

> [!info] Modelo de seguridad de Capabilities
> El modelo de `Capabilities` busca implementar el principio de "mínimo privilegio". En lugar de otorgar todos los privilegios de `root` a un proceso que solo necesita una función específica (como abrir un puerto bajo), se le puede conceder solo esa capacidad (`CAP_NET_BIND_SERVICE`). Esto reduce la superficie de ataque y limita el daño potencial si el proceso es comprometido.
