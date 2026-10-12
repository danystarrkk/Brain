---
tags:
  - sistemas
  - redes
tipo: concepto
estado: terminado
---

# Conceptos Fundamentales de Redes

Los cimientos sobre los que se apoya todo lo demás en redes: el **modelo en capas** (OSI y TCP/IP), el **encapsulado** que conecta esas capas, la dirección **IP** y la **MAC**, el transporte **TCP/UDP** con su handshake, los puertos comunes y la aritmética de **subredes**. Al final, una metodología de diagnóstico que usa todo lo anterior.

> [!info] Por qué el modelo en capas importa de verdad
> Cada capa resuelve un problema concreto y **habla solo con las capas contiguas**. Eso da tres ventajas prácticas: (1) puedes diagnosticar *en qué capa* está el fallo ("no hay ARP = capa 2, hay ARP pero no ruta = capa 3, hay ruta pero el puerto no responde = capa 4"), (2) las herramientas se mapean a capas (`arp-scan` → L2, `ping`/`traceroute` → L3, `nmap` → L4, `curl` → L7), y (3) puedes reemplazar una capa sin tocar las demás (wifi por cable, TCP sobre IP v6 sobre cualquier enlace).

## El modelo OSI de 7 capas

El modelo **OSI** (*Open Systems Interconnection*) es una estructura conceptual de **siete capas** que describe la comunicación entre dispositivos. Es el marco didáctico con el que se habla de redes.

| Capa | Nombre | PDU | Dirección | Protocolos/ejemplos | Herramientas típicas |
|---|---|---|---|---|---|
| 7 | **Aplicación** | Datos | — | HTTP(S), DNS, SMTP, FTP, SSH, SNMP | `curl`, `nmap -sV`, navegadores |
| 6 | **Presentación** | Datos | — | TLS/SSL, codificación (UTF-8, Base64), compresión | `openssl s_client`, `ffuf` |
| 5 | **Sesión** | Datos | — | RPC, sesiones SMB, sockets | `wireshark` (follow stream) |
| 4 | **Transporte** | Segmentos (TCP) / Datagramas (UDP) | **Puertos** | TCP, UDP | `ss`, `nmap -sU`, `nc -zv` |
| 3 | **Red** | Paquetes | **IP** | IPv4, IPv6, ICMP, IPsec, enrutamiento | `ip route`, `ping`, `traceroute`, `nmap -sn` |
| 2 | **Enlace** | Tramas | **MAC** | Ethernet (802.3), WiFi (802.11), ARP, VLANs | `arp-scan`, [[netdiscover]], `ip link` |
| 1 | **Física** | Bits | — | Cable, fibra, radio, voltajes | Medidores, LEDs, [[Wireshark y Tshark]] en la interfaz |

1. **Capa física (1)**: transmisión de bits por el medio (cobre, fibra, radio). Define especificaciones eléctricas, mecánicas y de procedimiento para activar, mantener y desactivar el enlace físico.
2. **Capa de enlace de datos (2)**: transferencia confiable de datos **entre dispositivos del mismo segmento**. Detección de errores de trama (CRC) y acceso al medio. Se subdivide en **LLC** (control de enlace lógico) y **MAC** (control de acceso al medio — de ahí salen las direcciones MAC). Operan aquí switches, bridges y las tarjetas de red.
3. **Capa de red (3)**: **enrutamiento de paquetes a través de redes distintas**, con direcciones lógicas (IP). Operan los routers. También ICMP (`ping`, `traceroute`).
4. **Capa de transporte (4)**: entrega entre **procesos finales** (puertos), con control de flujo, congestión y (en TCP) confiabilidad.
5. **Capa de sesión (5)**: establecimiento, mantenimiento y terminación de sesiones de diálogo.
6. **Capa de presentación (6)**: representación de los datos: cifrado, compresión, serialización — que los datos sean legibles para la aplicación receptora.
7. **Capa de aplicación (7)**: servicios de red para el usuario final: correo, web, transferencia de archivos, resolución de nombres.

> [!tip] Mnemotecnica de las 7 capas (de abajo a arriba)
> "**P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way" → Physical, Data, Network, Transport, Session, Presentation, Application. (Invertida: "**A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing".)

## El modelo TCP/IP: el que realmente corre

El modelo **TCP/IP** (de facto, el de Internet) agrupa las 7 capas OSI en 4 (o 5):

| TCP/IP | Capas OSI que absorbe | Protocolos |
|---|---|---|
| **Acceso a la red** (o Enlace) | 1 + 2 | Ethernet, WiFi, ARP, PPP |
| **Internet** | 3 | IPv4, IPv6, ICMP, ICMPv6, IPsec |
| **Transporte** | 4 | TCP, UDP |
| **Aplicación** | 5 + 6 + 7 | HTTP(S), DNS, SMTP, FTP, SSH, TLS (aquí se sitúa en la práctica) |

> [!info] OSI vs TCP/IP
> OSI es el **marco conceptual** de 7 capas; TCP/IP es la **implementación real** de 4/5 capas. En la práctica se usan juntos: se diagnostica "en capa OSI" (L2/L3/L4/L7 es jerga universal) pero el tráfico real es TCP/IP. TLS es el ejemplo clásico de divergencia: OSI lo sitúa en la capa 6 (presentación), pero en el stack real va **entre la aplicación y el transporte** (https://tls.example → TCP:443 con TLS dentro del flujo TCP).

### Encapsulación y desencapsulación

Al enviar, cada capa **añade su cabecera** (y la de abajo, sus datos como payload); al recibir, cada capa **lee y remueve** la suya:

```
Aplicación:   [ HTTP: GET / ]                                  ← datos
Transporte:   [TCP hdr | HTTP]                                 ← + puerto, seq/ack, checksum
Internet:     [IP hdr | TCP hdr | HTTP]                        ← + IP origen/destino, TTL
Enlace:       [Eth hdr | IP hdr | TCP hdr | HTTP | FCS]        ← + MAC origen/destino, CRC
Física:       bits por el medio
```

| Al enviar | Al recibir |
|---|---|
| Encapsulación: cada capa envuelve los datos de la superior con su cabecera (y trailer en L2) | Desencapsulación: cada capa valida su cabecera, la quita y entrega el payload de vuelta |
| La cabecera L2 **cambia en cada salto** (MAC del siguiente router) | La cabecera L3 **se conserva** de extremo a extremo (salvo NAT, que la reescribe) |
| MTU limita el tamaño: >1500 B IP = fragmentación (o PMTUD) | La reensambla la capa destino |

## Direcciones IP (IPv4 e IPv6)

Las direcciones IP identifican **dispositivos (interfaces) lógicos** en una red. Existen dos versiones: **IPv4** (32 bits, decimal puntado: `192.168.0.1`) e **IPv6** (128 bits, hexadecimal: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`), esta última desplegada gradualmente para superar el agotamiento del espacio IPv4.

> [!info] Estructura de las direcciones IP
> IPv4: cuatro **octetos** (grupos de 8 bits) separados por puntos; cada octeto va de 0 a 255 → 32 bits en total. IPv6: notación hexadecimal en **ocho grupos de 16 bits** (cuatro dígitos) separados por dos puntos → 128 bits. La escala: 2³² ≈ 4 294 967 296 IPv4 (poco más de 4 mil millones) vs 2¹²⁸ ≈ 3,4×10³⁸ IPv6 — espacio prácticamente ilimitado.

Mediante `ifconfig` (clásico) podemos verificar nuestra IP:

![[Pasted image 20250821092208.png]]

> [!tip] `ifconfig` → `ip addr`
> `ifconfig` pertenece a `net-tools` y está deprecado; en sistemas modernos puede no existir. El reemplazo es `ip addr show` (o `ip -br a`) del paquete `iproute2` — ver [[Tipos de Interfaces de Red]] y [[Comandos Básicos de Sistema y Red en Linux]]. Las capturas siguientes usan `ifconfig`; la lectura es idéntica.

Vamos a convertir nuestra IP a binario:

```bash
echo "$(echo "obase=2; 192" | bc).168.0.180"
```

> [!tip] Uso de `bc` para conversión numérica
> `bc` (*basic calculator*) es una calculadora de línea de comandos con precisión arbitraria. Dentro de ella, `obase=2` fija la **base de salida** a binario (`ibase` es la de entrada). Es la forma canónica de convertir decimales a binario/hex sin depender de `printf`.

![[Pasted image 20250821092444.png]]

El valor de 192 en binario es `11000000`; lo mismo con todos los octetos:

```bash
echo "$(echo "obase=2; 192" | bc).$(echo "obase=2; 168" | bc).$(echo "obase=2; 0" | bc).$(echo "obase=2; 180" | bc)"
```

![[Pasted image 20250821092931.png]]

Tenemos 4 octetos de 8 bits = 32 bits por dirección. Respecto a IPv4 tenemos `2^32 = 4294967296` direcciones teóricas, pero ya hay más dispositivos humanos conectados que direcciones — por eso existe IPv6 con `2^128`.

> [!info] Escasez de IPv4 y la solución IPv6
> El agotamiento (IANA delegó los últimos bloques /8 a los RIRs en 2011) impulsó coexistencia: **dual-stack** (ambos protocolos a la vez), **tunelización** (6to4, Teredo) y **traducción** (NAT64/DNS64). Además, IPv4 sobrevivió en gran medida gracias a **NAT** (ver [[Tipos de IP y Dirección MAC]]), que multiplica la vida útil del espacio privado a costa de complejidad y de romper el modelo extremo-a-extremo.

### Dónde vive cada cosa en el stack

| Protocolo | Capa OSI / TCP-IP | Por qué |
|---|---|---|
| **ARP** | 2 (a veces 2/3) | Mapea IP→MAC **dentro de un segmento**; no puede viajar por routers |
| **IP (v4/v6)** | 3 / Internet | Enrutado extremo a extremo |
| **ICMP / ICMPv6** | 3 (viaja encapsulado en IP) | Diagnóstico: `ping`, `traceroute`, "destino inalcanzable"; ICMPv6 trae ND (el ARP de IPv6) |
| **TCP / UDP** | 4 / Transporte | Multiplexación por puertos |
| **DNS** | 7 (UDP/53 y TCP/53) | Consultas pequeñas por UDP, transferencias/zonas por TCP |
| **HTTP / HTTPS** | 7 (TLS entre 7 y 4) | Texto sobre TCP/80; TLS envuelve HTTP sobre TCP/443 |
| **TLS/SSL** | 6 según OSI; **entre app y transporte** en la práctica | Cifra el stream TCP completo antes de que lo vea la aplicación |
| **DHCP** | 7 (UDP 67/68) | Configure/lease sobre broadcast (por eso vive "arriba", aunque configura la capa 3) |
| **SSH, FTP, SMTP, SNMP** | 7 | Servicios de aplicación sobre TCP (SNMP además UDP 161/162) |
| **PPP, VLAN (802.1Q)** | 2 | Enlaces y segmentación de enlace |

## Direcciones MAC (OUI y NIC): el otro identificador

La dirección **MAC** es un identificador hexadecimal de **12 dígitos** (**6 bytes / 48 bits**), en notación con dos puntos, asignado a cada interfaz de red para la **capa 2**. Mediente `ifconfig` (o `ip link`) la vemos junto a la IP:

![[Pasted image 20250821093954.png]]

Los primeros **6 dígitos** (digamos **00:0c:29**) identifican al fabricante: el **OUI** (*Organizationally Unique Identifier*), asignado por el comité de registro del **IEEE**. Los seis dígitos restantes los asigna el fabricante al **NIC** (Network Interface Controller) — el número de serie de esa interfaz. Es decir: **3 bytes de OUI + 3 bytes de serie = los 48 bits**. Para buscar fabricante hacen falta al menos los primeros **3 bytes**; herramientas como [[Macchanger Herramienta de Modificación MAC|macchanger]] la visualizan y manipulan.

> [!warning] Suplantación de direcciones MAC (MAC spoofing)
> Aunque se las llama "quemadas" en el hardware (firmware de la NIC), las MAC pueden modificarse a nivel de software en la mayoría de los sistemas. El MAC spoofing se usa para evadir filtros de MAC, ocultar la identidad de un dispositivo o suplantar a otro en la red. La estructura completa (bit U/L local, multicast, spoofing) está en [[Tipos de IP y Dirección MAC]].

Muchas implementaciones usan la MAC para identificar dispositivos en la red, por ejemplo [[Arp Scan Herramienta de Descubrimiento|arp-scan]]:

```bash
sudo arp-scan -I eth0 --local-net
```

![[Pasted image 20250821095020.png]]

Con IP, MAC y fabricante de cada host vivo del segmento. El descubrimiento masivo en LAN usa también [[netdiscover]].

## TCP y UDP a fondo

Protocolos **TCP** (*Transmission Control Protocol*) y **UDP** (*User Datagram Protocol*) son los dos protocolos de transporte de Internet.

| Criterio | **TCP** | **UDP** |
|---|---|---|
| Conexión | **Orientado a conexión**: handshake antes de datos | **Sin conexión**: manda y ya |
| Confianza | Entrega **confiable**: ACK, retransmisión, orden, deduplicación | **Best-effort**: puede perder, duplicar, reordenar — sin avisar |
| Orden de datos | Garantizado (reordenación con nº de secuencia) | Sin garantía |
| Control de flujo | Ventana deslizante (emisor no inunda al receptor) | No |
| Control de congestión | Sí (slow start, cwnd…) | No (la app lo gestiona o sufre) |
| Cabecera | 20 B mínima (hasta 60) | 8 B fijas |
| Overhead | Alto (handshake + ACKs por tramo) | Mínimo |
| Modelo | Byte-stream confiable | Datagramas discretos |
| Cierre | Four-way handshake (FIN/ACK) | No existe |
| Uso típico | Web, SSH, correo, transferencias, bases de datos | DNS, DHCP, VoIP, video, gaming, streaming, SNMP |

### El Three-Way Handshake (TCP)

Procedimiento para establecer la conexión, sincronizando números de **secuencia** (SYN) y de **reconocimiento** (ACK):

1. **SYN**: el cliente envía `SYN` con su **ISN** (*Initial Sequence Number*, num. de secuencia inicial).
2. **SYN-ACK**: el servidor responde con su propio **ISN** (`SYN`) y reconoce el del cliente (`ACK = ISN_cliente + 1`).
3. **ACK**: el cliente reconoce el ISN del servidor (`ACK = ISN_servidor + 1`).

![[Pasted image 20250821100608.png]]

> [!info] Números de secuencia y reconocimiento
> El ISN no es 0: lo elige cada extremo de forma pseudoaleatoria (protección contra spoofing/hijacking de conexiones). Durante la transferencia, cada byte consume un número de secuencia; el ACK confirma "he recibido todo hasta aquí". Si falta un segmento, el receptor reenvía el ACK con el último contiguo recibido (retransmisión por *fast retransmit* o por timeout). El cierre usa el intercambio simétrico **FIN ↔ ACK** (four-way), y puede quedar conexiones en `TIME_WAIT` (2×MSL) — de ahí los puertos "atrapados" en `ss` tras cerrar conexiones.

Wireshark muestra el intercambio exacto: SYN → SYN,ACK → ACK, y después PSH/ACK con datos y ACKs puros — ver [[Wireshark y Tshark]] y [[Tshark]] para capturarlo con `tshark -f 'tcp'`.

### UDP: el transporte mínimo

UDP solo añade 8 bytes (puertos origen/destino, longitud, checksum — opcional en IPv4). Sin handshake, el primer datagrama **ya lleva datos**. Por eso: latencia mínima (VoIP, gaming), tolerancia a pérdida (un paquete de vídeo perdido no bloquea el stream) y facilidad para **broadcast/multicast** (que TCP no soporta). A cambio, todo lo que TCP da gratis (orden, retransmisión, flujo) lo implementa la aplicación si lo necesita (QUIC sobre UDP es el ejemplo moderno: reordena y cifra en la app).

## Puertos comunes

Los puertos pertenecen a la capa 4 y multiplexan servicios dentro de un host (0–65535 por protocolo).

**TCP comunes:**

| Puerto | Servicio | Función |
|---|---|---|
| 21 | **FTP** | Control de transferencia de archivos (el dato va por 20 u otro canal) |
| 22 | **SSH** | Sesiones remotas cifradas (también SCP/SFTP) |
| 23 | **Telnet** | Sesión remota en claro — obsoleto por seguridad |
| 25 | **SMTP** | Envío de correo |
| 53 | **DNS** | Consultas TCP para respuestas grandes / transferencias de zona |
| 80 | **HTTP** | Web en claro |
| 110 | **POP3** | Recepción de correo |
| 143 | **IMAP** | Recepción de correo sincronizada |
| 443 | **HTTPS** | Web con TLS |
| 445 | **SMB** | Compartición de archivos/imprenta (Windows/Unix) |
| 3306 / 5432 | **MySQL / PostgreSQL** | Bases de datos |
| 3389 | **RDP** | Escritorio remoto de Windows |
| 5900 | **VNC** | Escritorio remoto |

**UDP comunes:**

| Puerto | Servicio | Función |
|---|---|---|
| 53 | **DNS** | Consultas normales (el 99% del tráfico DNS) |
| 67/68 | **DHCP** | Asignación de IP y parámetros (67 server, 68 client) |
| 69 | **TFTP** | Transferencia de archivos sin autenticación (config de switches, PXE) |
| 123 | **NTP** | Sincronización de relojes |
| 161/162 | **SNMP** | Administración/monitoreo (162 = traps) |
| 514 | **Syslog** | Envío centralizado de logs |
| 500 / 4500 | **IKE/IPsec** | Llaves de VPN |
| 1900 | **SSDP/UPnP** | Descubrimiento de dispositivos (malo en redes ajenas) |

> [!info] Registro de puertos IANA
> Los **bien conocidos** (0–1023) los asigna la IANA a servicios estándar (requieren privilegios para bindear en Linux: `CAP_NET_BIND_SERVICE` o root — ver [[Linux Capabilities]]). Los **registrados** (1024–49151) son de aplicaciones específicas; los **dinámicos/privados** (49152–65535) los usan los clientes para sus conexiones salientes. En laboratorios, los servicios se mueven a puertos no estándar: por eso un escaneo que solo mira el 80/443 se pierde cosas (ver [[Nmap Guía Completa]]).

## Subnetting: la idea central

**Subnetting** es dividir una red IP en subredes más pequeñas usando **máscaras**: los bits de la máscara en `1` son de **red**, los que están en `0` son de **host**. Una máscara `255.255.255.0` = primeros 3 octetos de red, último de hosts.

**CIDR** (*Classless Inter-Domain Routing*) reemplazó las clases fijas A/B/C: la dirección se escribe con su **prefijo** `/<n>` = cantidad de bits de red. `192.168.1.1/24` es la IP `192.168.1.1` en la red cuya máscara tiene 24 bits en 1 (`11111111.11111111.11111111.00000000` = `255.255.255.0`).

> [!info] Beneficios de CIDR
> - **Eficiencia**: bloques de tamaño real (no /8 enteros para quien necesita 10 IPs).
> - **Tablas de enrutamiento más pequeñas**: agregación de rutas (un /12 en vez de 256 /16).
> - **Escalabilidad**: el crecimiento de Internet no cabría con el esquema de clases.

El detalle completo de cálculo (network ID, broadcast, hosts utilizables, rangos privados, NAT, IPv6) está en [[Tipos de IP y Dirección MAC]]; aquí resolvemos un ejemplo entero paso a paso con la hoja de cálculo de apoyo:

![[Subnetting.xlsx]]

### Clases de direcciones (histórico que aún se usa como referencia)

| Clase | Primer octeto | Máscara por defecto | Rango | Uso original |
|---|---|---|---|---|
| **A** | 1–126 | `255.0.0.0` (/8) | `0.0.0.0`–`126.255.255.255` | Grandes organizaciones (p. ej. `10.52.36.11`) |
| **B** | 128–191 | `255.255.0.0` (/16) | `128.0.0.0`–`191.255.255.255` | Medianas (p. ej. `172.16.52.63`) |
| **C** | 192–223 | `255.255.255.0` (/24) | `192.0.0.0`–`223.255.255.255` | Pequeñas (p. ej. `192.168.123.132`) |
| D / E | 224–255 | — | Multicast / experimentación | — |

> [!warning] Las clases son historia, no regla
> Hoy nadie asigna por clases: se usa CIDR con cualquier prefijo (/22, /26, /28…). Las clases solo sirven para leer diagramas antiguos y para entender de dónde salieron los rangos privados (RFC 1918).

### Rangos privados (RFC 1918)

| Clase lógica | Rango | Prefijo | Tamaño |
|---|---|---|---|
| Grande | `10.0.0.0` – `10.255.255.255` | `/8` | 16 777 216 direcciones |
| Media | `172.16.0.0` – `172.31.255.255` | `/12` | 1 048 576 |
| Pequeña | `192.168.0.0` – `192.168.255.255` | `/16` | 65 536 |

No son enrutables en Internet: su tráfico sale traducido por **NAT** en el router (ver [[Tipos de IP y Dirección MAC]]).

### Ejemplo completo: 192.168.1.0/26

**1. Máscara**: 26 bits de red → `11111111.11111111.11111111.11000000` = **255.255.255.192**.

**2. Hosts**: quedan 32−26 = **6 bits** de host → 2⁶ = 64 direcciones, menos red y broadcast: **62 hosts utilizables** (`2^n − 2`).

**3. Network ID** (AND de IP y máscara en binario):

IP: `11000000.10101000.00000001.00000000`
Máscara: `11111111.11111111.11111111.11000000`
AND: `11000000.10101000.00000001.00000000` = **192.168.1.0**

![[Pasted image 20250821113837.png]]

**4. Broadcast**: bits de host a 1: `11000000.10101000.00000001.00111111` = **192.168.1.63**.

Resultado: subred `192.168.1.0/26`, rango utilizable **192.168.1.1 – 192.168.1.62**, broadcast `192.168.1.63`. (Truco rápido: el "salto" por subred es 256−192 = **64**: subredes en .0, .64, .128, .192.)

### Tabla de referencia rápida de prefijos

| Prefijo | Máscara | Hosts (2^(32−n)−2) | Uso típico |
|---|---|---|---|
| /24 | 255.255.255.0 | 254 | LAN de oficina/casa |
| /25 | 255.255.255.128 | 126 | Dos LAN iguales |
| /26 | 255.255.255.192 | 62 | VLAN pequeña |
| /27 | 255.255.255.224 | 30 | VLAN de servidores |
| /28 | 255.255.255.240 | 14 | DMZ pequeña |
| /30 | 255.255.255.252 | 2 | Enlace punto a punto (router↔router) |
| /32 | 255.255.255.255 | 1 | Una sola interfaz (túneles, loopbacks) |
| /16 | 255.255.0.0 | 65 534 | Red corporativa grande |
| /12 | 255.240.0.0 | 1 048 574 | Rango privado clase B entero |

## Metodología de diagnóstico de red

Cuando "la red no va", se recorre el stack **de abajo hacia arriba** — cada paso descarta un nivel:

| Paso | Qué se verifica | Comando | Interpretación del fallo |
|---|---|---|---|
| 1. Interfaz activa | ¿Existe y está UP? | `ip -br link`, `ip -br a` | `DOWN` → cable/driver; `NO-CARRIER` → cable o switch |
| 2. Dirección local | ¿Tiene IP y máscara correctas? | `ip a show eth0` | Sin IP → DHCP caído o config estática mal |
| 3. Ruta por defecto | ¿Hay gateway? | `ip route`, `ip route get 8.8.8.8` | `default` ausente → sin salida a otras redes |
| 4. Vecino L2 (gateway) | ¿Se resuelve la MAC? | `ip neigh` | Entrada `FAILED`/incompleta → mismo segmento roto o ARP fallando |
| 5. Salto local | ¿Responde el gateway? | `ping -c2 $(ip route | awk '/default/{print $3}')` | Falla dentro del LAN → problema local, no del ISP |
| 6. Salto externo | ¿Hay ruta y hops? | `traceroute 8.8.8.8` (o `tracepath`) | Se corta en un salto X → el problema está en el enlace o router X |
| 7. DNS | ¿Resuelve nombres? | `getent hosts example.com`, `nslookup`, `dig` | IP directa funciona pero nombre no → DNS |
| 8. Servicio | ¿Responde el puerto? | `curl -sv https://example.com`, `nc -zv host 443` | Conectividad OK pero `Connection refused` → servicio caído; `timeout` → firewall/descartado |
| 9. TLS/HTTP | ¿El servicio hace lo que debe? | `openssl s_client`, `curl -sSI` | Certificado vencido, 5xx, redirect… |

```bash
# Cadena corta de diagnóstico (reutilizable)
ip -br a && ip route && ip neigh
ping -c2 127.0.0.1 && ping -c2 $(ip route | awk '/default/{print $3}') && ping -c2 8.8.8.8
getent hosts example.com && curl -sSI https://example.com | head -3
```

> [!tip] `ping` manda ICMP, no TCP
> Un host que no responde a `ping` puede estar perfectamente vivo: muchos firewalls descartan ICMP. Por eso el paso 8 usa la conexión real al puerto del servicio. Regla de diagnóstico: **prueba siempre el protocolo que el usuario usa** — ICMP falla ≠ red caída.

## Errores y confusiones comunes

1. **Confundir MAC con IP**: la MAC (L2) identifica la interfaz **en su segmento** y cambia de router a router; la IP (L3) identifica la interfaz **de extremo a extremo** y se conserva (salvo NAT).
2. **Dar por muerto un host por no responder a ping** — ICMP bloqueado; usar `nmap -Pn` con el puerto real (ver [[Nmap Guía Completa]]).
3. **Clases como si fueran CIDR vigente**: `172.16.0.0/12` es privado completo, pero `172.32.x` es público — el rango privado es solo 172.16–172.31.
4. **Olvidar el −2 de hosts** (red + broadcast) al dimensionar VLANs.
5. **Pensar que `traceroute` "muestra la ruta real completa"**: usa TTL decreciente y respuestas ICMP; hops pueden no responder (`* * *`) sin que el enlace esté roto (firewall), y el camino de vuelta puede ser distinto (asimetría).
6. **Puerto bien conocido ≠ servicio real**: en el laboratorio, SSH puede estar en 2222 — verificar con `nmap -sV` o `ss -tlnp`, no asumir por número.

## Cheatsheet

```bash
# Capas → herramientas
ip link / ip a (L2/L3) · ip route / ping / traceroute (L3) · ss / nc -zv / nmap (L4) · curl / openssl (L7)
# Diagnóstico en orden
ip -br a → ip route → ip neigh → ping gateway → ping 8.8.8.8 → getent hosts → curl puerto
# Números clave
TCP handshake: SYN → SYN/ACK → ACK     ·  FIN/ACK ×2 para cerrar
Privados: 10/8 · 172.16/12 · 192.168/16      ·  hosts = 2^(32−prefijo) − 2
192.168.1.0/26 = máscara .192, salto 64, hosts 1–62, broadcast .63
Puertos: 21 ftp · 22 ssh · 23 telnet · 25 smtp · 53 dns · 67/68 dhcp · 80 http · 110 pop3
         123 ntp · 143 imap · 161 snmp · 443 https · 445 smb · 3306 mysql · 3389 rdp
```

## Relacionadas

- [[Tipos de IP y Dirección MAC]] — matemática de subredes, NAT, IPv6, MAC y DHCP a fondo
- [[Tipos de Interfaces de Red]] — `ip link/addr/route`, tipos de interfaz, MTU
- [[Comandos Básicos de Sistema y Red en Linux]] — `ss`, `ip`, `ping`, `curl`, `nc` en el catálogo diario
- [[Nmap Guía Completa]] — llevar el diagnóstico a reconocimiento de puertos y servicios
- [[Reconocimiento y Escaneo de Redes y Web]] — metodología completa de reconocimiento
- [[Arp Scan Herramienta de Descubrimiento]] y [[netdiscover]] — descubrimiento en capa 2
- [[Wireshark y Tshark]] y [[Tshark]] — ver el handshake y el encapsulado reales
- [[Macchanger Herramienta de Modificación MAC]] — manipular la identidad L2

## Referencias

- `man 7 tcp`, `man 7 udp`, `man 7 ip`, `man 7 arp`, `man 8 ip`, `man 8 traceroute`
- RFC 791 (IPv4), RFC 793 (TCP), RFC 768 (UDP), RFC 1918 (privados), RFC 8200 (IPv6): <https://www.rfc-editor.org/>
- ITU-T X.200 (modelo OSI): <https://www.itu.int/rec/T-REC-X.200>
- Cisco Networking Academy (subnetting y OSI): <https://skillsforall.com/>
- Wireshark Wiki — "Following TCP Streams": <https://wiki.wireshark.org/FollowStream>
