---
aliases: ["Gobuster", "gobuster", "Fuzzing de Directorios", "Enumeración de Directorios"]
tags: [tool/gobuster]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[WFuzz]]"
  - "[[WhatWeb Herramienta de Fingerprinting]]"
  - "[[Nmap Guía Completa]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Enumeración de Servicios y CMS]]"
  - "[[WPScan]]"
  - "[[HTTP Host header attacks]]"
  - "[[Ataques de Transferencia de Zona DNS (AXFR)]]"
  - "[[Burp Suite Escaneo Dirigido y Estructuras de Datos No Estándar]]"
fuente: "https://github.com/OJ/gobuster"
---

# Gobuster: Enumeración de Directorios, Subdominios y VHosts

**Gobuster** es un brute-forcer escrito en Go, multihilo y sin dependencias, con modos para **directorios/archivos (`dir`), subdominios (`dns`), virtual hosts (`vhost`), parámetros (`fuzz`), buckets S3 (`s3`)** y TFTP. Es la herramienta de reconocimiento activo web más usada en CTF y pentests: convierte una URL base en el mapa real de la superficie de ataque en segundos.

> [!info] Por qué Gobuster y no un crawler
> Un crawler solo sigue enlaces visibles; Gobuster **fuerza** rutas que no enlazan a ningún lado: `/backup.zip`, `/.git/config`, `/admin`, `/api/internal`. La hipótesis es que existen rutas conocidas por nombre — por eso el éxito depende de la **wordlist**, no de la inteligencia del algoritmo. Para fuzzing de parámetros/valores, mira [[WFuzz]].

## Instalación

```bash
# Kali / Debian
sudo apt update && sudo apt install gobuster

# Go (última versión — ver README del repositorio para el módulo exacto)
go install github.com/OJ/gobuster/v3@latest

# Docker (aislado)
docker run -v /usr/share/wordlists:/wordlists ojg/gobuster dir -u http://target -w /wordlists/common.txt

# Verificar versión (¡afecta a los flags!)
gobuster version
gobuster -h
```

> [!warning] Deriva de versiones (v3 vs v4)
> Los flags de Gobuster **cambiaron entre v3 y v4**: algunas opciones pasaron a modo, otras se renombraron o se eliminaron. Antes de automatizar un comando, confirma con `gobuster dir -h` en **tu** instalación; el error típico es `unknown flag: --xyz`.

## Modos de operación

| Modo | Qué descubre | Ejemplo mínimo |
|---|---|---|
| `dir` | Directorios, archivos y backups en una URL | `gobuster dir -u http://t.com -w wl.txt` |
| `dns` | Subdominios (brute-force DNS) | `gobuster dns -d t.com -w wl.txt` |
| `vhost` | Virtual hosts del servidor (Host header) | `gobuster vhost -u http://t.com -w wl.txt` |
| `fuzz` | Cualquier punto `FUZZ` (params, paths, headers) | `gobuster fuzz -u http://t.com/?q=FUZZ -w wl.txt` |
| `s3` | Nombres de buckets S3 | `gobuster s3 -w wl.txt` |
| `tftp` | Recursos TFTP (según versión) | `gobuster tftp -w wl.txt` |

## Tabla de flags

### Flags globales (todos los modos)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-w`, `--wordlist` | Wordlist a usar (`-` = stdin) | `-w /usr/share/wordlists/dirb/common.txt` |
| `-t`, `--threads` | Hilos concurrentes (defecto 10) | `-t 50` |
| `-o`, `--output` | Guardar resultados en fichero | `-o resultados.txt` |
| `-q`, `--quiet` | Solo resultados, sin banner | `-q` |
| `-v`, `--verbose` | Salida detallada (incluye "missing" en algunos modos) | `-v` |
| `-k`, `--insecure` | Ignorar certificado TLS autofirmado | `-k` |
| `-p`, `--proxy` | Proxy (Burp/ZAP) | `-p http://127.0.0.1:8080` |
| `-H`, `--header` | Header HTTP custom (repetible) | `-H "X-Forwarded-For: 127.0.0.1"` |
| `-c`, `--cookie` | Cookies | `-c "PHPSESSID=abc; role=user"` |
| `-a`, `--user-agent` | User-Agent propio | `-a "Mozilla/5.0"` |
| `-U` / `-P` | Usuario/contraseña de HTTP Basic Auth | `-U admin -P admin` |
| `--timeout` | Timeout HTTP (s) | `--timeout 10s` |
| `-r`, `--follow-redirect` | Seguir redirecciones | `-r` |
| `-h`, `--help` | Ayuda **del modo** (la relevante) | `gobuster dir -h` |

### Modo `dir` (directorios y archivos)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-u`, `--url` | URL base objetivo | `-u http://target.local` |
| `-x`, `--extensions` | Extensiones a probar por cada palabra | `-x php,html,js,bak,zip` |
| `-s`, `--status-codes` | Códigos a considerar **válidos** (lista) | `-s 200,301,403` |
| `--exclude-length` | Excluir respuestas por longitud exacta (false positives) | `--exclude-length 123` |
| `-e`, `--extended` | Modo extendido (muestra más info de cada hallazgo) | `-e` |
| `-f`, `--follow-redirect` | Seguir 301/302 (importante para ver el destino real) | `-f` |
| `-n`, `--no-status` | Ocultar códigos de estado en la salida (limpiar) | `-n` |
| `-l`, `--show-length` | Mostrar longitud de contenido | `-l` |
| `-d`, `--debug` | Modo debug (peticiones/errores internos) | `-d` |

| Flags heredados de notas/versiones previas — **verificar con `gobuster dir -h`** |
|---|
| `-b` (códigos a excluir), `--delay` (pausa entre peticiones), `--wildcard` (wildcards), `--discover-backup` (buscar copias `*.bak`), `-U/-P` (auth). Si tu versión no los tiene, aplica el efecto equivalente con `-s`/`--exclude-length`, con throttle externo o con la wordlist. |

### Modo `dns` (subdominios)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-d`, `--domain` | Dominio objetivo | `-d target.local` |
| `-w`, `--wordlist` | Lista de subdominios | `-w seclists/subdomains-top1million-5000.txt` |
| `-r`, `--resolver` | Servidor DNS propio (evita el de sistema) | `-r 8.8.8.8` |
| `--show-ips` / `-i` | Imprime las IPs resueltas | `--show-ips` |
| `-t`, `--threads` | Hilos | `-t 30` |
| `-o` | Salida | `-o subdominios.txt` |

### Modo `vhost` (virtual hosts)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-u`, `--url` | IP/base URL del servidor | `-u http://10.10.10.10` |
| `-w`, `--wordlist` | Nombres candidatos de vhost | `-w vhosts.txt` |
| `--append-domain` | Añade el dominio base a cada palabra (si existe) | `--append-domain` |
| `-v` | Verbose (ver el Host enviado) | `-v` |

> [!tip] El truco de vhost: la IP base
> Cuando el DNS no resuelve (`dev.target.local`), añade al `/etc/hosts` la IP y accede por nombre — o prueba directamente con `gobuster vhost -u http://10.10.10.10 -w vhosts.txt`: cada intento manda `Host: palabra`. VHosts ocultos suelen devolver **200 con tamaño distinto** al default. Relacionado: [[HTTP Host header attacks]].

### Modo `fuzz` (puntos `FUZZ` arbitrarios)

```bash
gobuster fuzz -u "http://target/api/users/FUZZ" -w ids.txt
gobuster fuzz -u "http://target/?lang=FUZZ" -w /usr/share/wordlists/dirb/common.txt
```
La palabra se sustituye en **cualquier** posición de la URL: parámetros, paths, versiones, cabeceras vía `-H`.

### Modo `s3`

```bash
gobuster s3 -w buckets.txt --region us-east-1 --no-ssl
```
Solo prueba nombres; **la existencia no implica que sea público** — verificar permisos con un `GET`/`LIST` manual.

## Selección de wordlists

| Wordlist | Ruta (Kali) | Cuándo |
|---|---|---|
| `common.txt` (dirb) | `/usr/share/wordlists/dirb/common.txt` | **Primer pase**, rápido y efectivo |
| `big.txt` (dirb) | `/usr/share/wordlists/dirb/big.txt` | Segundo pase, más a fondo |
| `directory-list-2.3-small.txt` | `/usr/share/wordlists/dirbuster/` | Estándar de la industria (dirbuster) |
| `raft-small-words.txt` | `/usr/share/seclists/Discovery/Web-Content/` | Moderna, buena con CMS |
| `raft-large-words.txt` | igual | Auditorías grandes (tiempo ∝ tamaño) |
| Extensiones | — | `-x php,asp,aspx,jsp,txt,bak,zip,git` según stack |
| Subdominios | `/usr/share/seclists/Discovery/DNS/` | modo `dns` |

**Estrategia:** `common.txt` → filtrar FP → `-x` sobre hallazgos → `big.txt`/raft → recursion manual sobre directorios interesantes. Wordlist de 100k con `-t 10` contra un WAF es la mejor forma de ser bloqueado.

## Workflow: de la URL al mapa de ataque

### Paso 0 — Línea base (evitar falsos positivos)

```bash
curl -s -o /dev/null -w "%{http_code} %{size_download}\n" http://target/noexiste123
# 404 0       → normal: 404 con tamaño 0
# 200 1234    → ¡ojo! custom 200 para todo: false positives masivos
```

### Paso 1 — Enumeración inicial

```bash
gobuster dir -u http://target.local \
  -w /usr/share/wordlists/dirb/common.txt \
  -t 30 -o gobuster_common.txt
```

### Paso 2 — Filtrar y profundizar

```bash
# Extensiones típicas de stack LAMP/Windows
gobuster dir -u http://target.local -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,js,bak,zip,sql -s 200,301,401,403 -t 30

# Si hay páginas 200 "fantasma" todas del mismo tamaño:
gobuster dir -u http://target.local -w ... --exclude-length 1234
```

### Paso 3 — Recursión sobre hallazgos interesantes

```bash
# Gobuster no recursa por defecto (o requiere flag según versión); encadena tú:
gobuster dir -u http://target.local -w common.txt -q | \
  grep -E "Status: (200|301|403)" | awk '{print $1}' | while read d; do
    echo "== $d"
    gobuster dir -u "http://target.local$d" -w common.txt -q
  done
```

### Paso 4 — Otras capas

```bash
gobuster dns -d target.local -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --show-ips
gobuster vhost -u http://10.10.10.10 -w vhosts.txt --append-domain
gobuster fuzz -u "http://target.local/search?q=FUZZ" -w raft-small-words.txt
```

### Paso 5 — Análisis de resultados

| Salida | Qué significa | Acción |
|---|---|---|
| `/admin (Status: 301) [Size: 178]` | Existe y redirige (normalmente a `/admin/`) | Revisar destino con `-r` o curl |
| `(Status: 200) [Size: 4820]` | Contenido real | Abrir, medir, comparar con la línea base |
| `(Status: 403)` | Existe pero prohibido (WAF/directorio index off) | Bypass: `-H "X-Forwarded-For: 127.0.0.1"`, `/*`, `%2e`, etc. |
| `(Status: 401)` | Requiere auth | Fuerza bruta con credenciales (ver [[THC Hydra]]) |
| `(Status: 500)` | Error del servidor al procesar | Indicio de endpoint dinámico — probar parámetros |
| `429 Too Many Requests` | Rate limit/WAF | Bajar `-t`, rotar UA, pausar |
| Todos los paths → mismo `200 [Size: N]` | **Wildcard** del servidor | False positives: filtrar por tamaño o cambiar de wordlist |
| `/index.php.bak` | Backup expuesto | Extraer, buscar credenciales (`grep -i pass`) |

## Recetas: los 10 comandos reales

```bash
# 1. Barrido básico (el que se corre siempre)
gobuster dir -u http://target.local -w /usr/share/wordlists/dirb/common.txt

# 2. Con extensiones y códigos válidos
gobuster dir -u http://target.local -w /usr/share/wordlists/dirb/common.txt -x php,js,txt -s 200,301,403

# 3. Rápido, en silencio y con informe
gobuster dir -u http://target.local -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -t 60 -q -o out.txt

# 4. Detrás de login (cookie)
gobuster dir -u http://target.local/app -w common.txt -c "role=admin; sid=abc123"

# 5. Zona protegida por Basic Auth
gobuster dir -u http://target/admin -w common.txt -U admin -P admin123

# 6. Ignorar cert autofirmado + header anti-proxy
gobuster dir -u https://target.local -k -H "X-Forwarded-For: 127.0.0.1" -w common.txt

# 7. Subdominios
gobuster dns -d target.local -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --show-ips -t 30

# 8. Virtual hosts contra la IP
gobuster vhost -u http://10.10.10.10 -w vhosts.txt --append-domain -v

# 9. Fuzz de un parámetro
gobuster fuzz -u "http://target.local/page?file=FUZZ" -w /usr/share/wordlists/dirb/common.txt

# 10. Filtrar falsos positivos por longitud
gobuster dir -u http://target.local -w big.txt --exclude-length 1735 -q -o limpio.txt
```

**Detalle de la receta 2:** `-x` concatena la extensión a cada palabra (`admin` → `admin.php`, `admin.txt`…), `-s` restringe a códigos que *tú* consideras hallazgo (sin él, Gobuster trae un default que incluye 401/403 y satura).

## Detección y contramedidas (defensor)

| Señal | Dónde se ve | Mitigación |
|---|---|---|
| Ráfaga de GETs con patrones de wordlist (`/admin`, `/.git`, `/backup`) | Logs de acceso, WAF (ModSecurity, Cloudflare) | Rate limit por IP, reglas de path peligrosos |
| User-Agent de Gobuster o UA rotando sin navegador | `access.log` | Bloquear UA, exigir UA consistente con TLS fingerprint |
| Cientos de 404 en <1 min | Umbral de IDS web | Fail2ban / WAF burst rules |
| `429` repetidos | Propio del servidor | Indica que la defensa ya actuó: bajar ritmo |
| Enumeration de `/.git/config`, `/.env` | Alerta crítica | Bloquear rutas de desarrollo en el reverse proxy |
| Sin tráfico "inocente" previo | Un escaneo desde cero destaca | Rotar IPs legítimas, usar crawling primero |

> [!tip] Del lado del pentester
> Primero `robots.txt`, `sitemap.xml` y el propio DOM (sin ruido); Gobuster después y con `-t` moderado. En clientes reales el rate de peticiones va en el **alcance firmado**.

## Errores comunes (mensaje → arreglo)

1. **`unknown flag: --xyz` / `Error: flag provided but not defined`** — deriva de versión (v3 vs v4).
   → `gobuster <modo> -h` y adaptar el comando.
2. **`x509: certificate signed by unknown authority`** — TLS autofirmado.
   → `-k`.
3. **Falsos positivos masivos (todo `200`)** — custom 404 o wildcard.
   → Línea base con `curl` + `--exclude-length <N>` o `-s 301,403` para recortar.
4. **`429 Too Many Requests` / conexión cortada** — WAF/rate limit.
   → Bajar `-t` (10 o menos), añadir `-H`/UA, pausar; no insistir.
5. **`context deadline exceeded` / timeouts** — `--timeout` muy bajo o servidor lento.
   → Subir `--timeout 15s` y bajar hilos.
6. **Resultados vacíos con URL con puerto/ruta rara** — URL mal formada o sin esquema.
   → Incluir `http://` y la ruta exacta: `-u http://target:8080/app/`.
7. **`Error: 404 Not Found` para todo** — servidor no responde a la IP/Host esperado.
   → Probar con `curl -H "Host: dev.target.local"`; quizá el sitio es vhost-only → modo `vhost`.
8. **Escaneo eterno** — wordlist gigante contra servidor lento.
   → `common.txt` primero; `-t` razonable; `-q -o` para revisar después.
9. **Directorio encontrado pero 404 al entrar** — redirección con sesión/cookies.
   → Repetir con `-c` de sesión o `-r`.

## Cheatsheet — los 10 comandos del día a día

```bash
gobuster dir -u http://target.local -w /usr/share/wordlists/dirb/common.txt
gobuster dir -u http://target.local -w common.txt -x php,js,txt,bak
gobuster dir -u http://target.local -w big.txt -t 60 -q -o out.txt
gobuster dir -u http://target.local -w common.txt --exclude-length 1735
gobuster dir -u https://target.local -k -c "sid=abc" -w common.txt
gobuster dns -d target.local -w subdomains.txt --show-ips
gobuster vhost -u http://10.10.10.10 -w vhosts.txt --append-domain
gobuster fuzz -u "http://target/?q=FUZZ" -w common.txt
gobuster dir -u http://target -w common.txt -p http://127.0.0.1:8080
gobuster <modo> -h
```

| Tarea | Comando |
|---|---|
| Primer pase | `gobuster dir -u <URL> -w .../dirb/common.txt` |
| Con extensiones | `-x php,html,js,bak` |
| Guardar informe | `-o out.txt -q` |
| Evitar FP | `--exclude-length <n>` o `-s 200,301,403` |
| Detrás de sesión | `-c "k=v"` / `-U user -P pass` |
| Por Burp | `-p http://127.0.0.1:8080` |
| Subdominios | `gobuster dns -d <dominio> -w <wl> --show-ips` |
| VHosts | `gobuster vhost -u <IP> -w <wl>` |
| Param fuzz | `gobuster fuzz -u ".../x?q=FUZZ" -w <wl>` |
| Ayuda real | `gobuster dir -h` |

## Gobuster vs ffuf vs dirsearch vs feroxbuster vs WFuzz

| Criterio | **Gobuster** | ffuf | dirsearch | feroxbuster | [[WFuzz]] |
|---|---|---|---|---|---|
| Lenguaje | Go | Go | Python | Python | Python |
| Velocidad | Muy alta | **Máxima** (motor de fuzz) | Media | Media-Alta | Media |
| Recursión | Limitada/manual | No (por diseño) | **Sí** | **Sí** | No |
| Automatch FP | `--exclude-length` | `--mc/--fs` (match/filter size) | Sí | Auto-calibración | Baseline manual |
| Modos extra | dns, vhost, fuzz, s3 | cualquier posición `FUZZ` | dirs/files | dirs + extensions | cualquier fuzz |
| Salida | texto/JSON (según versión) | JSON | JSON/CSV | JSON | múltiples formatos |
| Curva | Baja | Baja-Media | Baja | Baja | Media |
| Mejor para | Rapidez y simplicidad | Fuzzing de verdad | Auditorías con recursion | Crawling + fuzz | Integración con scripts/[[Burp Suite Escaneo Dirigido y Estructuras de Datos No Estándar]] |

## Referencias

- [Repositorio oficial (OJ/gobuster)](https://github.com/OJ/gobuster) — README con ejemplos por modo y changelog de flags
- [SecLists](https://github.com/danielmiessler/SecLists) — wordlists de directorios/subdominios
- HackTricks — *Web Content Discovery* para bypass de 403 y estrategias
- GTFOBins no aplica (no explota binarios locales), pero sí la fase de reconocimiento de [[Reconocimiento y Escaneo de Redes y Web]]

## Relacionadas

- [[WFuzz]] — fuzzing profundo de parámetros/valores
- [[WhatWeb Herramienta de Fingerprinting]] — identificar tecnologías de lo encontrado
- [[Enumeración de Servicios y CMS]] — metodología de enumeración
- [[WPScan]] / [[JoomScan Herramienta de Escaneo]] — cuando el hallazgo es un CMS
- [[Nmap Guía Completa]] — servicios que sostienen la web (443, 8080, 8443)
- [[HTTP Host header attacks]] — explotación de vhosts descubiertos
- [[Ataques de Transferencia de Zona DNS (AXFR)]] — vía alternativa a brute-force DNS
