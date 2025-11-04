-----
**Gobuster** es una herramienta de fuerza bruta escrita en Go que se utiliza principalmente para:

- Descubrimiento de directorios y archivos en servidores web
- Descubrimiento de subdominios
- Escaneo de DNS
- Fuerza bruta de S3 buckets, VHosts virtuales, y más

## Tabla de opciones principales de Gobuster

### Opciones Generales

| Opción       | Descripción                              | Ejemplo                             |
| ------------ | ---------------------------------------- | ----------------------------------- |
| `-h`         | Muestra ayuda                            | `gobuster -h`                       |
| `-v`         | Modo verbose                             | `gobuster dir -v`                   |
| `-q`         | Modo quiet (silencioso)                  | `gobuster dir -q`                   |
| `-t`         | Número de hilos (threads)                | `gobuster dir -t 50`                |
| `-w`         | Wordlist a usar                          | `gobuster dir -w wordlist.txt`      |
| `-o`         | Archivo de output                        | `gobuster dir -o results.txt`       |
| `-s`         | Códigos de estado a considerar válidos   | `gobuster dir -s 200,301,302`       |
| `-b`         | Códigos de estado a considerar inválidos | `gobuster dir -b 404`               |
| `-c`         | Cookies HTTP                             | `gobuster dir -c "session=abc123"`  |
| `-x`         | Extensiones a probar                     | `gobuster dir -x php,txt,html`      |
| `-a`         | User-Agent personalizado                 | `gobuster dir -a "Mozilla/5.0"`     |
| `-k`         | Ignorar verificación SSL                 | `gobuster dir -k`                   |
| `-n`         | No mostrar contenido (No status)         | `gobuster dir -n`                   |
| `-d`         | Mostrar longitud de contenido            | `gobuster dir -d`                   |
| `-U`         | Usuario para autenticación               | `gobuster dir -U admin`             |
| `-P`         | Password para autenticación              | `gobuster dir -P password`          |
| `-H`         | Headers personalizados                   | `gobuster dir -H "X-Header: value"` |
| `--timeout`  | Timeout de conexión                      | `gobuster dir --timeout 10s`        |
| `--delay`    | Delay entre requests                     | `gobuster dir --delay 100ms`        |
| `--wildcard` | Manejar wildcards DNS                    | `gobuster dns --wildcard`           |

### Modos de Operación

|Modo|Descripción|Ejemplo|
|---|---|---|
|`dir`|Descubrimiento de directorios/archivos|`gobuster dir -u http://example.com`|
|`dns`|Descubrimiento de subdominios|`gobuster dns -d example.com`|
|`s3`|Descubrimiento de buckets S3|`gobuster s3 -w wordlist.txt`|
|`vhost`|Descubrimiento de virtual hosts|`gobuster vhost -u http://example.com`|
|`fuzz`|Fuzzing de parámetros|`gobuster fuzz -u http://example.com/FUZZ`|
|`tftp`|Fuerza bruta TFTP|`gobuster tftp -w wordlist.txt`|

### Opciones Específicas por Modo

#### Modo DIR (Directorios)

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-u`|URL target|`gobuster dir -u http://example.com`|
|`-f`|Agregar slash al final|`gobuster dir -f`|
|`-l`|Mostrar longitud de contenido|`gobuster dir -l`|
|`--discover-backup`|Descubrir backups|`gobuster dir --discover-backup`|
|`--exclude-length`|Excluir por longitud|`gobuster dir --exclude-length 123`|

#### Modo DNS (Subdominios)

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-d`|Dominio target|`gobuster dns -d example.com`|
|`-r`|DNS server personalizado|`gobuster dns -r 8.8.8.8`|
|`-i`|Mostrar direcciones IP|`gobuster dns -i`|
|`--show-ips`|Mostrar IPs de subdominios|`gobuster dns --show-ips`|
|`--search`|Buscar subdominios en resultados|`gobuster dns --search`|

#### Modo S3 (Buckets AWS)

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--region`|Región AWS|`gobuster s3 --region us-east-1`|
|`--no-ssl`|No usar SSL|`gobuster s3 --no-ssl`|