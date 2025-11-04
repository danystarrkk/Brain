-------
**OpenSSL** es un toolkit de código abierto que implementa los protocolos SSL y TLS, además de proporcionar una amplia gama de funciones criptográficas. Es la librería más utilizada para seguridad en internet.

## Tabla de Comandos Principales de OpenSSL

### Comandos Básicos

|Comando|Descripción|Ejemplo|
|---|---|---|
|`version`|Versión de OpenSSL|`openssl version`|
|`help`|Ayuda general|`openssl help`|
|`list`|Lista comandos disponibles|`openssl list`|
|`-help`|Ayuda específica de comando|`openssl genrsa -help`|

### Generación de Claves y Certificados

|Comando|Descripción|Ejemplo|
|---|---|---|
|`genrsa`|Generar clave RSA|`openssl genrsa -out key.pem 2048`|
|`genpkey`|Generar clave privada|`openssl genpkey -algorithm RSA`|
|`req`|Generar CSR (Certificate Signing Request)|`openssl req -new -key key.pem -out csr.pem`|
|`x509`|Manejar certificados X.509|`openssl x509 -in cert.pem -text`|
|`ca`|Autoridad Certificadora|`openssl ca -in csr.pem -out cert.pem`|

### Operaciones Criptográficas

|Comando|Descripción|Ejemplo|
|---|---|---|
|`enc`|Cifrado/Descifrado|`openssl enc -aes-256-cbc -in file.txt -out file.enc`|
|`dgst`|Generar hash|`openssl dgst -sha256 file.txt`|
|`rsautl`|Operaciones RSA|`openssl rsautl -encrypt -in file.txt -out file.enc`|
|`pkeyutl`|Operaciones con claves|`openssl pkeyutl -encrypt -in file.txt -out file.enc`|

### Pruebas de Conectividad

|Comando|Descripción|Ejemplo|
|---|---|---|
|`s_client`|Cliente SSL/TLS|`openssl s_client -connect example.com:443`|
|`s_server`|Servidor SSL/TLS|`openssl s_server -cert cert.pem -key key.pem`|
|`s_time`|Medir performance SSL|`openssl s_time -connect example.com:443`|

### Conversión de Formatos

|Comando|Descripción|Ejemplo|
|---|---|---|
|`pkcs12`|Manejar formatos PKCS#12|`openssl pkcs12 -export -out cert.pfx`|
|`pkcs8`|Convertir formato clave privada|`openssl pkcs8 -topk8 -in key.pem -out key.pk8`|