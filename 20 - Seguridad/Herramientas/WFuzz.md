---
tags:
  - ciberseguridad
  - herramientas
  - wfuzz
tipo: herramienta
estado: terminado
---

# WFuzz: Fuzzing Web Genérico

**WFuzz** (Web Fuzzer) es un fuzzer web **orientado a marcadores**: colocás la palabra mágica `FUZZ` (y `FUZ2Z`, `FUZ3Z`…) en **cualquier punto** de la petición — URL, body, headers, cookies, auth — y WFuzz sustituye allí cada entrada de tus wordlists/payloads, midiendo las respuestas con un motor de match/filtros muy expresivo. Es la herramienta para descubrir parámetros ocultos, endpoints, IDs válidos, credenciales, valores de cabeceras y cualquier inyección que admita diccionario.

> [!info] Por qué WFuzz y no un dirbuster
> Gobuster/ffuf resuelven un caso: rutas con una wordlist. WFuzz resuelve **cualquier posición de cualquier protocolo HTTP**: `-u http://t/?lang=FUZZ`, `-d '{"user":"FUZZ","pass":"FUZ2Z"}'`, `-H "X-Forwarded-For: FUZZ"`, `--basic FUZZ:FUZ2Z`… Además trae **encoders** y **plugins** integrados. El costo: más sintaxis y una curva de filtrado que hay que dominar (pero que es su superpoder). Para directorios "a secas", [[Gobuster]] es más rápido de lanzar; para param fuzzing y ataques de credenciales web, WFuzz/ffuf.

## Instalación

```bash
# Kali (viene preinstalado en las últimas versiones)
sudo apt update && sudo apt install wfuzz
wfuzz --version

# pip (si el paquete del repo está desactualizado)
sudo pip3 install wfuzz

# Docker
docker run -v $(pwd)/wl:/wordlists xmendez/wfuzz -c -w /wordlists/list.txt http://target/FUZZ
```

> [!warning] Fragilidad de la instalación pip
> El bug de instalación más frecuente de WFuzz viene de `pip` actualizando dependencias (`requests`, `lxml`…): wfuzz falla al arrancar con un `ImportError` tras cualquier upgrade del sistema. Si `wfuzz` rompe sin tocarlo, reinstala con `pip3 install --force-reinstall wfuzz` o usa el paquete de Kali (aislado en su venv de sistema).

## Modelo mental: marcadores + motor de match

1. **Marcadores**: la petición debe contener `FUZZ` (o `FUZ2Z`, `FUZ3Z` para más payloads). Varias `-w`/`-z` se asignan en orden a los marcadores.
2. **Payloads**: `-w` (wordlist) o `-z tipo,arg1[,arg2]` (cualquier generador).
3. **Línea base**: WFuzz mide `code / lines / words / chars` de cada respuesta. El matcher por defecto muestra lo "interesante" (ver warning) — vos defines qué es interesante con los filtros.
4. **Filtros**: ocultar/mostrar por código, líneas, palabras, tamaño o expresiones (`--filter "code!=200 and chars>500"`).

> [!warning] La línea base manda (falsos positivos #1)
> Antes de fuzzear, medí una respuesta inexistente conocida: `curl -s -o /dev/null -w "%{size_download}\n" http://target/NOEXISTE123`. Si el servidor devuelve 200 con el mismo tamaño para todo (soft-404), tus filtros tienen que excluir ese tamaño (`--hs <N>` o `--filter "chars!=<N>"`), o verás mil "hallazgos" que no son nada. Igual que con [[Gobuster]] y `--exclude-length`.

## Tabla de flags

### Básicos y salida

| Flag | Significado | Ejemplo |
|---|---|---|
| `-h`, `--help` | Ayuda | `wfuzz -h` |
| `-c` | Salida coloreada | `-c` |
| `-v` | Verbose (incluye cabeceras) | `-v` |
| `-Z` | Dry-run: no envía peticiones | `-Z` |
| `-f nombre,printer` | Guarda resultados: `raw`, `csv`, `json`, `html`, `md`, `sitemap`… | `-f out.json,json` |
| `-e, --list-plugins` | Lista plugins/encoders/printers/payloads disponibles | `-e encoders` |
| `--no-color` / `--no-cache` | Sin color / sin cachear payloads | `--no-cache` |

### Objetivo y petición

| Flag | Significado | Ejemplo |
|---|---|---|
| `-u, --url` | URL objetivo (conteniendo `FUZZ`) | `-u http://t/?q=FUZZ` |
| `-X, --method` | Método HTTP | `-X POST` |
| `-d, --data` | Cuerpo (soporta marcadores) | `-d "user=FUZZ&pass=FUZ2Z"` |
| `-H, --header` | Cabecera (repetible, acepta `FUZZ`) | `-H "X-Forwarded-For: FUZZ"` |
| `-b, --cookie` | Cookies | `-b "sid=FUZZ; role=1"` |
| `--basic` / `--digest` / `--ntlm` | Auth HTTP con marcadores | `--basic admin:FUZZ` |
| `--proxy` | Proxy (Burp/ZAP) | `--proxy 127.0.0.1:8080` |
| `--follow` | Seguir redirecciones | `--follow` |
| `--req-delay` / `--conn-delay` | Timeout de respuesta / de conexión (segundos) | `--conn-delay 10` |
| `-t, --threads` | Hilos (default 10) | `-t 30` |
| `-s, --delay` | **Pausa entre peticiones (ms)** — el throttle real | `-s 200` |

### Payloads

| Flag | Significado | Ejemplo |
|---|---|---|
| `-w, --wordlist` | Wordlist (repetible; la 2ª llena `FUZ2Z`) | `-w users.txt -w pass.txt` |
| `-z, --payload` | Payload tipado: `-z tipo,arg1[,opt]` | `-z range,1-1000` |
| `--zP` | Parámetros del payload (p. ej. `--zP "keyword=ID"` para renombrar el marcador) | `--zP keyword=item` |
| `--slice` | Recorta la entrada del payload antes de enviarla | `--slice 0:16` |
| `-enc` / `--encoder` | Encoder global aplicado a todos los payloads | `-enc urlencode` |

### Tipos de payload (`-z`)

| Tipo | Qué genera | Ejemplo |
|---|---|---|
| `file` | Líneas de un archivo | `-z file,/usr/share/wordlists/dirb/common.txt` |
| `list` | Valores literales separados por guion | `-z list,admin-test-guest` |
| `range` | Rango numérico (con step opcional) | `-z range,1-1000` |
| `hex` | Rango hexadecimal | `-z hex,00-ff` |
| `ipnet` | IPs de una red | `-z ipnet,192.168.0.0/24` |
| `iprange` | Rango de IPs | `-z iprange,192.168.0.1-192.168.0.254` |
| `burp` | Entrada exportada de Burp | `-z burp,items.xml` |
| `null` | Payload vacío (posicionar `FUZZ` sin sustituir nada) | `-z null` |

### Encoders (`-e encoders` lista los tuyos)

| Encoder | Efecto | Uso típico |
|---|---|---|
| `urlencode` | URL-encoding | Evasión de WAF, inyección |
| `urlencode2` / doble | URL-encoding dos veces | WAF con decodificación simple |
| `base64` | Base64 | Autenticación Basic, filtros débiles |
| `hex` | Hexadecimal | `0x…` en consultas/headers |
| `html`, `html2` | Entidades HTML | XSS con filtros |
| `md5`, `sha1`, `sha256` | Hashes | Comparar contra dumps de hashes |
| `base64encode/encode_once…` | Variantes | Ver `-e encoders` en tu versión |

Uso por payload (último campo de `-z`) o global (`-enc`):

```bash
wfuzz -z file,/path/wordlist.txt,urlencode -u "http://t/search?q=FUZZ"
wfuzz -z file,sqli.txt -enc urlencode -u "http://t/item?id=FUZZ"
```

> [!tip] Apilar encoders
> WFuzz permite encoders compuestos (p. ej. `urlencode/base64` según versión): se listan encadenados y se aplican en orden. Verificá la cadena disponible con `-e encoders` y probando `wfuzz -z list,test,md5/base64` antes de un ataque largo — la salida te muestra el payload resultante con `-v`.

### Motor de match/filtros (el corazón de WFuzz)

| Flag | Significado |
|---|---|
| `--hc`, `--hw`, `--hl`, `--hs` | **Hide** por código / palabras / líneas / tamaño (accepta listas: `--hc 404,403`) |
| `--sc`, `--sw`, `--sl`, `--ss` | **Show** solo las que coincidan con código / palabras / líneas / tamaño |
| `--hf` / `--sh` por hash… | (`--mf/--ms` variantes según versión — verifica con `wfuzz -h`) |
| `--filter "expr"` | Expresión completa: `code`, `lines`, `words`, `chars`, `content`, `md5` con `=, !=, >, <, ~, !~` |
| `--prefilter "expr"` | Filtra **payloads** antes de enviarlos |
| `--postfilter "expr"` | Filtra resultados después del matcher |
| `--slice` | Recorta/parsa el payload (también dentro de `--filter` vía `payload`) |

```bash
# Ocultar el 404 "blando" y los 30x
wfuzz --hc 404,302,301 ...
# Solo códigos 200 con tamaño distinto al de la línea base
wfuzz --sc 200 --hs 1735 ...
# Expresión compuesta: interesantes = 200 grandes, o 500, o con "admin" en el body
wfuzz --filter "code=200 and chars>1000 or code=500 or content~'admin'" ...
```

> [!info] El matcher por defecto
> Sin flags de filtrado, WFuzz muestra respuestas con código en {200, 400, 401, 403, 500} y valores no-cero en líneas/palabras/tamaño. En cuanto tocás `--sc/--hc/...` **ese default se reemplaza por el tuyo**: no "se suma". Si tu filtrado desaparece todo, revisa que la línea base de la respuesta 404 no esté dentro de lo que pediste mostrar.

### Autenticación y plugins

| Flag | Significado | Ejemplo |
|---|---|---|
| `--script=<plugin>` | Carga un plugin (request/response) del directorio de plugins | `--script=<nombre>` |
| `--script-args` | Argumentos para plugins | `--script-args k=v` |
| `--script-help` | Ayuda de un plugin | `--script-help <nombre>` |
| `-t/-s` + `--proxy` | Combina con plugins para inspección en Burp | — |

## Recetas (copy-paste)

```bash
# 1. Enumeración de rutas con línea base (estilo Gobuster con filtros)
wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc 404,403 \
  -f rutas.csv,csv http://target.local/FUZZ

# 2. Descubrir parámetros GET ocultos
wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc 404 \
  -u "http://target.local/page?FUZZ=1"

# 3. Fuzzear valores de un parámetro concreto (LFI básico)
wfuzz -w /usr/share/wordlists/vulns/lfi.txt --hc 404 \
  -u "http://target.local/index.php?page=FUZZ"

# 4. Login: dos payloads (usuario × contraseña)
wfuzz -c -w users.txt -w passwords.txt -d "username=FUZZ&password=FUZ2Z" \
  --hc 403 --hs 5120 -t 5 -f login.json,json http://target.local/login.php
#   ^ el -t 5 bajo evita lockouts; el --hs 5120 filtra el tamaño de "login incorrecto"

# 5. Enumerar IDs (IDOR / BOLA) con rango numérico
wfuzz -z range,1-500 --hc 404 -u "http://target.local/api/orders/FUZZ" \
  -H "Authorization: Bearer <token>"

# 6. Fuzz de headers (host virtual, IP bypass, cache poisoning)
wfuzz -w seclists/bypass.txt --hc 404,403 \
  -H "X-Original-URL: FUZZ" http://target.local/admin

# 7. Autenticación Basic (usuario fuzz + pass fija; el inverso con FUZZ en la pass)
wfuzz -w users.txt --basic FUZZ:secret http://target.local/manager

# 8. JSON API POST (marcadores dentro del JSON)
wfuzz -X POST -H "Content-Type: application/json" \
  -d '{"user":"FUZZ","pass":"FUZ2Z"}' -w users.txt -w pass.txt \
  --filter "code=200 and chars>2" http://target.local/api/login

# 9. Evasión con encoder + filtro por hash de contenido
wfuzz -z file,wordlist.txt,base64 --filter "code!=403 and md5!='<md5_404>'" \
  -u "http://target.local/auth" -X POST -d "cred=FUZZ"

# 10. Proxy a Burp para depurar la plantilla antes del ataque real
wfuzz -c -w users.txt -d "u=FUZZ&p=x" --proxy 127.0.0.1:8080 --hc 404 http://target.local/login
```

**Detalle de la receta 4:** cada `-w` se asigna por orden a los marcadores (`FUZZ`, `FUZ2Z`, `FUZ3Z`…). `--hc 403` elimina el bloqueo por cuenta inexistente; `--hs 5120` debe salir de medir la respuesta fallida real con `curl`. Sin esa medición, confías en el azar.

**Detalle de la receta 9:** `--filter` con `md5!='…'` compara el hash del body contra la plantilla del 404 — el filtro más robusto cuando los tamaños varían por tokens CSRF (aunque esos tokens también pueden romper el filtro por chars: para eso existe `content~'patrón'`).

## Interpretación de la salida

```
000000001:  c200   26 L     43 W        749 Ch       "admin"
000000002:  c404   10 L     18 W        285 Ch       "backup"
```

| Columna | Significado |
|---|---|
| `00000000N` | Índice del payload (posición en la wordlist — útil para reproducir) |
| `c200` / `c404` | Código HTTP (prefijo `c`) |
| `L` / `W` / `Ch` | Líneas / palabras / bytes del body — los tres ejes del filtrado |
| `"…"` | Payload enviado (ya decodificado) |
| Con `-v` | Se añaden cabeceras de respuesta (server, set-cookie…) — imprescindible para afinar filtros |

## Detección y contramedidas (defensor)

| Señal | Origen | Mitigación |
|---|---|---|
| Cientos de GETs con parámetros de diccionario (`?id=`, `?page=`, `?user=`) | recetas 2–5 | WAF con reglas de param mining; rate limit |
| Mismos POSTs de login con usuarios/creds rotando (receta 4) | fuerza bruta web | Bloqueo por cuenta + CAPTCHA; alertas por velocidad por usuario |
| Headers anómalos (`X-Original-URL`, `X-Forwarded-For` raros) | receta 6 | Reglas de header smuggling en el reverse proxy |
| UA por defecto de Python/requests y threading alto | cualquier receta | Fail2ban con patrón wfuzz; UA consistente |
| `429`/cortes | Ya activo | WFuzz no reintenta solo: reprograma, no insistas |

## Errores comunes (mensaje → arreglo)

1. **`ImportError: cannot import name ...` al arrancar** — dependencias de Python rotas tras un upgrade del sistema.
   → `pip3 install --force-reinstall wfuzz` o el paquete de Kali (aislado). El más reportado tras actualizar `requests`/`lxml`.
2. **"Todo es hallazgo" / resultados masivos** — línea base mal medida (soft-404) o matcher por defecto reemplazado mal.
   → Baseline con `curl`, y usar `--hs <tamaño_404>` o `--filter "md5!='<md5_404>'"`.
3. **0 resultados cuando hay hallazgos** — `--hs`/`--hc` se comen la respuesta buena (tamaño/ código coinciden con el filtro).
   → Quitar filtros, pasar a `-v`, comparar manualmente good vs bad response.
4. **`FUZZ` en body/JSON no se sustituye** — falta el header de content-type o el marcador está mal escrito (`FUZZZ`).
   → `-H "Content-Type: application/json"` y marcadores exactos: `FUZZ`, `FUZ2Z`, `FUZ3Z`.
5. **`429 Too Many Requests` / corta el servidor** — rate limit.
   → `-s 200` (pausa ms), `-t 5`, o rotación de IP; no hay reintento mágico.
6. **Encoders no aplicados** — `-e` lista plugins; el encoder va como **último campo de `-z`** o con `-enc`.
   → `-z file,list.txt,urlencode`; verificar payload resultante con `-v`.
7. **Dos `-w` pero solo se rota una** — falta el segundo marcador `FUZ2Z` en la petición.
   → La petición debe contener tantos marcadores como listas.
8. **`--proxy` no muestra nada en Burp** — proxy mal formado (falta esquema) o Burp solo escucha loopback.
   → `--proxy 127.0.0.1:8080` (WFuzz añade el esquema) y cert CA de Burp instalado si el objetivo es HTTPS.

## WFuzz vs ffuf vs Gobuster

| Criterio | [[Gobuster]] | ffuf | **WFuzz** |
|---|---|---|---|
| Marcadores | Sólo URL (y `-H` según modo) | Cualquier posición | **Cualquier posición** (URL, body, headers, cookies, auth) |
| Velocidad | Muy alta | **Máxima** (motor propio, raw sockets) | Media (Python) |
| Filtrado por atributos | `--exclude-length` | `-mc/-fc/-ms/-fs/-ml/-fl` (match/filter) | `--hc/--sc/--filter` + **expresiones con regex y md5** |
| Encoders integrados | No | No | **Sí** (base64, urlencode, hashes…) |
| Múltiples payloads | No | Limitado (2 vía `-ac`?) — no: ffuf soporta `-w a -w b` con `FUZZ`/`FUZ2Z` | **Sí, ilimitados** |
| Plugins/scripts | No | No | Sí (plugins de request/response) |
| Printers/salida | texto | JSON | raw/csv/json/html/md |
| Curva | Muy baja | Baja | **Media-alta** |
| Mejor para | Directorios rapidísimamente | Fuzzing web de alto volumen | Param fuzzing, creds, encoders, todo lo demás |

En la práctica se usan juntos: [[Gobuster]]/ffuf para rutas, WFuzz para profundizar parámetros y credenciales de lo encontrado — la misma sinergia que describen [[Enumeración de Servicios y CMS]] y [[Reconocimiento y Escaneo de Redes y Web]].

## Cheatsheet

```bash
wfuzz -c -w common.txt --hc 404 http://t/FUZZ
wfuzz -c -w common.txt --hc 404 -u "http://t/page?FUZZ=1"
wfuzz -w params.txt --hc 404 -u "http://t/?FUZZ=1"
wfuzz -c -w users.txt -w pass.txt -d "u=FUZZ&p=FUZ2Z" -t 5 --hc 403 --hs <size> http://t/login
wfuzz -z range,1-500 --hc 404 -u "http://t/api/FUZZ" -H "Authorization: Bearer T"
wfuzz -z file,list.txt,urlencode -u "http://t/?q=FUZZ"
wfuzz --filter "code=200 and chars>1000" -w big.txt http://t/FUZZ
wfuzz -c -w users.txt -d "u=FUZZ&p=x" --proxy 127.0.0.1:8080 http://t/login
wfuzz -f out.json,json -w common.txt http://t/FUZZ
wfuzz -e encoders    # inventario local
wfuzz -h             # ayuda real de tu versión
```

| Tarea | Comando |
|---|---|
| Línea base | `curl -s -o /dev/null -w "%{size_download}\n" http://t/NOEXISTE` |
| Filtro por defecto reemplazado | `--hc/--sc` + `--hs/--ss` |
| Expresiones | `--filter "code=200 and chars!=N or content~'x'"` |
| Credenciales | `-w users -w pass -d "u=FUZZ&p=FUZ2Z" -t 5` |
| Encoder | `-z file,wl.txt,urlencode` o `-enc base64` |
| Throttle | `-s <ms>` |
| Informe | `-f out.json,json` |
| Depurar plantilla | `--proxy 127.0.0.1:8080 -v` |

## Referencias

- [Repositorio oficial (xmendez/wfuzz)](https://github.com/xmendez/wfuzz) — wiki con payloads, plugins y filter language
- `wfuzz -h` / `wfuzz -e encoders` — inventario vivo de tu instalación
- SecLists — wordlists de parámetros (`Discovery/Web-Content/burp-parameter-names.txt`) y valores

## Relacionadas

- [[Gobuster]] — descubrimiento de rutas (hermano directo, ver su tabla comparativa)
- [[THC Hydra]] — brute force de servicios de red; para forms web el pipeline es WFuzz/Hydra según el caso
- [[WPScan]] — cuando el objetivo es WordPress y hay que enumerar además de fuzzear
- [[Abuso de APIs]] — el param/ID fuzzing como fase de auditoría de APIs (recetas 5 y 8)
- [[Inyección NoSQL]] — payloads de operadores `$ne`/`$gt` vía WFuzz en JSON bodies
- [[Burp Suite Escaneo Dirigido y Estructuras de Datos No Estándar]] — depurar plantillas por proxy antes de lanzar el fuzz
- [[WhatWeb Herramienta de Fingerprinting]] — contexto de la fase de reconocimiento web
