-----
**WhatWeb** es una herramienta de fingerprinting de sitios web que identifica:

- Sistemas de Gestión de Contenido (CMS)
- Plataformas de blogging
- Frameworks JavaScript/CSS
- Servidores web
- Módulos de Apache
- Dispositivos de red
- Y más de 1800 plugins diferentes

## Tabla de opciones principales de WhatWeb

### Opciones Básicas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-h`|Muestra ayuda|`whatweb -h`|
|`--help`|Ayuda avanzada|`whatweb --help`|
|`-v`|Modo verbose|`whatweb -v example.com`|
|`--version`|Muestra versión|`whatweb --version`|
|`-a`|Nivel de agresividad (1-4)|`whatweb -a 3 example.com`|
|`--aggression`|Nivel de agresividad|`whatweb --aggression 3`|
|`-i`|Archivo de input con URLs|`whatweb -i urls.txt`|
|`--input-file`|Archivo de input|`whatweb --input-file=urls.txt`|

### Opciones de Output

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-q`|Modo quiet (silencioso)|`whatweb -q example.com`|
|`--quiet`|Modo quiet|`whatweb --quiet example.com`|
|`-v`|Verbose output|`whatweb -v example.com`|
|`--verbose`|Output verbose|`whatweb --verbose example.com`|
|`--color`|Output con colores|`whatweb --color=always example.com`|
|`--no-errors`|Suprime mensajes de error|`whatweb --no-errors example.com`|
|`-o`|Output a archivo|`whatweb -o results.txt example.com`|
|`--output`|Output a archivo|`whatweb --output=results.txt`|

### Formatos de Output

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--log-json`|Output en JSON|`whatweb --log-json=results.json`|
|`--log-xml`|Output en XML|`whatweb --log-xml=results.xml`|
|`--log-html`|Output en HTML|`whatweb --log-html=results.html`|
|`--log-csv`|Output en CSV|`whatweb --log-csv=results.csv`|
|`--log-brief`|Output breve|`whatweb --log-brief=results.txt`|
|`--log-verbose`|Output verbose|`whatweb --log-verbose=results.txt`|

### Opciones de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--url-prefix`|Prefijo para URLs|`whatweb --url-prefix=https://`|
|`--url-suffix`|Sufijo para URLs|`whatweb --url-suffix=/admin`|
|`--follow-redirect`|Sigue redirects|`whatweb --follow-redirect=always`|
|`--max-redirects`|Máximo de redirects|`whatweb --max-redirects=10`|
|`--user-agent`|User-Agent personalizado|`whatweb --user-agent="CustomUA"`|
|`--header`|Headers personalizados|`whatweb --header="Cookie: test=1"`|
|`--cookie`|Cookies|`whatweb --cookie="session=abc"`|
|`--proxy`|Usar proxy|`whatweb --proxy=127.0.0.1:8080`|
|`--proxy-user`|Usuario de proxy|`whatweb --proxy-user=user:pass`|

### Opciones de Plugins

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--list-plugins`|Lista todos los plugins|`whatweb --list-plugins`|
|`--info-plugins`|Info de todos los plugins|`whatweb --info-plugins`|
|`--search-plugins`|Busca plugins|`whatweb --search-plugins=wordpress`|
|`--plugins`|Plugins específicos|`whatweb --plugins=wordpress,apache`|
|`--grep`|Filtra output por plugin|`whatweb --grep=wordpress`|
|`--custom-plugin`|Plugin personalizado|`whatweb --custom-plugin=plugin.rb`|

### Opciones de Rendimiento

|Opción|Descripción|Ejemplo|
|---|---|---|
|`--threads`|Número de hilos|`whatweb --threads=10`|
|`--concurrency`|Nivel de concurrencia|`whatweb --concurrency=20`|
|`--timeout`|Timeout de conexión|`whatweb --timeout=30`|
|`--open-timeout`|Timeout de apertura|`whatweb --open-timeout=10`|
|`--read-timeout`|Timeout de lectura|`whatweb --read-timeout=20`|

## Niveles de Agresividad

|Nivel|Descripción|Comportamiento|
|---|---|---|
|`1`|Stealthy|Solo requests iniciales, pasivo|
|`2`|Default|Algunas requests adicionales|
|`3`|Aggressive|Requests más intrusivas|
|`4`|Heavy|Máxima intrusividad, muchas requests|