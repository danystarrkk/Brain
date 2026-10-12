---
aliases: ["Droopscan", "droopescan", "Drupscan", "Escáner Drupal"]
tags: [tool/droopescan]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Enumeración de Servicios y CMS]]"
  - "[[WPScan]]"
  - "[[JoomScan Herramienta de Escaneo]]"
  - "[[Tabla de Opciones Principales de MageScan]]"
  - "[[WhatWeb Herramienta de Fingerprinting]]"
  - "[[Gobuster]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
fuente: "https://github.com/droope/droopescan"
---

# Droopscan (droopescan): Escáner de Seguridad para Drupal y CMS

**Droopscan** —cuyo nombre real del binario es **`droopescan`**— es un escáner de vulnerabilidades y enumeración de componentes para CMS, con foco inicial en **Drupal** y soporte añadido de Joomla, WordPress, Magento, SilverStripe y Piano. Detecta versión core, módulos/temas instalados (sin autenticación) y los CVEs asociados, a partir de firmas de ficheros y metadatos públicos.

> [!info] Qué hace exactamente (y qué no)
> droopescan **no fuerza credenciales ni ejecuta exploits**: manda peticiones HTTP a rutas conocidas (ficheros de versión, `.info`, `CHANGELOG.txt`, metadatos `X-Generator`/`generator`) y cruza las respuestas con su base de firmas. La salida te dice *"esta instalación tiene Drupal 7.34 y el módulo ctools X"*, y de ahí el salto a CVE es manual. Eso lo diferencia de WPScan/JoomScan, que sí llevan base de exploits y brute force.

## Instalación

```bash
# Kali Linux
sudo apt update && sudo apt install droopescan

# Alternativa pip (cualquier distro, en venv)
python3 -m pip install droopescan

# Verificar
droopescan --version
droopescan --help
```

> [!tip] Actualizar las firmas ANTES de escanear
> Las detecciones dependen de ficheros de firmas descargados del servidor de Droope. Actualízalas siempre:
> ```bash
> droopescan update
> ```
> (En versiones con comprobación automática, `--disable-update-check` permite saltársela cuando trabajas sin red/blanco aislado.)

## Sintaxis: subcomandos, no flags sueltos

```text
droopescan <subcomando> ...
droopescan scan <drupal|joomla|wordpress|magento|silverstripe|piano> -u <URL>
droopescan update          # refresca firmas de plugins/temas/versiones
droopescan inspect <plugin>  # versiones conocidas de un plugin concreto
droopescan plugins <cms>     # lista los plugins que droopescan sabe detectar
```

| Subcomando | Función |
|---|---|
| `scan <cms>` | Escaneo principal — **el CMS va primero**, no después del `-u` |
| `update` | Descarga actualizaciones de la base de firmas |
| `inspect` | Muestra qué versiones de un plugin/módulo conoce la base |
| `plugins` | Enumera los plugins conocidos (para diseñar tus propias comprobaciones) |

## Tabla de opciones del escaneo (`scan`)

| Opción | Significado | Ejemplo |
|---|---|---|
| `-u`, `--url` | URL objetivo (con esquema) | `-u https://target.com` |
| `-e`, `--enumerate` | Qué enumerar: `all` (defecto), `plugins`, `themes`, `files`, `users` según CMS | `-e plugins` |
| `--timeout` | Timeout HTTP por petición (segundos) | `--timeout 30` |
| `--cookie` | Cookie de sesión (para escanear atrás de login) | `--cookie "SESS=abc123"` |
| `--user-agent` | User-Agent personalizado | `--user-agent "Mozilla/5.0"` |
| `--disable-update-check` | No comprueba actualizaciones al arrancar | — |
| `--version` (global) | Versión de droopescan | `droopescan --version` |
| `-h`, `--help` | Ayuda (incluye los valores reales de `-e` **para cada CMS**) | `droopescan scan drupal -h` |

> [!warning] Corrección respecto a notas previas
> - **`droopescan` no tiene `--brute` ni `--modules`/`--themes` como flags de scan**: la enumeración es `-e <tipo>` y la actualización es el subcomando `droopescan update`, no `--update`.
> - **No genera `--json`/`--xml`/`--csv` nativos**: la salida es texto en stdout → redirige con `> informe.txt` o `tee`.
> - Para proxy, usa el binario bajo proxychains (`proxychains droopescan scan ...`) en vez de confiar en un flag `-p` que puede no existir en tu versión.
> - El listado completo de flags válidos siempre sale de `droopescan scan <cms> -h`.

## Workflow: de la URL al CVE

### Paso 1 — Confirmar que es Drupal

```bash
# Cabeceras y fingerprints rápidos
curl -sI https://target.com | grep -iE "x-generator|x-drupal|x-powered-by"
whatweb https://target.com -a 3
```
Señales: `X-Generator: Drupal 7 (https://www.drupal.org)`, `X-Drupal-Cache`, rutas `?q=`, `/sites/default/files`, `/misc/drupal.js`.

### Paso 2 — Escanear

```bash
droopescan scan drupal -u https://target.com
# Enumeración acotada (menos peticiones, más rápido)
droopescan scan drupal -u https://target.com -e plugins --timeout 20
```

### Paso 3 — Leer la salida

```text
[+] Version found:
    7.34           https://target.com/CHANGELOG.txt
[+] Plugins found:
    ctools         https://target.com/sites/all/modules/ctools/ctools.info
    views          https://target.com/sites/all/modules/views/views.info
[+] Themes found:
    bartik         ...
[+] Files found:
    /sites/default/files/...
```

| Sección | Qué significa | Qué hacer |
|---|---|---|
| `Version found` | Versión del core detectada por fichero/meta | Cruzar con CVEs de Drupal core |
| `Plugins found` (módulos) | Módulos contrib instalados **y su ruta accesible** | Cada módulo = superficie extra; versionar por `.info` |
| `Themes found` | Temas instalados | Menor impacto, sirve para fingerprint fino |
| `Files found` | Ficheros sensibles/expuestos | Revisar a mano: backups, `.git`, SQL volcados |
| `Users found` | Enumeración de usuarios (si el CMS la permite) | Alimenta ataques de login (ver [[THC Hydra]]) |
| Sin resultados | Redirección/403 o firmas desactualizadas | Actualizar (`droopescan update`) y comprobar con `-v`/curl manual |

> [!warning] Falsos positivos de versión
> Un `CHANGELOG.txt` accesible lista **todas** las versiones: droopescan puede reportar una versión que no es la real. **Verifica siempre** con una segunda evidencia (meta `generator`, fichero que solo existe en esa versión, o `inspect`). Un CVE erróneo en un informe es un hallazgo inválido.

### Paso 4 — Del CVE a la verificación manual

| Versión Drupal | Hallazgos clásicos | Referencia |
|---|---|---|
| < 7.58 / < 8.5.1 | **Drupalgeddon2** (SA-CORE-2018-002, CVE-2018-7600) | Exploit público, RCE sin auth |
| 7.x/8.x sin parche | Drupalgeddon3 (CVE-2018-7601), form API bypass | — |
| < 8.5.10 | SA-CORE-2018-004 (CVE-2018-14773, filtrado de rutas) | — |
| < 8.6.10 / 8.5.12 | SA-CORE-2019-003 (CVE-2019-6341, archivo subido / render array) | — |
| Módulo desactualizado | CVE del módulo concreto (search por nombre + "drupal vulnerability") | `droopescan inspect <modulo>` para versión |

Flujo: `droopescan` → versión + módulos → **verificación manual** (abrir el fichero, mirar meta) → exploit en laboratorio controlado → remediación recomendada (actualizar/core parche).

## Recetas: los 5 usos reales

```bash
# 1. Escaneo completo estándar (lo que se corre siempre)
droopescan update && droopescan scan drupal -u https://target.com

# 2. Solo módulos, rápido (para informes de superficie)
droopescan scan drupal -u https://target.com -e plugins --timeout 15

# 3. Detrás de login (zona de administración)
droopescan scan drupal -u https://target.com --cookie "SESS1=abc123def456"

# 4. Guardar para informe
droopescan scan drupal -u https://target.com | tee informe_drupal.txt

# 5. A través de proxy de auditoría (Burp/ZAP)
proxychains -q droopescan scan drupal -u http://target.local
```

**Explicación:** `update` sincroniza firmas (sin esto, módulos nuevos no se detectan); `-e plugins` recorta el escaneo a la enumeración que más aporta; `--cookie` salta el login; `tee` conserva color y guarda copia; proxychains fuerza el tráfico por 127.0.0.1:8080 sin depender de flags.

## Detección y contramedidas (defensor)

| Señal | Dónde se ve | Mitigación |
|---|---|---|
| Ráfaga de GET a rutas fijas (`CHANGELOG.txt`, `.info`, `misc/drupal.js`) | WAF/HTTP logs con patrón de escáner, UA de droopescan | Bloquear `CHANGELOG.txt` y `.info` externamente; reglas WAF por path |
| `droopescan`/UA desconocido | Access logs, ModSecurity | Bloquear UA y rate-limit por IP |
| Enumeración de módulos sin auth | Alto ratio 404/200 en `/sites/all/modules/*` | 404 personalizado estable + limitación de tasa |
| Exposición de `/sites/default/files` | Listado de directorios | `Options -Indexes` en Apache |
| Base desactualizada (escáner no detecta) | Falsos negativos en auditorías defensivas | Mantener firmas actualizadas y contrastar con WPScan/WhatWeb |

## Checklist manual post-escaneo (Drupal)

Un escáner termina la *enumeración*; la validación es manual. Antes de reportar o pasar a explotación:

| # | Comprobación | Cómo | Si sale mal/está expuesto |
|---|---|---|---|
| 1 | Versión real del core | `curl -s <URL>/core/CHANGELOG.txt \| head` (D8+) o meta generator | Confirmar CVE contra advisories |
| 2 | `settings.php` accesible | `curl -sI <URL>/sites/default/settings.php` | 200 = **crítico** (DB creds) |
| 3 | Backup de configuración | probar `settings.php.bak`, `.save`, `.old`, `settings.php~` | Extraer credenciales |
| 4 | Archivos de actualización | `/update.php`, `/install.php`, `/authorize.php` | Deben estar bloqueados |
| 5 | Listado de directorios | `/sites/default/files/` con `Options -Indexes` off | Fuga de subidas |
| 6 | Archivos de desarrollo | `/.git/config`, `/.env`, `web.config` | Información sensible |
| 7 | Enumeración de usuarios | `/user/1`, patrones de error en `/user/login` | Alimenta [[THC Hydra]] (si aplica en alcance) |
| 8 | Panel admin | `/user/login`, `/admin` renombrado o no | Superficie de fuerza bruta |
| 9 | Módulos sin actualizar | Cruce `droopescan -e plugins` con el changelog de cada módulo | CVE por módulo |
| 10 | Firma de error / stack trace | Forzar error (param inválido) | Versiones de PHP/Drupal en trazas |

> [!tip] Criterio de informe
> Cada hallazgo lleva: evidencia HTTP exacta (request/response), versión confirmada con **dos** fuentes, CVE o configuración afectada, impacto y remediación (parche a la versión X). Sin versión confirmada, no hay CVE en el informe.

## Errores comunes (mensaje → arreglo)

1. **`droopescan: command not found`** — no instalado o no en PATH.
   → `sudo apt install droopescan` o `python3 -m pip install droopescan`.
2. **`ConnectionError` / timeouts constantes** — URL sin esquema o con WAF que lame.
   → Incluir `https://`, subir `--timeout`, o pasar por proxychains/Burp para ver qué devuelve.
3. **`No versions found` con un sitio que sí es Drupal** — firmas viejas o redirección a home.
   → `droopescan update`; probar con la URL final (sin redirecciones) y con `-e all`.
4. **Escaneo muy lento / cortado por WAF** — demasiadas peticiones seguidas.
   → Usar `-e plugins` (menos peticiones), subir `--timeout`, y/o encajar bajo proxy con pausa manual.
5. **Versión "encontrada" que no cuadra con el sitio** — CHANGELOG.txt o meta falsos positivos.
   → Verificación manual con curl de un fichero de versión único (ver Paso 4).
6. **Módulo conocido no reportado** — no está en la base o su ruta es no estándar.
   → `droopescan plugins drupal` para ver qué cubre; si no aparece, enumerar a mano con `Gobuster`/`gobuster dir -x info,inc`.

## Cheatsheet — los 10 comandos del día a día

```bash
droopescan --version
droopescan --help
droopescan update
droopescan scan drupal -u https://target.com
droopescan scan drupal -u https://target.com -e plugins
droopescan scan joomla -u https://target.com
droopescan scan wordpress -u https://target.com
droopescan inspect ctools
droopescan scan drupal -u https://target.com --cookie "SESS1=abc"
droopescan scan drupal -u https://target.com | tee informe.txt
```

| Tarea | Comando |
|---|---|
| Actualizar firmas | `droopescan update` |
| Escaneo completo Drupal | `droopescan scan drupal -u <URL>` |
| Solo módulos | `... -e plugins` |
| Versiones de un plugin | `droopescan inspect <plugin>` |
| Ver qué sabe detectar | `droopescan plugins drupal` |
| Tras login | `... --cookie "SESS1=..."` |
| Guardar salida | `... \| tee informe.txt` |
| Por proxy | `proxychains droopescan scan drupal -u <URL>` |
| Ayuda real | `droopescan scan drupal -h` |
| Cross-check fingerprint | `whatweb <URL> -a 3` |

## Alternativas

| Herramienta | CMS | Fuerza bruta | Base CVEs | Salida estructurada | Nota |
|---|---|---|---|---|---|
| **droopescan** | Drupal, WP, Joomla, Magento… | No | Firmas de versión | Texto | Rápido, sin auth, ideal primera pasada |
| [[WPScan]] | WordPress | **Sí** (users/plugins) | Sí (API token) | JSON | El equivalente serio para WP |
| [[JoomScan Herramienta de Escaneo]] | Joomla | Sí (admin) | Sí | JSON/XML | Foco Joomla |
| [[Tabla de Opciones Principales de MageScan]] | Magento | No | Parcial | — | Magento |
| [[WhatWeb Herramienta de Fingerprinting]] | Cualquiera | No | No | JSON/CSV | Solo identifica, no versiona módulos |
| Manual + [[Gobuster]] | Cualquiera | — | — | — | Para rutas no estándar y confirmación |

## Errores de concepto a no cometer

1. **Confundir el binario con la nota**: se ejecuta `droopescan`, no "droopscan".
2. **Esperar exploits**: droopescan reporta *pistas*, no explota. La verificación es manual.
3. **Sin `update`**, la detección de módulos recientes es ciega.
4. **Escaneo de un cliente real sin autorización**: las peticiones a `CHANGELOG.txt`/`.info` son ruidosas y registrables.

## Referencias

- [Repositorio oficial (droope/droopescan)](https://github.com/droope/droopescan) — README, FAQs y contributing
- [Drupal Security Advisories](https://www.drupal.org/security) — SA-CORE-* (Drupalgeddon, etc.)
- [Exploit-DB](https://www.exploit-db.com/) — búsqueda `drupal` para validar exposición
- HackTricks — *CMS enumeration* para encadenar con [[Enumeración de Servicios y CMS]]

## Relacionadas

- [[Enumeración de Servicios y CMS]] — metodología general de CMS
- [[WPScan]] / [[JoomScan Herramienta de Escaneo]] / [[Tabla de Opciones Principales de MageScan]] — hermanos por CMS
- [[WhatWeb Herramienta de Fingerprinting]] — paso previo de identificación
- [[Gobuster]] — enumeración de rutas cuando el CMS no responde a firmas
- [[Reconocimiento y Escaneo de Redes y Web]] — dónde encaja en la metodología
