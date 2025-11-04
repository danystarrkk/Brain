------
**Nmap** (Network Mapper) es una herramienta de código abierto para exploración de redes y auditoría de seguridad. Se utiliza para descubrir hosts y servicios en una red, identificando así lo que se denomina "mapa de la red". Nmap ofrece características avanzadas para el escaneo de puertos, detección de sistemas operativos, detección de versiones de servicios, detección de vulnerabilidades y scripting con NSE (Nmap Scripting Engine).
## Tabla de opciones principales de Nmap

### Opciones de Escaneo de Hosts

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-sL`|List scan - solo lista los objetivos|`nmap -sL 192.168.1.0/24`|
|`-sn`|Ping scan - solo descubre hosts activos|`nmap -sn 192.168.1.0/24`|
|`-Pn`|Treat all hosts as online - omite el descubrimiento|`nmap -Pn 192.168.1.1`|
|`-PS`|TCP SYN ping|`nmap -PS80,443 192.168.1.1`|
|`-PA`|TCP ACK ping|`nmap -PA80,443 192.168.1.1`|
|`-PU`|UDP ping|`nmap -PU53 192.168.1.1`|
|`-PY`|SCTP INIT ping|`nmap -PY 192.168.1.1`|
|`-PE`|ICMP echo ping|`nmap -PE 192.168.1.1`|
|`-PP`|ICMP timestamp ping|`nmap -PP 192.168.1.1`|
|`-PM`|ICMP netmask ping|`nmap -PM 192.168.1.1`|
|`-PO`|IP Protocol ping|`nmap -PO 192.168.1.1`|

### Opciones de Escaneo de Puertos

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-p`|Especifica puertos o rangos|`nmap -p 80,443,22-25 192.168.1.1`|
|`-p-`|Escanea todos los puertos (1-65535)|`nmap -p- 192.168.1.1`|
|`-F`|Fast mode - puertos más comunes|`nmap -F 192.168.1.1`|
|`-r`|Escanea puertos secuencialmente|`nmap -r 192.168.1.1`|
|`--top-ports`|Escanea los N puertos más comunes|`nmap --top-ports 100 192.168.1.1`|

### Técnicas de Escaneo

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-sS`|TCP SYN scan (stealth)|`nmap -sS 192.168.1.1`|
|`-sT`|TCP connect scan|`nmap -sT 192.168.1.1`|
|`-sU`|UDP scan|`nmap -sU 192.168.1.1`|
|`-sN`|TCP NULL scan|`nmap -sN 192.168.1.1`|
|`-sF`|TCP FIN scan|`nmap -sF 192.168.1.1`|
|`-sX`|TCP Xmas scan|`nmap -sX 192.168.1.1`|
|`-sA`|TCP ACK scan|`nmap -sA 192.168.1.1`|
|`-sW`|TCP Window scan|`nmap -sW 192.168.1.1`|
|`-sM`|TCP Maimon scan|`nmap -sM 192.168.1.1`|
|`--scanflags`|Escaneo con flags personalizados|`nmap --scanflags URGACKPSHRSTSYNFIN 192.168.1.1`|

### Detección de Versión y SO

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-sV`|Detección de versiones de servicios|`nmap -sV 192.168.1.1`|
|`--version-intensity`|Intensidad de detección (0-9)|`nmap -sV --version-intensity 5 192.168.1.1`|
|`--version-light`|Detección ligera (intensidad 2)|`nmap -sV --version-light 192.168.1.1`|
|`--version-all`|Máxima intensidad (intensidad 9)|`nmap -sV --version-all 192.168.1.1`|
|`-O`|Detección de sistema operativo|`nmap -O 192.168.1.1`|
|`--osscan-limit`|Limita OS detection a objetivos prometedores|`nmap -O --osscan-limit 192.168.1.1`|
|`--osscan-guess`|Adivina el OS más agresivamente|`nmap -O --osscan-guess 192.168.1.1`|

### Scripting con NSE (Nmap Scripting Engine)

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-sC`|Ejecuta scripts por defecto|`nmap -sC 192.168.1.1`|
|`--script`|Ejecuta scripts específicos|`nmap --script http-title 192.168.1.1`|
|`--script-args`|Argumentos para los scripts|`nmap --script http-title --script-args http.useragent="CustomUA" 192.168.1.1`|
|`--script-trace`|Muestra toda la comunicación del script|`nmap --script http-title --script-trace 192.168.1.1`|
|`--script-updatedb`|Actualiza la base de datos de scripts|`nmap --script-updatedb`|

### Categorías de Scripts NSE

|Categoría|Descripción|Ejemplo|
|---|---|---|
|`auth`|Scripts de autenticación|`--script auth`|
|`broadcast`|Descubrimiento de red por broadcast|`--script broadcast`|
|`brute`|Fuerza bruta contra servicios|`--script brute`|
|`default`|Scripts por defecto (equivale a -sC)|`--script default`|
|`discovery`|Descubrimiento de servicios y hosts|`--script discovery`|
|`dos`|Denial of Service (usar con cuidado)|`--script dos`|
|`exploit`|Explotación de vulnerabilidades|`--script exploit`|
|`external`|Scripts que usan recursos externos|`--script external`|
|`fuzzer`|Fuzzing de servicios|`--script fuzzer`|
|`intrusive`|Scripts intrusivos (detectables)|`--script intrusive`|
|`malware`|Detección de malware|`--script malware`|
|`safe`|Scripts seguros y no intrusivos|`--script safe`|
|`version`|Detección de versiones mejorada|`--script version`|
|`vuln`|Detección de vulnerabilidades|`--script vuln`|

### Opciones de Timing y Rendimiento

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-T0`|Paranoid (muy lento, evasión)|`nmap -T0 192.168.1.1`|
|`-T1`|Sneaky (muy lento)|`nmap -T1 192.168.1.1`|
|`-T2`|Polite (lento, menos ruido)|`nmap -T2 192.168.1.1`|
|`-T3`|Normal (default)|`nmap -T3 192.168.1.1`|
|`-T4`|Aggressive (rápido)|`nmap -T4 192.168.1.1`|
|`-T5`|Insane (muy rápido, puede ser poco fiable)|`nmap -T5 192.168.1.1`|
|`--min-hostgroup`|Tamaño mínimo del grupo de hosts|`nmap --min-hostgroup 64 192.168.1.0/24`|
|`--min-rate`|Velocidad mínima de paquetes por segundo|`nmap --min-rate 100 192.168.1.1`|
|`--max-rate`|Velocidad máxima de paquetes por segundo|`nmap --max-rate 100 192.168.1.1`|

### Evasión y Ofuscación

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-f`|Fragmenta paquetes|`nmap -f 192.168.1.1`|
|`--mtu`|Especifica el MTU para fragmentación|`nmap --mtu 24 192.168.1.1`|
|`-D`|Decoy scanning (escaneo con señuelos)|`nmap -D RND:5 192.168.1.1`|
|`-S`|Spoof source address|`nmap -S 192.168.1.99 192.168.1.1`|
|`-e`|Especifica la interfaz de red|`nmap -e eth0 192.168.1.1`|
|`--source-port`|Spoof source port|`nmap --source-port 53 192.168.1.1`|
|`--data-length`|Añade datos aleatorios a los paquetes|`nmap --data-length 100 192.168.1.1`|
|`--ttl`|Establece TTL personalizado|`nmap --ttl 64 192.168.1.1`|
|`--spoof-mac`|Spoof MAC address|`nmap --spoof-mac 0 192.168.1.1`|

### Salida y Reportes

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-oN`|Salida normal (texto)|`nmap -oN resultado.txt 192.168.1.1`|
|`-oX`|Salida XML|`nmap -oX resultado.xml 192.168.1.1`|
|`-oS`|Salida script kiddie|`nmap -oS resultado.txt 192.168.1.1`|
|`-oG`|Salida grepeable|`nmap -oG resultado.txt 192.168.1.1`|
|`-oA`|Todas las salidas (normal, XML, grepeable)|`nmap -oA resultado 192.168.1.1`|
|`-v`|Aumenta el nivel de verbosidad|`nmap -v 192.168.1.1`|
|`-vv`|Verbosidad alta|`nmap -vv 192.168.1.1`|
|`--reason`|Muestra la razón del estado del puerto|`nmap --reason 192.168.1.1`|
|`--open`|Muestra solo puertos abiertos|`nmap --open 192.168.1.1`|
|`--packet-trace`|Muestra todos los paquetes enviados/recibidos|`nmap --packet-trace 192.168.1.1`|

### Opciones Diversas

|Opción|Descripción|Ejemplo|
|---|---|---|
|`-6`|Escaneo de IPv6|`nmap -6 2001:db8::1`|
|`-A`|Escaneo agresivo (OS, version, script, traceroute)|`nmap -A 192.168.1.1`|
|`--datadir`|Especifica directorio de datos de Nmap|`nmap --datadir /ruta/custom 192.168.1.1`|
|`--servicedb`|Especifica archivo de servicios personalizado|`nmap --servicedb miservicios.txt 192.168.1.1`|
|`--versiondb`|Especifica archivo de versiones personalizado|`nmap --versiondb miversiones.txt 192.168.1.1`|
|`--system-dns`|Usa el resolver DNS del sistema|`nmap --system-dns 192.168.1.1`|
|`--dns-servers`|Especifica servidores DNS personalizados|`nmap --dns-servers 8.8.8.8 192.168.1.`|