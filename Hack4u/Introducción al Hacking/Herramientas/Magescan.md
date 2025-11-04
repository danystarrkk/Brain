----

## Tabla de Opciones Principales de MageScan

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-u`|URL objetivo|`magescan -u https://example.com`|
|`--url`|URL objetivo|`magescan --url https://example.com`|
|`-h`|Muestra ayuda|`magescan -h`|
|`--help`|Ayuda completa|`magescan --help`|
|`-v`|Modo verbose|`magescan -v`|
|`--version`|Muestra versión|`magescan --version`|
|`--update`|Actualizar base de datos|`magescan --update`|

### Opciones de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`scan:all`|Escaneo completo|`magescan scan:all`|
|`scan:version`|Detectar versión|`magescan scan:version`|
|`scan:patches`|Verificar parches|`magescan scan:patches`|
|`scan:modules`|Enumerar módulos|`magescan scan:modules`|
|`scan:unreachable`|Buscar paths inaccesibles|`magescan scan:unreachable`|
|`scan:sitemap`|Analizar sitemap|`magescan scan:sitemap`|
|`scan:server`|Información del servidor|`magescan scan:server`|

### Opciones de Output

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-o`|Output file|`magescan -o resultados.txt`|
|`--output`|Output file|`magescan --output scan.txt`|
|`--format`|Formato de output|`magescan --format json`|
|`--json`|Output JSON|`magescan --json output.json`|
|`--xml`|Output XML|`magescan --xml output.xml`|

### Opciones de Conexión

| Opción         | Descripción              | Ejemplo                               |
| -------------- | ------------------------ | ------------------------------------- |
| `-t`           | Timeout de conexión      | `magescan -t 30`                      |
| `--timeout`    | Timeout específico       | `magescan --timeout 45`               |
| `--user-agent` | User agent personalizado | `magescan --user-agent "Mozilla/5.0"` |
| `--proxy`      | Usar proxy               | `magescan --proxy http://proxy:8080`  |
| `--cookie`     | Cookie personalizada     | `magescan --cookie "admin=value"`     |