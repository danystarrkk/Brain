----
**WPScan** es un escáner de seguridad de WordPress que permite identificar vulnerabilidades, plugins/themes desactualizados, usuarios, y otros problemas de seguridad en sitios WordPress.

## Tabla de Opciones Principales de WPScan

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--url`|URL objetivo|`wpscan --url https://example.com`|
|`--help`|Muestra ayuda|`wpscan --help`|
|`--version`|Muestra versión|`wpscan --version`|
|`--verbose`|Output verbose|`wpscan --verbose`|
|`--output`|Output file|`wpscan --output resultados.txt`|
|`--format`|Formato de output|`wpscan --format json`|

### Opciones de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--enumerate`|Enumeración específica|`wpscan --enumerate u`|
|`--plugins`|Escanear plugins|`wpscan --plugins all`|
|`--themes`|Escanear themes|`wpscan --themes all`|
|`--config-backups`|Buscar backups de config|`wpscan --config-backups`|
|`--db-exports`|Buscar exports de DB|`wpscan --db-exports`|
|`--medias`|Enumerar medias|`wpscan --medias`|

### Opciones de Fuerza Bruta

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--usernames`|Lista de usuarios|`wpscan --usernames users.txt`|
|`--passwords`|Lista de passwords|`wpscan --passwords pass.txt`|
|`--password-attack`|Tipo de ataque|`wpscan --password-attack wp-login`|
|`--threads`|Número de hilos|`wpscan --threads 20`|
|`--request-timeout`|Timeout requests|`wpscan --request-timeout 30`|

### Opciones de API

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--api-token`|Token de API WPScan|`wpscan --api-token YOUR_TOKEN`|
|`--update`|Actualizar base de datos|`wpscan --update`|
|`--disable-tls-checks`|Desactivar checks TLS|`wpscan --disable-tls-checks`|