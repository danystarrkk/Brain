---
aliases: Gobuster
tags:
  - hacking/herramientas
  - hacking/reconocimiento
estado: 🟢 Terminado
---
# Gobuster

**Gobuster** es una herramienta de fuerza bruta escrita en Go que se utiliza principalmente para:

- Descubrimiento de directorios y archivos en servidores web
- Descubrimiento de subdominios
- Escaneo de DNS
- Fuerza bruta de S3 buckets, VHosts virtuales, y más

> [!info] Rendimiento y Aplicación
> Gobuster, al estar escrito en Go, es conocido por su alta velocidad y eficiencia, lo que lo convierte en una opción popular para tareas de enumeración rápida en entornos de pentesting y CTFs. Es una herramienta fundamental en la fase de reconocimiento activo para mapear la superficie de ataque de una aplicación web o infraestructura.

## Tabla de Opciones Principales de Gobuster

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

> [!tip] Gestión de Conexiones
> Las opciones `--timeout` y `--delay` son cruciales para evitar la detección por sistemas de seguridad (WAFs, IPS/IDS) y para manejar la carga en el servidor objetivo. Un `delay` adecuado puede simular un comportamiento más humano, mientras que un `timeout` evita que el escaneo se quede colgado en conexiones lentas.

> [!info] Wildcards DNS
> Un wildcard DNS (`*.example.com`) permite que cualquier subdominio no definido explícitamente apunte a una dirección IP específica. La opción `--wildcard` en Gobuster DNS es vital para filtrar estos resultados y evitar falsos positivos, ya que de lo contrario, cada entrada de la wordlist podría resolver a la misma IP del wildcard.

### Modos de Operación

| Modo    | Descripción                       | Ejemplo                             |
| ------- | --------------------------------- | ----------------------------------- |
| `dir`   | Descubrimiento de directorios/archivos | `gobuster dir -u http://example.com`|
| `dns`   | Descubrimiento de subdominios     | `gobuster dns -d example.com`       |
| `s3`    | Descubrimiento de buckets S3      | `gobuster s3 -w wordlist.txt`       |
| `vhost` | Descubrimiento de virtual hosts   | `gobuster vhost -u http://example.com`|
| `fuzz`  | Fuzzing de parámetros             | `gobuster fuzz -u http://example.com/FUZZ`|
| `tftp`  | Fuerza bruta TFTP                 | `gobuster tftp -w wordlist.txt`     |

### Opciones Específicas por Modo

#### Modo DIR (Directorios)

| Opción            | Descripción                       | Ejemplo                             |
| ----------------- | --------------------------------- | ----------------------------------- |
| `-u`              | URL target                        | `gobuster dir -u http://example.com`|
| `-f`              | Agregar slash al final            | `gobuster dir -f`                   |
| `-l`              | Mostrar longitud de contenido     | `gobuster dir -l`                   |
| `--discover-backup`| Descubrir backups                 | `gobuster dir --discover-backup`    |
| `--exclude-length`| Excluir por longitud              | `gobuster dir --exclude-length 123` |

> [!tip] Estrategias de Wordlist y Extensiones
> Para el modo `dir`, es fundamental elegir una wordlist adecuada (ej. `common.txt`, `dirb.txt`, `big.txt` de SecLists). La opción `-x` (extensiones) es muy útil para refinar la búsqueda, por ejemplo, `-x php,html,js,bak` para aplicaciones web específicas. Combinar `-x` con `--discover-backup` puede revelar archivos sensibles como `index.php.bak` o `config.php.old`.

> [!tip] Filtrado por Longitud
> La opción `--exclude-length` es extremadamente útil para filtrar páginas de error personalizadas que devuelven un código de estado 200 OK pero tienen una longitud de contenido constante. Primero, se debe identificar la longitud de una página "no encontrada" (ej. `gobuster dir -u http://example.com/nonexistentpage -l`), y luego usar esa longitud para excluirla de los resultados.

#### Modo DNS (Subdominios)

| Opción      | Descripción                       | Ejemplo                             |
| ----------- | --------------------------------- | ----------------------------------- |
| `-d`        | Dominio target                    | `gobuster dns -d example.com`       |
| `-r`        | DNS server personalizado          | `gobuster dns -r 8.8.8.8`           |
| `-i`        | Mostrar direcciones IP            | `gobuster dns -i`                   |
| `--show-ips`| Mostrar IPs de subdominios        | `gobuster dns --show-ips`           |
| `--search`  | Buscar subdominios en resultados  | `gobuster dns --search`             |

> [!info] Expansión de la Superficie de Ataque
> La enumeración de subdominios es un paso crítico en el reconocimiento, ya que cada subdominio puede representar una aplicación o servicio diferente, a menudo con configuraciones de seguridad más débiles o vulnerabilidades únicas. Esto puede llevar a descubrimientos de APIs internas, paneles de administración o versiones de software desactualizadas.

#### Modo S3 (Buckets AWS)

| Opción     | Descripción                       | Ejemplo                             |
| ---------- | --------------------------------- | ----------------------------------- |
| `--region` | Región AWS                        | `gobuster s3 --region us-east-1`    |
| `--no-ssl` | No usar SSL                       | `gobuster s3 --no-ssl`              |

> [!warning] Consideraciones al Enumerar S3 Buckets
> La enumeración de buckets S3 puede ser ruidosa y potencialmente detectada por los sistemas de monitoreo de AWS. Es crucial entender que la existencia de un bucket no implica necesariamente que sea público o vulnerable. Después de identificar un bucket, se deben verificar sus permisos (lectura, escritura, listado) para determinar si hay alguna exposición de datos o configuración incorrecta.