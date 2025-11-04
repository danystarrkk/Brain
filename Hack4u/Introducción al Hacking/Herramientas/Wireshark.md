-----------
**Wireshark** es el analizador de protocolos de red más utilizado del mundo. Es una herramienta de código abierto que permite capturar y analizar el tráfico de red en tiempo real, así como examinar archivos de captura (PCAP). Permite una inspección profunda de cientos de protocolos, con capacidad de decodificación y análisis a nivel de paquete.

## Tabla de opciones principales de Wireshark (Interfaz Gráfica - GUI)

|Sección/Función|Descripción|Acceso/Ubicación|
|---|---|---|
|**Captura (Capture)**|Inicia una nueva captura en tiempo real|Menú Capture → Options|
|**Interfaces**|Lista las interfaces de red disponibles para capturar|Menú Capture → Options|
|**Filtros de Captura**|Filtra el tráfico durante la captura (ej: `tcp port 80`)|Barra "Capture Filter" en Capture Options|
|**Filtros de Visualización**|Filtra paquetes ya capturados (ej: `ip.src==192.168.1.1`)|Barra de filtro principal|
|**Flujos de Conversación**|Muestra conversaciones entre hosts (TCP, UDP, etc.)|Statistics → Conversations|
|**Seguimiento de Flujos**|Reconstruye streams/flujos de comunicación TCP|Click derecho → Follow → TCP Stream|
|**Estadísticas**|Diversas herramientas de análisis estadístico|Menú Statistics|
|**Expert Info**|Análisis experto de problemas en la captura|Analyze → Expert Info|
|**Colorización**|Reglas para colorear paquetes según protocolo/estado|View → Coloring Rules|
|**Decodificación como**|Forza la interpretación de un tráfico como otro protocolo|Click derecho → Decode As...|
|**Exportar Objetos**|Extrae archivos transferidos (HTTP, SMB, etc.)|File → Export Objects|

## Tabla de opciones principales de Wireshark (Línea de Comandos - `tshark`)

`tshark` es la versión en terminal de Wireshark. Es extremadamente potente para automatización.

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-i`|Especifica la interfaz de captura|`tshark -i eth0`|
|`-f`|Filtro de captura (sintaxis BPF)|`tshark -f "tcp port 443"`|
|`-Y`|Filtro de visualización (post-captura)|`tshark -r captura.pcap -Y "http"`|
|`-w`|Escribe la captura en un archivo|`tshark -i eth0 -w salida.pcap`|
|`-r`|Lee un archivo de captura (PCAP)|`tshark -r captura.pcap`|
|`-V`|Salida verbose (muestra todos los detalles del paquete)|`tshark -r captura.pcap -V`|
|`-T fields`|Salida personalizada con campos específicos|`tshark -r captura.pcap -T fields -e frame.number -e ip.src`|
|`-c`|Limita el número de paquetes a capturar|`tshark -i eth0 -c 100`|
|`-a`|Condición de parada (ej: duración)|`tshark -i eth0 -a duration:60`|
|`-S`|Separa la salida por paquetes (con `-V`)|`tshark -r captura.pcap -V -S`|
|`-G`|Genera reportes o ayuda|`tshark -G fields` (lista campos disponibles)|
|`-d`|Forza la decodificación de un tráfico como un protocolo|`tshark -d tcp.port==8888,http`|
|`-z`|Estadísticas (equivalente al menú Statistics)|`tshark -r captura.pcap -z conv,tcp`|
|`-o`|Establece una preferencia|`tshark -o "tcp.desegment_tcp_streams:TRUE"`|
|`-n`|Deshabilita la resolución de nombres (más rápido)|`tshark -i eth0 -n`|
|`-N`|Habilita la resolución de nombres de ciertos tipos|`tshark -i eth0 -N m` (resuelve MAC)|
|`-D`|Lista las interfaces disponibles|`tshark -D`|
|`-F`|Especifica el formato del archivo de salida|`tshark -r captura.pcap -F pcapng -w nueva.pcapng`|
|`-h`|Muestra la ayuda|`tshark -h`|