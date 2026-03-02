---
aliases: ["Nmap", "Network Mapper", "Escaneo de Redes"]
tags: [hacking/herramientas, hacking/reconocimiento]
estado: 🟢 Terminado
---

Nmap (Network Mapper) es una herramienta de código abierto para la exploración de redes y la auditoría de seguridad. Se utiliza para descubrir hosts y servicios en una red, identificando así lo que se denomina el "mapa de la red". Nmap ofrece características avanzadas para el escaneo de puertos, la detección de sistemas operativos, la detección de versiones de servicios, la detección de vulnerabilidades y el scripting con NSE (Nmap Scripting Engine).

> [!info] La importancia de Nmap
> Nmap es una herramienta fundamental en el arsenal de cualquier profesional de ciberseguridad. Permite obtener una visión detallada de la infraestructura de red de un objetivo, lo cual es crucial para las fases de reconocimiento y enumeración en pruebas de penetración y auditorías de seguridad. Su flexibilidad y el potente Nmap Scripting Engine (NSE) lo hacen indispensable.

## Tabla de Opciones Principales de Nmap

### Opciones de Escaneo de Hosts

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-sL` | List scan - solo lista los objetivos | `nmap -sL 192.168.1.0/24` |
| `-sn` | Ping scan - solo descubre hosts activos | `nmap -sn 192.168.1.0/24` |
| `-Pn` | Treat all hosts as online - omite el descubrimiento | `nmap -Pn 192.168.1.1` |
| `-PS` | TCP SYN ping | `nmap -PS80,443 192.168.1.1` |
| `-PA` | TCP ACK ping | `nmap -PA80,443 192.168.1.1` |
| `-PU` | UDP ping | `nmap -PU53 192.168.1.1` |
| `-PY` | SCTP INIT ping | `nmap -PY 192.168.1.1` |
| `-PE` | ICMP echo ping | `nmap -PE 192.168.1.1` |
| `-PP` | ICMP timestamp ping | `nmap -PP 192.168.1.1` |
| `-PM` | ICMP netmask ping | `nmap -PM 192.168.1.1` |
| `-PO` | IP Protocol ping | `nmap -PO 192.168.1.1` |

> [!info] Escaneo de Hosts (`-sn`)
> La opción `-sn` (anteriormente `-sP`) realiza un "ping scan" para determinar qué hosts están activos en una red sin realizar un escaneo de puertos completo. Esto es útil para una rápida enumeración de hosts antes de proceder con escaneos más detallados. Utiliza una combinación de solicitudes ICMP echo, paquetes TCP SYN a puertos comunes (80, 443) y paquetes TCP ACK, así como solicitudes UDP a puertos comunes (53, 161).

> [!warning] Omisión de Descubrimiento (`-Pn`)
> La opción `-Pn` (anteriormente `-P0`) es crucial cuando se sospecha que los hosts objetivo están detrás de un *firewall* que bloquea las solicitudes de ping (ICMP). Al usar `-Pn`, Nmap asume que todos los hosts están activos y procede directamente con el escaneo de puertos, lo que puede aumentar significativamente el tiempo de escaneo si muchos hosts no están realmente en línea.

> [!info] Tipos de Ping TCP/UDP/ICMP
> Las opciones `-PS`, `-PA`, `-PU`, `-PE`, `-PP`, `-PM`, `-PO` permiten personalizar los métodos de descubrimiento de hosts.
> *   `-PS` (TCP SYN Ping): Envía un paquete SYN a un puerto específico. Si recibe un SYN/ACK, el host está activo.
> *   `-PA` (TCP ACK Ping): Envía un paquete ACK. Si recibe un RST, el host está activo. Útil para atravesar *firewalls* con estado.
> *   `-PU` (UDP Ping): Envía un paquete UDP. Si recibe un ICMP Port Unreachable, el puerto está cerrado; si no hay respuesta, el host podría estar activo.
> *   `-PE`, `-PP`, `-PM` (ICMP Pings): Variaciones de ICMP para el descubrimiento de hosts, útiles cuando los *firewalls* permiten ciertos tipos de ICMP.

### Opciones de Escaneo de Puertos

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-p` | Especifica puertos o rangos | `nmap -p 80,443,22-25 192.168.1.1` |
| `-p-` | Escanea todos los puertos (1-65535) | `nmap -p- 192.168.1.1` |
| `-F` | Fast mode - puertos más comunes | `nmap -F 192.168.1.1` |
| `-r` | Escanea puertos secuencialmente | `nmap -r 192.168.1.1` |
| `--top-ports` | Escanea los N puertos más comunes | `nmap --top-ports 100 192.168.1.1` |

> [!warning] Escaneo de Todos los Puertos (`-p-`)
> Escanear todos los 65535 puertos TCP/UDP con `-p-` puede ser extremadamente lento, especialmente en redes con latencia o *firewalls*. Se recomienda usarlo solo cuando sea estrictamente necesario o en combinación con opciones de *timing* agresivas (`-T4`, `-T5`) en entornos controlados.

> [!info] Modo Rápido (`-F`)
> La opción `-F` escanea los 100 puertos más comunes según la base de datos de Nmap (`nmap-services`). Esto proporciona un equilibrio entre velocidad y cobertura para un reconocimiento inicial.

### Técnicas de Escaneo

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-sS` | TCP SYN scan (stealth) | `nmap -sS 192.168.1.1` |
| `-sT` | TCP connect scan | `nmap -sT 192.168.1.1` |
| `-sU` | UDP scan | `nmap -sU 192.168.1.1` |
| `-sN` | TCP NULL scan | `nmap -sN 192.168.1.1` |
| `-sF` | TCP FIN scan | `nmap -sF 192.168.1.1` |
| `-sX` | TCP Xmas scan | `nmap -sX 192.168.1.1` |
| `-sA` | TCP ACK scan | `nmap -sA 192.168.1.1` |
| `-sW` | TCP Window scan | `nmap -sW 192.168.1.1` |
| `-sM` | TCP Maimon scan | `nmap -sM 192.168.1.1` |
| `--scanflags` | Escaneo con *flags* TCP personalizados | `nmap --scanflags URGACKPSHRSTSYNFIN 192.168.1.1` |

> [!info] TCP SYN Scan (`-sS`)
> También conocido como "half-open scan", es la técnica de escaneo por defecto y más popular. Envía un paquete SYN y espera un SYN/ACK (puerto abierto) o RST (puerto cerrado). Si recibe un SYN/ACK, Nmap envía un RST para cerrar la conexión sin completar el *handshake* TCP, lo que lo hace menos ruidoso y más difícil de registrar por algunos sistemas de detección de intrusiones (IDS) o *firewalls* básicos.

> [!info] TCP Connect Scan (`-sT`)
> Esta técnica completa el *handshake* TCP de tres vías (SYN, SYN/ACK, ACK) para determinar si un puerto está abierto. Es menos sigilosa que el SYN scan y deja más rastros en los logs del sistema objetivo, pero es útil cuando el usuario no tiene privilegios para enviar paquetes RAW (como en algunos sistemas operativos o configuraciones de red).

> [!info] UDP Scan (`-sU`)
> El escaneo UDP es inherentemente más lento y menos fiable que el TCP scan debido a la naturaleza sin conexión del protocolo UDP. Nmap envía paquetes UDP a los puertos objetivo. Si no recibe respuesta, el puerto puede estar abierto o filtrado. Si recibe un mensaje ICMP "Port Unreachable", el puerto está cerrado. La ausencia de respuesta es ambigua y requiere más tiempo para determinar el estado.

> [!info] Escaneos Sigilosos (NULL, FIN, Xmas)
> Las opciones `-sN` (NULL), `-sF` (FIN) y `-sX` (Xmas) son técnicas de escaneo sigilosas que manipulan los *flags* TCP. Se basan en el comportamiento de la RFC 793 para sistemas operativos compatibles:
> *   **NULL Scan (`-sN`)**: Envía un paquete sin *flags* TCP activados.
> *   **FIN Scan (`-sF`)**: Envía un paquete con el *flag* FIN activado.
> *   **Xmas Scan (`-sX`)**: Envía un paquete con los *flags* FIN, PSH y URG activados (como un "árbol de Navidad" de *flags*).
> En sistemas compatibles con la RFC 793, si el puerto está cerrado, se envía un RST. Si el puerto está abierto, no se envía ninguna respuesta. Esto puede ayudar a evadir *firewalls* que solo monitorean los *flags* SYN.

> [!info] TCP ACK Scan (`-sA`)
> El ACK scan no determina si un puerto está abierto o cerrado, sino si está filtrado o no filtrado por un *firewall*. Envía un paquete ACK. Si recibe un RST, el puerto es "no filtrado" (el *firewall* no lo bloquea). Si no hay respuesta o se recibe un ICMP "Communication Administratively Prohibited", el puerto es "filtrado". Es útil para mapear reglas de *firewall*.

> [!tip] Escaneo con *Flags* Personalizados (`--scanflags`)
> La opción `--scanflags` permite crear paquetes TCP con cualquier combinación de *flags* (SYN, ACK, FIN, RST, PSH, URG, ECE, CWR). Esto es útil para pruebas avanzadas de *firewalls* y para intentar evadir sistemas de detección de intrusiones que buscan patrones específicos. Por ejemplo, `--scanflags SYNACK` enviaría un paquete con ambos *flags* SYN y ACK activados.

### Detección de Versión y SO

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-sV` | Detección de versiones de servicios | `nmap -sV 192.168.1.1` |
| `--version-intensity` | Intensidad de detección (0-9) | `nmap -sV --version-intensity 5 192.168.1.1` |
| `--version-light` | Detección ligera (intensidad 2) | `nmap -sV --version-light 192.168.1.1` |
| `--version-all` | Máxima intensidad (intensidad 9) | `nmap -sV --version-all 192.168.1.1` |
| `-O` | Detección de sistema operativo | `nmap -O 192.168.1.1` |
| `--osscan-limit` | Limita OS detection a objetivos prometedores | `nmap -O --osscan-limit 192.168.1.1` |
| `--osscan-guess` | Adivina el OS más agresivamente | `nmap -O --osscan-guess 192.168.1.1` |

> [!info] Detección de Versiones (`-sV`)
> La detección de versiones (`-sV`) es crucial para identificar vulnerabilidades específicas. Nmap envía una serie de sondas a los puertos abiertos y compara las respuestas con una base de datos de *fingerprints* de servicios (`nmap-service-probes`). Esto permite identificar el nombre del servicio, la versión, el protocolo e incluso el sistema operativo subyacente en algunos casos.

> [!info] Detección de Sistema Operativo (`-O`)
> La detección de SO (`-O`) utiliza una combinación de técnicas, incluyendo el análisis de *fingerprints* TCP/IP (como el tamaño de la ventana TCP, el orden de los *flags* TCP, el valor TTL inicial, etc.) y respuestas ICMP. Nmap compara estas características con su base de datos (`nmap-os-db`) para identificar el sistema operativo y, a menudo, la versión exacta del *kernel*.

### Scripting con NSE (Nmap Scripting Engine)

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-sC` | Ejecuta scripts por defecto | `nmap -sC 192.168.1.1` |
| `--script` | Ejecuta scripts específicos | `nmap --script http-title 192.168.1.1` |
| `--script-args` | Argumentos para los scripts | `nmap --script http-title --script-args http.useragent="CustomUA" 192.168.1.1` |
| `--script-trace` | Muestra toda la comunicación del script | `nmap --script http-title --script-trace 192.168.1.1` |
| `--script-updatedb` | Actualiza la base de datos de scripts | `nmap --script-updatedb` |

> [!info] El Poder del NSE
> El Nmap Scripting Engine (NSE) es una de las características más potentes de Nmap. Permite a los usuarios escribir (o usar los miles de scripts existentes) para automatizar una amplia gama de tareas, desde la detección de vulnerabilidades y la enumeración avanzada hasta la explotación simple y la interacción con servicios de red. Los scripts están escritos en lenguaje Lua.

> [!tip] Uso de Comodines con `--script`
> Puedes usar comodines para ejecutar grupos de scripts. Por ejemplo:
> *   `nmap --script "http-*" 192.168.1.1` (ejecuta todos los scripts HTTP)
> *   `nmap --script "vuln,discovery" 192.168.1.1` (ejecuta scripts de las categorías `vuln` y `discovery`)
> *   `nmap --script "not intrusive" 192.168.1.1` (ejecuta todos los scripts excepto los intrusivos)

> [!info] Argumentos Comunes de Scripts (`--script-args`)
> Muchos scripts NSE aceptan argumentos para personalizar su comportamiento. Algunos ejemplos comunes incluyen:
> *   `http.useragent`: Para cambiar el User-Agent en solicitudes HTTP.
> *   `smbuser`, `smbpass`: Para credenciales de autenticación SMB.
> *   `vulns.showall`: Para mostrar todas las vulnerabilidades encontradas, no solo las de alta confianza.

### Categorías de Scripts NSE

| Categoría | Descripción | Ejemplo |
|---|---|---|
| `auth` | Scripts de autenticación | `--script auth` |
| `broadcast` | Descubrimiento de red por broadcast | `--script broadcast` |
| `brute` | Fuerza bruta contra servicios | `--script brute` |
| `default` | Scripts por defecto (equivale a -sC) | `--script default` |
| `discovery` | Descubrimiento de servicios y hosts | `--script discovery` |
| `dos` | Denial of Service (usar con cuidado) | `--script dos` |
| `exploit` | Explotación de vulnerabilidades | `--script exploit` |
| `external` | Scripts que usan recursos externos | `--script external` |
| `fuzzer` | Fuzzing de servicios | `--script fuzzer` |
| `intrusive` | Scripts intrusivos (detectables) | `--script intrusive` |
| `malware` | Detección de malware | `--script malware` |
| `safe` | Scripts seguros y no intrusivos | `--script safe` |
| `version` | Detección de versiones mejorada | `--script version` |
| `vuln` | Detección de vulnerabilidades | `--script vuln` |

> [!warning] Scripts `dos`, `exploit`, `intrusive`
> Las categorías `dos`, `exploit` e `intrusive` contienen scripts que pueden ser destructivos, generar mucho ruido en la red, o ser detectados fácilmente por sistemas de seguridad. Úsalos con extrema precaución y solo en entornos autorizados y controlados, como laboratorios de pruebas.

> [!info] Scripts `default` y `safe`
> La categoría `default` (`-sC`) incluye scripts que son considerados seguros, rápidos y útiles para la mayoría de los escaneos. La categoría `safe` es un subconjunto de `default` y garantiza que los scripts no son intrusivos ni causarán problemas.

### Opciones de Timing y Rendimiento

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-T0` | Paranoid (muy lento, evasión) | `nmap -T0 192.168.1.1` |
| `-T1` | Sneaky (muy lento) | `nmap -T1 192.168.1.1` |
| `-T2` | Polite (lento, menos ruido) | `nmap -T2 192.168.1.1` |
| `-T3` | Normal (default) | `nmap -T3 192.168.1.1` |
| `-T4` | Aggressive (rápido) | `nmap -T4 192.168.1.1` |
| `-T5` | Insane (muy rápido, puede ser poco fiable) | `nmap -T5 192.168.1.1` |
| `--min-hostgroup` | Tamaño mínimo del grupo de hosts | `nmap --min-hostgroup 64 192.168.1.0/24` |
| `--min-rate` | Velocidad mínima de paquetes por segundo | `nmap --min-rate 100 192.168.1.1` |
| `--max-rate` | Velocidad máxima de paquetes por segundo | `nmap --max-rate 100 192.168.1.1` |

> [!info] Plantillas de Timing (`-T0` a `-T5`)
> Nmap ofrece plantillas de *timing* para ajustar la velocidad y el sigilo del escaneo:
> *   `-T0` (Paranoid): Muy lento, envía paquetes muy espaciados para evadir IDS/IPS.
> *   `-T1` (Sneaky): Lento, similar a `-T0` pero un poco más rápido.
> *   `-T2` (Polite): Más lento que el normal, reduce la carga en la red y es menos propenso a ser detectado.
> *   `-T3` (Normal): El modo por defecto, un equilibrio entre velocidad y sigilo.
> *   `-T4` (Aggressive): Rápido, asume una red fiable y puede ser ruidoso.
> *   `-T5` (Insane): Muy rápido, puede perder algunos resultados en redes inestables o con alta latencia.
> La elección de la plantilla depende del entorno y los objetivos del escaneo.

> [!tip] Control Fino de Velocidad (`--min-rate`, `--max-rate`)
> Para un control más granular sobre la velocidad del escaneo, puedes usar `--min-rate` y `--max-rate` para especificar el número mínimo y máximo de paquetes por segundo. Esto es útil para evitar la detección por parte de sistemas de seguridad o para acelerar escaneos en redes de alta velocidad.

### Evasión y Ofuscación

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-f` | Fragmenta paquetes | `nmap -f 192.168.1.1` |
| `--mtu` | Especifica el MTU para fragmentación | `nmap --mtu 24 192.168.1.1` |
| `-D` | Decoy scanning (escaneo con señuelos o *decoys*) | `nmap -D RND:5 192.168.1.1` |
| `-S` | Spoof *source address* (suplantar dirección de origen) | `nmap -S 192.168.1.99 192.168.1.1` |
| `-e` | Especifica la interfaz de red | `nmap -e eth0 192.168.1.1` |
| `--source-port` | Spoof *source port* (suplantar puerto de origen) | `nmap --source-port 53 192.168.1.1` |
| `--data-length` | Añade datos aleatorios a los paquetes | `nmap --data-length 100 192.168.1.1` |
| `--ttl` | Establece TTL personalizado | `nmap --ttl 64 192.168.1.1` |
| `--spoof-mac` | Spoof *MAC address* (suplantar dirección MAC) | `nmap --spoof-mac 0 192.168.1.1` |

> [!info] Fragmentación de Paquetes (`-f`, `--mtu`)
> La fragmentación de paquetes divide los paquetes de Nmap en fragmentos más pequeños. Esto puede ayudar a evadir *firewalls* o IDS/IPS que inspeccionan solo los encabezados de los paquetes o que tienen dificultades para reensamblar paquetes fragmentados, aunque los sistemas modernos son más robustos contra esta técnica. `--mtu` permite especificar el tamaño máximo de unidad de transmisión.

> [!info] Decoy Scanning (`-D`)
> El *decoy scanning* (`-D`) permite enviar paquetes desde múltiples direcciones IP falsas (señuelos) junto con la dirección IP real del atacante. Esto dificulta que los sistemas de detección de intrusiones identifiquen la fuente real del escaneo, ya que los logs mostrarán múltiples IPs realizando la actividad.
`RND:5` genera 5 direcciones IP aleatorias.

> [!warning] Suplantación de Dirección IP (`-S`)
> La suplantación de la dirección IP de origen (`-S`) es una técnica avanzada que requiere privilegios de *root* y un conocimiento profundo de la red. Nmap enviará paquetes con la dirección IP especificada, pero no podrá recibir las respuestas directamente a menos que se configure un *sniffing* o se esté en la misma red que la IP suplantada.

> [!info] Suplantación de Puerto de Origen (`--source-port`)
> Al especificar un puerto de origen común (como 53 para DNS, 80 para HTTP, 443 para HTTPS), se puede intentar hacer que el escaneo parezca tráfico legítimo para algunos *firewalls* o sistemas de detección de intrusiones.

> [!info] Añadir Datos Aleatorios (`--data-length`)
> Añadir datos aleatorios a los paquetes puede ayudar a evadir *firewalls* o IDS/IPS que buscan patrones específicos en el tamaño o contenido de los paquetes.

> [!tip] Suplantación de Dirección MAC (`--spoof-mac`)
> Cambiar la dirección MAC (`--spoof-mac`) puede ser útil en redes locales para evadir la detección basada en la dirección física del adaptador de red. `0` genera una MAC aleatoria.

### Salida y Reportes

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-oN` | Salida normal (texto) | `nmap -oN resultado.txt 192.168.1.1` |
| `-oX` | Salida XML | `nmap -oX resultado.xml 192.168.1.1` |
| `-oS` | Salida *script kiddie* (formato menos estructurado) | `nmap -oS resultado.txt 192.168.1.1` |
| `-oG` | Salida *grepeable* (fácil de procesar con `grep`) | `nmap -oG resultado.txt 192.168.1.1` |
| `-oA` | Todas las salidas (normal, XML, *grepeable*) | `nmap -oA resultado 192.168.1.1` |
| `-v` | Aumenta el nivel de verbosidad (muestra más detalles) | `nmap -v 192.168.1.1` |
| `-vv` | Verbosidad alta (muestra aún más detalles) | `nmap -vv 192.168.1.1` |
| `--reason` | Muestra la razón del estado del puerto (ej. `syn-ack`, `rst-ack`) | `nmap --reason 192.168.1.1` |
| `--open` | Muestra solo puertos abiertos (y sus estados) | `nmap --open 192.168.1.1` |
| `--packet-trace` | Muestra todos los paquetes enviados/recibidos (para depuración) | `nmap --packet-trace 192.168.1.1` |

> [!tip] Salida XML (`-oX`)
> La salida XML es ideal para el procesamiento automatizado. Muchas herramientas de análisis de resultados de Nmap (como `xsltproc` o herramientas personalizadas) pueden consumir este formato para generar informes o integrar los resultados en otros flujos de trabajo.

> [!tip] Salida Grepeable (`-oG`)
> El formato *grepeable* (`-oG`) es un formato de salida simple y de una línea por host, diseñado para ser fácilmente procesado por herramientas de línea de comandos como `grep`, `awk` o `cut`. Es muy útil para extraer rápidamente información específica de los resultados del escaneo.

> [!info] Salida Agresiva (`-oA`)
> La opción `-oA` es muy conveniente, ya que guarda los resultados en los tres formatos principales (normal, XML y *grepeable*) con el nombre base especificado. Esto asegura que se tenga una copia de los resultados en formatos adecuados para lectura humana y procesamiento automatizado.

> [!tip] Verbosidad (`-v`, `-vv`) y Razón (`--reason`)
> Aumentar la verbosidad (`-v`, `-vv`) es útil para entender el progreso del escaneo y depurar problemas. La opción `--reason` proporciona información valiosa sobre por qué Nmap determinó un puerto como abierto, cerrado o filtrado, lo cual es fundamental para el análisis de *firewalls*.

### Opciones Diversas

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-6` | Escaneo de IPv6 | `nmap -6 2001:db8::1` |
| `-A` | Escaneo agresivo (detección de SO, versiones, scripts por defecto y *traceroute*) | `nmap -A 192.168.1.1` |
| `--datadir` | Especifica el directorio de datos de Nmap | `nmap --datadir /ruta/custom 192.168.1.1` |
| `--servicedb` | Especifica un archivo de servicios personalizado | `nmap --servicedb miservicios.txt 192.168.1.1` |
| `--versiondb` | Especifica un archivo de versiones personalizado | `nmap --versiondb miversiones.txt 192.168.1.1` |
| `--system-dns` | Usa el *resolver* DNS del sistema | `nmap --system-dns 192.168.1.1` |
| `--dns-servers` | Especifica servidores DNS personalizados | `nmap --dns-servers 8.8.8.8 192.168.1.1` |

> [!info] Escaneo Agresivo (`-A`)
> La opción `-A` es un atajo para activar varias opciones de escaneo avanzadas: detección de SO (`-O`), detección de versiones (`-sV`), ejecución de scripts por defecto (`-sC`) y *traceroute* (`--traceroute`). Es una opción muy completa para obtener una gran cantidad de información en un solo escaneo, pero también es más ruidosa y lenta.

> [!tip] Control de Resolución DNS (`--system-dns`, `--dns-servers`)
> Controlar cómo Nmap resuelve los nombres de dominio es importante. `--system-dns` fuerza a Nmap a usar el *resolver* DNS configurado en el sistema operativo, mientras que `--dns-servers` permite especificar servidores DNS específicos, lo cual es útil para pruebas de DNS o para evitar *resolvers* lentos o filtrados.