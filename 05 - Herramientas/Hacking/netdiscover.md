---
aliases: ["netdiscover"]
tags:
  - hacking/herramientas
  - redes/comandos
estado: 🟢 Terminado
---
**netdiscover** es una herramienta de descubrimiento de hosts y exploración de redes que utiliza sondeos ARP (Address Resolution Protocol) para identificar dispositivos activos en una red local. Es especialmente útil para redes donde el protocolo ICMP (Internet Control Message Protocol, comúnmente conocido como ping) está bloqueado, ya que ARP es fundamental para el funcionamiento de Ethernet.

> [!info] Funcionamiento de ARP
> ARP opera en la capa de enlace de datos (Capa 2 del modelo OSI) y es crucial para la comunicación dentro de una red local. Cuando un host necesita enviar un paquete IP a otro host en la misma subred, pero solo conoce su dirección IP, utiliza ARP para resolver esa dirección IP a una dirección MAC (Media Access Control). Envía una solicitud ARP broadcast preguntando "¿Quién tiene esta IP? Dime tu MAC". El host con esa IP responde con su dirección MAC, permitiendo la comunicación directa a nivel de hardware. Esto lo hace efectivo para el descubrimiento de hosts incluso cuando los firewalls bloquean ICMP.

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

> [!tip] Modo Pasivo (`-p`) para Evasión
> El modo pasivo es valioso en escenarios de reconocimiento sigiloso. Al solo escuchar el tráfico ARP existente en la red, `netdiscover` no genera ningún paquete propio, lo que reduce significativamente la probabilidad de ser detectado por sistemas de detección de intrusiones (IDS/IPS) que monitorean actividades de escaneo activas. Sin embargo, su efectividad depende del volumen de tráfico ARP en la red; en redes con poca actividad, puede tardar más en descubrir hosts.

### Opciones de Configuración de Red

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-d`|Usa la base de datos de fabricantes personalizada|`netdiscover -d fabricantes.txt`|
|`-g`|Usa este gateway (para modo pasivo)|`netdiscover -g 192.168.1.1`|
|`-m`|Usa este archivo de máscaras|`netdiscover -m masks.txt`|
|`-F`|Filtro de expresiones regulares para fabricantes|`netdiscover -F "Cisco|Dell"`|
|`-P`|Imprime en formato adecuado para parsing|`netdiscover -P`|
|`-N`|No muestra el header|`netdiscover -N`|

> [!info] Formato para Parsing (`-P`)
> La opción `-P` es extremadamente útil para la automatización y la integración con otros scripts o herramientas. Al imprimir los resultados en un formato fácilmente parseable (generalmente una línea por host con campos delimitados), permite extraer direcciones IP, MAC y nombres de fabricantes de manera programática. Esto es ideal para alimentar listas de hosts a otras herramientas de escaneo o para generar informes personalizados.

### Opciones de Rendimiento y Tiempo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-t`|Timeout entre solicitudes ARP (ms)|`netdiscover -t 100`|
|`-T`|Timeout de espera de respuesta (ms)|`netdiscover -T 500`|
|`-S`|Habilita sleep entre rondas|`netdiscover -S`|
|`-A`|Habilita modo adaptativo|`netdiscover -A`|

> [!tip] Modo Adaptativo (`-A`)
> El modo adaptativo ajusta dinámicamente los tiempos de espera y los intervalos de sondeo basándose en la latencia y la respuesta de la red. Esto puede mejorar la eficiencia del escaneo en redes con diferentes características de rendimiento, evitando timeouts excesivos en redes rápidas y asegurando la detección en redes más lentas o congestionadas. Es una buena opción para optimizar el equilibrio entre velocidad y fiabilidad.

### Opciones de Información y Help

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-h`|Muestra ayuda completa|`netdiscover -h`|
|`-V`|Muestra versión|`netdiscover -V`|
|`-v`|Modo verbose|`netdiscover -v`|
|`-j`|Muestra información de la interfaz|`netdiscover -j`|
