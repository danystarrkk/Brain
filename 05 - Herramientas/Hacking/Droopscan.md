---
aliases: Droopscan
tags:
  - hacking/web
  - hacking/herramientas
estado: 🟢 Terminado
---
# Droopscan: Escáner de Seguridad para Drupal

**Droopscan** es un escáner de seguridad especializado para Drupal que permite identificar vulnerabilidades, módulos desactualizados, configuraciones inseguras y otros problemas de seguridad en sitios Drupal.

> [!info] ¿Qué es Drupal?
> Drupal es un sistema de gestión de contenido (CMS) de código abierto y gratuito escrito en PHP. Es ampliamente utilizado para construir sitios web y aplicaciones robustas, ofreciendo una gran flexibilidad y una arquitectura modular. Su popularidad lo convierte en un objetivo frecuente para atacantes, de ahí la necesidad de herramientas como Droopscan para asegurar sus implementaciones.

## Tabla de Opciones Principales de Droopscan

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-u` | URL objetivo | `droopscan -u https://example.com` |
| `--url` | URL objetivo | `droopscan --url https://example.com` |
| `-h` | Muestra ayuda | `droopscan -h` |
| `--help` | Ayuda completa | `droopscan --help` |
| `-v` | Modo verbose | `droopscan -v` |
| `--version` | Muestra versión | `droopscan --version` |
| `--update` | Actualizar base de datos de vulnerabilidades y módulos conocidos. | `droopscan --update` |

> [!tip] Actualización Constante
> Es crucial mantener la base de datos de Droopscan actualizada (`--update`) para asegurar que pueda detectar las últimas vulnerabilidades y versiones de módulos/temas. Las bases de datos de vulnerabilidades se actualizan constantemente con nuevos CVEs y exploits.

### Opciones de Escaneo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-e` | Enumeración específica de componentes (ej. `modules`, `themes`, `users`). | `droopscan -e modules` |
| `--enumerate` | Enumerar todos los componentes o un tipo específico. | `droopscan --enumerate all` |
| `--modules` | Escanear módulos instalados y verificar sus versiones. | `droopscan --modules all` |
| `--themes` | Escanear temas (themes) instalados y verificar sus versiones. | `droopscan --themes all` |
| `--users` | Enumerar usuarios válidos en el sistema Drupal. | `droopscan --users` |
| `--brute` | Realizar un ataque de fuerza bruta contra usuarios enumerados o predefinidos. | `droopscan --brute` |

> [!warning] Enumeración de Usuarios y Fuerza Bruta
> La enumeración de usuarios (`--users`) y los ataques de fuerza bruta (`--brute`) son técnicas intrusivas. Deben realizarse únicamente en entornos controlados y con la autorización explícita del propietario del sistema. Un uso no autorizado puede tener consecuencias legales.

### Opciones de Salida (Output)

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-o` | Especificar un archivo para guardar los resultados del escaneo. | `droopscan -o resultados.txt` |
| `--output` | Especificar un archivo para guardar los resultados del escaneo. | `droopscan --output scan.txt` |
| `--json` | Exportar los resultados en formato JSON. | `droopscan --json output.json` |
| `--xml` | Exportar los resultados en formato XML. | `droopscan --xml output.xml` |
| `--csv` | Exportar los resultados en formato CSV. | `droopscan --csv output.csv` |

> [!info] Formatos de Salida
> La capacidad de exportar resultados en diferentes formatos (JSON, XML, CSV) es extremadamente útil para la integración con otras herramientas de análisis, sistemas de gestión de vulnerabilidades (VMS) o para el procesamiento automatizado de datos.

### Opciones de Conexión

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-t` | Establecer el tiempo máximo de espera para una conexión (en segundos). | `droopscan -t 30` |
| `--timeout` | Establecer el tiempo máximo de espera para una conexión (en segundos). | `droopscan --timeout 45` |
| `-p` | Usar un servidor proxy HTTP/SOCKS para la conexión. | `droopscan -p http://proxy:8080` |
| `--proxy` | Usar un servidor proxy HTTP/SOCKS para la conexión. | `droopscan --proxy socks5://127.0.0.1:9050` |
| `--cookie` | Añadir una cookie personalizada a las solicitudes HTTP. Útil para sesiones autenticadas. | `droopscan --cookie "session=value"` |
| `--user-agent` | Establecer un User-Agent personalizado para las solicitudes HTTP. | `droopscan --user-agent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"` |

> [!tip] Uso de Proxies y Cookies
> El uso de proxies (`-p`, `--proxy`) es fundamental para anonimizar el tráfico o para escanear a través de firewalls. Las cookies (`--cookie`) son esenciales para escanear sitios que requieren autenticación, permitiendo a Droopscan acceder a áreas protegidas y realizar un análisis más profundo. Un `User-Agent` personalizado puede ayudar a evadir detecciones básicas de WAFs o a simular un navegador específico.