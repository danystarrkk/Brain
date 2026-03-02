---
aliases: ["tshark"]
tags:
  - hacking/herramientas
  - redes/comandos
estado: 🟢 Terminado
---
# Tshark

**Tshark** es el analizador de protocolos de red de línea de comandos de la suite Wireshark. Ofrece todas las capacidades de análisis de Wireshark, pero desde la terminal, lo que lo hace ideal para automatización, servidores remotos y procesamiento de grandes volúmenes de datos de red.

> [!info] Tshark y Wireshark
> Tshark es la versión de línea de comandos de Wireshark. Ambos utilizan la misma biblioteca de captura de paquetes (`libpcap` en sistemas tipo Unix y `WinPcap`/`Npcap` en Windows) y el mismo motor de análisis de protocolos, lo que garantiza la coherencia en el análisis de datos. Su naturaleza de CLI lo hace indispensable para scripts y entornos sin interfaz gráfica.

## Tabla de opciones principales de Tshark

### Opciones de Captura

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-i` | Especifica la interfaz de captura | `tshark -i eth0` |
| `-f` | Filtro de captura (sintaxis BPF) | `tshark -f "tcp port 80"` |
| `-s` | Snaplength (tamaño de captura por paquete) | `tshark -s 1500` |
| `-p` | Modo promiscuo no habilitado | `tshark -p` |
| `-I` | Modo monitor (inalámbrico) | `tshark -I` |
| `-B` | Tamaño del buffer de captura | `tshark -B 10` |
| `-L` | Lista enlaces/capas de datos disponibles | `tshark -L` |
| `-D` | Lista interfaces disponibles | `tshark -D` |
| `--time-stamp-type` | Tipo de timestamp | `tshark --time-stamp-type=adapter_unsynced` |

> [!tip] Filtros BPF (Berkeley Packet Filter)
> Los filtros BPF (`-f`) son muy eficientes porque se aplican directamente en el kernel antes de que los paquetes sean procesados por Tshark. Esto reduce la carga de CPU y la cantidad de datos que Tshark necesita manejar. Ejemplos comunes incluyen `host 192.168.1.1`, `port 22`, `src host 10.0.0.1 and dst port 80`, `tcp[13] & 0x02 != 0` (para SYN flags).

### Opciones de Archivo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-w` | Escribe captura en archivo | `tshark -w captura.pcap` |
| `-r` | Lee captura desde archivo | `tshark -r captura.pcap` |
| `-V` | Salida detallada (detalles completos) | `tshark -r captura.pcap -V` |
| `-F` | Formato de archivo de salida | `tshark -F pcapng -w salida.pcapng` |
| `--capture-comment` | Añade comentario a la captura | `tshark --capture-comment="Test capture"` |
| `-g` | Seguimiento de paquetes (packet numbers) | `tshark -r file.pcap -g` |

### Opciones de Filtrado

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-Y` | Filtro de visualización (display filter) | `tshark -Y "http"` |
| `-2` | Realiza filtrado en dos pasadas | `tshark -2 -Y "complex.filter"` |
| `-R` | Filtro de visualización (obsoleto; usar -Y) | `tshark -R "tcp.port==80"` |
| `--enable-protocol` | Habilita protocolo específico | `tshark --enable-protocol=PROTO` |
| `--disable-protocol` | Deshabilita protocolo específico | `tshark --disable-protocol=PROTO` |

> [!info] Diferencia entre filtros BPF y de visualización
> Los filtros BPF (`-f`) se aplican *antes* de que Tshark procese el paquete, descartando el tráfico no deseado en la capa de captura. Los filtros de visualización (`-Y`) se aplican *después* de que Tshark ha capturado y decodificado el paquete, permitiendo filtrar por campos de protocolo más complejos. Para un rendimiento óptimo, usa BPF para descartar grandes volúmenes de tráfico irrelevante y filtros de visualización para un análisis más granular.

### Opciones de Salida y Formato

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-T` | Formato de salida (fields, json, pdml, etc.) | `tshark -T fields` |
| `-E` | Opciones para el formateador de salida | `tshark -E separator=,` |
| `-e` | Campo a mostrar (con -T fields) | `tshark -e frame.number -e ip.src` |
| `-O` | Especifica protocolos para mostrar detalles | `tshark -O http,tcp` |
| `-P` | Escribe información de paquetes en stdout | `tshark -P` |
| `-S` | Separador de líneas para la salida | `tshark -S "\n"` |
| `-X` | Opción extensión | `tshark -X lua_script:script.lua` |
| `--color` | Coloreado de salida | `tshark --color` |

> [!tip] Extracción de campos para scripting
> La combinación `-T fields -e <campo1> -e <campo2> ...` es extremadamente útil para extraer datos específicos de los paquetes y procesarlos con scripts (Bash, Python, etc.). Por ejemplo, `tshark -r capture.pcap -T fields -e ip.src -e ip.dst -e tcp.port` puede generar una lista CSV de conexiones.

### Opciones de Estadísticas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-z` | Estadísticas diversas | `tshark -z io,stat,60` |
| `-q` | Modo silencioso (solo estadísticas) | `tshark -q -z conv,tcp` |
| `--export-objects` | Exporta objetos (HTTP, SMB, etc.) | `tshark --export-objects=http,./dir/` |

> [!info] Análisis rápido con `-z`
> La opción `-z` permite generar resúmenes estadísticos sin necesidad de procesar manualmente los paquetes. Es ideal para obtener una visión general rápida del tráfico, como la distribución de protocolos, conversaciones de red o estadísticas HTTP.

### Opciones de Rendimiento y Configuración

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-a` | Condición de parada de la captura | `tshark -a duration:60` |
| `-b` | Condición de rotación de archivos | `tshark -b filesize:100000` |
| `-c` | Número de paquetes a capturar | `tshark -c 1000` |
| `-n` | Deshabilita la resolución de nombres | `tshark -n` |
| `-N` | Habilita la resolución de nombres específica | `tshark -N m` (MAC) |
| `-d` | Decodifica el tráfico como un protocolo específico | `tshark -d tcp.port==8888,http` |
| `-C` | Configuración de perfil | `tshark -C MyProfile` |
| `-H` | Archivo de hosts personalizado | `tshark -H hostsfile` |
| `-o` | Preferencia de configuración | `tshark -o "tcp.desegment_tcp_streams:TRUE"` |
| `-M` | Estadísticas de paquetes perdidos | `tshark -M` |
| `-K` | Keytab para el descifrado | `tshark -K keytabfile` |

> [!tip] Decodificación de puertos no estándar
> La opción `-d` es crucial para analizar aplicaciones que utilizan puertos no estándar para protocolos conocidos. Por ejemplo, si un servidor web corre en el puerto 8888, `tshark -d tcp.port==8888,http` le indicará a Tshark que interprete el tráfico en ese puerto como HTTP.

### Opciones de Depuración y Misceláneas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-h` | Muestra la ayuda | `tshark -h` |
| `--version` | Muestra la versión | `tshark --version` |
| `-G` | Genera informes/manuales | `tshark -G fields` |
| `-J` | Lista filtros de protocolo | `tshark -J http` |
| `-U` | Instrucciones de reloj para PDUs | `tshark -U seconds` |
| `-A` | Delimitador personalizado | `tshark -A "===="` |
| `--export-psk` | Exporta claves PSK | `tshark --export-psk psk.txt` |

## Formatos de Salida (-T)

| Formato | Descripción | Ejemplo |
|---|---|---|
| `fields` | Muestra campos específicos | `-T fields -e frame.number` |
| `json` | Genera salida JSON | `-T json` |
| `jsonraw` | JSON con valores sin procesar (raw) | `-T jsonraw` |
| `ek` | Formato JSON para Elasticsearch (EK) | `-T ek` |
| `pdml` | XML (Packet Details Markup Language) | `-T pdml` |
| `ps` | Formato PostScript | `-T ps` |
| `psml` | XML (Packet Summary Markup Language) | `-T psml` |
| `text` | Texto plano (predeterminado) | `-T text` |
| `tabs` | Salida separada por tabulaciones | `-T tabs` |
| `latex` | Tabla en formato LaTeX | `-T latex` |

## Opciones de Estadísticas (-z)

| Estadística | Descripción | Ejemplo |
|---|---|---|
| `io,stat` | Estadísticas de E/S | `-z io,stat,30` |
| `conv,tcp` | Estadísticas de conversaciones TCP | `-z conv,tcp` |
| `conv,udp` | Estadísticas de conversaciones UDP | `-z conv,udp` |
| `http,stat` | Estadísticas HTTP | `-z http,stat` |
| `dns,tree` | Estadísticas DNS | `-z dns,tree` |
| `smb,srt` | Estadísticas SMB | `-z smb,srt` |
| `protocols` | Distribución de protocolos | `-z protocols` |
| `hits,tree` | Árbol de hits | `-z hits,tree` |
| `ends,tcp` | Estadísticas de endpoints TCP | `-z ends,tcp` |