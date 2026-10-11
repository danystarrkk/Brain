---
aliases: WPScan
tags:
  - hacking/herramientas
  - hacking/web
estado: 🟢 Terminado
---
# WPScan: Escáner de Seguridad para WordPress

**WPScan** es un escáner de seguridad de WordPress de código abierto y gratuito que permite identificar vulnerabilidades, plugins/temas desactualizados, usuarios, y otros problemas de seguridad en sitios WordPress. Es una herramienta esencial para auditores de seguridad y administradores de sistemas que buscan proteger sus instalaciones de WordPress.

> [!info] **Base de Datos de Vulnerabilidades**
> WPScan utiliza una base de datos de vulnerabilidades mantenida por la comunidad y por el equipo de WPScan. Esta base de datos se actualiza constantemente con nuevas vulnerabilidades conocidas en el core de WordPress, plugins y temas. El uso de un token de API (gratuito para uso no comercial) mejora significativamente la precisión y exhaustividad de los resultados al acceder a la base de datos completa.

## Tabla de Opciones Principales de WPScan

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--url` | URL objetivo del sitio WordPress. | `wpscan --url https://example.com` |
| `--help` | Muestra la ayuda y las opciones disponibles. | `wpscan --help` |
| `--version` | Muestra la versión actual de WPScan. | `wpscan --version` |
| `--verbose` | Habilita la salida detallada para mostrar más información durante el escaneo. | `wpscan --verbose` |
| `--output` | Especifica un archivo para guardar los resultados del escaneo. | `wpscan --output resultados.txt` |
| `--format` | Define el formato de salida de los resultados (ej. `json`, `xml`, `cli`). | `wpscan --format json` |

> [!tip] **Uso de HTTPS**
> Siempre que sea posible, utiliza `https://` en la URL objetivo para asegurar que el escaneo se realice sobre una conexión cifrada, reflejando el entorno de producción real.

### Opciones de Escaneo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--enumerate` | Realiza una enumeración específica de componentes. Los valores comunes incluyen `u` (usuarios), `vp` (plugins vulnerables), `vt` (temas vulnerables), `ap` (todos los plugins), `at` (todos los temas), `dbe` (exportaciones de base de datos), `cb` (copias de seguridad de configuración). | `wpscan --enumerate u,vp` |
| `--plugins` | Escanea plugins instalados. Puede ser `all` para todos, o una lista separada por comas. | `wpscan --plugins all` |
| `--themes` | Escanea temas instalados. Puede ser `all` para todos, o una lista separada por comas. | `wpscan --themes all` |
| `--config-backups` | Busca copias de seguridad de archivos de configuración (`wp-config.php.bak`, etc.). | `wpscan --config-backups` |
| `--db-exports` | Busca exportaciones de base de datos (`.sql`, `.sql.zip`, etc.). | `wpscan --db-exports` |
| `--medias` | Enumera archivos multimedia que podrían revelar información sensible. | `wpscan --medias` |

### Opciones de Fuerza Bruta

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--usernames` | Proporciona un archivo de lista de usuarios para intentar ataques de fuerza bruta. | `wpscan --usernames users.txt` |
| `--passwords` | Proporciona un archivo de lista de contraseñas para ataques de fuerza bruta. | `wpscan --passwords pass.txt` |
| `--password-attack` | Especifica el tipo de ataque de fuerza bruta (ej. `wp-login` para el formulario de inicio de sesión). | `wpscan --password-attack wp-login` |
| `--threads` | Define el número de hilos concurrentes para acelerar el ataque de fuerza bruta. | `wpscan --threads 20` |
| `--request-timeout` | Establece el tiempo de espera en segundos para las solicitudes HTTP. | `wpscan --request-timeout 30` |

> [!warning] **Consideraciones Legales y Éticas**
> La realización de ataques de fuerza bruta sin el consentimiento explícito del propietario del sitio es ilegal y poco ético. Utiliza estas opciones únicamente en entornos controlados y con autorización previa.

### Opciones de API

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--api-token` | Proporciona tu token de API para acceder a la base de datos completa de vulnerabilidades de WPScan. | `wpscan --api-token YOUR_TOKEN` |
| `--update` | Actualiza la base de datos local de vulnerabilidades de WPScan. | `wpscan --update` |
| `--disable-tls-checks` | Desactiva la verificación de certificados TLS/SSL. Útil para sitios con certificados autofirmados o inválidos en entornos de laboratorio, pero no recomendado en producción. | `wpscan --disable-tls-checks` |

> [!info] **Obtención del API Token**
> Puedes obtener un token de API gratuito registrándote en el sitio web de WPScan (wpscan.com). Este token es crucial para obtener los resultados más precisos y actualizados de las vulnerabilidades.
