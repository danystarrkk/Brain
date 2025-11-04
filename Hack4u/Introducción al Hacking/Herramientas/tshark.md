-----------
**tshark** es el analizador de protocolos de red en línea de comandos de la suite Wireshark. Ofrece todas las capacidades de análisis de Wireshark pero desde la terminal, lo que lo hace ideal para automatización, servidores remotos, y procesamiento de grandes volúmenes de datos de red.

## Tabla de opciones principales de tshark

### Opciones de Captura

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-i`|Especifica la interfaz de captura|`tshark -i eth0`|
|`-f`|Filtro de captura (sintaxis BPF)|`tshark -f "tcp port 80"`|
|`-s`|Snaplength (tamaño de captura por paquete)|`tshark -s 1500`|
|`-p`|Modo promiscuo no habilitado|`tshark -p`|
|`-I`|Modo monitor (inalámbrico)|`tshark -I`|
|`-B`|Tamaño del buffer de captura|`tshark -B 10`|
|`-L`|Lista enlaces/capas disponibles|`tshark -L`|
|`-D`|Lista interfaces disponibles|`tshark -D`|
|`--time-stamp-type`|Tipo de timestamp|`tshark --time-stamp-type=adapter_unsynced`|

### Opciones de Archivo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-w`|Escribe captura en archivo|`tshark -w captura.pcap`|
|`-r`|Lee captura desde archivo|`tshark -r captura.pcap`|
|`-V`|Salida verbose (detalles completos)|`tshark -r captura.pcap -V`|
|`-F`|Formato de archivo de salida|`tshark -F pcapng -w salida.pcapng`|
|`--capture-comment`|Añade comentario a la captura|`tshark --capture-comment="Test capture"`|
|`-g`|Seguimiento de paquetes (packet numbers)|`tshark -r file.pcap -g`|

### Opciones de Filtrado

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-Y`|Filtro de visualización (display filter)|`tshark -Y "http"`|
|`-2`|Realiza filtrado en dos pasadas|`tshark -2 -Y "complex.filter"`|
|`-R`|Filtro de visualización (obsoleto, usar -Y)|`tshark -R "tcp.port==80"`|
|`--enable-protocol`|Habilita protocolo específico|`tshark --enable-protocol=PROTO`|
|`--disable-protocol`|Deshabilita protocolo específico|`tshark --disable-protocol=PROTO`|

### Opciones de Salida y Formato

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-T`|Formato de salida (fields, json, pdml, etc.)|`tshark -T fields`|
|`-E`|Opciones para el formateador de salida|`tshark -E separator=,`|
|`-e`|Campo a mostrar (con -T fields)|`tshark -e frame.number -e ip.src`|
|`-O`|Especifica protocolos para mostrar detalles|`tshark -O http,tcp`|
|`-P`|Escribe información de paquetes en stdout|`tshark -P`|
|`-S`|Separador de líneas para output|`tshark -S "\n"`|
|`-X`|Opción extensión|`tshark -X lua_script:script.lua`|
|`--color`|Coloreado de salida|`tshark --color`|

### Opciones de Estadísticas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-z`|Estadísticas varias|`tshark -z io,stat,60`|
|`-q`|Modo quiet (solo estadísticas)|`tshark -q -z conv,tcp`|
|`--export-objects`|Exporta objetos (HTTP, SMB, etc.)|`tshark --export-objects=http,./dir/`|

### Opciones de Rendimiento y Configuración

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-a`|Condición de parada de captura|`tshark -a duration:60`|
|`-b`|Condición de rotación de archivos|`tshark -b filesize:100000`|
|`-c`|Número de paquetes a capturar|`tshark -c 1000`|
|`-n`|Deshabilita resolución de nombres|`tshark -n`|
|`-N`|Habilita resolución de nombres específica|`tshark -N m` (MAC)|
|`-d`|Decodifica tráfico como protocolo|`tshark -d tcp.port==8888,http`|
|`-C`|Configuración de perfil|`tshark -C MyProfile`|
|`-H`|Fichero de hosts personalizado|`tshark -H hostsfile`|
|`-o`|Preferencia de configuración|`tshark -o "tcp.desegment_tcp_streams:TRUE"`|
|`-M`|Estadísticas de paquetes perdidos|`tshark -M`|
|`-K`|Keytab para descifrado|`tshark -K keytabfile`|

### Opciones de Depuración y Misceláneas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-h`|Muestra ayuda|`tshark -h`|
|`--version`|Muestra versión|`tshark --version`|
|`-G`|Genera reportes/manuales|`tshark -G fields`|
|`-J`|Lista filtros de protocolo|`tshark -J http`|
|`-U`|Instrucciones de reloj PDUs|`tshark -U seconds`|
|`-A`|Delimitador personalizado|`tshark -A "===="`|
|`--export-psk`|Exporta PSK keys|`tshark --export-psk psk.txt`|

## Formatos de Salida (-T)

|Formato|Descripción|Ejemplo|
|---|---|---|
|`fields`|Campos específicos|`-T fields -e frame.number`|
|`json`|Salida JSON|`-T json`|
|`jsonraw`|JSON con valores raw|`-T jsonraw`|
|`ek`|Elasticsearch JSON|`-T ek`|
|`pdml`|XML Packet Details Markup Language|`-T pdml`|
|`ps`|PostScript|`-T ps`|
|`psml`|XML Packet Summary Markup Language|`-T psml`|
|`text`|Texto plano (default)|`-T text`|
|`tabs`|Separado por tabs|`-T tabs`|
|`latex`|LaTeX table|`-T latex`|

## Opciones de Estadísticas (-z)

|Estadística|Descripción|Ejemplo|
|---|---|---|
|`io,stat`|Estadísticas I/O|`-z io,stat,30`|
|`conv,tcp`|Conversaciones TCP|`-z conv,tcp`|
|`conv,udp`|Conversaciones UDP|`-z conv,udp`|
|`http,stat`|Estadísticas HTTP|`-z http,stat`|
|`dns,tree`|Estadísticas DNS|`-z dns,tree`|
|`smb,srt`|Estadísticas SMB|`-z smb,srt`|
|`protocols`|Distribución de protocolos|`-z protocols`|
|`hits,tree`|Árbol de hits|`-z hits,tree`|
|`ends,tcp`|Endpoints TCP|`-z ends,tcp`|