---
aliases:
  - SMBMap
tags:
  - hacking/herramientas
  - redes/comandos
estado: 🟢 Terminado
---
# SMBMap: Enumeración de Shares y Permisos SMB

**SMBMap** es una herramienta que permite enumerar shares SMB y permisos en dominios Windows. Es especialmente útil para pentesting y auditorías de seguridad, ya que puede identificar shares accesibles y permisos de escritura.

> [!info] Protocolo SMB
> Server Message Block (SMB) es un protocolo de red utilizado para proporcionar acceso compartido a archivos, impresoras y puertos serie entre nodos de una red. Es fundamental en entornos Windows y su configuración incorrecta puede llevar a fugas de información o acceso no autorizado. SMBMap interactúa directamente con este protocolo para descubrir recursos compartidos.

## Tabla de Opciones Principales de SMBMap

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-h` | Muestra la ayuda de la herramienta. | `smbmap -h` |
| `-H` | Especifica el host o la dirección IP objetivo. | `smbmap -H 192.168.1.100` |
| `-u` | Define el nombre de usuario para la autenticación. | `smbmap -u usuario` |
| `-p` | Proporciona la contraseña para el usuario especificado. | `smbmap -p password` |
| `-d` | Especifica el dominio de Windows para la autenticación. | `smbmap -d DOMINIO` |
| `-P` | Define el puerto SMB a utilizar (por defecto es 445). | `smbmap -P 445` |
| `-v` | Habilita el modo verbose, mostrando más detalles durante la ejecución. | `smbmap -v` |
| `-q` | Habilita el modo silencioso, suprimiendo la mayoría de los mensajes de salida. | `smbmap -q` |

> [!tip] Autenticación NTLM
> SMBMap soporta autenticación NTLM. Para entornos de Active Directory, es común usar el formato `DOMINIO\usuario` o especificar el dominio con `-d`. Si no se proporciona un usuario o contraseña, SMBMap intentará una conexión anónima o nula.

### Opciones de Enumeración

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-L` | Lista todos los shares SMB disponibles en el host objetivo. | `smbmap -L` |
| `-r` | Lista el contenido de un share específico de forma recursiva (un nivel de profundidad). | `smbmap -r SHARE` |
| `-R` | Lista el contenido de un share específico de forma recursiva profunda (todos los niveles). | `smbmap -R SHARE` |
| `-x` | Ejecuta un comando arbitrario en el share si se tienen los permisos adecuados. | `smbmap -x "dir"` |
| `--dir-only` | Muestra solo directorios al listar el contenido de un share. | `smbmap --dir-only` |

> [!warning] Ejecución de Comandos (`-x`)
> La opción `-x` es extremadamente potente y requiere permisos de escritura en el share o acceso a un servicio que permita la ejecución remota. Su uso debe ser cauteloso y siempre dentro de un entorno autorizado, ya que puede tener un impacto significativo en el sistema objetivo.

### Opciones de Archivos

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--download` | Descarga un archivo específico desde un share SMB. | `smbmap --download SHARE\\FILE.txt` |
| `--upload` | Sube un archivo local a un share SMB remoto. | `smbmap --upload /ruta/local/archivo.txt SHARE\\REMOTE_FILE.txt` |
| `--delete` | Elimina un archivo específico de un share SMB. | `smbmap --delete SHARE\\FILE.txt` |
| `--skip` | Omite la enumeración de shares predeterminados (como IPC$, ADMIN$, C$). | `smbmap --skip` |

> [!info] Shares Administrativos
> Shares como `IPC$`, `ADMIN$`, y `C$` son shares administrativos predeterminados en Windows. `IPC$` se usa para la comunicación entre procesos, `ADMIN$` mapea al directorio `Windows`, y `C$` mapea a la raíz del disco `C:`. A menudo, estos shares requieren credenciales administrativas para ser accedidos.

### Opciones de Salida (Output)

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-o` | Guarda la salida de SMBMap en un archivo especificado. | `smbmap -o resultados.txt` |
| `-f` | Filtra los shares mostrados basándose en permisos específicos (ej. "READ", "WRITE"). | `smbmap -f "READ"` |
| `-g` | Realiza una enumeración de grupos de usuarios en el sistema objetivo. | `smbmap -g` |
| `--admin` | Muestra solo los shares que requieren privilegios administrativos para su acceso. | `smbmap --admin` |

> [!tip] Filtrado de Permisos
> La opción `-f` es muy útil para identificar rápidamente shares con permisos de escritura (`-f "WRITE"`) o lectura (`-f "READ"`), lo cual es crucial para la fase de reconocimiento y escalada de privilegios.