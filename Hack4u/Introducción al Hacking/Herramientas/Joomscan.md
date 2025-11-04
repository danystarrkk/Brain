-----
**JoomScan** (OWASP Joomla Vulnerability Scanner) es una herramienta de escaneo de seguridad diseñada específicamente para detectar vulnerabilidades y problemas de configuración en sitios Joomla!.

## Tabla de Opciones Principales de JoomScan

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-u`|URL objetivo|`joomscan -u https://example.com`|
|`--url`|URL objetivo|`joomscan --url https://example.com`|
|`-h`|Muestra ayuda|`joomscan -h`|
|`--help`|Ayuda completa|`joomscan --help`|
|`-v`|Modo verbose|`joomscan -v`|
|`--version`|Muestra versión|`joomscan --version`|
|`--update`|Actualizar base de datos|`joomscan --update`|

### Opciones de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-ec`|Enumerar componentes|`joomscan -ec`|
|`-ep`|Enumerar plugins|`joomscan -ep`|
|`-em`|Enumerar módules|`joomscan -em`|
|`-et`|Enumerar templates|`joomscan -et`|
|`-f`|Fuerza bruta de admin|`joomscan -f`|
|`-a`|Agente de usuario personalizado|`joomscan -a "Mozilla/5.0"`|
|`-r`|Rango de IPs|`joomscan -r 192.168.1.0/24`|

### Opciones de Output

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-o`|Output file|`joomscan -o resultados.txt`|
|`--log`|Log file|`joomscan --log scan.log`|
|`--json`|Output JSON|`joomscan --json output.json`|
|`--xml`|Output XML|`joomscan --xml output.xml`|

### Opciones de Proxy y Conexión

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-p`|Usar proxy|`joomscan -p http://proxy:8080`|
|`--proxy`|Proxy específico|`joomscan --proxy socks5://127.0.0.1:9050`|
|`-t`|Timeout de conexión|`joomscan -t 30`|
|`--timeout`|Timeout específico|`joomscan --timeout 45`|
|`--cookie`|Cookie personalizada|`joomscan --cookie "session=value"`|