---
aliases:
  - smbclient
tags:
  - hacking/herramientas
  - linux/comandos
  - redes/protocolos
estado: 🟢 Terminado
---

# smbclient: Cliente de Línea de Comandos para SMB/CIFS

`smbclient` es un cliente de línea de comandos para sistemas SMB/CIFS que permite acceder a recursos compartidos de red, transferir archivos y administrar servicios SMB como si fuera un sistema de archivos local. Es una herramienta fundamental para la interacción con servidores Windows y Samba desde entornos Linux/Unix.

## Opciones Principales de `smbclient`

### Opciones de Conexión

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-L` | Listar recursos compartidos del servidor | `smbclient -L //servidor` |
| `-U` | Especificar usuario y, opcionalmente, contraseña | `smbclient -U usuario%password` |
| `-W` | Especificar workgroup o dominio | `smbclient -W WORKGROUP` |
| `-I` | Especificar la dirección IP del servidor | `smbclient -I 192.168.1.100` |
| `-p` | Especificar el puerto TCP (por defecto es 445) | `smbclient -p 445` |
| `-N` | Conexión sin requerir contraseña (acceso anónimo) | `smbclient -N` |
| `-A` | Archivo de autenticación para credenciales | `smbclient -A authfile` |
| `-k` | Usar autenticación Kerberos | `smbclient -k` |

> [!info] Enumeración con `-L`
> La opción `-L` es crucial para la fase de reconocimiento. Permite enumerar los recursos compartidos (shares) disponibles en un servidor SMB sin necesidad de autenticación si el servidor lo permite, o con credenciales si son necesarias. Esto revela posibles puntos de acceso o información sensible.
>
> > [!example] Ejemplo de enumeración
> > ```bash
> > smbclient -L //192.168.1.100 -N
> > ```
> > Este comando intentará listar los shares de `192.168.1.100` de forma anónima.
>
> [!tip] Formato de usuario y contraseña
> Para `-U`, el formato `usuario%password` es útil para scripts. Si solo se proporciona `usuario`, `smbclient` solicitará la contraseña interactivamente.
>
> [!info] Acceso Anónimo con `-N`
> La opción `-N` es vital para probar accesos anónimos a shares. Muchos servidores mal configurados permiten el acceso anónimo a shares como `IPC$` o `NETLOGON`, que pueden contener información valiosa.
>
> [!info] Archivo de Autenticación (`-A`)
> Un archivo de autenticación (`authfile`) es un archivo de texto plano que contiene las credenciales en el formato:
> ```
> username = <nombre_de_usuario>
> password = <contraseña>
> domain = <dominio_o_workgroup>
> ```
> Esto es útil para automatización y para evitar que las credenciales aparezcan en el historial de comandos. Asegúrate de proteger este archivo adecuadamente.
>
> [!info] Autenticación Kerberos (`-k`)
> La opción `-k` permite a `smbclient` utilizar tickets Kerberos para la autenticación, lo cual es común en entornos de Active Directory. Requiere que el sistema cliente esté configurado para Kerberos (por ejemplo, con `kinit`).

### Opciones de Comportamiento

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-c` | Ejecutar un comando interno y salir | `smbclient //servidor/share -U user -c "ls; get file.txt"` |
| `-t` | Establecer un tiempo de espera (timeout) para la conexión | `smbclient -t 30` |
| `-d` | Establecer el nivel de depuración (debug) | `smbclient -d 3` |
| `-s` | Especificar un archivo de configuración `smb.conf` alternativo | `smbclient -s smb.conf` |
| `-D` | Establecer el directorio inicial al conectarse al share | `smbclient //servidor/share -D /sub/dir` |
| `-T` | Utilizar comandos de `tar` para operaciones de backup/restore | `smbclient //servidor/share -T` |

> [!tip] Automatización con `-c`
> La opción `-c` es extremadamente útil para la automatización de tareas. Permite ejecutar una secuencia de comandos internos de `smbclient` sin entrar en el modo interactivo, lo que es ideal para scripts de extracción de datos o subida de payloads. Los comandos se separan por punto y coma (`;`).

## Comandos Internos de `smbclient`

Una vez conectado a un recurso compartido (share), `smbclient` ofrece una interfaz de línea de comandos interactiva con los siguientes comandos internos:

### Comandos de Navegación

| Comando | Descripción | Ejemplo |
|---|---|---|
| `ls` | Listar archivos y directorios | `ls` |
| `cd` | Cambiar el directorio de trabajo actual | `cd directorio` |
| `pwd` | Mostrar el directorio de trabajo actual | `pwd` |
| `mkdir` | Crear un nuevo directorio | `mkdir nuevo_dir` |
| `rmdir` | Eliminar un directorio vacío | `rmdir viejo_dir` |

### Comandos de Archivos

| Comando | Descripción | Ejemplo |
|---|---|---|
| `get` | Descargar un archivo del share al sistema local | `get archivo.txt` |
| `put` | Subir un archivo del sistema local al share | `put archivo.txt` |
| `mget` | Descargar múltiples archivos (soporta wildcards) | `mget *.txt` |
| `mput` | Subir múltiples archivos (soporta wildcards) | `mput *.txt` |
| `rename` | Renombrar un archivo o directorio | `rename viejo_nombre nuevo_nombre` |
| `rm` | Eliminar un archivo | `rm archivo.txt` |

> [!tip] Transferencia de Archivos
> Los comandos `get` y `put` son esenciales para la exfiltración y la implantación de archivos. `mget` y `mput` son muy eficientes para manejar grupos de archivos, especialmente cuando se utilizan wildcards (`*`).

### Comandos de Información

| Comando | Descripción | Ejemplo |
|---|---|---|
| `du` | Mostrar el uso de disco del directorio actual | `du` |
| `df` | Mostrar el espacio libre en el share | `df` |
| `allinfo` | Mostrar toda la información disponible sobre un archivo o directorio | `allinfo archivo.txt` |
| `setmode` | Cambiar los permisos de un archivo o directorio | `setmode archivo +r` |

### Comandos Avanzados

| Comando | Descripción | Ejemplo |
|---|---|---|
| `tar` | Realizar operaciones de backup/restore utilizando el formato `tar` | `tar c backup.tar *` |
| `blocksize` | Establecer el tamaño de bloque para transferencias | `blocksize 1024` |
| `printmode` | Establecer el modo de impresión (ej. `text`, `raw`) | `printmode text` |
| `queue` | Mostrar la cola de trabajos de impresión | `queue` |

> [!warning] Uso de `tar` para Exfiltración/Infiltración
> El comando `tar` dentro de `smbclient` es una característica potente para transferir grandes volúmenes de datos o estructuras de directorios completas. Puede ser utilizado para crear un archivo `tar` en el servidor (`tar c backup.tar *`) o para extraer uno (`tar x backup.tar`). Esto es particularmente útil en escenarios de pentesting para exfiltrar directorios completos de forma eficiente.