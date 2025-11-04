--------
**netdiscover** es una herramienta de descubrimiento de hosts y exploración de redes que utiliza sondeos ARP (Address Resolution Protocol) para identificar dispositivos activos en una red local. Es especialmente útil para redes donde el protocolo ICMP (ping) está bloqueado, ya que ARP es fundamental para el funcionamiento de Ethernet.

## Tabla de opciones principales de netdiscover

### Opciones de Modo de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-i`|Especifica la interfaz de red|`netdiscover -i eth0`|
|`-r`|Escanea un rango de IP específico|`netdiscover -r 192.168.1.0/24`|
|`-l`|Lee archivo con rangos de IP a escanear|`netdiscover -l ips.txt`|
|`-p`|Modo pasivo (solo escucha, no envía paquetes)|`netdiscover -p`|
|`-f`|Modo rápido (escaneo con intervalos reducidos)|`netdiscover -f`|
|`-s`|Modo silencioso (solo muestra resultados)|`netdiscover -s`|
|`-c`|Número de veces a escanear cada IP|`netdiscover -c 5`|
|`-L`|Habilita modo loop (escaneo continuo)|`netdiscover -L`|

### Opciones de Configuración de Red

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-d`|Usa la base de datos de fabricantes personalizada|`netdiscover -d fabricantes.txt`|
|`-g`|Usa este gateway (para modo pasivo)|`netdiscover -g 192.168.1.1`|
|`-m`|Usa este archivo de máscaras|`netdiscover -m masks.txt`|
|`-F`|Filtro de expresiones regulares para fabricantes|`netdiscover -F "Cisco|Dell"`|
|`-P`|Imprime en formato adecuado para parsing|`netdiscover -P`|
|`-N`|No muestra el header|`netdiscover -N`|

### Opciones de Rendimiento y Tiempo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-t`|Timeout entre solicitudes ARP (ms)|`netdiscover -t 100`|
|`-T`|Timeout de espera de respuesta (ms)|`netdiscover -T 500`|
|`-S`|Habilita sleep entre rondas|`netdiscover -S`|
|`-A`|Habilita modo adaptativo|`netdiscover -A`|

### Opciones de Información y Help

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-h`|Muestra ayuda completa|`netdiscover -h`|
|`-V`|Muestra versión|`netdiscover -V`|
|`-v`|Modo verbose|`netdiscover -v`|
|`-j`|Muestra información de la interfaz|`netdiscover -j`|