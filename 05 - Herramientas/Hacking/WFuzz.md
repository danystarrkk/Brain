---
aliases: ["WFuzz", "Web Fuzzer"]
tags:
  - hacking/herramientas
  - hacking/web
estado: 🟢 Terminado
---
**WFuzz** (Web Fuzzer) es una herramienta de fuzzing web diseñada para realizar ataques de fuerza bruta contra aplicaciones web. Es extremadamente flexible y permite fuzzear parámetros, encabezados (headers), cookies y cualquier parte de una petición HTTP.

> [!info] Fuzzing en Ciberseguridad
> El fuzzing es una técnica de prueba de software que implica inyectar datos semi-aleatorios o malformados en una aplicación para descubrir vulnerabilidades como desbordamientos de búfer, inyecciones SQL, XSS o errores de lógica. WFuzz automatiza este proceso para aplicaciones web, permitiendo la enumeración de recursos, parámetros ocultos y credenciales.

## Opciones Principales de WFuzz

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-h` | Muestra ayuda | `wfuzz -h` |
| `--help` | Ayuda avanzada | `wfuzz --help` |
| `-c` | Salida coloreada | `wfuzz -c` |
| `-v` | Modo verbose | `wfuzz -v` |
| `-f` | Salida en formato específico | `wfuzz -f output.txt` |
| `-o` | Formato de salida (html, csv, json, etc.) | `wfuzz -o html` |
| `--script` | Ejecuta script de scripting | `wfuzz --script=script.py` |

> [!tip] Salida Coloreada y Verbose
> El uso de `-c` (coloreado) y `-v` (verbose) es fundamental para una mejor legibilidad y comprensión de los resultados durante el fuzzing, especialmente en terminales donde se procesan grandes volúmenes de peticiones.

### Opciones de Fuzzing

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-w` | Wordlist para fuzzing | `wfuzz -w wordlist.txt` |
| `-z` | Especifica tipo de payload | `wfuzz -z file,wordlist.txt` |
| `-Z` | Fuzzing paralelo (múltiples wordlists) | `wfuzz -z file,list1.txt -z file,list2.txt` |
| `-d` | Datos POST para fuzzing | `wfuzz -d "user=FUZZ&pass=FUZ2Z"` |
| `-X` | Método HTTP | `wfuzz -X POST` |
| `-H` | Encabezados HTTP | `wfuzz -H "Header: Value"` |
| `--hc` | Ocultar respuestas por código | `wfuzz --hc 404` |
| `--hl` | Ocultar respuestas por líneas | `wfuzz --hl 10` |
| `--hw` | Ocultar respuestas por palabras | `wfuzz --hw 25` |
| `--hs` | Ocultar respuestas por regex | `wfuzz --hs "Not found"` |
| `-t` | Número de hilos (threads) | `wfuzz -t 50` |
| `-s` | Retraso entre requests (ms) | `wfuzz -s 100` |

> [!warning] Ocultar Respuestas
> Las opciones `--hc`, `--hl`, `--hw`, `--hs` son cruciales para filtrar el "ruido" de las respuestas HTTP. Por ejemplo, ocultar códigos 404 (Not Found) permite enfocarse en respuestas que indican la existencia de un recurso o un comportamiento diferente, lo que es clave para la enumeración efectiva.

### Opciones de Payload

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-z` | Tipo de payload (file, range, list, etc.) | `wfuzz -z range,1-1000` |
| `--zP` | Parámetros para el payload | `wfuzz --zP "sep=;"` |
| `--slice` | Filtra resultados del payload | `wfuzz --slice "results[0:10]"` |
| `-e` | Encoders para payload | `wfuzz -e encoders` |
| `--prefilter` | Prefiltra payloads | `wfuzz --prefilter "len>3"` |

> [!info] Personalización de Payloads
> WFuzz permite una gran flexibilidad en la generación de payloads. La opción `--zP` es útil para ajustar el comportamiento de payloads específicos, como definir un separador para listas o un formato para rangos. Los encoders (`-e`) son esenciales para evadir filtros de seguridad o para probar diferentes representaciones de los datos.

### Opciones de Filtrado

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--sc` | Mostrar respuestas por código | `wfuzz --sc 200` |
| `--sl` | Mostrar respuestas por líneas | `wfuzz --sl 5` |
| `--sw` | Mostrar respuestas por palabras | `wfuzz --sw 10` |
| `--ss` | Mostrar respuestas por regex | `wfuzz --ss "success"` |
| `--filter` | Filtro personalizado | `wfuzz --filter "code=200 and lines>5"` |

> [!tip] Filtrado Inteligente
> A diferencia de las opciones de "ocultar", las opciones `--sc`, `--sl`, `--sw`, `--ss` permiten mostrar *solo* las respuestas que cumplen con ciertos criterios. Esto es ideal para identificar rápidamente respuestas interesantes, como códigos 200 OK con un número de líneas o palabras específico que difiere de las respuestas por defecto.

### Opciones de Autenticación y Proxy

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--basic` | Autenticación básica | `wfuzz --basic user:pass` |
| `--ntlm` | Autenticación NTLM | `wfuzz --ntlm domain:user:pass` |
| `--proxy` | Usar proxy | `wfuzz --proxy 127.0.0.1:8080` |
| `--follow` | Seguir redirecciones | `wfuzz --follow` |
| `--req-delay` | Retraso máximo por request | `wfuzz --req-delay 10` |

> [!info] Integración con Proxies
> La opción `--proxy` es vital para integrar WFuzz con herramientas como Burp Suite o OWASP ZAP. Esto permite interceptar, modificar y analizar las peticiones y respuestas de WFuzz en tiempo real, facilitando la depuración y el análisis de vulnerabilidades.

## Tipos de Payloads (-z)

| Tipo | Descripción | Ejemplo |
|---|---|---|
| `file` | Desde archivo | `-z file,wordlist.txt` |
| `range` | Rango numérico | `-z range,1-1000` |
| `list` | Lista de valores | `-z list,admin-test-guest` |
| `hex` | Rango hexadecimal | `-z hex,00-ff` |
| `ipnet` | Rango de IPs | `-z ipnet,192.168.1.0/24` |
| `iprange` | Rango de IPs | `-z iprange,192.168.1.1-192.168.1.100` |
| `burp` | Desde Burp | `-z burp,file.xml` |
| `null` | Payload vacío | `-z null` |

> [!tip] Versatilidad de Payloads
> La variedad de tipos de payloads (`-z`) es una de las mayores fortalezas de WFuzz. Desde archivos de wordlists (`file`) hasta rangos numéricos (`range`) o direcciones IP (`ipnet`), esta flexibilidad permite adaptar el fuzzing a casi cualquier escenario, desde la enumeración de directorios hasta la búsqueda de IDs de usuario válidos.

## Encoders (-e)

| Encoder | Descripción |
|---|---|
| `base64` | Codificación Base64 |
| `urlencode` | Codificación URL |
| `hex` | Codificación Hexadecimal |
| `md5` | Hash MD5 |
| `sha1` | Hash SHA1 |
| `html` | Codificación HTML |
| `none` | Sin codificación |

> [!warning] Evasión de Filtros
> Los encoders son fundamentales para la evasión de filtros de seguridad web (WAFs, IDS/IPS). Al codificar los payloads de diferentes maneras (ej., URL encoding doble, Base64), se puede intentar eludir las reglas de detección que buscan patrones específicos en texto plano. Es crucial probar diferentes encoders para cada tipo de inyección.