---
aliases:
  - JoomScan
  - OWASP Joomla Vulnerability Scanner
tags:
  - hacking/herramientas
  - hacking/web
estado: 🟢 Terminado
---
**JoomScan** (OWASP Joomla Vulnerability Scanner) es una herramienta de escaneo de seguridad diseñada específicamente para detectar vulnerabilidades y problemas de configuración en sitios Joomla!.

> [!info] JoomScan es una herramienta crucial en el arsenal de un pentester o auditor de seguridad web, ya que Joomla! es uno de los Sistemas de Gestión de Contenidos (CMS) más populares. Su enfoque específico permite identificar debilidades comunes como versiones desactualizadas, configuraciones inseguras, y vulnerabilidades en extensiones (componentes, módulos, plugins y plantillas).

## Tabla de Opciones Principales de JoomScan

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-u` | URL objetivo | `joomscan -u https://example.com` |
| `--url` | URL objetivo | `joomscan --url https://example.com` |
| `-h` | Muestra ayuda | `joomscan -h` |
| `--help` | Ayuda completa | `joomscan --help` |
| `-v` | Modo verboso | `joomscan -v` |
| `--version` | Muestra versión | `joomscan --version` |
| `--update` | Actualizar base de datos de vulnerabilidades | `joomscan --update` |

> [!tip] Mantener la base de datos de JoomScan actualizada (`--update`) es fundamental para asegurar que la herramienta pueda detectar las últimas vulnerabilidades conocidas y exploits. Las bases de datos de vulnerabilidades se actualizan constantemente con nuevos CVEs y fallos de seguridad.

### Opciones de Escaneo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-ec` | Enumerar componentes | `joomscan -ec` |
| `-ep` | Enumerar plugins | `joomscan -ep` |
| `-em` | Enumerar módulos | `joomscan -em` |
| `-et` | Enumerar plantillas | `joomscan -et` |
| `-f` | Fuerza bruta de credenciales de administrador | `joomscan -f` |
| `-a` | Agente de usuario personalizado | `joomscan -a "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"` |
| `-r` | Rango de IPs para escanear | `joomscan -r 192.168.1.0/24` |

> [!warning] El uso de la opción `-f` para fuerza bruta de credenciales de administrador debe realizarse con extrema precaución y **únicamente en entornos autorizados y controlados**. Esta acción puede generar una gran cantidad de tráfico, bloquear cuentas de usuario y ser detectada por sistemas de detección de intrusiones (IDS/IPS), además de tener implicaciones legales si se realiza sin consentimiento explícito.

### Opciones de Salida

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-o` | Archivo de salida | `joomscan -o resultados.txt` |
| `--log` | Archivo de registro | `joomscan --log scan.log` |
| `--json` | Salida JSON | `joomscan --json output.json` |
| `--xml` | Salida XML | `joomscan --xml output.xml` |

> [!info] Las opciones de salida estructurada como `--json` y `--xml` son invaluable para la automatización y la integración con otras herramientas. Permiten parsear los resultados de JoomScan de manera programática, facilitando el análisis posterior, la generación de informes personalizados o la alimentación de dashboards de seguridad.

### Opciones de Proxy y Conexión

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-p` | Usar proxy HTTP/S | `joomscan -p http://proxy:8080` |
| `--proxy` | Proxy específico (HTTP/S, SOCKS4, SOCKS5) | `joomscan --proxy socks5://127.0.0.1:9050` |
| `-t` | Tiempo de espera de conexión (segundos) | `joomscan -t 30` |
| `--timeout` | Tiempo de espera específico (segundos) | `joomscan --timeout 45` |
| `--cookie` | Cookie personalizada para la sesión | `joomscan --cookie "session=value; PHPSESSID=abc123def456"` |

> [!tip] El uso de proxies (`-p` o `--proxy`) es esencial para mantener el anonimato durante las pruebas de penetración, evadir bloqueos de IP, o para enrutar el tráfico a través de herramientas como Burp Suite o OWASP ZAP para una inspección más profunda del tráfico HTTP/S. Esto permite analizar las solicitudes y respuestas en tiempo real y modificar parámetros.