---
aliases: ["SSLScan"]
tags:
  - hacking/herramientas
  - redes/protocolos
estado: 🟢 Terminado
---
# SSLScan

SSLScan es una herramienta de línea de comandos diseñada para analizar servidores **SSL/TLS**. Proporciona información detallada sobre la configuración de seguridad de un servidor, incluyendo suites de cifrado soportadas, protocolos y posibles vulnerabilidades.

> [!info] Protocolos SSL/TLS
> **SSL** (Secure Sockets Layer) y **TLS** (Transport Layer Security) son protocolos criptográficos para comunicaciones seguras. 
> * **Obsoletos:** SSLv2, SSLv3 (vulnerable a POODLE), TLSv1.0 y TLSv1.1.
> * **Recomendados:** TLSv1.2 o TLSv1.3.

---

## 🛠️ Tabla de Opciones Principales

### 1. Opciones Básicas
| Opción | Descripción | Ejemplo |
| :--- | :--- | :--- |
| `--help` | Muestra el menú de ayuda. | `sslscan --help` |
| `--version` | Muestra la versión instalada. | `sslscan --version` |
| `--targets=` | Especifica un archivo con lista de objetivos. | `sslscan --targets=file.txt` |
| `--no-colour` | Desactiva la salida en colores. | `sslscan --no-colour` |
| `--verbose` | Activa el modo detallado (logs extra). | `sslscan --verbose` |
| `--show-certificate`| Muestra la información completa del certificado. | `sslscan --show-certificate` |
| `--show-times` | Muestra los tiempos de respuesta de cada handshake. | `sslscan --show-times` |

### 2. Opciones de Conexión
| Opción | Descripción | Ejemplo |
| :--- | :--- | :--- |
| `--port=` | Escanea un puerto específico (defecto 443). | `sslscan --port=8443` |
| `--starttls=` | Indica el protocolo para STARTTLS (smtp, pop3, etc). | `sslscan --starttls=smtp` |
| `--timeout=` | Tiempo de espera máximo de conexión (segundos). | `sslscan --timeout=5` |
| `--sleep=` | Retraso en milisegundos entre cada prueba. | `sslscan --sleep=1` |
| `--proxy=` | Configura un servidor proxy para el escaneo. | `sslscan --proxy=host:port` |
| `--proxy-auth=` | Credenciales para el proxy (usuario:pass). | `sslscan --proxy-auth=u:p` |

### 3. Opciones de Escaneo (Protocolos)
| Opción | Descripción | Ejemplo |
| :--- | :--- | :--- |
| `--ssl2` | Fuerza la prueba de SSLv2. | `sslscan --ssl2` |
| `--ssl3` | Fuerza la prueba de SSLv3. | `sslscan --ssl3` |
| `--tls1` | Fuerza la prueba de TLSv1.0. | `sslscan --tls1` |
| `--tls1-1` | Fuerza la prueba de TLSv1.1. | `sslscan --tls1-1` |
| `--tls1-2` | Fuerza la prueba de TLSv1.2. | `sslscan --tls1-2` |
| `--tls1-3` | Fuerza la prueba de TLSv1.3. | `sslscan --tls1-3` |
| `--pk=` | Ruta a la clave privada del cliente. | `sslscan --pk=key.pem` |
| `--cert=` | Ruta al certificado del cliente. | `sslscan --cert=cert.pem` |

---

> [!warning] Escaneo Activo
> El uso de **SSLScan** realiza un escaneo activo. Se establecen conexiones reales, lo que puede ser detectado por **IDS/IPS**. Asegúrate de tener autorización explícita antes de escanear sistemas ajenos.

> [!tip] Interpretación de Resultados
> Al analizar los resultados, prioriza revisar:
> - **Protocolos obsoletos:** La presencia de SSL o TLS < 1.2 es un hallazgo crítico.
> - **Cifrados débiles:** Algoritmos como RC4, DES o claves de < 128 bits.
> - **Vulnerabilidades:** Identifica Heartbleed o CCS Injection de forma rápida.

---

### 4. Opciones de Salida
| Opción | Descripción | Ejemplo |
| :--- | :--- | :--- |
| `--xml=` | Guarda los resultados en formato XML. | `sslscan --xml=out.xml` |
| `--json=` | Guarda los resultados en formato JSON. | `sslscan --json=out.json` |
| `--pretty` | Formatea el JSON para que sea legible. | `sslscan --pretty` |
| `--no-failed` | Oculta las pruebas que no tuvieron éxito. | `sslscan --no-failed` |