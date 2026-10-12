---
aliases:
  - Tipos de IP y MAC
  - IP Privada y Pública
  - NAT
  - Dirección MAC
  - OUI
  - IPv6
  - DHCP
tags:
  - sys/redes
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Conceptos Fundamentales de Redes]]"
  - "[[Tipos de Interfaces de Red]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
  - "[[Macchanger Herramienta de Modificación MAC]]"
  - "[[Arp Scan Herramienta de Descubrimiento]]"
  - "[[netdiscover]]"
  - "[[Nmap Guía Completa]]"
fuente: "https://www.rfc-editor.org/rfc/rfc1918"
---

# Tipos de IP y Dirección MAC

Todo lo que hay que saber sobre **direcciones**: la estructura de IPv4 y su aritmética de subredes (red, broadcast, hosts), la distinción público/privado con **NAT**, la notación **CIDR**, la estructura de **IPv6** con sus tipos de dirección, y la dirección **MAC** de capa 2 — estructura, OUI, bit local y spoofing. Cierra con los dos flujos que conectan todo: el **lease de DHCP** y la **resolución de DNS**.

> [!info] IP vs MAC: dos capas, dos identidades
> La **IP** (capa 3) es **lógico y configurable**: identifica una interfaz en el enrutado global, cambia al moverse o por DHCP. La **MAC** (capa 2) es **físico y local**: identifica la tarjeta dentro de su segmento y viaja solo de salto en salto (el router reescribe las cabeceras L2, la de L3 se conserva — ver encapsulación en [[Conceptos Fundamentales de Redes]]).

## Dirección IP: público vs privado

Existen dos tipos de direcciones IP: **pública** y **privada**.

### Dirección IP privada

Las direcciones IP privadas son asignadas localmente por nuestro router. Solo se usan dentro de una red privada: no son enrutables en Internet ni accesibles directamente desde fuera, a menos que alguien esté en nuestro mismo segmento de red.

> [!info] Rangos de IP Privadas (RFC 1918)
> | Rango | Prefijo | Tamaño | Nota |
> |---|---|---|---|
> | `10.0.0.0` – `10.255.255.255` | `/8` | 16 777 216 | Una única red enorme |
> | `172.16.0.0` – `172.31.255.255` | `/12` | 1 048 576 | **16** redes /16 contiguas |
> | `192.168.0.0` – `192.168.255.255` | `/16` | 65 536 | **256** redes /16 |
>
> Permiten que las organizaciones tengan redes internas sin agotar direcciones públicas. Otras reservas útiles: `127.0.0.0/8` (loopback), `169.254.0.0/16` (link-local por APIPA: DHCP falló y el host se aut asigna), `100.64.0.0/10` (CGNAT de ISPs), `192.0.0.0/24` y multicast `224.0.0.0/4`.

### Dirección IP pública

Las direcciones IP públicas son **únicas** y se asignan al dispositivo (o router) con acceso directo a Internet; son globalmente visibles y enrutables. Las asignan los **RIR** (ARIN, RIPE NAC, APNIC, LACNIC, AFRINIC) a los ISPs, que las delegan a sus clientes. Pueden ser **dinámicas** (las cambia el ISP periódicamente, vía DHCP) o **estáticas** (fijas, para servidores y servicios que necesitan una dirección constante).

> [!tip] NAT: el multiplicador que salvó a IPv4
> La **NAT** (*Network Address Translation*) permite que muchos dispositivos privados compartan **una** IP pública. Cuando un host interno envía tráfico a Internet, el router reemplaza la IP de origen por su pública y registra la traducción (IP:puerto → IP:puerto) en su tabla; al volver la respuesta, reescribe el destino interno y la reenvía. Por eso los puertos efímeros importan y por eso un escaneo externo solo ve al **router**, nunca a los hosts de dentro.

### Tipos de NAT

| Tipo | Nombre | Qué traduce | Escenario |
|---|---|---|---|
| **SNAT / MASQUERADE** | Source NAT (suvariante masquerade) | IP de **origen** saliente | El caso doméstico/empresa: todos salen con la IP pública del router. `MASQUERADE` es SNAT con IP dinámica (se adapta si cambia la del WAN) |
| **DNAT** | Destination NAT | IP de **destino** entrante | Port-forwarding: el router manda el tráfico a su puerto 8080→ servidor interno 10.0.0.5:80 |
| **PAT / NAT sobrepuertos** | Port Address Translation | Origen + **puerto** | Multiplexa muchos clientes en una IP pública usando puertos distintos (tabla de 4-tuple). Es lo que hace que 50 equipos "quepan" en 1 IP |
| **CGNAT** | Carrier-grade NAT (`100.64.0.0/10`) | ISP hace NAT otra vez | El cliente ni siquiera tiene IP pública real: doble NAT (rompe hosting directo, P2P) |
| **NAT64 / DNS64** | Traducción entre familias | IPv6↔IPv4 | Hosts IPv6 puros acceden a IPv4 |

```bash
# Ver el NAT en Linux (el router de un lab suele ser la máquina con dos interfaces)
sysctl net.ipv4.ip_forward          # 1 = el host actúa como router
iptables -t nat -L -n -v            # reglas SNAT/DNAT (nft list table ip nat en sistemas nuevos)
conntrack -L 2>/dev/null | head     # tabla de conexiones NAT (si conntrack está cargado)
```

> [!warning] NAT y seguridad: no es un firewall
> NAT oculta los hosts internos (por eso un escaneo externo no los ve), pero **no inspecciona ni filtra** por sí mismo: el filtrado real es `iptables`/`nftables`/firewalld. Y los hosts internos salen **todos con la misma IP**: para un incidente forense "quién bajó X" depende de los logs de aplicación o del DHCP (quién tuvo esa IP en ese momento) — ver el flujo DHCP más abajo.

## Estructura de IPv4 y aritmética de subred

IPv4: 32 bits en cuatro octetos decimal-punteado (`192.168.1.180` = `11000000.10101000.00000001.10110100`). La **máscara** separa la porción de **red** (bits en 1) de la de **host** (bits en 0).

### Clases (historia) vs CIDR (realidad)

| | Sistema de clases (histórico, pre-1993) | CIDR (actual, RFC 1519) |
|---|---|---|
| Tamaño de red | Fijo por clase A/B/C (8/16/24 bits) | **Cualquier** prefijo: /12, /22, /26, /28… |
| Desperdicio | Brutal (una universidad con clase A = 16M IPs) | Eficiente: bloques del tamaño exacto |
| Ejemplo | 172.16.x.x "es clase B" → /16 fijo | 172.16.5.0/24: una LAN cualquiera |
| Enrutamiento | Rutas por clase | **Longest-prefix match**: la ruta más específica gana |

Las clases quedaron como **vocabulario heredado**: los rangos privados del RFC 1918 se definen por rangos clase A/B/C, y los examenes/laboratorios aún hablan de "clases". Pero el cálculo real siempre es por prefijo.

### CIDR: notación y equivalencias

| CIDR | Máscara | Bits host | IPs totales | Hosts usables (−2) | Salto entre subredes |
|---|---|---|---|---|---|
| /24 | 255.255.255.0 | 8 | 256 | 254 | 1 en el 4º octeto |
| /25 | 255.255.255.128 | 7 | 128 | 126 | 128 |
| /26 | 255.255.255.192 | 6 | 64 | 62 | 64 |
| /27 | 255.255.255.224 | 5 | 32 | 30 | 32 |
| /28 | 255.255.255.240 | 4 | 16 | 14 | 16 |
| /29 | 255.255.255.248 | 3 | 8 | 6 | 8 |
| /30 | 255.255.255.252 | 2 | 4 | **2** | 4 (enlaces p2p) |
| /31 | 255.255.255.254 | 1 | 2 | 2 (RFC 3021) | 2 |
| /32 | 255.255.255.255 | 0 | 1 | 1 (identifica interfaz) | — |

### Receta de cálculo (cuatro pasos)

Para una IP `A.B.C.D/n`:

1. **Máscara**: n bits en 1 → convertir por octetos. /26 → `11111111.11111111.11111111.11000000` = `255.255.255.192`.
2. **Network ID**: AND bit a bit entre IP y máscara. Truco sin binario: con prefijo ≥ /24, el "salto" es `256 − valor_del_octeto_máscara`; los network IDs son múltiplos del salto: 0, 64, 128, 192… El network ID es el múltiplo **menor o igual** que D.
3. **Broadcast**: network ID con todos los bits de host en 1 → el **siguiente** network ID − 1. Ej.: si el network ID es .0 con salto 64 → broadcast .63.
4. **Hosts utilizables**: de network ID+1 a broadcast−1; total `2^(32−n) − 2`.

**Ejemplo resuelto**: `192.168.1.100/26` → máscara `.192`, salto 64 → network IDs .0, .64, .128, .192 → `100` cae entre 64 y 128 → red **192.168.1.64/26**, rango utilizable **.65–.126**, broadcast **.127**.

**Ejemplo /28**: `10.0.5.20/28` → salto 16 → red `10.0.5.16/28`, hosts `.17`–`.30`, broadcast `.31`, 14 hosts.

> [!tip] La tabla de /25–/30 de memoria
> Los octetos de máscara "de un octeto" valen la pena memorizarlos: `128 192 224 240 248 252 254` para /25…/31. Con el salto (`256 − máscara`) y la regla "el network ID es el múltiplo del salto más cercano por debajo", todo el subnetting mental se vuelve aritmética de secundaria. El flujo completo con AND en binario está desarrollado paso a paso en [[Conceptos Fundamentales de Redes]].

## IPv6: estructura y tipos de dirección

IPv6: **128 bits**, ocho grupos hexadecimales de 16 bits separados por `::` (el `::` comprime un solo bloque de ceros consecutivos):

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
2001:db8:85a3::8a2e:370:7334            ← forma comprimida equivalente
```

`2^128 ≈ 3,4×10^38` direcciones: suficiente para cada dispositivo del mundo con "IP para cada grano de arena" — el problema que IPv4 no resolvía.

### Tipos de direcciones IPv6

| Tipo | Prefijo | Alcance | Descripción |
|---|---|---|---|
| **Unicast global** | `2000::/3` (p. ej. `2001:db8::/32`) | Internet | Equivalente moderno a la IP pública: enrutable globalmente |
| **Link-local** | `fe80::/10` | **Solo el enlace** (segmento físico) | Cada interfaz la genera **sola** al activarse (EUI-64 o privacidad). Obligatoria: la comunicación local y el neighbor discovery la usan. No se enrutan nunca |
| **ULA** (*Unique Local Address*) | `fc00::/7` (`fd00::/8` práctico) | Sitio privado | El "RFC 1918 de IPv6": equivalentes a las IPs privadas, no enrutables globalmente (bit L=1: `fd12:3456::/48`) |
| **Loopback** | `::1/128` | Host | El `127.0.0.1` de IPv6 |
| **Multicast** | `ff00::/8` | Variable | `ff02::1` todos los nodos del enlace, `ff02::2` todos los routers, MLD… |
| **Compatibilidad (legacy)** | `::/96`, `::ffff:0:0/96` | — | IPv4-mapped para stacks duales; `6to4`, Teredo — en desuso frente a NAT64 |

| Diferencia clave con IPv4 | Detalle |
|---|---|
| Sin broadcast | Reemplazado por **multicast** (y anycast para servicios) |
| SLAAC | Los hosts se autoconfiguran: toman el prefijo del router (RA) y arman su IPv6 + derivan la MAC (o usan direcciones de **privacidad**, rotadas — por eso verás varias IPv6 globales en `ip a`) |
| Neighbor Discovery | El "ARP de IPv6" vive en ICMPv6 (no hay ARP): `ip neigh` muestra la tabla ND |
| Seguridad | IPsec incorporado al diseño (uso opcional en la práctica); sin NAT "gratuito" |

```bash
ip -6 addr show wlo1        # ver global (2001:…), link-local (fe80::) y ULA (fd…)
ip -6 route                 # rutas v6 (incluida la default vía RA)
ping6 -c2 ff02::1%wlo1      # ping multicast a todos los nodos del enlace (con %interfaz)
ping -6 2001:4860:4860::8888  # probar salida v6 global
```

## Dirección MAC: estructura y matices

La dirección **MAC** (*Media Access Control*) identifica una interfaz en **capa 2**: 48 bits (6 bytes), 12 dígitos hexadecimales en notación con dos puntos (`d4:25:8b:2d:d8:d6`).

![[Pasted image 20250821093954.png]]

Mediante `ifconfig` (o `ip link`) vemos la MAC de cada interfaz.

- Los primeros **6 dígitos** (**00:0c:29** en el ejemplo clásico de VMware) identifican al fabricante: el **OUI** (*Organizationally Unique Identifier*), asignado por el comité de registro del **IEEE**. Se pueden mirar los primeros **3 bytes** para buscar el fabricante en la base del IEEE.
- Los últimos **6 dígitos** los asigna el fabricante al controlador de red (**NIC**): el número de serie de esa interfaz.

O sea: 24 bits de OUI + 24 bits de serie = los 48 bits. Para buscar fabricante se requieren al menos los primeros **3 bytes** (**6 caracteres**); herramientas como [[Macchanger Herramienta de Modificación MAC|macchanger]] la visualizan y manipulan, y varias implementaciones las usan para identificar dispositivos en la red:

```bash
sudo arp-scan -I eth0 --local-net
```

![[Pasted image 20250821095020.png]]

Con `arp-scan` (o [[netdiscover]]) vemos IP, MAC y fabricante de cada host vivo del segmento — el descubrimiento L2 completo.

### Bit I/G y bit U/L: los dos bits que "no son del fabricante"

Los dos bits menos significativos del **primer byte** tienen significado especial:

| Bit | Nombre | Significado |
|---|---|---|
| bit 0 (LSB del primer octeto) | **I/G** (*Individual/Group*) | 0 = dirección **individual** (normal); 1 = **multicast/grupo** (p. ej. `01:00:5e:…` es multicast IPv4 Ethernet) |
| bit 1 | **U/L** (*Universal/Local*) | 0 = asignada **universalmente** por el fabricante (OUI real); **1 = administrada localmente** (asignada por software) |

Una MAC "localmente administrada" tiene el segundo hex del primer byte con el bit U/L activo: por ejemplo `d4` (11010100) → versión local `d6` (11010110). Por eso al hacer spoofing verás MACs que "no existen" en la base IEEE: **el bit local está activado**. También lo usan software legítimo: VMs (VMware/VirtualBox generan MACs locales), direcciones de privacidad de IPv6 (RFC 7217) y tethering móvil.

> [!warning] Suplantación de direcciones MAC (MAC spoofing)
> Aunque se las llama "quemadas en el hardware" (firmware de la NIC), las MAC pueden modificarse a nivel de software en la mayoría de los sistemas. Usos: evadir filtros de acceso por MAC, ocultar la identidad del dispositivo, suplantar a otro equipo en el LAN (ataques de impersonación, ataques al switch/802.1X). Herramientas: `macchanger` (ver [[Macchanger Herramienta de Modificación MAC]]), `ip link set dev eth0 address XX:XX:XX:XX:XX:XX`, o el propio `ifconfig eth0 hw ether …`. Límites: el spoofing solo afecta al **segmento local** (la identificación a nivel Internet usa la IP), y el tráfico sigue saliendo por la NIC real (switches con port-security, 802.1X o logs DHCP pueden atraparlo).

```bash
ip link show wlo1                       # ver la MAC actual (campo 'link/ether')
sudo ip link set wlo1 down && sudo ip link set wlo1 address 00:11:22:33:44:55 && sudo ip link set wlo1 up
macchanger -r wlo1                      # MAC aleatoria
macchanger -p wlo1                      # restaurar la de fábrica
```

## Flujo DHCP: de encender a tener IP

**DHCP** (*Dynamic Host Configuration Protocol*, UDP 67 server / 68 client) entrega la IP y toda la configuración de red: máscara, gateway, DNS, NTP, dominio, tiempo de lease. El intercambio es el famoso **DORA**:

```
Cliente ── DHCPDISCOVER  (broadcast, "¿hay servidor?")          →
        ← DHCPOFFER     (broadcast: IP ofrecida + parámetros)  
        ── DHCPREQUEST   (broadcast: "acepto esa IP")           →
        ← DHCPACK        (broadcast: lease confirmado + TTL)    
```

| Paso | Mensaje | Detalle |
|---|---|---|
| 1 | **DISCOVER** | Cliente sin IP: manda a `255.255.255.255` (y multicast `224.0.0.11` en v6) buscando servidores |
| 2 | **OFFER** | El servidor reserva la IP y ofrece parámetros (lease time, máscara, gateway…) |
| 3 | **REQUEST** | El cliente acepta la oferta (y si compiten varios servidores, rechaza implícitamente a los demás) |
| 4 | **ACK** | El servidor confirma y activa la reserva en su base |

| Concepto | Significado |
|---|---|
| **Lease** | Tiempo de vigencia (p. ej. 24 h). El cliente renueva con **unicast** a la mitad del lease (REBIND/RENEW) |
| **Reserva/estática** | `host fija { hardware ethernet MAC; option address 10.0.0.50; }` — misma MAC = siempre la misma IP (clave forense: MAC→IP→hora) |
| **Relay (helper)** | Routers reenvían el broadcast DHCP a un servidor en otra red (`ip helper-address`) |
| **Lease file** | `/var/lib/dhcp/dhcpd.leases` (ISC) — **quién tuvo qué IP en qué momento**, el log dorado de incidentes |

```bash
# Cliente
ip a                                    # 169.254.x = DHCP falló (APIPA)
journalctl -u NetworkManager | grep -i dhcp   # historial del lease
dhclient -v eth0                        # forzar renew verboso (si usas dhclient)
nmcli con show "Wired connection 1" | grep -i address   # lease actual (NetworkManager)
# Servidor (auditoría de configuración)
cat /etc/dhcp/dhcpd.conf | grep -A3 host
```

> [!info] DHCP y seguridad
> El protocolo es **de confianza por diseño**: un cliente acepta el primer OFFER que oiga, y no hay autenticación. Un atacante con acceso L2 puede montar un **rogue DHCP server** (o responder más rápido que el legítimo) y entregar gateway/DNS falsos (redirección de todo el tráfico — *evil twin*/MITM). Contramedidas: DHCP snooping en el switch, 802.1X, y ARP estático en lo crítico.

## Flujo DNS: de nombre a IP

**DNS** (UDP/53, TCP/53 para respuestas >512 B o transferencias) resuelve nombres a direcciones — y viceversa (PTR). La consulta típica de un cliente local, con los pasos y cachés que implican cada salto:

| Paso | Quién | Qué pasa | Caché correspondiente |
|---|---|---|---|
| 1 | Navegador/app | ¿Tengo la IP en memoria? | Caché del navegador |
| 2 | libc (getaddrinfo) | ¿El nombre está en `/etc/hosts` o nsswitch lo manda directo? | `/etc/hosts` (precedencia según `/etc/nsswitch.conf`: `hosts: files dns`) |
| 3 | Stub resolver local | Pasárselo al servidor configurado en `/etc/resolv.conf` | Caché del stub (systemd-resolved) |
| 4 | Resolutor recursivo | ¿Tengo el resultado en caché (TTL válido)? | Caché del resolutor (unbound, dnsmasq, el del ISP) |
| 5 | Recursivo → **raíz** (`.`) | "¿quién maneja `.com`?" → delega a los TLD nameservers | — |
| 6 | Recursivo → **TLD** (.com) | "¿quién maneja `example.com`?" → los NS autoritativos del dominio | — |
| 7 | Recursivo → **NS autoritativo** | Respuesta final con el registro A/AAAA (y CNAME, MX, TXT… resueltos en cadena) | — |
| 8 | Respuesta hacia arriba | Cada nivel cachea según el **TTL** del registro y responde al cliente | Se guardan todas las capas |

```bash
getent hosts example.com        # resolución como la ve el sistema (con nsswitch)
nslookup example.com            # consulta básica (usa el resolutor configurado)
dig example.com A +short        # consulta directa a un nameserver
dig @1.1.1.1 example.com A      # preguntar a un servidor concreto
dig example.com +trace          # ver la cascada raíz → TLD → autoritativo
dig -x 8.8.8.8                  # PTR inverso (IP → nombre)
resolvectl status                # estado de systemd-resolved (caché, upstream)
```

| Registro | Función |
|---|---|
| `A` / `AAAA` | Nombre → IPv4 / IPv6 |
| `CNAME` | Alias canónico (apunta a otro nombre) |
| `MX` | Servidores de correo del dominio |
| `NS` | Nameservers autoritativos |
| `TXT` | Texto libre (SPF, DKIM, verificaciones) |
| `PTR` | Inverso: IP → nombre (zona `in-addr.arpa` / `ip6.arpa`) |

> [!tip] DNS en el diagnóstico
> "No va la web pero el ping a IP sí" es casi siempre DNS (paso 7 de la metodología de [[Conceptos Fundamentales de Redes]]). Comparar respuestas entre resolutores (`dig @8.8.8.8` vs `@1.1.1.1` vs el del ISP) separa fallo local de fallo upstream; `dig +trace` diagnostica delegaciones rotas. En laboratorios ofensivos, el DNS también es superficie de ataque (cache poisoning, DNS rebinding, tunneling) — ver [[Reconocimiento y Escaneo de Redes y Web]].

## Cheatsheet

```bash
# Ver y calcular
ip a · ip -br a · ip route        # IP local, máscara, gateway
# Privados: 10/8 · 172.16/12 · 192.168/16 · link-local 169.254/16 · CGNAT 100.64/10
# Subnetting: salto = 256 − máscara · red = múltiplo del salto ≤ IP · broadcast = red + salto − 1
#            hosts = 2^(32−n) − 2 · /30 = 2 hosts (p2p) · /26 = 62 · /28 = 14
# IPv6: 2000::/3 global · fe80::/10 link-local · fd00::/7 ULA · ::1 loopback · ff00::/8 multicast
ip -6 a · ping -6 host · ping6 ff02::1%eth0
# MAC: 48 bits = OUI(24) + serie(24) · bit I/G (bit0) · bit U/L local (bit1) · multicast 01:…
ip link show · ip link set eth0 address XX:XX:XX:XX:XX:XX · macchanger -r eth0
# DHCP: DORA (DISCOVER→OFFER→REQUEST→ACK) · UDP 67/68 · leases en /var/lib/dhcp/dhcpd.leases
# DNS: hosts → resolutor → raíz → TLD → autoritativo · dig @servidor nombre +short · dig -x ip
```

## Relacionadas

- [[Conceptos Fundamentales de Redes]] — OSI, encapsulado, TCP/UDP, subnetting paso a paso, diagnóstico
- [[Tipos de Interfaces de Red]] — interfaces, `ip link/addr/route`, MTU, `/sys/class/net`
- [[Comandos Básicos de Sistema y Red en Linux]] — `ip`, `ss`, `ping`, `curl` en el catálogo
- [[Macchanger Herramienta de Modificación MAC]] — visualización y manipulación de MAC
- [[Arp Scan Herramienta de Descubrimiento]] y [[netdiscover]] — descubrimiento L2 con OUI
- [[Nmap Guía Completa]] — escaneo sobre redes calculadas aquí
- [[Reconocimiento y Escaneo de Redes y Web]] — reconocimiento completo (incluye DNS)

## Referencias

- `man 7 inet`, `man 7 ipv6`, `man 7 arp`, `man 8 dhclient`, `man 1 dig`
- RFC 1918 (direcciones privadas): <https://www.rfc-editor.org/rfc/rfc1918>
- RFC 4291 (arquitectura de direccionamiento IPv6): <https://www.rfc-editor.org/rfc/rfc4291>
- RFC 2131 (DHCP): <https://www.rfc-editor.org/rfc/rfc2131>
- IEEE OUI lookup: <https://standards-oui.ieee.org/>
- ArchWiki — DHCP / DNS: <https://wiki.archlinux.org/title/Dynamic_Host_Configuration_Protocol>
