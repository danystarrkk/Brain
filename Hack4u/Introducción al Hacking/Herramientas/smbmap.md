-----
**SMBMap** es una herramienta que permite enumerar shares SMB y permisos en dominios Windows. Es especialmente útil para pentesting y auditorías de seguridad, ya que puede identificar shares accesibles y permisos de escritura.

## Tabla de Opciones Principales de SMBMap

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-h`|Muestra ayuda|`smbmap -h`|
|`-H`|Host objetivo|`smbmap -H 192.168.1.100`|
|`-u`|Usuario|`smbmap -u usuario`|
|`-p`|Password|`smbmap -p password`|
|`-d`|Dominio|`smbmap -d DOMINIO`|
|`-P`|Puerto SMB|`smbmap -P 445`|
|`-v`|Modo verbose|`smbmap -v`|
|`-q`|Modo silencioso|`smbmap -q`|

### Opciones de Enumeración

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-L`|Listar shares|`smbmap -L`|
|`-r`|Listar recursivo|`smbmap -r SHARE`|
|`-R`|Listar recursivo profundo|`smbmap -R SHARE`|
|`-x`|Ejecutar comando|`smbmap -x "command"`|
|`--dir-only`|Solo directorios|`smbmap --dir-only`|

### Opciones de Archivos

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--download`|Descargar archivo|`smbmap --download FILE`|
|`--upload`|Subir archivo|`smbmap --upload LOCAL REMOTE`|
|`--delete`|Eliminar archivo|`smbmap --delete FILE`|
|`--skip`|Saltar shares default|`smbmap --skip`|

### Opciones de Output

| Opción    | Descripción       | Ejemplo                    |
| --------- | ----------------- | -------------------------- |
| `-o`      | Output file       | `smbmap -o resultados.txt` |
| `-f`      | Filter shares     | `smbmap -f "READ"`         |
| `-g`      | Group enumeration | `smbmap -g`                |
| `--admin` | Solo shares admin | `smbmap --admin`           |