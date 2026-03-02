---
aliases:
  - masscan
tags:
  - hacking/herramientas
  - hacking/reconocimiento
estado: 🟢 Terminado
---
Masscan (Mass IP Port Scanner) es un escáner de puertos de altísima velocidad diseñado para escanear grandes rangos de IP en tiempos récord. Utiliza transmisión asíncrona y personalización de paquetes a nivel de kernel, permitiendo escanear toda Internet en minutos desde una sola máquina.

> [!info] Rendimiento de Masscan
> La capacidad de Masscan para escanear a velocidades extremas se debe a su implementación de un motor de escaneo asíncrono. A diferencia de escáneres tradicionales que esperan respuestas antes de enviar el siguiente paquete, Masscan envía paquetes de forma continua y procesa las respuestas a medida que llegan. La "personalización de paquetes a nivel de kernel" significa que Masscan puede construir y enviar paquetes IP/TCP/UDP directamente, sin pasar por la pila de red estándar del sistema operativo, lo que reduce la latencia y aumenta el rendimiento. Esto es crucial para escanear redes masivas o incluso toda la internet en cuestión de minutos.

## Tabla de Opciones Principales de Masscan

### Opciones Básicas de Escaneo

| Opción    | Descripción                                | Ejemplo                               |
| --------- | ------------------------------------------ | ------------------------------------- |
| `-p`      | Puertos a escanear                         | `masscan -p80,443 192.168.1.0/24`     |
| `--ports` | Equivalente a `-p`                         | `masscan --ports 1-1000`              |
| `--rate`  | Paquetes por segundo a enviar              | `masscan --rate 100000`               |
| `-iL`     | Lee objetivos desde archivo                | `masscan -iL targets.txt`             |
| `-oB`     | Salida en formato binario                  | `masscan -oB output.bin`              |
| `-oL`     | Salida en formato lista                    | `masscan -oL output.txt`              |
| `-oJ`     | Salida en formato JSON                     | `masscan -oJ output.json`             |
| `-oG`     | Salida en formato grepable                 | `masscan -oG output.grep`             |
| `-oX`     | Salida en formato XML                      | `masscan -oX output.xml`              |
| `--echo`  | Muestra la configuración actual de Masscan | `masscan --echo`                      |

> [!tip] Formatos de Salida
> Masscan ofrece múltiples formatos de salida para facilitar la integración con otras herramientas o el análisis posterior. El formato JSON (`-oJ`) es ideal para el procesamiento programático, mientras que el formato grepable (`-oG`) es útil para filtrar rápidamente resultados con herramientas de línea de comandos como `grep` o `awk`. El formato binario (`-oB`) es el más eficiente para almacenar grandes volúmenes de datos de escaneo y puede ser leído posteriormente con la opción `--readscan`.

### Opciones de Configuración de Red

| Opción           | Descripción                                  | Ejemplo                               |
| ---------------- | -------------------------------------------- | ------------------------------------- |
| `--adapter`      | Interfaz de red a usar                       | `masscan --adapter eth0`              |
| `--adapter-ip`   | IP específica del adaptador                  | `masscan --adapter-ip 192.168.1.100`  |
| `--adapter-port` | Puerto fuente a usar                         | `masscan --adapter-port 40000`        |
| `--adapter-mac`  | Dirección MAC a usar                         | `masscan --adapter-mac 00:11:22:33:44:55` |
| `--router-ip`    | IP del router/gateway                        | `masscan --router-ip 192.168.1.1`     |
| `--router-mac`   | Dirección MAC del router/gateway             | `masscan --router-mac 00:11:22:33:44:55` |
| `--source-ip`    | IP de origen personalizada para los paquetes | `masscan --source-ip 1.2.3.4`         |

> [!warning] Spoofing de IP con `--source-ip`
> La opción `--source-ip` permite falsificar la dirección IP de origen de los paquetes enviados. Esto puede ser útil para evadir la detección o para realizar escaneos desde una dirección IP que no es la de la máquina atacante. Sin embargo, al falsificar la IP de origen, Masscan no podrá recibir las respuestas directamente, ya que estas se dirigirán a la IP falsificada. Esto es útil para escaneos "fire-and-forget" donde solo se busca la reacción de los objetivos, no la respuesta completa. Para recibir respuestas, se debe configurar un sniffer o un router para redirigir el tráfico.

### Opciones de Rendimiento y Tuning

| Opción                | Descripción                                | Ejemplo                         |
| --------------------- | ------------------------------------------ | ------------------------------- |
| `--rate`              | Tasa de paquetes por segundo               | `masscan --rate 1000000`        |
| `--wait`              | Segundos a esperar al final del escaneo    | `masscan --wait 5`              |
| `--connection-timeout`| Tiempo de espera para la conexión          | `masscan --connection-timeout 10` |
| `--retries`           | Intentos de reenvío de paquetes            | `masscan --retries 2`           |
| `--packet-trace`      | Muestra paquetes enviados y recibidos      | `masscan --packet-trace`        |
| `--seed`              | Semilla para generación aleatoria de IPs/puertos | `masscan --seed 12345`          |
| `--regress`           | Modo de regresión (para pruebas internas)  | `masscan --regress`             |

> [!info] Ajuste de la Tasa (`--rate`)
> La opción `--rate` es crítica para el rendimiento y la discreción. Una tasa alta (millones de paquetes/segundo) permite escanear rápidamente grandes segmentos de red, pero puede generar mucho ruido y ser detectada fácilmente por sistemas de detección de intrusiones (IDS/IPS). Una tasa baja puede ser más sigilosa, pero el escaneo tardará mucho más. El valor óptimo depende de la capacidad de la red, el hardware y los objetivos del escaneo. Masscan puede saturar enlaces de red si la tasa es demasiado alta.

### Opciones de Escaneo Avanzado

| Opción            | Descripción                                  | Ejemplo                           |
| ----------------- | -------------------------------------------- | --------------------------------- |
| `--banners`       | Captura banners de servicios                 | `masscan --banners`               |
| `--heartbleed`    | Detecta la vulnerabilidad Heartbleed         | `masscan --heartbleed`            |
| `--nmap`          | Muestra comandos Nmap equivalentes           | `masscan --nmap`                  |
| `--ping`          | Incluye descubrimiento de hosts (ICMP/ARP)   | `masscan --ping`                  |
| `--exclude`       | Excluye IPs o rangos de IPs del escaneo      | `masscan --exclude 192.168.1.100` |
| `--excludefile`   | Excluye IPs desde un archivo                 | `masscan --excludefile exclude.txt` |
| `--randomize-hosts`| Aleatoriza el orden de las IPs objetivo      | `masscan --randomize-hosts`       |
| `--randomize-ports`| Aleatoriza el orden de los puertos a escanear| `masscan --randomize-ports`       |

> [!info] Captura de Banners (`--banners`)
> La captura de banners es una técnica de reconocimiento pasivo que implica recolectar la información que un servicio envía automáticamente al establecer una conexión. Estos banners a menudo revelan el tipo de servicio, la versión del software y, en ocasiones, el sistema operativo subyacente. Esta información es invaluable para identificar posibles vulnerabilidades asociadas a versiones específicas de software.

> [!warning] Detección de Heartbleed (`--heartbleed`)
> La opción `--heartbleed` permite a Masscan intentar detectar la vulnerabilidad Heartbleed (CVE-2014-0160) en servicios SSL/TLS. Esta vulnerabilidad en OpenSSL permitía a un atacante leer hasta 64 KB de memoria de un servidor o cliente vulnerable, exponiendo potencialmente claves privadas, credenciales y otros datos sensibles. Aunque es una vulnerabilidad antigua, aún puede encontrarse en sistemas no parcheados.

> [!tip] Sinergia con Nmap (`--nmap`)
> Masscan es excelente para el escaneo rápido de puertos a gran escala, identificando qué puertos están abiertos. Sin embargo, no realiza un análisis profundo de los servicios. La opción `--nmap` es útil porque, una vez que Masscan ha identificado los puertos abiertos, puede generar comandos Nmap para realizar un escaneo más detallado (detección de versiones, scripts de vulnerabilidad, etc.) en esos puertos específicos, combinando la velocidad de Masscan con la profundidad de Nmap.

### Opciones de Output y Formatos

| Opción             | Descripción                                | Ejemplo                        |
| ------------------ | ------------------------------------------ | ------------------------------ |
| `-v`               | Salida detallada (verbose)                 | `masscan -v`                   |
| `--status-updates` | Intervalo de actualizaciones de estado     | `masscan --status-updates 1`   |
| `--iflist`         | Lista las interfaces de red disponibles    | `masscan --iflist`             |
| `--resume`         | Reanuda un escaneo desde un archivo de estado | `masscan --resume paused.conf` |
| `--readscan`       | Lee un archivo binario de escaneo          | `masscan --readscan scan.bin`  |

> [!info] Reanudación de Escaneos (`--resume`)
> La opción `--resume` es extremadamente útil para escaneos de larga duración o para aquellos que se interrumpen. Masscan puede guardar su estado de escaneo en un archivo (generalmente con la extensión `.conf` o `.masscan`) y luego reanudar desde ese punto, evitando tener que empezar de nuevo y ahorrando tiempo y recursos. Esto es especialmente valioso en entornos de laboratorio o en escaneos de rangos IP muy extensos.