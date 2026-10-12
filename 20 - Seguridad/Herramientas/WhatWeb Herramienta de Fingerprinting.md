---
tags:
  - ciberseguridad
  - herramientas
  - whatweb-fingerprinting
tipo: herramienta
estado: terminado
---

# WhatWeb: Fingerprinting de Tecnologías Web

**WhatWeb** identifica **qué tecnología** hay detrás de una URL: CMS, frameworks, servidores web, bibliotecas JS/CSS, paneles de administración, dispositivos de red y más de 1800 plugins de firma. Devuelve tecnologías **con sus versiones** cuando las firmas lo permiten — la entrada natural del reconocimiento web: antes de atacar algo hay que saber qué es.

> [!info] Por qué WhatWeb
> Todo lo demás del reconocimiento web presupone esta respuesta: si WhatWeb dice "WordPress 5.8 + Apache + jQuery 3.5", tus wordlists, tus CVEs y tus herramientasCMS ([[WPScan]], [[Tabla de Opciones Principales de MageScan]]) cambian. Es rápido (modo pasivo = 1 petición) y su ventaja sobre httpx/Wappalyzer es la **extensibilidad**: más de 1800 plugins y la posibilidad de escribir los tuyos en Ruby.

## Instalación

```bash
# Kali (incluido por defecto)
sudo apt update && sudo apt install whatweb
whatweb --version

# Desde fuente (últimos plugins)
git clone https://github.com/urbanadventurer/WhatWeb.git && cd WhatWeb
sudo apt install ruby ruby-dev
bundle install          # o: gem install addressable rexical trollop
./whatweb --version

# Gem (binario en el PATH)
sudo gem install whatweb
```

> [!tip] Ruby de Kali vs rbenv
> El paquete de Kali usa el Ruby del sistema (funciona out of the box). Si instalás gems con rbenv/rvm, tenés que invocar el WhatWeb clonado con `./whatweb` (su propio Ruby), no confiar en el `whatweb` del PATH — una de las causas de "ejecuta otra versión de la que creés".

## Modelo de plugins

Cada tecnología la detecta un **plugin** (Ruby) que aplica reglas sobre: cabeceras HTTP (`Server`, `X-Powered-By`), cuerpo HTML (meta generator, scripts, comentarios), URLs, cookies, favicon-hasheado, y —según nivel de agresividad— peticiones extra (HEAD/GET a rutas probables). La salida lista las tecnologías con su **evidencia** (la firma que coincidió).

```bash
whatweb https://target.local
# http://target.local [200 OK] Apache/2.4.41 [(Ubuntu)], HTTPServer[Ubuntu][Apache/2.4.41], PHP[7.4.3], ...

whatweb --list-plugins | wc -l     # inventario completo
whatweb --search-plugins=nginx      # plugins que mencionan nginx
whatweb --info-plugins HTTPServer   # qué reglas usa un plugin concreto
```

## Selección de objetivos

| Sintaxis | Qué hace | Ejemplo |
|---|---|---|
| `whatweb <url>` | Un objetivo | `whatweb http://target.local` |
| `-i, --input-file=<archivo>` | Un objetivo por línea (acepta IPs, rangos CIDR, nombres) | `-i objetivos.txt` |
| `--url-prefix` / `--url-suffix` | Prefija/sufija a cada línea de `-i` | `--url-prefix=https://` |
| `--url-pattern` | Inserta `%s` en la plantilla | `--url-pattern=http://%s:8443` |

## Niveles de agresividad (`-a`, el concepto clave de WhatWeb)

| Nivel | Nombre | Semántica | Peticiones aprox. | Cuándo |
|---|---|---|---|---|
| `1` | **Stealthy** | Solo la(s) petición(es) inicial(es). Sin extras. | 1 | Objetivos sensibles, primer barrido, producción |
| `2` | **Default/Enrage** | Algunos plugins hacen peticiones HEAD extra para confirmar (custom-404, banners). | 2–10 | Uso general |
| `3` | **Aggressive** | Plugins adicionales: GETs a rutas probables, hash de favicon, detección de errores, chequeos de versión. | 10s | Lab / con permiso |
| `4` | **Heavy** | Máxima intensidad: rutas forzadas, comprobaciones invasivas de plugins/paneles. | 100s | Solo con alcance explícito |

```bash
whatweb -a 1 http://target.local        # un paquete: lo más discreto
whatweb -a 3 http://target.local        # firma fina con versiones
whatweb --aggression 4 -t 10 -i lab.txt # lote en laboratorio
```

> [!warning] Agresividad = ruido y riesgo
> Los niveles 3–4 dejan en los logs decenas de peticiones por host con rutas de fingerprinting (`/favicon.ico` versionado, rutas de paneles, versiones forzadas). En producción puede activar WAF/IDS y, contra servidores frágiles, duplicar el coste de cada página. El nivel 1–2 es el estándar profesional; el 3 se usa cuando el 2 no identifica y hay permiso; el 4, en lab.

## Tabla de flags

### Salida

| Flag | Significado | Ejemplo |
|---|---|---|
| `-q, --quiet` | Solo resultados, sin banner/progreso | `-q` |
| `-v, --verbose` | Salida detallada: plugin por plugin, con detalle de cada hallazgo | `-v` |
| `--no-errors` | Silencia errores (útil en lotes) | `--no-errors` |
| `--color` / `--no-color` | Fuerza color / lo quita | `--color` |
| `-o, --output <fmt> <archivo>` | Formato de log + ruta (sintaxis antigua, aún válida) | `-o brief out.txt` |

### Formatos de log (`--log-<fmt>=<archivo>`)

| Formato | Contenido | Uso |
|---|---|---|
| `--log-brief` | Una línea por target: tecnologías detectadas | Lectura humana, informes rápidos |
| `--log-verbose` | Detalle completo de cada plugin/evidencia | Análisis forense del fingerprinting |
| `--log-json` | JSON (array de targets con `plugins` y `objects`) | **Automatización** (`jq`) |
| `--log-xml` | XML | Integración con pipelines Java/antiguos |
| `--log-csv` | CSV | Hojas de cálculo de inventario |
| `--log-html` | Reporte HTML con hipervínculos | Entrega al cliente |

```bash
whatweb -a 2 --log-json=informe.json -i objetivos.txt
jq -r '.[] | select(.plugins | test("WordPress")) | .target' informe.json
```

**Interpretación del JSON:** cada elemento del array es un target; `plugins` es un objeto donde cada clave es un plugin y sus valores contienen `string` (evidencia), `version` y a veces `accounts`/`categories`. Es el campo fiable para filtrar por versión (`"version":"5.8.1"`).

### HTTP

| Flag | Significado | Ejemplo |
|---|---|---|
| `-U, --user-agent` | User-Agent propio | `-U "Mozilla/5.0"` |
| `--header` | Header adicional (repetible) | `--header="Cookie: a=b"` |
| `--cookie` | Cookies | `--cookie="PHPSESSID=abc"` |
| `--proxy` | Proxy (Burp/ZAP/SOCKS) | `--proxy=127.0.0.1:8080` |
| `--proxy-user` | Credenciales del proxy | `--proxy-user=user:pass` |
| `-t, --threads` | Hilos (default 25) | `-t 10` |
| `--open-timeout` | Timeout de apertura de conexión (segundos) | `--open-timeout=10` |
| `--read-timeout` | Timeout de lectura de respuesta (segundos) | `--read-timeout=20` |

### Plugins

| Flag | Significado | Ejemplo |
|---|---|---|
| `--list-plugins` (`-l`) | Lista todos los plugins | `--list-plugins` |
| `--info-plugins` | Info (reglas) de todos | `--info-plugins` |
| `--info-plugin <nombre>` | Info de un plugin | `--info-plugin WordPress` |
| `--search-plugins=<texto>` | Busca plugins por nombre/texto | `--search-plugins=drupal` |
| `--plugins=<l1,l2>` | Restringe el set de plugins activos | `--plugins=Apache,PHP` |
| `--grep=<texto\|regex>` | **Muestra solo targets cuyo resultado contenga el patrón** | `--grep=OpenSSH` |
| `--bitwise <n1,n2…>` | Filtra valores numéricos de resultados aplicando máscara AND bitwise (firmas numéricas/hashes) | `--bitwise=0x10` |
| `--custom-plugin=<def>` | Plugin propio inline (Ruby: `strings`, `regex`, `md5`, `favicon`…) | ver abajo |

```bash
# "Dame solo los que corren OpenSSH viejo" sobre un lote ya escaneado en verbose
whatweb --log-verbose -i web.txt | whatweb --log-brief --grep="OpenSSH/7."

# Plugin inline: detecta un marker propio en el body
whatweb --custom-plugin='MyMarker { strings { body: "X-INTERNAL-FLAG" } }' http://target.local
```

> [!tip] `--grep` también funciona sobre logs
> WhatWeb puede releer sus propios logs con `-i`: combinar `--log-verbose` + `--grep` convierte al WhatWeb en un mini-filtro de inventario corporativo (p. ej. `--grep="Apache/2.4.49"` para cazar el CVE-2021-41773 a escala).

## Workflows

### 1. Fingerprint de un objetivo

```bash
whatweb -a 2 -v http://target.local
```

Lectura de la salida brief:

```
http://target.local [301 Moved Permanently] → https://target.local/
https://target.local [200 OK] Apache/2.4.41 [(Ubuntu)], Country[RESERVED][GB], HTTPServer[Ubuntu][Apache/2.4.41],
  PHP[7.4.3], RedirectLocation[https://target.local/], UncommonHeaders[x-powered-by], WordPress[5.8.1]
```

| Evidencia | Cómo leerla |
|---|---|
| `[200 OK]` / `[301 …] → destino` | Estado y redirección final (si el destino es otro host, repetir contra él) |
| `Apache/2.4.41 [(Ubuntu)]` | Servidor + SO: traducir a CVEs de Apache/Ubuntu |
| `PHP[7.4.3]` | Versión exacta de PHP (EOL desde nov 2022 → buscar CVEs) |
| `WordPress[5.8.1]` | CMS + versión → pipeline a [[WPScan]] |
| `Country[RESERVED][GB]` | GeoIP (fuente MaxMind) — orientativa, no fiable |
| `RedirectLocation[...]` | Hacia dónde va el 301 — seguir con WhatWeb en destino |

### 2. Inventario de una red

```bash
whatweb -a 1 -q --log-json=inv.json -i urls.txt --no-errors
jq -r '.[] | "\(.target)\t\(.plugins | keys | join(","))"' inv.json | column -t -s$'\t'
```

### 3. Caza de versiones vulnerables

```bash
whatweb -a 2 -i web.txt --log-brief - | grep -E "Apache/2\.4\.49|OpenSSH/[0-6]\.|PHP/5\."
```

## Detección y contramedidas (defensor)

| Señal | Origen | Mitigación |
|---|---|---|
| Una petición con UA de WhatWeb (o Ruby) | nivel 1 | UA bloqueable (evadible con `-U`) |
| Ráfaga de HEAD/GET a favicon, generadores meta y rutas de paneles | niveles 2–3 | WAF con reglas de fingerprinting; rate limit |
| Decenas de GETs a paths de plugins por host | nivel 4 | Bloqueo de rutas de paneles (`/wp-admin`, `/admin/`, `/phpmyadmin`) |
| Sonda de custom-404 (GET a un path aleatorio y comparación) | nivel 2 | Respuesta 404 personalizada consistente reduce la precisión del detector |
| Escaneo masivo con `-i` de rangos completos | inventario | IDS de scanning; geo/IP reputation |

> [!tip] Defensa desde el diseño
> Quitar el banner `Server`/`X-Powered-By` y el meta `generator` no "arregla" nada (las firmas profundas siguen funcionando: favicon hash, CSS/JS paths), pero **baja la tasa de identificación casual** y obliga al atacante a niveles más ruidosos. Versiones ocultas ≠ versiones parcheadas: parcheá siempre.

## Errores comunes (mensaje → arreglo)

1. **`WhatWeb requires Ruby >= 2.0` / syntax errors al arrancar** — Ruby viejo del sistema o gemas mezcladas.
   → Kali actual (`ruby --version` ≥ 3.x) o el clon con su propio bundler (`bundle install && ./whatweb`).
2. **`bundler: command not found: whatweb` o el binario no es el tuyo** — gem vs clon vs paquete en conflicto.
   → `which whatweb` y `./whatweb` explícito desde el clon.
3. **`SSL_connect ... certificate verify failed` en HTTPS con cert autofirmado** — WhatWeb **no expone flag para omitir la verificación TLS**.
   → Workaround: interceptar por un proxy local que valide por vos (`--proxy=127.0.0.1:8080` con el CA de Burp instalado), o lab con cert firmado por el CA del sistema.
4. **0 resultados / timeouts contra sitios lentos** — timeouts por defecto insuficientes o `-a` alto con `threads` bajo.
   → `--open-timeout=20 --read-timeout=40`; nivel 1 para probar conectividad.
5. **Resultados "Unknown" o demasiado genéricos** — el sitio está tras CDN/WAF que normaliza banners.
   → `-a 3` para firmas profundas; cruzar con cabeceras reales vía `curl -svI` y con [[Nmap Guía Completa]] (`http-enum`, `http-headers`).
6. **Lote gigante que tarda horas** — `-t 25` por defecto contra hosts lentos + niveles altos.
   → `-a 1` para el barrido base, `-a 2`/`-3` solo en hosts seleccionados; `--no-errors` y `-q` para reducir I/O de consola.
7. **`--log-json` vacío** — se pasó `-o` con formato antiguo o la ruta no es escribible.
   → `--log-json=./informe.json` (ruta absoluta relativa al cwd actual, revisar permisos).

## WhatWeb vs httpx vs Wappalyzer vs herramientas por CMS

| Criterio | **WhatWeb** | httpx | Wappalyzer (ext/node) | [[WPScan]] | [[Tabla de Opciones Principales de MageScan]] |
|---|---|---|---|---|---|
| Naturaleza | CLI Ruby, activo | CLI Go, activo-pasivo | Extensión navegador / librería | CLI Ruby | CLI PHP |
| Detec. de tecnología | **1800+ firmas, con versión** | Básica (título, tech probe limitada) | Buena (DOM + requests) | Solo WP | Solo Magento |
| Agresividad configurable | **Sí (1–4)** | Banda propia (`-fr`…) | Pasiva (navegador) | Enum + ataques | Comandos scan |
| Plugins propios | **Sí (Ruby, --custom-plugin)** | No | Sí (regex) | No | No |
| Salida machine | json/xml/csv/html/sql | json | json | json | json |
| Fuerza bruta | No | No | No | **Sí** | No |
| Velocidad lote | Alta | **Muy alta** | n/a | Media | Media |
| Mejor para | Fingerprint a fondo | Probar quién responde rápido | Uso desde navegador | WordPress completo | Magento completo |

Flujo típico: httpx (quién responde) → WhatWeb `-a 2` (qué es) → herramienta por CMS → [[Gobuster]]/[[WFuzz]] (qué más hay). Metodología completa en [[Enumeración de Servicios y CMS]].

## Cheatsheet

```bash
whatweb http://target.local                          # fingerprint básico (-a 2)
whatweb -a 1 http://target.local                     # stealthy: 1 paquete
whatweb -a 3 -v http://target.local                  # agresivo con detalle
whatweb -i urls.txt -q --log-json=out.json --no-errors
whatweb --log-brief --grep="OpenSSH" -i verbose.txt
whatweb --list-plugins | head
whatweb --search-plugins=wordpress
whatweb --info-plugin WordPress
whatweb --plugins=Apache,PHP,WordPress http://target.local
whatweb -t 10 --proxy=127.0.0.1:8080 http://target.local
whatweb --open-timeout=20 --read-timeout=40 https://slow.local
whatweb --user-agent="Mozilla/5.0" http://target.local
```

| Tarea | Comando |
|---|---|
| Un target rápido | `whatweb http://t` |
| Stealth | `-a 1` |
| Versiones (con permiso) | `-a 3 -v` |
| Lote + JSON | `-i lote.txt -q --log-json=out.json` |
| Cazar versión concreta | `--log-verbose` + `--grep="patrón"` |
| Por proxy | `--proxy=127.0.0.1:8080` |
| Ayuda real | `whatweb --help` |

## Referencias

- [Repositorio oficial (urbanadventurer/WhatWeb)](https://github.com/urbanadventurer/WhatWeb) — wiki con niveles de agresión, formatos de log y guía de plugins propios
- [Base de plugins / signatures](https://github.com/urbanadventurer/WhatWeb/tree/master/plugins) — leer firmas para escribir las tuyas
- `whatweb --help` — referencia viva en tu instalación

## Relacionadas

- [[Enumeración de Servicios y CMS]] — la fase en la que WhatWeb es el paso "¿qué es esto?"
- [[Reconocimiento y Escaneo de Redes y Web]] — el flujo completo de reconocimiento
- [[Nmap Guía Completa]] — fingerprint TCP/TLS complementario (`-sV`, scripts `http-*`)
- [[WPScan]] / [[Tabla de Opciones Principales de MageScan]] / [[JoomScan Herramienta de Escaneo]] / [[Droopscan]] — profundización cuando el CMS está identificado
- [[Gobuster]] / [[WFuzz]] — la siguiente capa: rutas y parámetros de lo identificado
