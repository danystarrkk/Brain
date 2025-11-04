-----

**SSLScan** es una herramienta de línea de comandos diseñada para analizar servidores SSL/TLS. Proporciona información detallada sobre la configuración SSL de un servidor, including cipher suites soportadas, protocolos, y vulnerabilidades.

## Tabla de Opciones Principales de SSLScan

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--help`|Muestra ayuda|`sslscan --help`|
|`--version`|Muestra versión|`sslscan --version`|
|`--targets=`|Archivo con targets|`sslscan --targets=file.txt`|
|`--no-colour`|Desactiva colores|`sslscan --no-colour`|
|`--verbose`|Modo verbose|`sslscan --verbose`|
|`--show-certificate`|Muestra certificado|`sslscan --show-certificate`|
|`--show-times`|Muestra tiempos|`sslscan --show-times`|

### Opciones de Conexión

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--port=`|Puerto específico|`sslscan --port=8443`|
|`--starttls=`|STARTTLS protocol|`sslscan --starttls=smtp`|
|`--timeout=`|Timeout conexión|`sslscan --timeout=5`|
|`--sleep=`|Delay entre tests|`sslscan --sleep=1`|
|`--proxy=`|Usar proxy|`sslscan --proxy=host:port`|
|`--proxy-auth=`|Auth proxy|`sslscan --proxy-auth=user:pass`|

### Opciones de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--ssl2`|Test SSLv2|`sslscan --ssl2`|
|`--ssl3`|Test SSLv3|`sslscan --ssl3`|
|`--tls1`|Test TLSv1.0|`sslscan --tls1`|
|`--tls1-1`|Test TLSv1.1|`sslscan --tls1-1`|
|`--tls1-2`|Test TLSv1.2|`sslscan --tls1-2`|
|`--tls1-3`|Test TLSv1.3|`sslscan --tls1-3`|
|`--pk=`|Client private key|`sslscan --pk=key.pem`|
|`--cert=`|Client certificate|`sslscan --cert=cert.pem`|

### Opciones de Output

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--xml=`|Output XML|`sslscan --xml=output.xml`|
|`--json=`|Output JSON|`sslscan --json=output.json`|
|`--pretty`|JSON pretty print|`sslscan --json=out.json --pretty`|
|`--no-failed`|Oculta tests fallidos|`sslscan --no-failed`|