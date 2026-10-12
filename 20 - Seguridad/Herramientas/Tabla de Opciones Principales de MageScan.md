---
aliases:
  - MageScan
  - MageScan Opciones
  - MageScan Comandos
  - Fingerprinting de Magento
tags: [tool/magescan]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[WhatWeb Herramienta de Fingerprinting]]"
  - "[[WPScan]]"
  - "[[JoomScan Herramienta de Escaneo]]"
  - "[[Droopscan]]"
  - "[[Enumeración de Servicios y CMS]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Gobuster]]"
  - "[[Nmap Guía Completa]]"
fuente: "https://github.com/stymie/magescan"
---

# MageScan: Fingerprinting y Auditoría de Magento

**MageScan** es una herramienta de fingerprinting orientada exclusivamente a sitios **Magento** (Open Source / Commerce): detecta la presencia del CMS, estima su **versión**, comprueba la aplicación de parches de seguridad conocidos (SUPEE), descubre rutas y módulos instalados, parsea el sitemap y extrae información del servidor — todo sin autenticación y solo con peticiones HTTP legítimas.

> [!info] Por qué una herramienta específica de Magento
> Un fingerprinting genérico como [[WhatWeb Herramienta de Fingerprinting]] te dirá "hay un CMS aquí"; MageScan responde las preguntas que importan **después**: *¿qué versión exacta corre?*, *¿le aplicaron SUPEE-6788 / SUPEE-5344?*, *¿qué módulos de terceros están expuestos?*, *¿hay rutas de admin, API o backups al descubierto?* Para WordPress el equivalente es [[WPScan]], para Joomla [[JoomScan Herramienta de Escaneo]] y para Drupal [[Droopscan]] — ver [[Enumeración de Servicios y CMS]] para la metodología completa por CMS.

## Qué detecta (y cómo)

| Señal | Cómo lo obtiene | Riesgo si coincide con una versión vieja |
|---|---|---|
| **Versión del core** | Firmas en CSS/JS embebidos (`media/css_secure/...`), meta generator, rutas de skin | Versiones Magento 1 < 1.14.3 / 2.2.x < 2.2.8 tienen vulnerabilidades públicas conocidas (RCE, SQLi, XXE en el API) |
| **Parches SUPEE** | Comprueba la presencia/ausencia de archivos y comportamientos asociados a cada SUPEE (p. ej. SUPEE-5344, SUPEE-6788, SUPEE-8788, SUPEE-9767, SUPEE-10266, SUPEE-10415, SUPEE-10570, SUPEE-10752, SUPEE-10888, SUPEE-10975, SUPEE-11086, SUPEE-11155, SUPEE-11219) | Un Magento 1 sin SUPEE-6788 permite bypass de autenticación y ejecución remota |
| **Rutas inaccesibles** (*unreachable*) | Fuerza rutas comunes de Magento y detecta respuestas 200 vs custom-404 | `/admin`, `/api.php`, `/downloader/`, `/app/etc/local.xml`, backups de BD expuestos |
| **Módulos** | Detecta rutas y assets de módulos de terceros (p. ej. módulos de pago, extensiones comerciales) | Módulos abandonados = superficie de ataque sin parches |
| **Sitemap** | Descarga y parsea `sitemap.xml`/`sitemap_index.xml` | Expone el catálogo completo de URLs, incluidas áreas "ocultas" |
| **Servidor** | Cabeceras HTTP (`Server`, `X-Powered-By`, `Set-Cookie`, HSTS, etc.) | Banner con versión exacta de Apache/Nginx/PHP |

## Instalación

> [!warning] No viene en Kali
> `apt install magescan` **no existe**: MageScan no está en los repositorios de Kali/Debian. Se instala como paquete PHP vía Composer o como `.phar`. Requiere PHP ≥ 7.x con las extensiones `curl` y `dom`.

```bash
# Opción A — PHAR (la más cómoda y portátil)
wget https://github.com/stymie/magescan/releases/latest/download/magescan.phar
php magescan.phar --version

# Opción B — Composer (global)
composer global require stymie/magescan
export PATH="$PATH:$HOME/.composer/vendor/bin"   # o ~/.config/composer/vendor/bin
magescan --version

# Opción C — clonar y usar desde el repo
git clone https://github.com/stymie/magescan.git && cd magescan
composer install
php bin/magescan scan:all https://target.local
```

> [!warning] Dos proyectos históricos, una interfaz
> El proyecto original (`aoepeople/magescan`) está **abandonado**; el mantenido es [`stymie/magescan`](https://github.com/stymie/magescan). Las notas antiguas de este vault mezclaban flags que no existen en ninguno de los dos (ver "Errores comunes"). Antes de automatizar, confirma la sintaxis con `php magescan.phar list` en **tu** instalación: los comandos son comandos de Symfony Console (`scan:xxx`), no flags tipo `--json`.

## Catálogo de comandos

MageScan (stymie) se organiza en **subcomandos Symfony Console**, no en banderas sueltas:

| Comando | Qué hace | Ejemplo |
|---|---|---|
| `scan:all <url>` | Ejecuta todos los escaneos disponibles sobre el objetivo | `php magescan.phar scan:all https://target.local` |
| `scan:version <url>` | Detecta la versión de Magento por firmas en HTML/CSS/JS | `php magescan.phar scan:version https://target.local` |
| `scan:patches <url>` | Comprueba qué parches SUPEE están aplicados | `php magescan.phar scan:patches https://target.local` |
| `scan:unreachable <url>` | Fuerza rutas comunes de Magento y reporta accesibles/no accesibles | `php magescan.phar scan:unreachable https://target.local` |
| `scan:sitemap <url>` | Descarga y analiza el sitemap | `php magescan.phar scan:sitemap https://target.local` |
| `scan:server <url>` | Resume cabeceras del servidor (web server, PHP, cookies, seguridad) | `php magescan.phar scan:server https://target.local` |
| `list` | Lista todos los comandos disponibles | `php magescan.phar list` |
| `help <comando>` | Ayuda de un subcomando | `php magescan.phar help scan:all` |

### Opciones globales (compartidas por todos los `scan:`)

| Opción | Significado | Ejemplo |
|---|---|---|
| `--format=cli\|json\|markdown` | Formato de salida (por defecto `cli`) | `--format=json` |
| `--output=<archivo>` | Escribe el resultado en fichero en vez de stdout | `--output=informe.json` |
| `-h`, `--help` | Ayuda del comando | `php magescan.phar scan:all -h` |
| `-V`, `--version` | Versión de MageScan | — |

> [!tip] Sin `--update` ni `--proxy` nativos
> A diferencia de lo que decían notas antiguas, MageScan **no** tiene flags propios de proxy ni de actualización de base de datos: las firmas viajan dentro del propio código (por eso hay que actualizar el paquete con `composer global update` o bajando el `.phar` nuevo). Para salir por un proxy, exporta `HTTPS_PROXY=http://127.0.0.1:8080` y usá el proxy de Burp/ZAP como en cualquier cliente curl/PHP.

## Interpretación de la salida

Salida típica de `scan:all` (modo `cli`):

```
Checking version...
 Magento version identified: 1.9.2.4

Checking patches...
 SUPEE-5344      Not detected
 SUPEE-6788      Not detected
 SUPEE-8788      Unknown
 ...

Checking unreachable paths...
 /admin                  Not Found (404)
 /api.php                OK (200)
 /downloader/index.php   OK (200)     ← ¡índice de Magento Connect accesible!
 /app/etc/local.xml     Not Found (404)

Server information...
 HTTPServer: Apache/2.4.10 (Ubuntu)
 X-Powered-By: PHP/5.5.9-1ubuntu4.21
```

| Campo | Cómo leerlo |
|---|---|
| `Magento version identified: X` | Versión estimada. Si es "Unknown", la firma fue eliminada en versiones recientes o el tema oculta los assets — prueba `scan:version` contra `/media/css/` o usa [[WhatWeb Herramienta de Fingerprinting]] como respaldo |
| `Not detected` (patch) | El parche **no** está aplicado → vulnerabilidad pública asociada abierta |
| `OK (200)` en rutas sensibles | Ruta accesible: revisar manualmente con curl/navegador antes de reportar (puede ser redirect o landing genérica) |
| `Not Found (404)` | Ruta inexistente — Magento 1 suele devolver 404 con página CMS en vez de 404 real, interpreta con cuidado el cuerpo |
| Cabeceras del servidor | Versiones exactas de webserver/PHP → buscar CVEs del stack subyacente |

## Workflow: de cero a informe de Magento

### Paso 1 — ¿Es Magento?

```bash
# Confirmación rápida cruzada (nunca confíes de una sola herramienta)
whatweb https://target.local -v
curl -sk https://target.local/ | grep -i -E "mage|skin/frontend"
curl -sk -o /dev/null -w "%{http_code}\n" https://target.local/media/js/prototype/prototype.js
# 200 → casi seguro Magento 1
```

### Paso 2 — Versión y parches

```bash
php magescan.phar scan:version  https://target.local
php magescan.phar scan:patches https://target.local
```

### Paso 3 — Superficie (rutas, módulos, sitemap)

```bash
php magescan.phar scan:unreachable https://target.local
php magescan.phar scan:sitemap     https://target.local
```

### Paso 4 — Informe machine-readable

```bash
php magescan.phar scan:all https://target.local --format=json --output=magescan.json
jq '.version, .patches[] | select(.status != "applied")' magescan.json
```

### Paso 5 — Profundizar

- Rutas sensibles descubiertas → [[Gobuster]] con wordlist de Magento (`SecLists/Discovery/Web-Content/Magento.txt`).
- Plugins/módulos → comprobar versiones y exploits públicos; para WordPress migrar a [[WPScan]].
- Todo dentro de la fase de enumeración: [[Enumeración de Servicios y CMS]].

## Recetas (copy-paste)

```bash
# 1. Barrido completo con informe JSON
php magescan.phar scan:all https://target.local --format=json --output=scan.json

# 2. Solo versión (rápido, para scripts)
php magescan.phar scan:version https://target.local | grep -i "version identified"

# 3. Solo parches, en markdown para el informe
php magescan.phar scan:patches https://target.local --format=markdown --output=patches.md

# 4. Rutas sensibles antes/después de cada cambio de alcance
php magescan.phar scan:unreachable https://target.local

# 5. Por un proxy de interceptación (Burp/ZAP)
HTTPS_PROXY=http://127.0.0.1:8080 php magescan.phar scan:all https://target.local

# 6. Todo sobre un lote de objetivos (loop)
for u in $(cat magento_targets.txt); do
  php magescan.phar scan:all "$u" --format=json --output="$(echo "$u" | tr -c 'a-zA-Z0-9' '_').json"
done
```

**Detalle de la receta 1:** `scan:all` ejecuta los demás `scan:*` en secuencia — es el comando de "primer pase"; los individuales sirven para re-ejecutar solo una comprobación sin repetir todo el tráfico.

## Detección y contramedidas (defensor)

| Señal que ve el defensor | Origen en MageScan | Mitigación |
|---|---|---|
| Ráfaga de GETs a rutas de Magento (`/downloader/`, `/api.php`, `/app/etc/local.xml`) | `scan:unreachable` | WAF/ModSecurity con reglas de rutas Magento; bloquear `/app`, `/downloader` en el reverse proxy |
| Peticiones a `media/css_secure/`, `media/js/prototype/` | `scan:version` | Son assets legítimos; el abuso se ve por volumen — rate limit por IP |
| Descarga de `sitemap.xml` | `scan:sitemap` | Normal y esperado; si el catálogo es sensible, exigir auth |
| Sin UA de navegador / UA por defecto de PHP-cURL | todas | Bloquear o alertar por UA sospechosa (aunque es evadible) |
| Varias herramientas en secuencia (WhatWeb + MageScan + Gobuster) | workflow completo | Fail2ban/WAF burst rules; correlación de eventos |

> [!warning] Alcance y legal
> `scan:unreachable` fuerza decenas de rutas: en producción puede tripear alarmas y, en servidores frágiles, generar 500s. Ejecútalo solo con autorización escrita y fuera de horario crítico — mismo criterio que para [[Gobuster]].

## Errores comunes (mensaje → arreglo)

1. **`Could not open input file: magescan`** — el binario no está en el PATH o no usas el `.phar`.
   → Ejecuta con intérprete explícito: `php magescan.phar scan:all <url>` o corrige el PATH de Composer global.
2. **`PHP Fatal error: Uncaught Error: Class 'CurlHandle' not found` / `Call to undefined function curl_init()`** — falta la extensión curl.
   → `sudo apt install php-curl php-xml` (la extensión `dom` también es obligatoria para `scan:sitemap`).
3. **`Failed to execute ... scan:all` / comando desconocido** — estás invocando flags estilo `-u https://...` de la nota antigua, que **no existen**.
   → La URL es argumento posicional del subcomando: `scan:all <url>`. Ver `php magescan.phar list`.
4. **Versión `Unknown` siempre** — el sitio está tras un CDN/WAF que muta los assets, o es Magento 2 (MageScan está orientado a Magento 1; en M2 la detección es débil).
   → Cruzar con [[WhatWeb Herramienta de Fingerprinting]] (`whatweb -a 3`) y con las firmas de `/static/version`/`/media/_cache` de Magento 2.
5. **`SSL certificate problem: self signed certificate`** — cert autofirmado en lab.
   → `export CURLOPT_SSL_VERIFYPEER=false` no funciona (flag interno no expuesto): usa un lab con cert válido autofirmado en el CA store del sistema o un proxy local que lo resuelva.
6. **Resultados parciales / 403 en todo** — el WAF bloquea al cliente PHP-cURL.
   → Rotar User-Agent (`HTTPS_PROXY` + Burp con UA de navegador) y reducir el alcance a comandos individuales.

## Comparación rápida: herramientas de fingerprinting por CMS

| Criterio | **MageScan** | [[WhatWeb Herramienta de Fingerprinting]] | [[WPScan]] | [[JoomScan Herramienta de Escaneo]] | [[Droopscan]] |
|---|---|---|---|---|---|
| Alcance | Solo Magento | Cualquier web (1800+ plugins) | Solo WordPress | Solo Joomla | Solo Drupal |
| Versión exacta | Sí (firmas Magento) | A veces (banner/firmas) | Sí (+ API de vulns) | Sí | Parcial |
| Parches/vulns | Sí (SUPEE) | No | Sí (base WPScan) | Sí (base de datos) | Débil |
| Fuerza bruta | No | No | Sí (usuarios/creds) | Sí | No |
| Salida JSON | Sí | Sí | Sí | Sí (texto) | No |
| Mejor para | Auditoría Magento profunda | Identificación general rápida | WordPress completo | Joomla | Drupal (legacy) |

## Cheatsheet

```bash
php magescan.phar scan:all https://target.local
php magescan.phar scan:version https://target.local
php magescan.phar scan:patches https://target.local
php magescan.phar scan:unreachable https://target.local
php magescan.phar scan:sitemap https://target.local
php magescan.phar scan:server https://target.local
php magescan.phar scan:all https://t.local --format=json --output=scan.json
php magescan.phar list
```

| Tarea | Comando |
|---|---|
| Primer pase | `scan:all <url>` |
| Versión sola (scripts) | `scan:version <url>` |
| Parches para el informe | `scan:patches --format=markdown --output=p.md` |
| Rutas expuestas | `scan:unreachable <url>` |
| Informe automatable | `--format=json --output=` |
| Ayuda real | `php magescan.phar help scan:all` |

## Referencias

- [Repositorio oficial (stymie/magescan)](https://github.com/stymie/magescan) — comandos, requisitos PHP y changelog
- [Magento Open Source](https://magento.com) / [Adobe Commerce release notes](https://experienceleague.adobe.com/docs/commerce-operations/release-notes/overview.html) — versiones y SUPEE correspondientes
- HackTricks — *CMS Identification* y sección Magento para el post-descubrimiento

## Relacionadas

- [[WhatWeb Herramienta de Fingerprinting]] — confirmación genérica de tecnología cuando MageScan no identifica versión
- [[Enumeración de Servicios y CMS]] — metodología de la que forma parte esta fase
- [[WPScan]] / [[JoomScan Herramienta de Escaneo]] / [[Droopscan]] — hermanos por CMS
- [[Gobuster]] — profundizar en rutas descubiertas (`/downloader`, backups)
- [[Nmap Guía Completa]] — servicios subyacentes (80/443/8443) y scripts NSE de respaldo
- [[Reconocimiento y Escaneo de Redes y Web]] — dónde encaja el fingerprinting en el flujo completo
