---
tags:
  - sistemas
  - redes
  - interfaces-red
tipo: concepto
estado: terminado
---

# Tipos de Interfaces de Red

Una **interfaz de red** (*network interface*) es el punto por donde el kernel envía y recibe tráfico. Existe una interfaz **física** por cada tarjeta de red (Ethernet, wifi) y muchas **virtuales** creadas por software: loopback, bridges, túneles, VLANs, pares veth de contenedores. Este note cubre los tipos, los flags de estado, el modelo de nomenclatura (incluida la nomenclatura predecible de systemd), la inspección vía `ip` y `/sys/class/net`, y lo básico de MTU.

> [!info] El modelo del kernel: todo es una interfaz
> El kernel no distingue "hardware" de "software" en la capa de red: una bridge o un túnel WireGuard es una interfaz con la misma estructura interna que una Ethernet (`net_device`). Por eso `ip link` lista juntas a `lo`, `eth0`, `docker0` y `wg0`: el stack de red (IPv4/IPv6, iptables/nftables, sockets) las trata uniformemente. Los **namespaces de red** (`ip netns`) dan un paso más: instancias completas y aisladas de este árbol de interfaces, que es como funcionan los contenedores.

## Catálogo de tipos de interfaz

| Tipo | Nombre típico | Origen | Función | Uso común |
|---|---|---|---|---|
| **Loopback** | `lo` | Kernel (siempre) | Retroalimentación local: todo lo mandado a 127.0.0.1/::1 lo procesa el propio host sin tocar cable | Servicios locales, pruebas, IPC |
| **Ethernet** | `eth0`, `ens33`, `enp3s0` | Hardware (driver) | Red cableada | LAN |
| **Wi-Fi** | `wlan0`, `wlp2s0`, `wlo1` | Hardware (driver) | Red inalámbrica | LAN inalámbrica, modo monitor (ver abajo) |
| **Dummy** | `dummy0` | Software | Interfaz virtual "de mentira": descarta todo lo que se le manda, siempre UP | Pruebas, IPs de conveniencia sin hardware |
| **Bridge** | `br0`, `docker0`, `virbr0` | Software | **Switch L2** en software: agrupa varias interfaces en un mismo dominio de broadcast | Virtualización, Docker, libvirt |
| **Veth** | `veth0`/`veth1`, `eth0@if2` | Software | **Par** de interfaces conectadas por dentro: lo que entra en una sale por la otra — el cable virtual | Conectar contenedores/VMs a bridges y namespaces |
| **TUN** | `tun0` | Software | Túnel **L3**: trabaja con paquetes IP | VPN (OpenVPN, WireGuard) |
| **TAP** | `tap0` | Software | Túnel **L2**: trabaja con tramas Ethernet (como si fuera una Ethernet) | VPN con tramas, QEMU/KVM networking |
| **VLAN** | `eth0.10`, `enp3s0.100` | Software (802.1Q) | Sub-interfaz con **tag VLAN**: varias redes lógicas sobre un mismo cable | Segmentación de red |
| **VXLAN** | `vxlan0` | Software | Túnel L2 sobre UDP (VNI de 24 bits) | Redes de datacenter, overlays de Kubernetes |
| **WireGuard** | `wg0` | Software | Interfaz de VPN moderna (crypto-routing) | VPN |
| **MACVLAN/MACVTAP** | `macvlan0` | Software | Múltiples MACs sobre una interfaz física | Contenedores con IP propia en la misma red |
| **Bond / Team** | `bond0` | Software | Agregación de enlaces (LACP): dos cables como uno (redundancia/throughput) | Servidores con HA de red |
| **PPP / PDPP** | `ppp0` | Software | Enlace punto a punto (autenticación) | ADSL antiguo, algunas VPN |

> [!tip] Reconocer el tipo a partir del nombre
> El prefijo revela la clase: `en*` Ethernet cableada, `wl*` wifi, `eth*`/`wlan*` nombres clásicos (kernel renombramiento), `lo` loopback, `docker0`/`virbr0` bridges de Docker/libvirt, `tun*`/`tap*` túneles, `*.<vlan>` subinterfaz con tag. `ip -br link` y `ethtool -i eth0` (driver) lo confirman.

## Interfaz Loopback (`lo`)

Interfaz **especial**, presente en todos los sistemas Unix/Linux, **sin ningún dispositivo físico asociado**. Toda la tráfico que se envía a sus direcciones lo procesa el sistema operativo sin salir de la máquina.

> [!info] Direcciones loopback
> En IPv4: `127.0.0.1/8` (todo el rango 127.0.0.0/8 es loopback, no solo `.1`). En IPv6: `::1/128`. El kernel trata el rango 127/8 entero como local: cualquier 127.x.x.x responde a ping sin configuración.

Usos principales:

- **Pruebas internas y diagnósticos**: verificar que el stack de red funciona (`ping -c1 127.0.0.1`) sin dependencia de hardware.
- **Comunicación interna entre procesos**: servicios locales se anclan a `127.0.0.1` (bases de datos, cachés, APIs de desarrollo) sin exponerse a la red.
- **Desarrollo y pruebas de software**: servidores locales, mocks, automatización.
- **Simulación de redes**: combinado con namespaces y bridges permite montar topologías completas en un host.

> [!tip] `lo` y seguridad
> En ciberseguridad, `lo` permite probar exploits/payloads contra servicios locales sin exponerlos, y es clave en análisis de malware (comunicación con "servidores" simulados). En hardening: un servicio que **solo** debe escuchar en `lo` (`127.0.0.1`) está aislado de la red externa por diseño; si lo ves en `0.0.0.0`, algo anda mal (ver `ss -tlnp`).

## Interfaz "normal": Ethernet y Wi-Fi

Este tipo no tiene nombre fijo: varía según el hardware y el sistema. Es la interfaz creada cuando hay una tarjeta Wi-Fi o Ethernet.

> [!info] Nomenclatura: clásica vs predecible (systemd)
> - **Clásica** (kernel legacy): `eth0`, `eth1`… para Ethernet y `wlan0`… para wifi. Asignada por **orden de detección** — inestable entre reinicios (hoy pueden invertirse).
> - **Predecible** (systemd/`udev`, por defecto en distros modernas): nombres basados en la **topología física**: `enp3s0` (PCI, bus 3, slot 0), `wlp2s0` (wifi PCI), `ens33` (slot en BIOS enumerado), `enx78e7d1ea46da` (USB, por MAC). Estables entre reinicios.
> - **Renombrar a mano**: `ip link set ens33 name eth0` (efímero) o por `udev`/NetworkManager (`/etc/systemd/network/*.link`, `nmcli connection modify … interface-name`). Reglas udev en `/etc/udev/rules.d/70-persistent-net.rules` (clásico) o drop-ins `.link` (moderno).
> - Los **nombres por MAC** (`enx…`) estabilizan USB: el nombre depende del dispositivo físico, no del puerto.

Funciones principales: conexión a LAN y WAN, acceso a recursos compartidos (SMB, NFS), comunicación entre dispositivos, acceso a servicios y desarrollo con conectividad.

> [!warning] Seguridad de las interfaces físicas
> Son el principal punto de entrada/salida de ataques: corresponde configurar firewalls, segmentación, VPNs y monitoreo de tráfico. En laboratorios ofensivos son la interfaz de trabajo para reconocimiento y escaneo (ver [[Nmap Guía Completa]], [[Arp Scan Herramienta de Descubrimiento]], [[netdiscover]]).

## `ip link`: capa 2 — estado, flags, MTU

```bash
ip link                     # lista todas las interfaces
ip -br link                 # resumido (name, state, MAC)
ip link show eth0           # detalle de una
ip link set eth0 up         # activar    (down: apagar)
ip link set eth0 mtu 1400   # cambiar MTU
ip link set eth0 name lan0  # renombrar (debe estar abajo)
```

Salida real de `ip -br link` en un host:

```
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
wlo1             UP             d4:25:8b:2d:d8:d6 <BROADCAST,MULTICAST,UP,LOWER_UP>
docker0          DOWN           4a:00:5b:f0:54:aa <NO-CARRIER,BROADCAST,MULTICAST,UP>
```

### Flags de estado: qué significa cada uno

| Flag | Significado |
|---|---|
| `UP` | El administrador la activó (`ip link set up`): el kernel puede usarla |
| `LOWER_UP` | El **enlace físico** está activo (cable conectado, señal wifi) — operstate `up` |
| `RUNNING` | Disponible para usar (equivalente a LOWER_UP en la práctica) |
| `NO-CARRIER` | Sin portadora: el cable está fuera o el switch no da enlace — la interfaz puede estar `UP` pero **no transmite** (caso `docker0` arriba) |
| `PROMISC` | Modo **promiscuo**: captura **todas** las tramas del segmento, no solo las destinadas a su MAC. Normal en bridges; sospechoso en hosts de usuario (posible sniffing — ver [[Wireshark y Tshark]]) |
| `ALLMULTI` | Acepta todos los paquetes multicast |
| `MULTICAST` | Soporta multicast (habilitado por defecto en Ethernet) |
| `BROADCAST` | Soporta broadcast |
| `LOOPBACK` | Interfaz de retorno |
| `MULTICAST`/`DYNAMIC` | `DYNAMIC`: interfaz creada dinámicamente (p. ej. PPP) |

Estado administrativo vs operacional: `UP` es la **intención** del admin; `LOWER_UP`/`RUNNING` es la **realidad del enlace**. La columna de estado (`UNKNOWN` en `lo`, `UP`/`DOWN` en `-br`) es el *operstate* del kernel: `up`, `down`, `dormant`, `error`.

## `ip addr`: capa 3 — direcciones

```bash
ip addr show                # todas las direcciones
ip addr show eth0           # de una interfaz
ip a add 192.168.1.50/24 dev eth0     # añadir IP (efímero: no sobrevive reboot)
ip a del 192.168.1.50/24 dev eth0     # quitar
ip a flush dev eth0          # quitar todas (¡con cuidado!)
```

- Cada interfaz puede tener **varias direcciones** (IPv4 y varias IPv6: global, link-local).
- La IP lleva **prefijo** (`/24`): la máscara es un atributo de la dirección, no de la interfaz.
- La configuración hecha con `ip` es **efímera** (se pierde al reboot): la persistencia la da NetworkManager (`nmcli`), systemd-networkd (`/etc/systemd/network/*.network`) o netplan (Ubuntu).
- Direccionamiento de IPv6: `fe80::…/64` (link-local) lo asigna el kernel solo al activar la interfaz; la global llega por SLAAC o DHCPv6 (ver [[Tipos de IP y Dirección MAC]]).

## `ip route`: capa 3 — el mapa de decisiones

```bash
ip route                      # tabla de rutas IPv4
ip -6 route                   # IPv6
ip route show default         # la ruta por defecto (gateway)
ip route get 8.8.8.8          # ¿por qué ruta/interface saldría ese destino?
ip route add 10.10.10.0/24 via 192.168.1.1 dev eth0   # ruta estática (efímera)
ip route del 10.10.10.0/24
```

```
default via 192.168.1.1 dev wlo1 proto dhcp metric 100
192.168.1.0/24 dev wlo1 proto kernel scope link src 192.168.18.34
```

Cómo lee el kernel una ruta: **prefijo más largo gana** (longest-prefix match). `via` = siguiente salto (gateway), `dev` = interfaz de salida, `src` = IP origen de las conexiones nuevas, `metric` = coste (menor = preferida entre rutas iguales), `proto dhcp` = quién la instaló. `ip route get` es la herramienta de diagnóstico definitiva: responde exactamente qué decisión tomaría el kernel (incluye el resultado del lookup de fib).

## `/sys/class/net`: el kernel expone las interfaces

Cada interfaz es un directorio en sysfs (ver [[Estructura de Directorios de Linux]]) con atributos legibles desde scripts **sin `ip`**:

```bash
ls /sys/class/net                     # lista de interfaces
cat /sys/class/net/eth0/address       # MAC
cat /sys/class/net/eth0/mtu           # MTU actual
cat /sys/class/net/eth0/flags         # flags en hexadecimal
cat /sys/class/net/eth0/operstate     # up / down / dormant
cat /sys/class/net/eth0/speed         # 1000 (Mb/s, Ethernet)
cat /sys/class/net/eth0/carrier       # 1 = hay portadora
ls /sys/class/net/eth0/statistics/    # rx_bytes, tx_bytes, errors, drops…
```

`ip link`/`ethtool` leen exactamente estos archivos; escribir en algunos (p. ej. `/sys/class/net/eth0/mtu`) equivale al comando `ip link set`.

## Wireless: modos de la interfaz wifi

Una interfaz wifi puede operar en varios modos (`iw dev <iface> set type <modo>`):

| Modo | Comportamiento |
|---|---|
| `managed` (por defecto) | Conecta a puntos de acceso como cliente normal (NetworkManager) |
| `monitor` | Captura **todos** los frames 802.11 del aire incluidos los management (beacons, probes, handshakes 4-way) — base de airecracking y de [[Wireshark y Tshark]] inalámbrico |
| `ad-hoc` | Red peer-to-peer sin AP (IBSS) |
| `mesh` | Malla 802.11s |
| `master` | La interfaz actúa como **punto de acceso** (hostapd) |

```bash
iw dev                    # info de la interfaz wifi (sustituye a iwconfig)
iw dev wlo1 link          # conectada a qué BSSID, señal, bitrate
iw dev wlo1 scan | grep -E 'SSID|signal'   # escaneo de APs cercanos
iwconfig wlo1 mode monitor   # (wireless-tools, legacy; preferir iw)
```

> [!warning] Modo monitor: requisitos y legalidad
> Activar `monitor` requiere controlar la interfaz (sudo) y driver que lo soporte; no toda wifi lo permite. Capturar tráfico **propio o autorizado** es una habilidad de auditoría; capturar tráfico ajeno sin permiso es ilegal en la mayoría de jurisdicciones.

## MTU: el tamaño máximo de trama

La **MTU** (*Maximum Transmission Unit*) es el payload máximo que la interfaz admite por paquete/trama — **1500 B** por defecto en Ethernet (trama 1518 con cabeceras), 9000 con *jumbo frames*, 1500 típico en wifi, 1420–1480 comunes en VPN/túneles (por la cabecera extra que encapsula).

| Situación | Ajuste habitual |
|---|---|
| VPN (WireGuard 80 B de overhead, IPsec) | Bajar la MTU de la interfaz wg0/ipsec (p. ej. 1420) para evitar fragmentación |
| PPPoE (ADSL) | 1492 |
| `ping` grande falla pero pequeño funciona | PMTU blocked: `ping -M do -s 1472 host` para probar; ICMP "fragmentation needed" filtrado por algún firewall |
| Contenedores/Docker MTU raro | Fijar MTU del bridge y veths (1500 o 1450 con overlay) |

```bash
ip link set dev eth0 mtu 1400            # cambiar
ping -M do -s 1472 -c2 8.8.8.8           # probar MTU real sin fragmentar (1472+28=1500)
ethtool -k eth0 | grep -i mtu            # offloads (GSO/TSO) relacionados
```

> [!info] Fragmentación vs PMTUD
> Si un paquete excede la MTU de un salto intermedio y tiene el flag DF (*Don't Fragment*), el router lo descarta y responde ICMP tipo 3 código 4 ("fragmentation needed"). El emisor reduce la MTU para ese destino (**Path MTU Discovery**). Cuando firewalls filtran ese ICMP, la PMTUD "se rompe": conexiones que envían paquetes grandes se cuelgan — síntoma clásico de "SSH entra pero scp grande no va".

## Inspección y diagnóstico: la rutina

```bash
ip -br a                          # estado + IPs de un vistazo
ip -br link                      # estado + MAC + flags
ip route                          # ¿hay default? ¿por dónde?
ip route get 1.1.1.1             # decisión exacta del kernel
ip neigh                          # tabla ARP/ND: vecinos L2 conocidos
ethtool eth0                      # velocidades, negociación, cable
ethtool -S eth0 | grep -i err     # contadores de errores del driver
cat /sys/class/net/eth0/statistics/rx_errors   # errores por interfaz
dmesg | grep -i link             # eventos de enlace (cable, velocidad)
tcpdump -ni eth0 icmp            # ver el tráfico en la interfaz (o [[Wireshark y Tshark]])
```

## Errores comunes

1. **`Cannot find device "eth0"`** — el nombre real es otro (`ens33`, `enp3s0`, `wlo1`): `ip -br link` antes de nada.
2. **Interfaz `UP` pero `NO-CARRIER`** — cable fuera o puerto del switch muerto: `ethtool eth0` (Link detected: no).
3. **`ip a add` funciona y desaparece al reboot** — la config por `ip` es efímera; persistir con NetworkManager/networkd/netplan.
4. **Cambio de IP "solo funciona con `sudo`"** — `ip addr`/`ip link set` sobre interfaces ajenas requiere CAP_NET_ADMIN (root).
5. **MTU malo con VPN** — tráfico pequeño pasa, grande se cuelga: bajar MTU o revisar filtrado de ICMP (PMTUD).
6. **Renombrar interfaz en caliente rompe reglas udev/NetworkManager** — bajarla antes (`ip link set X down`) y actualizar la config persistente.
7. **PROMISC en un host normal** — no es un error, pero indica captura activa (sniffer o bridge); verificar con `tcpdump`/`ss` quién escucha.

## Cheatsheet

```bash
ip -br a · ip -br link · ip route · ip route get 8.8.8.8 · ip neigh
ip link set eth0 up|down|mtu 1400|name lan0      ip a add 10.0.0.5/24 dev eth0
iw dev · iw dev wlan0 scan · iwconfig (legacy)
/sys/class/net/eth0/{address,mtu,operstate,flags,statistics/}
ethtool eth0 · ethtool -S eth0 | grep -i err · dmesg | grep -i link

# Tipos: lo (loopback) · eth/wl (hardware) · dummy (virtual trivial) · br (switch L2)
#        veth (par de cable virtual) · tun(L3)/tap(L2) (túneles) · eth0.10 (VLAN tag 10)
# Estados: UP (admin) · LOWER_UP/RUNNING (enlace real) · NO-CARRIER (sin cable) · PROMISC (captura todo)
# MTU 1500 Ethernet; bajarla con VPN/PPPoE; probar: ping -M do -s 1472
```

## Relacionadas

- [[Conceptos Fundamentales de Redes]] — OSI, TCP/UDP, diagnóstico por capas
- [[Tipos de IP y Dirección MAC]] — direcciones, subredes, DHCP, DNS
- [[Comandos Básicos de Sistema y Red en Linux]] — `ip`/`ss` en el catálogo diario
- [[Estructura de Directorios de Linux]] — sysfs y `/sys/class/net`
- [[Macchanger Herramienta de Modificación MAC]] — cambiar la MAC (identidad L2)
- [[Arp Scan Herramienta de Descubrimiento]] y [[netdiscover]] — vecinos L2 via ARP
- [[Wireshark y Tshark]] — capturar en interfaces físicas y virtuales
- [[Nmap Guía Completa]] — escaneo desde interfaces específicas (`-e`)

## Referencias

- `man 8 ip`, `man 8 ip-link`, `man 7 netdevice`, `man 8 ethtool`, `man 8 iw`
- systemd predictable interface names: <https://www.freedesktop.org/software/systemd/man/systemd.net-naming-scheme.html>
- ArchWiki — Network interface: <https://wiki.archlinux.org/title/Network_interface>
- kernel.org — Network topologies (veth/bridge): <https://www.kernel.org/doc/html/latest/networking/>
