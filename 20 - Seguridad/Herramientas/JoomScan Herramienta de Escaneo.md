---
tags:
  - ciberseguridad
  - herramientas
  - joomscan-escaneo
tipo: herramienta
estado: terminado
---

# JoomScan: Escáner de Vulnerabilidades para Joomla (OWASP)

**JoomScan** (*OWASP Joomla Vulnerability Scanner*) es el escáner especializado en **Joomla!**: detecta la versión del core, enumera componentes/módulos/plantillas instalados y cruza esos hallazgos con una base de vulnerabilidades conocidas. Es la pieza de reconocimiento CMS equivalente a WPScan para WordPress y droopescan para Drupal (ver [[Enumeración de Servicios y CMS]]).

> [!info] Por qué un escáner específico de Joomla
> Joomla concentra su superficie en **componentes** (`/components/com_*/`), plantillas y extensiones de terceros: la mayoría de brechas graves no están en el core sino en una extensión concreta con una versión vieja. Un fingerprinting genérico (ver [[WhatWeb Herramienta de Fingerprinting]]) ve "Joomla"; JoomScan ve **qué componentes hay y qué versión exacta corre cada uno**, que es lo que se traduce en un CVE accionable.

## Instalación

```bash
# Kali Linux
sudo apt update && sudo apt install joomscan

# Verificar
joomscan --version
joomscan --help

# Actualizar la base de vulnerabilidades SIEMPRE antes del primer escaneo
joomscan --update
```

> [!warning] La base desactualizada es el fallo nº1
> Sin `joomscan --update` la herramienta trabaja con firmas antiguas: te pierdes CVEs recientes y reportas versiones "sin vulnerabilidad" que sí la tienen. Ejecútalo al inicio de cada sesión de trabajo y tras periodos de inactividad del laboratorio.

## Qué mira Joomla (rutas que JoomScan explota)

| Ruta / evidencia | Qué revela |
|---|---|
| `<meta name="generator" content="Joomla! ...">` | Versión candidata del core (en versiones recientes a veces sin número exacto) |
| `/manifests/files/joomla.xml` | **Versión exacta del core** (etiqueta `<version>`) si es accesible |
| `/administrator/` | Panel de administración (fuerza bruta de login — ver advertencia) |
| `/components/com_*/` | Lista de componentes instalados → superficie de extensiones |
| `/modules/`, `/templates/<nombre>/templateDetails.xml` | Módulos y plantillas con versiones |
| `/configuration.php` | Configuración (debe estar bloqueada; si es legible = hallazgo crítico) |
| `/language/*/*.xml`, `/media/jui/` | Fingerprint de versión/archivos típicos |
| Cabeceras `X-Generator`, `Set-Cookie` con `joomla_session` | Confirmación y sesiones |

## Tabla de opciones

### Núcleo (siempre disponibles)

| Opción | Significado | Ejemplo |
|---|---|---|
| `-u`, `--url` | URL objetivo (con esquema) | `joomscan -u https://target.com` |
| `--update` | Actualiza la base de vulnerabilidades | `joomscan --update` |
| `--version` | Versión del escáner | `joomscan --version` |
| `-h`, `--help` | Ayuda completa (la fuente real de flags en tu versión) | `joomscan --help` |
| `-v`, `--verbose` | Salida detallada (qué se comprueba) | `joomscan -u ... -v` |
| `-t`, `--timeout` | Timeout HTTP (segundos) | `-t 30` |
| `-p`, `--proxy` | Proxy HTTP/S (Burp, ZAP, tor) | `-p http://127.0.0.1:8080` |
| `-a`, `--user-agent` | User-Agent personalizado | `-a "Mozilla/5.0"` |
| `-o`, `--output` | Fichero de salida del informe | `-o informe.txt` |

### Enumeración y agresividad (confirmar con `joomscan --help` en tu versión)

| Opción documentada | Propósito |
|---|---|
| `-ec` | Enumerar **componentes** instalados |
| `-ep` | Enumerar **plugins** |
| `-em` | Enumerar **módulos** |
| `-et` | Enumerar **plantillas** (templates) |
| `-e` / `--enumerate` | Enumeración general (todas las categorías) |
| `-c` / `--cookie` | Cookie de sesión para zonas autenticadas |
| `--random-agent` / rotación de UA | Evitar bloqueos por User-Agent fijo |
| `-f` | Fuerza bruta de credenciales de administrador |
| `-r` | Rango de IPs / objetivo alternativo |
| `--json` / `--xml` | Salida estructurada (automatización de informes) |

> [!warning] Correcciones y advertencias sobre estas opciones
> - **`-f` (fuerza bruta de admin) es la opción más peligrosa operativamente**: genera tráfico de login masivo, dispara lockouts y alertas IDS. Úsala solo si el alcance firmado lo incluye explícitamente; para fuerza bruta de credenciales la herramienta estándar es [[THC Hydra]].
> - Los flags de enumeración/salida **cambian entre versiones de JoomScan**: verifica con `joomscan --help` antes de escribirlos en un script o informe.

## Workflows

### Flujo estándar: de la URL al CVE

```bash
# 1. Confirmar que el objetivo es Joomla (herramienta genérica)
whatweb https://target.com -a 3

# 2. Actualizar firmas
joomscan --update

# 3. Escaneo completo con verbosidad
joomscan -u https://target.com -v

# 4. Enumeración enfocada en extensiones
joomscan -u https://target.com -ec -em -et

# 5. Capturar tráfico por Burp para inspección manual
joomscan -u https://target.com -p http://127.0.0.1:8080 -v
```

### Interpretación de la salida

| Sección de salida | Qué significa | Siguiente paso |
|---|---|---|
| Versión del core detectada | Candidata a verificar | Confirmar con `/manifests/files/joomla.xml` o fichero de versión |
| Componentes listados (`com_*`) | Superficie real de ataque | Buscar cada `com_X version` en la base de datos/CVE |
| Plantillas detectadas | Posible vector (RCE en templates viejos) | Cruzar versión con advisories |
| `Vulnerability: ...` reportada | Coincidencia en la base de JoomScan | **Verificar manualmente** antes de reportar/explotar |
| Headers interesantes | Info disclosure (versiones en cabeceras) | Anotar para el informe |
| Sin resultados | Base vieja, WAF o falso negativo | `--update`, probar con proxy/UA, verificación manual |

> [!warning] Verificar antes de creer
> Los escáneres CMS mienten por completo en dos direcciones: **falsos positivos** (un fichero compartido dispara el CVE) y **falsos negativos** (versión mal detectada). Regla: toda vulnerabilidad reportada se confirma con una comprobación manual (abrir el fichero de versión, reproducir el patrón con curl) antes de tocar nada.

### Verificación manual de versión (complemento al escáner)

```bash
curl -s https://target.com/manifests/files/joomla.xml | grep -i "<version>"
curl -sI https://target.com/ | grep -i generator
curl -s https://target.com/templates/system/js/core.js | head -5
```

## Recetas: los 5 usos reales

```bash
# 1. Escaneo estándar para informe
joomscan -u https://target.com 2>&1 | tee joomscan_informe.txt

# 2. Solo componentes (rápido, para la matriz de superficie)
joomscan -u https://target.com -ec

# 3. Detrás de Burp para revisar peticiones
joomscan -u https://target.com -p http://127.0.0.1:8080 -v

# 4. Con sesión (zona autenticada / front-end con login)
joomscan -u https://target.com/administrator --cookie "joomla_user_state=logged_in; xyz=abc"

# 5. Tras actualizar la base + UA de navegador
joomscan --update && joomscan -u https://target.com -a "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
```

**Por qué el orden importa:** `--update` primero (sin firmas nuevas el escaneo miente), luego el escaneo, y el proxy cuando el objetivo tenga WAF o quieras auditar el propio tráfico del escáner.

## Detección y contramedidas (defensor)

| Señal del escáner | Dónde se ve | Mitigación |
|---|---|---|
| GET a `/manifests/files/joomla.xml`, `configuration.php`, rutas `com_*` | Access logs / WAF con patrón de escáner | Bloquear rutas de diagnóstico, WAF rules |
| User-Agent de JoomScan o UA inexistente | Logs | Bloquear UA, exigir UA coherente |
| Ráfaga de 404/200 sobre `/components/*` | Umbral de IDS web | Rate limit, fail2ban |
| Enumeración de `/administrator/` | Intentos de acceso al panel | Renombrar `/administrator`, 2FA, limitar por IP |
| Fuerza bruta (`-f`) | Cientos de 4625/4624 o intentos de login | Lockout policy, CAPTCHA, honeypot de login |
| Versión visible en `generator` | Fingerprint público | Quitar la versión del meta generator |

## Checklist manual post-escaneo (Joomla)

Tras `joomscan`, antes de reportar o explotar:

| # | Comprobación | Cómo | Qué aporta |
|---|---|---|---|
| 1 | Versión exacta del core | `curl -s <URL>/manifests/files/joomla.xml \| grep -i version` | Confirma/refuta lo detectado |
| 2 | Meta generator | `curl -sI <URL> \| grep -i generator` | Fingerprint rápido |
| 3 | Panel y ruta de admin | `/administrator/` (¿renombrado?) | Superficie de fuerza bruta |
| 4 | `configuration.php` | `curl -sI <URL>/configuration.php` | Legible = crítico |
| 5 | Backups de configuración | `configuration.php~`, `.bak`, `.txt`, `config.bak.php` | Credenciales |
| 6 | Instaladores/updates | `/administrator/index.php?option=com_joomlaupdate`, `/installation/` | Ruta de actualización expuesta |
| 7 | Componentes por versión | Cruce de `-ec` con el changelog de cada componente | CVEs reales |
| 8 | Ficheros de plantilla | `/templates/<t>/templateDetails.xml` | Versión de plantillas |
| 9 | Archivos de debug | `/api/index.php/v1/...` errors, `/tmp/`, `logs/` | Info disclosure |
| 10 | Fuerza bruta admin | Solo con `-f`/[[THC Hydra]] **y alcance firmado** | Credenciales por aging |

> [!tip] Matriz de severidad para el informe
> **Crítico**: `configuration.php` legible, RCE confirmada en componente, core con CVE RCE sin parchear. **Alto**: panel admin expuesto + componente con CVE de privilege escalation. **Medio**: componentes desactualizados sin vector demostrado, versiones visibles en headers. **Bajo**: fingerprint CMS visible. Siempre con evidencia HTTP y remediación concreta (actualizar a X).

## Errores comunes (mensaje → arreglo)

1. **`joomscan: command not found`** — no instalado.
   → `sudo apt install joomscan` (Kali/Debian) o instalar desde el [repo OWASP](https://github.com/OWASP/joomscan) con sus dependencias Perl.
2. **`Can't locate ... .pm` / errores de módulos Perl** — dependencias CPAN ausentes (instalación manual incompleta).
   → Usar el paquete de la distro o instalar los módulos que indica el error (`cpanm <Modulo>`).
3. **Escaneo vacío / "no version found"** — WAF, redirección o base vieja.
   → `joomscan --update`, probar con `-v` y proxy, y confirmar el fingerprint con curl.
4. **Timeouts (`timeout`/`connection`)** — servidor lento o el escáner va demasiado rápido.
   → Subir `-t 45`, bajar agresividad, ir por proxy con rate limit.
5. **Proxy no usado** — esquema incorrecto.
   → `http://127.0.0.1:8080` (no `https://`) y verificar que Burp/ZAP escucha en ese puerto.
6. **Versión detectada que no coincide con el sitio** — falso positivo por ficheros compartidos.
   → Verificar con `/manifests/files/joomla.xml` (ver receta de verificación manual).
7. **Bloqueo 403/429 en mitad del escaneo** — WAF.
   → Rotar UA, pausar, segmentar el escaneo por categorías (`-ec`, luego `-em`).

## Cheatsheet — los 10 comandos del día a día

```bash
joomscan --help
joomscan --version
joomscan --update
joomscan -u https://target.com
joomscan -u https://target.com -v
joomscan -u https://target.com -ec
joomscan -u https://target.com -ec -em -et
joomscan -u https://target.com -p http://127.0.0.1:8080
joomscan -u https://target.com -o informe.txt
curl -s https://target.com/manifests/files/joomla.xml | grep -i version
```

| Tarea | Comando |
|---|---|
| Actualizar base | `joomscan --update` |
| Escaneo completo | `joomscan -u <URL>` |
| Enumerar componentes | `joomscan -u <URL> -ec` |
| Verboso | `-v` |
| Por Burp | `-p http://127.0.0.1:8080` |
| Timeout | `-t 45` |
| Cookie de sesión | `--cookie "k=v"` |
| Informe | `-o informe.txt` |
| Confirmar versión a mano | `curl -s <URL>/manifests/files/joomla.xml` |
| Ayuda real | `joomscan --help` |

## Alternativas y comparativa

| Herramienta | CMS | Base CVEs | Fuerza bruta | Enumeración | Salida |
|---|---|---|---|---|---|
| **JoomScan** | Joomla | Sí (DB propia) | Sí (`-f`, con cautela) | Componentes/módulos/plantillas | texto/JSON/XML |
| [[WPScan]] | WordPress | Sí (API token) | Sí (users, passwords) | Plugins/temas/core | JSON |
| [[Droopscan]] | Drupal (+WP/Joomla) | Firmas de versión | No | Plugins/temas | texto |
| [[Tabla de Opciones Principales de MageScan]] | Magento | Parcial | No | Rutas/modulos | texto |
| [[WhatWeb Herramienta de Fingerprinting]] | Cualquiera | No | No | Solo fingerprint | JSON/CSV |
| [[Gobuster]] | — | — | — | Rutas genéricas (complemento) | texto |

**Flujo combinado recomendado:** `WhatWeb` (¿es Joomla?) → `joomscan --update && joomscan -u` (versión + componentes) → `Gobuster -x xml,php` para rutas no estándar → verificación manual del CVE → explotación solo en laboratorio autorizado.

## Referencias

- [Repositorio OWASP (OWASP/joomscan)](https://github.com/OWASP/joomscan) — código, issues y actualizaciones
- [Joomla Security Centre](https://developer.joomla.org/security-centre.html) — advisories oficiales (SA-*)
- [Exploit-DB](https://www.exploit-db.com/) — búsqueda `joomla`
- OWASP WSTG — secciones de fingerprinting y content discovery que enmarcan esta herramienta
- HackTricks — *CMS scanning* para encadenar con [[Enumeración de Servicios y CMS]]

## Relacionadas

- [[Enumeración de Servicios y CMS]] — metodología general de CMS
- [[WPScan]] / [[Droopscan]] / [[Tabla de Opciones Principales de MageScan]] — equivalentes por CMS
- [[WhatWeb Herramienta de Fingerprinting]] — paso previo de identificación
- [[Gobuster]] — enumeración complementaria de rutas
- [[THC Hydra]] — fuerza bruta de credenciales controlada (si el alcance lo permite)
- [[Reconocimiento y Escaneo de Redes y Web]] — contexto metodológico
