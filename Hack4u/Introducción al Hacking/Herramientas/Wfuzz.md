------
**WFuzz** (Web Fuzzer) es una herramienta de fuzzing web diseñada para realizar ataques de fuerza bruta contra aplicaciones web. Es extremadamente flexible y permite fuzzear parámetros, headers, cookies, y cualquier parte de una petición HTTP.

## Tabla de opciones principales de WFuzz

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-h`|Muestra ayuda|`wfuzz -h`|
|`--help`|Ayuda avanzada|`wfuzz --help`|
|`-c`|Output coloreado|`wfuzz -c`|
|`-v`|Modo verbose|`wfuzz -v`|
|`-f`|Output en formato específico|`wfuzz -f output.txt`|
|`-o`|Output format (html, csv, json, etc.)|`wfuzz -o html`|
|`--script`|Ejecuta script de scripting|`wfuzz --script=script.py`|

### Opciones de Fuzzing

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-w`|Wordlist para fuzzing|`wfuzz -w wordlist.txt`|
|`-z`|Especifica tipo de payload|`wfuzz -z file,wordlist.txt`|
|`-Z`|Fuzzing paralelo (múltiples wordlists)|`wfuzz -z file,list1.txt -z file,list2.txt`|
|`-d`|Data POST para fuzzing|`wfuzz -d "user=FUZZ&pass=FUZ2Z"`|
|`-X`|Método HTTP|`wfuzz -X POST`|
|`-H`|Headers HTTP|`wfuzz -H "Header: Value"`|
|`--hc`|Hide responses by code|`wfuzz --hc 404`|
|`--hl`|Hide responses by lines|`wfuzz --hl 10`|
|`--hw`|Hide responses by words|`wfuzz --hw 25`|
|`--hs`|Hide responses by regex|`wfuzz --hs "Not found"`|
|`-t`|Número de hilos (threads)|`wfuzz -t 50`|
|`-s`|Delay entre requests (ms)|`wfuzz -s 100`|

### Opciones de Payload

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-z`|Tipo de payload (file, range, list, etc.)|`wfuzz -z range,1-1000`|
|`--zP`|Parámetros para el payload|`wfuzz --zP "sep=;"`|
|`--slice`|Filtra resultados del payload|`wfuzz --slice "results[0:10]"`|
|`-e`|Encoders para payload|`wfuzz -e encoders`|
|`--prefilter`|Prefiltra payloads|`wfuzz --prefilter "len>3"`|

### Opciones de Filtrado

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--sc`|Show responses by code|`wfuzz --sc 200`|
|`--sl`|Show responses by lines|`wfuzz --sl 5`|
|`--sw`|Show responses by words|`wfuzz --sw 10`|
|`--ss`|Show responses by regex|`wfuzz --ss "success"`|
|`--filter`|Filtro personalizado|`wfuzz --filter "code=200 and lines>5"`|

### Opciones de Autenticación y Proxy

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--basic`|Autenticación básica|`wfuzz --basic user:pass`|
|`--ntlm`|Autenticación NTLM|`wfuzz --ntlm domain:user:pass`|
|`--proxy`|Usar proxy|`wfuzz --proxy 127.0.0.1:8080`|
|`--follow`|Follow redirects|`wfuzz --follow`|
|`--req-delay`|Delay máximo por request|`wfuzz --req-delay 10`|

## Tipos de Payloads (-z)

|Tipo|Descripción|Ejemplo|
|---|---|---|
|`file`|Desde archivo|`-z file,wordlist.txt`|
|`range`|Rango numérico|`-z range,1-1000`|
|`list`|Lista de valores|`-z list,admin-test-guest`|
|`hex`|Rango hexadecimal|`-z hex,00-ff`|
|`ipnet`|Rango de IPs|`-z ipnet,192.168.1.0/24`|
|`iprange`|Rango de IPs|`-z iprange,192.168.1.1-192.168.1.100`|
|`burp`|Desde Burp|`-z burp,file.xml`|
|`null`|Payload vacío|`-z null`|

## Encoders (-e)

|Encoder|Descripción|
|---|---|
|`base64`|Base64 encode|
|`urlencode`|URL encode|
|`hex`|Hexadecimal encode|
|`md5`|MD5 hash|
|`sha1`|SHA1 hash|
|`html`|HTML encode|
|`none`|Sin encoding|