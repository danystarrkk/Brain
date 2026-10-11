---
aliases:
  - SSLyze
tags:
  - hacking/herramientas
  - redes/protocolos
estado: 🟢 Terminado
---
# SSLyze: Analizador Rápido y Completo de Configuraciones SSL/TLS

**SSLyze** es un analizador rápido y completo de configuraciones SSL/TLS. Permite escanear servidores para identificar problemas de configuración, vulnerabilidades y verificar el cumplimiento de mejores prácticas de seguridad.

> [!info] Importancia del Escaneo SSL/TLS
> La seguridad de las comunicaciones en internet depende en gran medida de una configuración robusta de SSL/TLS. Herramientas como SSLyze son cruciales para identificar configuraciones débiles, cifrados obsoletos o vulnerabilidades conocidas que podrían ser explotadas por atacantes para interceptar o manipular datos.

## Tabla de Opciones Principales de SSLyze

### Opciones Básicas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--help` | Muestra ayuda | `sslyze --help` |
| `--version` | Muestra la versión | `sslyze --version` |
| `--targets_in` | Archivo con objetivos (targets) | `sslyze --targets_in=targets.txt` |
| `--json_out` | Salida en formato JSON | `sslyze --json_out=results.json` |
| `--xml_out` | Salida en formato XML | `sslyze --xml_out=results.xml` |
| `--text_out` | Salida en formato de texto plano | `sslyze --text_out=results.txt` |
| `--quiet` | Modo silencioso (suprime la salida detallada) | `sslyze --quiet` |
| `--timeout` | Tiempo de espera (timeout) para la conexión en segundos | `sslyze --timeout=5` |
| `--https_tunnel` | Usar un túnel HTTPS a través de un proxy | `sslyze --https_tunnel=proxy:8080` |

### Opciones de Escaneo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--regular` | Realiza un escaneo regular completo | `sslyze --regular` |
| `--certinfo` | Obtiene información básica de los certificados SSL/TLS | `sslyze --certinfo` |
| `--certinfo_full` | Obtiene información completa y detallada de los certificados | `sslyze --certinfo_full` |
| `--compression` | Verifica la presencia de vulnerabilidades de compresión (ej. CRIME) | `sslyze --compression` |
| `--reneg` | Verifica la capacidad de renegociación segura de SSL/TLS | `sslyze --reneg` |
| `--resum` | Verifica la reanudación de sesión (session resumption) | `sslyze --resum` |
| `--heartbleed` | Realiza el test de la vulnerabilidad Heartbleed | `sslyze --heartbleed` |
| `--openssl_ccs` | Realiza el test de la vulnerabilidad OpenSSL CCS Injection | `sslyze --openssl_ccs` |
| `--robot` | Realiza el test de la vulnerabilidad ROBOT | `sslyze --robot` |
| `--fallback` | Realiza el test de la vulnerabilidad TLS Fallback SCSV | `sslyze --fallback` |

> [!warning] Vulnerabilidades Comunes
> *   **Heartbleed (CVE-2014-0160)**: Una vulnerabilidad crítica en la implementación de OpenSSL que permitía a un atacante leer hasta 64KB de memoria del servidor, exponiendo claves privadas, credenciales y otros datos sensibles.
> *   **OpenSSL CCS Injection (CVE-2014-0224)**: Permite a un atacante forzar el uso de cifrados débiles o incluso descifrar el tráfico interceptado.
> *   **ROBOT (Return Of Bleichenbacher's Oracle Threat)**: Una vulnerabilidad que permite a un atacante realizar ataques de descifrado de tráfico en servidores que utilizan cifrados RSA con relleno PKCS#1 v1.5.
> *   **TLS Fallback SCSV (CVE-2014-0224)**: Protege contra ataques de degradación de protocolo (downgrade attacks) donde un atacante intenta forzar al cliente y al servidor a usar versiones más antiguas y menos seguras de TLS/SSL.

### Opciones de Cipher Suites

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--sslv2` | Realiza el test para el protocolo SSLv2 | `sslyze --sslv2` |
| `--sslv3` | Realiza el test para el protocolo SSLv3 | `sslyze --sslv3` |
| `--tlsv1` | Realiza el test para el protocolo TLSv1.0 | `sslyze --tlsv1` |
| `--tlsv1_1` | Realiza el test para el protocolo TLSv1.1 | `sslyze --tlsv1_1` |
| `--tlsv1_2` | Realiza el test para el protocolo TLSv1.2 | `sslyze --tlsv1_2` |
| `--tlsv1_3` | Realiza el test para el protocolo TLSv1.3 | `sslyze --tlsv1_3` |
| `--cipher_list` | Especifica una lista de cifrados personalizada para probar | `sslyze --cipher_list=ALL` |

> [!tip] Evolución de los Protocolos SSL/TLS
> SSLv2 y SSLv3 son protocolos obsoletos y altamente inseguros, propensos a ataques como POODLE. TLSv1.0 y TLSv1.1 también se consideran inseguros y están siendo deprecados. TLSv1.2 es actualmente el estándar mínimo recomendado, mientras que TLSv1.3 es la versión más reciente y segura, ofreciendo mejoras significativas en rendimiento y seguridad. Es crucial deshabilitar las versiones antiguas en los servidores.

### Opciones de Performance

| Opción | Descripción | Ejemplo |
|---|---|---|
| `--nb_retries` | Número de reintentos en caso de fallos de conexión | `sslyze --nb_retries=3` |
| `--http_get` | Realiza una solicitud HTTP GET después del handshake SSL/TLS | `sslyze --http_get` |
| `--starttls` | Permite escanear servicios que usan STARTTLS (ej. `smtp`, `ftp`, `xmpp`, `imap`, `pop3`, `ldap`, `rdp`) | `sslyze --starttls=smtp` |
| `--sni` | Especifica el nombre de host SNI (Server Name Indication) | `sslyze --sni=example.com` |
| `--early_data` | Realiza el test de 0-RTT (Zero Round Trip Time) para TLSv1.3 | `sslyze --early_data` |

> [!info] Server Name Indication (SNI)
> SNI es una extensión del protocolo TLS que permite a un cliente indicar el nombre de host al que intenta conectarse al inicio del proceso de handshake. Esto es esencial para servidores que alojan múltiples dominios con certificados SSL/TLS diferentes en la misma dirección IP, permitiendo al servidor presentar el certificado correcto.