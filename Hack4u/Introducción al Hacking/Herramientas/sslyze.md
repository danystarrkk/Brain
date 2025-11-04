------

**SSLyze** es un analizador rápido y completo de configuraciones SSL/TLS. Permite escanear servidores para identificar problemas de configuración, vulnerabilidades, y verificar el cumplimiento de mejores prácticas de seguridad.

## Tabla de Opciones Principales de SSLyze

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--help`|Muestra ayuda|`sslyze --help`|
|`--version`|Muestra versión|`sslyze --version`|
|`--targets_in`|Archivo con targets|`sslyze --targets_in=targets.txt`|
|`--json_out`|Output en JSON|`sslyze --json_out=results.json`|
|`--xml_out`|Output en XML|`sslyze --xml_out=results.xml`|
|`--text_out`|Output en texto|`sslyze --text_out=results.txt`|
|`--quiet`|Modo silencioso|`sslyze --quiet`|
|`--timeout`|Timeout de conexión|`sslyze --timeout=5`|
|`--https_tunnel`|Usar tunnel HTTPS|`sslyze --https_tunnel=proxy:8080`|

### Opciones de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--regular`|Escaneo regular completo|`sslyze --regular`|
|`--certinfo`|Información de certificados|`sslyze --certinfo`|
|`--certinfo_full`|Certificado completo|`sslyze --certinfo_full`|
|`--compression`|Chequear compresión|`sslyze --compression`|
|`--reneg`|Chequear renegociación|`sslyze --reneg`|
|`--resum`|Chequear resumption|`sslyze --resum`|
|`--heartbleed`|Test Heartbleed|`sslyze --heartbleed`|
|`--openssl_ccs`|Test CCS Injection|`sslyze --openssl_ccs`|
|`--robot`|Test ROBOT|`sslyze --robot`|
|`--fallback`|Test Fallback SCSV|`sslyze --fallback`|

### Opciones de Cipher Suites

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--sslv2`|Test SSLv2|`sslyze --sslv2`|
|`--sslv3`|Test SSLv3|`sslyze --sslv3`|
|`--tlsv1`|Test TLSv1.0|`sslyze --tlsv1`|
|`--tlsv1_1`|Test TLSv1.1|`sslyze --tlsv1_1`|
|`--tlsv1_2`|Test TLSv1.2|`sslyze --tlsv1_2`|
|`--tlsv1_3`|Test TLSv1.3|`sslyze --tlsv1_3`|
|`--cipher_list`|Cipher list personalizada|`sslyze --cipher_list=ALL`|

### Opciones de Performance

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--nb_retries`|Número de reintentos|`sslyze --nb_retries=3`|
|`--http_get`|Usar HTTP GET|`sslyze --http_get`|
|`--starttls`|STARTTLS protocols|`sslyze --starttls=smtp`|
|`--sni`|SNI hostname|`sslyze --sni=example.com`|
|`--early_data`|Test 0-RTT data|`sslyze --early_data`|