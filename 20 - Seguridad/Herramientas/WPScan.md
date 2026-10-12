---
aliases: ["WPScan", "wpscan", "WordPress Scanner"]
tags: [tool/wpscan]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[WhatWeb Herramienta de Fingerprinting]]"
  - "[[Tabla de Opciones Principales de MageScan]]"
  - "[[JoomScan Herramienta de Escaneo]]"
  - "[[Droopscan]]"
  - "[[WFuzz]]"
  - "[[THC Hydra]]"
  - "[[Gobuster]]"
  - "[[Enumeración de Servicios y CMS]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
fuente: "https://github.com/wpscanteam/wpscan"
---

# WPScan: Escáner de Seguridad de WordPress

**WPScan** es el escáner de seguridad para WordPress: enumera usuarios, plugins y temas (detectados **y** vulnerables según su base de datos mantenida por la comunidad), busca copias de configuración y exportaciones de base de datos, comprueba timthumbs y puede ejecutar ataques de contraseña contra el login (`wp-login`) y XML-RPC con throttling propio. Es el equivalente de [[Tabla de Opciones Principales de MageScan]] para el CMS más extendido del mundo.

> [!info] Por qué WPScan
> WordPress concentra el plugin/CMS más atacado del planeta: versiones viejas de core, de plugins abandonados y usuarios `admin` por defecto. WPScan no "finge" vulnerabilidades: cruza lo detectado con una **base de datos de vulnerabilidades** (CVEs y advisories) y con hashes de contenido de plugins/temas para estimar su versión incluso con los archivos ofuscados. La profundización va por aquí antes que por fuzzing ciego ([[WFuzz]]).

## Instalación

```bash
# Kali (empaquetado y actualizado con los repos)
sudo apt update && sudo apt install wpscan
wpscan --version

# Gem (siempre la última)
sudo gem install wpscan

# Docker (aislado, sin tocar el sistema)
docker run --rm wpscanteam/wpscan --url http://target.local

# Actualizar la base local de vulnerabilidades (siempre antes de cada uso)
wpscan --update
```

> [!warning] La base local caduca
> Sin `--update`, WPScan compara contra advisories viejos y **falsos negativos**: plugins vulnerables marcados como seguros. Ejecútalo al inicio de cada sesión de auditoría (la DB viaja en el gem/local DB, independiente del token API).

## API token: qué da y cuándo hace falta

| Sin token | Con token gratuito ([wpscan.com](https://wpscan.com/)) |
|---|---|
| Enumeración de usuarios, plugins, temas | Igual |
| Detección de versiones por hash | Igual |
| Match con vulnerabilidades **del core y plugins populares** | Match con la base **completa** (miles de advisories) |
| Límites de peticiones a la API | 25 peticiones/día (uso no comercial); planes de pago para más |

```bash
export WPSCAN_API_TOKEN=<tu-token>     # si exportas la variable, no hace falta el flag
wpscan --api-token <tu-token> --url https://target.local -e vp,vt
```

> [!info] El token es de cuota diaria
> Los `25/día` son peticiones al servicio de vulnerabilidades, no requests HTTP al objetivo. En un engagement con decenas de hosts, prioriza `-e vp,vt` solo en los targets con instalaciones confirmadas y reparte el uso; sin token, WPScan sigue siendo útil (enum + versiones), solo pierde el motor de vulns.

## Enumeración (`-e` / `--enumerate`)

| Valor | Qué enumera | Detalle |
|---|---|---|
| `u` | **Usuarios** | `?author=N` (redirects 301 → `/author/usuario/`) y endpoint REST `wp-json/wp/v2/users`. Rango por defecto `u[1-10]` |
| `u[1-100]` | Usuarios, author IDs 1–100 | Más amplio = más ruido, pero caza IDs altos |
| `p` | Plugins detectados | Lee el HTML del sitio + prueba rutas comunes de plugins |
| `t` | Temas detectados | Idem para temas (style.css con versión) |
| `vp` | Plugins **vulnerables** | Cruza los detectados con la DB (usa cuota API) |
| `vt` | Temas **vulnerables** | Ídem para temas |
| `a` | TimThumbs | Versiones de la librería de imágenes vulnerable |
| `b` | Copias de configuración | `wp-config.php.bak`, `.old`, `~`, etc. |
| `d` | Exportaciones de BD | `.sql`, `.sql.gz` accesibles |
| `m` | Media | Archivos multimedia accesibles (metadatos, renombrados) |

```bash
wpscan --url https://target.local --enumerate u,vp,vt
wpscan --url https://target.local --enumerate u[1-100],p,t
```

> [!tip] Enum en dos fases
> Primero `--enumerate u` (barato, 10 IDs, sin API) para confirmar que es WP y obtener usuarios reales; luego `-e vp,vt` con token solo si el objetivo lo merece. Lanzar `-e p,t` contra un sitio grande generacientos de peticiones a rutas de plugins — usa el nivel de ruido que el alcance permite.

## Ataques de contraseña (`-U`, `-P`, `--password-attack`)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-U, --usernames` | Lista (o usuario) de usuarios | `-U admin.txt` |
| `-P, --passwords` | Lista de contraseñas | `-P /usr/share/wordlists/rockyou.txt` |
| `--password-attack` | Método: `wp-login` (form), `xmlrpc` (un intento/petición), `xmlrpc-multicall` (agrupa intentos por petición — rápido pero ruidoso) | `--password-attack xmlrpc` |
| `-t, --max-threads` | Hilos concurrentes (default 5) | `-t 1` |
| `--throttling` | **Pausa en segundos entre peticiones** (default 0) | `--throttling 5` |
| `--stealthy` | Cambia UA/headers por petición (más lento, menos fingerprintable) | `--stealthy` |

```bash
# Spray prudente: 1 usuario × lista, con pausa de 5 s
wpscan --url https://target.local -U admin -P passwords.txt --password-attack wp-login --throttling 5 -t 1
```

> [!warning] Fuerza bruta contra WordPress = cuentas reales
> El login de WordPress no trae rate limit por defecto, pero los hosts gestionados y los WAF sí: un `-P` de rockyou sin `--throttling` bloquea la IP, tripa el sitio o **bloquea al usuario real**. En engagement: alcance escrito, usuario concreto, listas cortas y filtradas (`rockyou` entero rara vez está aprobado). `xmlrpc-multicall` multiplica intentos por conexión — es la vía más rápida **y** la que más ruido genera. Considera spraying con [[THC Hydra]]-estilo (una pass, muchos usuarios) cuando aplique lockout.

## Otras opciones de escaneo

| Flag | Significado | Ejemplo |
|---|---|---|
| `--force` | Escanear aunque WPScan no detecte WordPress | `--force` |
| `--exclude-content-hash` | Excluir temas/plugins por hash de contenido | `--exclude-content-hash` |
| `--plugins-detection` / `--plugins-version-detection` | Modo de detección de plugins: `mixed` (passive+aggressive), `passive`, `aggressive` | `--plugins-detection passive` |
| `--user-agent` / `-u` | UA propio (o usa `--stealthy`) | `-u "Mozilla/5.0"` |
| `--proxy` | Proxy Burp/ZAP | `--proxy http://127.0.0.1:8080` |
| `--proxy-userpass` | Credenciales del proxy | `--proxy-userpass u:p` |
| `--http-basic` | Basic auth delante del sitio | `--http-basic user:pass` |
| `--disable-tls-checks` | Omite verificación TLS (labs con cert autofirmado) | `--disable-tls-checks` |
| `--connect-timeout` / `--request-timeout` / `--response-timeout` | Timeouts de conexión/respuesta | `--request-timeout 30` |
| `--cookies-from-browser` | Reutiliza cookies del navegador (objetivo tras login) | `--cookies-from-browser firefox` |

## Salida

| Flag | Significado | Ejemplo |
|---|---|---|
| `--output` / `-o` | Archivo de resultado | `-o wpscan.txt` |
| `--format` | `cli` (defecto), `json`, `xml` | `--format json` |
| `--debug` | Salida de depuración | `--debug` |
| `--no-banner` / `--no-interaction` | Limpieza para pipelines | `--no-banner --no-interaction` |

```bash
wpscan --url https://target.local --enumerate u,vp,vt --api-token $WPSCAN_API_TOKEN \
  --format json --output scan.json --no-banner --no-interaction
jq -r '.data[].vulnerabilities[]? | .id' scan.json | sort -u     # IDs de vulns encontradas
```

**Interpretación JSON:** `data[]` contiene los recursos (plugins/temas/users/core); cada vulnerabilidad tiene `id` (CVE/advisory), `title`, `references` (enlaces) y a veces `poc`/`fixed_in` (versión corregida — la regla de remediación). `detection_timestamp` marca cuándo se vio.

## Workflow completo

1. **Confirmar WordPress** — `whatweb https://target.local` (ver [[WhatWeb Herramienta de Fingerprinting]]) o WPScan sin enum: detecta "WordPress" en el banner del HTML.
2. **Actualizar DB** — `wpscan --update`.
3. **Enum ligero** — `--enumerate u` (+ `--enumerate u[1-100]` si el sitio es activo).
4. **Enum profundo con token** — `-e vp,vt` (vulnerabilidades) y `-e p,t` si hace falta inventario completo.
5. **Superficie extra** — `-e a,b,d,m` (timthumbs, config backups, dumps, media).
6. **Credenciales** (si está en alcance) — spray con `--throttling` y verificación manual del par hallado en wp-login.
7. **Informe** — JSON + remediación por vulnerabilidad (`fixed_in`), usuarios hallados y riesgos de configuración.

## Interpretación de la salida (CLI)

```
[+] URL: https://target.local/
[+] WordPress 5.8.1
 [+] Interesting entry: https://target.local/readme.html (versión expuesta)
[+] Enumerating users ...
 [+] 2 users found:
 /author/admin/ (id:1) /author/jdoe/ (id:7)
[+] Enumerating vulnerable plugins ...
 [!] 3 vulnerabilities identified:
  contact-form-7 5.3.1 - CVE-2020-XXXX (cross-site request forgery)
[!] The main WAF could not be detected
```

| Marcador | Significado |
|---|---|
| `[+]` | Información/positivo de escaneo |
| `[!]` | **Hallazgo con riesgo** (vulnerabilidad, config débil) — el corazón del informe |
| `interesting entry` | Fichero expuesto detectado de pasada (readme, license, .git…) — abrirlo y confirmar |
| `no users found` | Puede ser real o que el WAF bloqueó `?author=`/REST: reintentar con `--stealthy` o `--proxy` |
| `fixed_in X.Y.Z` | Versión mínima corregida: la remediación exacta a reportar |

## Detección y contramedidas (defensor)

| Señal que ve WordPress/WAF | Origen en WPScan | Mitigación |
|---|---|---|
| `GET /?author=1..10` con redirect a `/author/usuario/` | enum `u` | Deshabilitar author archives, responder 404 a `?author=`, regla WAF |
| `GET /wp-json/wp/v2/users` | enum `u` (REST) | Restringir el endpoint a usuarios autenticados (plugin/`rest_authentication_errors`) |
| Cientos de 404 a `/wp-content/plugins/<nombre>/` | enum `p`, `vp` | Bloquear en el reverse proxy el escaneo de rutas de plugins |
| Ráfaga a `xmlrpc.php` con `system.multicall` | ataque xmlrpc-multicall | Deshabilitar `xmlrpc.php` (si no se usa) o limitar por IP |
| Intentos secuenciales en `wp-login.php` | ataque wp-login | WAF/rate limit, 2FA, limit_login_attempts, fail2ban |
| Peticiones a `wp-config.php.bak`, `.sql` | enum `b`, `d` | Bloquear backups/dumps en `.htaccess`/nginx y no dejarlos en webroot |

### Remediación (para el informe del cliente)

- Actualizar **core, plugins y temas** a `fixed_in` reportado; retirar plugins abandonados.
- Usuarios: eliminar `admin`, nombres predecibles y roles innecesarios; **2FA** y contraseñas largas.
- XML-RPC: deshabilitar si no se usa (muchos lo necesitan para apps: entonces limitar por IP).
- REST API: ocultar `/wp-json/wp/v2/users` a anónimos.
- Capa perimetral: bloquear escaneo de `/wp-content/plugins/` y copias `*.bak`, rate limit en `/wp-login.php`.
- Copias de configuración y `.sql` fuera del webroot (son hallazgos críticos si aparecen).

## Errores comunes (mensaje → arreglo)

1. **`Error: The remote site is down, requested URL does not exist`** — URL mal escrita, redirección rara o sitio caído.
   → Probar con `curl -sI https://target.local` y usar la URL exacta (sin redirigir a login).
2. **El sitio "no es WordPress" en pleno WordPress** — detecta por firmas HTML/`/wp-includes/`; themes headless o WAF que lo ocultan.
   → `--force` para continuar.
3. **`Your WPScan API Token is either invalid or has reached the daily limit`** — token mal copiado o cuota de 25/día agotada.
   → Revisar token; esperar al día siguiente o plan de pago; seguir sin él (enum y versiones funcionan).
4. **SSL: certificate verify failed** — cert autofirmado en lab.
   → `--disable-tls-checks` (solo lab/entorno controlado).
5. **Enum de usuarios vacío en un sitio vivo** — WAF bloquea `?author=` o REST cerrado.
   → `--stealthy`, `--proxy` con Burp para depurar, o enumerar por `sitemap.xml`/`/feed/?author=` alternativos.
6. **Ataque de passs que va "demasiado rápido"** — `--throttling` default 0 + WAF que corta en 30 s.
   → `--throttling 5 -t 1`; si hay WAF, cambiar de red o parar (no es evasión automática).
7. **`xmlrpc-multicall` no encuentra credenciales válidas que en wp-login sí funcionan** — multisite/redirects o plugins de login alterados.
   → Probar `--password-attack wp-login`; verificar a mano el par en el navegador.
8. **Gemas rotas tras actualizar Ruby** (`cannot load such file ...`) — dependencias del gem.
   → `gem install wpscan` de nuevo (reinstala bundler deps) o el paquete de Kali aislado.

## WPScan vs nuclei vs hermanos

| Criterio | **WPScan** | nuclei | [[Tabla de Opciones Principales de MageScan]] | [[JoomScan Herramienta de Escaneo]] |
|---|---|---|---|---|
| Alcance | WordPress | Cualquier cosa (templates) | Magento | Joomla |
| Profundidad CMS | **Máxima** (enum, versiones, vulns, creds) | Media (template por template) | Alta (versión/parches) | Alta |
| Base de vulns | Propia + token | **Plantillas de la comunidad** (miles) | SUPEE embebidos | Base propia |
| Ataques de pass | **Sí** (wp-login/xmlrpc) | No (solo detección) | No | Básico |
| Falso positivo | Bajo (hash de contenido) | Variable según template | Medio | Medio |
| Mantenimiento | Activo | Muy activo | Bajo | Medio |

Uso conjunto razonable: WPScan para el pipeline WordPress completo; nuclei para añadir plantillas de vulns modernas sobre lo descubierto (verificado una a una, nunca en automático contra producción).

> [!warning] Legal y ética (obligatorio leer)
> Enumerar usuarios y plugins es intrusivo (cientos de peticiones) y el ataque de contraseñas **bloquea cuentas reales** y puede activar alarmas. Todo esto se ejecuta solo con **autorización escrita** que lo mencione, en horario acordado, con throttling razonable y dejando constancia en el informe. La información de este note es material de estudio/CTF y de auditoría autorizada.

## Cheatsheet

```bash
wpscan --update
wpscan --url https://target.local
wpscan --url https://target.local --enumerate u
wpscan --url https://target.local --enumerate u[1-100]
wpscan --url https://target.local --enumerate vp,vt --api-token $WPSCAN_API_TOKEN
wpscan --url https://target.local -e a,b,d,m
wpscan --url https://target.local -U users.txt -P pass.txt --password-attack wp-login --throttling 5 -t 1
wpscan --url https://target.local --proxy http://127.0.0.1:8080 --stealthy
wpscan --url https://target.local --format json --output scan.json --no-banner --no-interaction
wpscan --url https://target.local --disable-tls-checks   # solo lab
wpscan --version
```

| Tarea | Comando |
|---|---|
| Antes de todo | `--update` |
| Enum users | `--enumerate u[1-100]` |
| Vulns | `-e vp,vt --api-token …` |
| Superficie extra | `-e a,b,d,m` |
| Spray pass (con freno) | `-U/-P --password-attack wp-login --throttling 5 -t 1` |
| Informe | `--format json --output` |
| Detrás de Burp | `--proxy http://127.0.0.1:8080` |
| Lab con cert roto | `--disable-tls-checks` |

## Referencias

- [Repositorio oficial (wpscanteam/wpscan)](https://github.com/wpscanteam/wpscan) — README, changelog y guía de opciones
- [WPScan Vulnerability Database](https://wpscan.com/) — token gratuito y advisories
- [Hardening WordPress (Códex oficial)](https://wordpress.org/support/article/hardening-wordpress/) — remediación oficial

## Relacionadas

- [[WhatWeb Herramienta de Fingerprinting]] — confirmación inicial de "es WordPress"
- [[Enumeración de Servicios y CMS]] — metodología de enumeración por CMS
- [[Tabla de Opciones Principales de MageScan]] / [[JoomScan Herramienta de Escaneo]] / [[Droopscan]] — hermanos por CMS
- [[WFuzz]] — param fuzzing sobre el sitio ya identificado
- [[THC Hydra]] — brute force de servicios de red (si el login WordPress no es el objetivo)
- [[Gobuster]] — rutas/archivos ocultos del webroot
- [[Reconocimiento y Escaneo de Redes y Web]] — el flujo completo en el que encaja
