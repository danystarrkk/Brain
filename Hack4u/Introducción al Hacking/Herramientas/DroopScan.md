-----
**Droopscan** es un escáner de seguridad especializado para Drupal que permite identificar vulnerabilidades, módulos desactualizados, configuraciones inseguras y otros problemas de seguridad en sitios Drupal.
## Tabla de Opciones Principales de Droopscan

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-u`|URL objetivo|`droopscan -u https://example.com`|
|`--url`|URL objetivo|`droopscan --url https://example.com`|
|`-h`|Muestra ayuda|`droopscan -h`|
|`--help`|Ayuda completa|`droopscan --help`|
|`-v`|Modo verbose|`droopscan -v`|
|`--version`|Muestra versión|`droopscan --version`|
|`--update`|Actualizar base de datos|`droopscan --update`|

### Opciones de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-e`|Enumeración específica|`droopscan -e modules`|
|`--enumerate`|Enumerar componentes|`droopscan --enumerate all`|
|`--modules`|Escanear módulos|`droopscan --modules all`|
|`--themes`|Escanear themes|`droopscan --themes all`|
|`--users`|Enumerar usuarios|`droopscan --users`|
|`--brute`|Fuerza bruta de usuarios|`droopscan --brute`|

### Opciones de Output

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-o`|Output file|`droopscan -o resultados.txt`|
|`--output`|Output file|`droopscan --output scan.txt`|
|`--json`|Output JSON|`droopscan --json output.json`|
|`--xml`|Output XML|`droopscan --xml output.xml`|
|`--csv`|Output CSV|`droopscan --csv output.csv`|

### Opciones de Conexión

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-t`|Timeout de conexión|`droopscan -t 30`|
|`--timeout`|Timeout específico|`droopscan --timeout 45`|
|`-p`|Usar proxy|`droopscan -p http://proxy:8080`|
|`--proxy`|Proxy específico|`droopscan --proxy socks5://127.0.0.1:9050`|
|`--cookie`|Cookie personalizada|`droopscan --cookie "session=value"`|
|`--user-agent`|User agent personalizado|`droopscan --user-agent "Mozilla/5.0"`|