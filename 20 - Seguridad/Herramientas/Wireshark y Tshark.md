---
aliases:
  - Wireshark
  - Tshark
  - Analizador de protocolos
  - Captura de red
tags:
  - hacking/herramientas
estado: 🟢 Terminado
---
# Wireshark y Tshark: Análisis de Tráfico de Red

**Wireshark** es el analizador de protocolos de red más utilizado del mundo. Es una herramienta de código abierto que permite capturar y analizar el tráfico de red en tiempo real, así como examinar archivos de captura (PCAP). Permite una inspección profunda de cientos de protocolos, con capacidad de decodificación y análisis a nivel de paquete.

> [!info] Fundamentos de Wireshark
> Wireshark se basa en la biblioteca `libpcap` (o `WinPcap`/`Npcap` en Windows) para la captura de paquetes. Esta biblioteca permite la intercepción de tráfico directamente desde la interfaz de red en modo promiscuo, lo que significa que puede ver todo el tráfico que pasa por la interfaz, no solo el destinado a su propia dirección MAC. Su capacidad de decodificación se extiende a miles de protocolos, desde la capa de enlace hasta la capa de aplicación, proporcionando una visibilidad sin precedentes en las comunicaciones de red.

## Tabla de Opciones Principales de Wireshark (Interfaz Gráfica - GUI)

| Sección/Función         | Descripción                                                                 | Acceso/Ubicación                                  |
| :---------------------- | :-------------------------------------------------------------------------- | :------------------------------------------------ |
| **Captura (Capture)**   | Inicia una nueva captura en tiempo real.                                    | Menú Capture → Options                            |
| **Interfaces**          | Lista las interfaces de red disponibles para capturar.                     | Menú Capture → Options                            |
| **Filtros de Captura**  | Filtra el tráfico durante la captura (ej: `tcp port 80`).                  | Barra "Capture Filter" en Capture Options         |
| **Filtros de Visualización** | Filtra paquetes ya capturados (ej: `ip.src==192.168.1.1`).                  | Barra de filtro principal                         |
| **Flujos de Conversación** | Muestra conversaciones entre hosts (TCP, UDP, etc.).                        | Statistics → Conversations                        |
| **Seguimiento de Flujos** | Reconstruye streams/flujos de comunicación TCP.                             | Click derecho → Follow → TCP Stream               |
| **Estadísticas**        | Diversas herramientas de análisis estadístico.                              | Menú Statistics                                   |
| **Expert Info**         | Análisis experto de problemas en la captura.                                | Analyze → Expert Info                             |
| **Colorización**        | Reglas para colorear paquetes según protocolo/estado.                       | View → Coloring Rules                             |
| **Decodificación como** | Fuerza la interpretación de un tráfico como otro protocolo.                 | Click derecho → Decode As...                      |
| **Exportar Objetos**    | Extrae archivos transferidos (HTTP, SMB, etc.).                             | File → Export Objects                             |

> [!tip] Diferencia entre Filtros de Captura y Visualización
> Los **filtros de captura** (Capture Filters) utilizan la sintaxis de Berkeley Packet Filter (BPF) y se aplican *antes* de que los paquetes sean escritos en el archivo de captura. Esto reduce el tamaño del archivo y la carga de procesamiento. Los **filtros de visualización** (Display Filters) utilizan la sintaxis propia de Wireshark y se aplican *después* de que los paquetes han sido capturados, permitiendo una búsqueda y análisis más flexibles sobre los datos ya almacenados.

> [!info] Seguimiento de Flujos (Follow Stream)
> La función "Follow TCP Stream" es crucial para el análisis forense de red. Permite reconstruir la conversación completa entre dos puntos finales para un protocolo específico (HTTP, FTP, etc.), mostrando los datos de la capa de aplicación en un formato legible. Esto es invaluable para extraer credenciales, archivos o entender la lógica de una interacción cliente-servidor.

## Tabla de Opciones Principales de Wireshark (Línea de Comandos - `tshark`)

`tshark` es la versión en terminal de Wireshark. Es extremadamente potente para automatización y análisis en servidores sin GUI.

| Opción      | Descripción                                                                 | Ejemplo                                                              |
| :---------- | :-------------------------------------------------------------------------- | :------------------------------------------------------------------- |
| `-i`        | Especifica la interfaz de captura.                                          | `tshark -i eth0`                                                     |
| `-f`        | Filtro de captura (sintaxis BPF).                                           | `tshark -f "tcp port 443"`                                           |
| `-Y`        | Filtro de visualización (post-captura).                                     | `tshark -r captura.pcap -Y "http"`                                   |
| `-w`        | Escribe la captura en un archivo.                                           | `tshark -i eth0 -w salida.pcap`                                      |
| `-r`        | Lee un archivo de captura (PCAP).                                           | `tshark -r captura.pcap`                                             |
| `-V`        | Salida verbose (muestra todos los detalles del paquete).                    | `tshark -r captura.pcap -V`                                          |
| `-T fields` | Salida personalizada con campos específicos.                                | `tshark -r captura.pcap -T fields -e frame.number -e ip.src`         |
| `-c`        | Limita el número de paquetes a capturar.                                    | `tshark -i eth0 -c 100`                                              |
| `-a`        | Condición de parada (ej: duración).                                         | `tshark -i eth0 -a duration:60`                                      |
| `-S`        | Separa la salida por paquetes (con `-V`).                                   | `tshark -r captura.pcap -V -S`                                       |
| `-G`        | Genera reportes o ayuda.                                                    | `tshark -G fields` (lista campos disponibles)                       |
| `-d`        | Fuerza la decodificación de un tráfico como un protocolo.                   | `tshark -d tcp.port==8888,http`                                      |
| `-z`        | Estadísticas (equivalente al menú Statistics).                              | `tshark -r captura.pcap -z conv,tcp`                                 |
| `-o`        | Establece una preferencia.                                                  | `tshark -o "tcp.desegment_tcp_streams:TRUE"`                         |
| `-n`        | Deshabilita la resolución de nombres (más rápido).                          | `tshark -i eth0 -n`                                                  |
| `-N`        | Habilita la resolución de nombres de ciertos tipos.                         | `tshark -i eth0 -N m` (resuelve MAC)                                 |
| `-D`        | Lista las interfaces disponibles.                                           | `tshark -D`                                                          |
| `-F`        | Especifica el formato del archivo de salida.                                | `tshark -r captura.pcap -F pcapng -w nueva.pcapng`                   |
| `-h`        | Muestra la ayuda.                                                           | `tshark -h`                                                          |

> [!warning] Consideraciones de Rendimiento con `tshark`
> Al usar `tshark` para capturas en vivo en entornos de alto tráfico, es crucial optimizar el rendimiento. Utilice filtros de captura (`-f`) lo más específicos posible para reducir la cantidad de datos procesados. Evite la resolución de nombres (`-n`) a menos que sea estrictamente necesario, ya que las consultas DNS/MAC pueden ralentizar significativamente la captura y el análisis. Para capturas prolongadas, considere rotar los archivos de salida (`-b filesize:X -b files:Y`) para evitar archivos PCAP excesivamente grandes.

> [!tip] Extracción de Campos Específicos con `tshark -T fields`
> La opción `-T fields` junto con `-e` es increíblemente útil para extraer información específica de los paquetes de forma programática. Por ejemplo, para obtener la dirección IP de origen, la IP de destino y el puerto de destino de todos los paquetes HTTP en un archivo PCAP:
> ```bash
> tshark -r captura.pcap -Y "http" -T fields -e ip.src -e ip.dst -e tcp.dstport
> ```
> Esto facilita la integración de `tshark` en scripts para automatizar el análisis de datos de red.