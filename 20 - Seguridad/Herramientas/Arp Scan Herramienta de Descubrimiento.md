---
aliases: ["arp-scan", "ARP Scan", "Escaneo ARP", "Descubrimiento de Hosts ARP"]
tags: [tool/arp-scan]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Nmap Guía Completa]]"
  - "[[netdiscover]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Macchanger Herramienta de Modificación MAC]]"
  - "[[Conceptos Fundamentales de Redes]]"
  - "[[Tipos de IP y Dirección MAC]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
fuente: "https://github.com/royhills/arp-scan"
---

# arp-scan: Herramienta de Descubrimiento de Hosts y Escaneo de Red

**arp-scan** descubre hosts vivos de una red local enviando solicitudes **ARP** (capa 2) por broadcast y registrando quién responde. Es la primera herramienta que se corre tras caer en una LAN: en milisegundos devuelve IP + MAC + fabricante de todo el segmento, **incluso para hosts que filtran ICMP**. No cruza routers: trabaja sobre el mismo dominio de broadcast.

> [!info] ¿Por qué arp-scan funciona contra firewalls?
> `ping` viaja en capa 3 (ICMP) y muchos firewalls lo descartan. ARP viaja en **capa 2 (enlace de datos)** y es *requisito* para que la red funcione: sin ARP no hay resolución IP→MAC y el host no puede hablar con nadie. Por eso casi ningún host bloquea ARP — bloquearlo es romperse a sí mismo. Además, arp-scan obtiene la **dirección MAC** (y de ahí el fabricante), algo que ICMP jamás dará.

## Instalación

```bash
# Kali / Debian
sudo apt update && sudo apt install arp-scan

# Ver versión y ayuda
arp-scan --version
arp-scan --help
```

> [!warning] Requiere privilegios
> arp-scan abre sockets crudos con libpcap: **necesita root** (`sudo`) o capabilities (`cap_net_raw`). Sin ello: `pcap_lookupdev: no suitable device found` o `Permission denied`.

## Cómo funciona (mecánica)

1. Lee la interfaz/IP propia → calcula la red (`-l`) o usa la que le pases (`192.168.1.0/24`).
2. Envía un **ARP request** (`who-has 192.168.1.7? tell 192.168.1.5`) a la dirección de broadcast `ff:ff:ff:ff:ff:ff`.
3. Cada host vivo responde con un **ARP reply** unicast: `192.168.1.7 is at aa:bb:cc:dd:ee:ff`.
4. Imime cada reply como una línea `IP  MAC  FABRICANTE` (el fabricante sale de la base OUI instalada con el paquete).
5. Al final, estadísticas: `N packets received, N packets dropped`.

Todo ocurre **dentro de tu segmento L2**: un router no reenvía ARP, por lo que `arp-scan 10.0.0.0/8` fuera de tu subred solo "escanea" lo que alcanza tu broadcast (y llena la tabla CAM del switch de intentos inútiles). Para redes remotas → [[Nmap Guía Completa]] con `-sn`.

## Tabla completa de opciones

### Escaneo y objetivos

| Opción | Significado | Ejemplo |
|---|---|---|
| `-l`, `--localnet` | Escanea la red local de la interfaz (IP+ máscara) | `sudo arp-scan -l` |
| `-F`, `--file=<f>` | Objetivos desde fichero (IP, rangos o CIDR por línea) | `sudo arp-scan -F hosts.txt` |
| (posicional) | Rangos/CIDR directamente | `sudo arp-scan 192.168.1.0/24 10.0.0.1-10.0.0.50` |
| `-I`, `--interface=<if>` | Interfaz a usar (obligatorio si hay varias) | `sudo arp-scan -I eth0 -l` |
| `-x`, `--exclude=<h1\|h2>` | Excluye hosts del escaneo | `--exclude=192.168.1.1` |
| `--excludefile=<f>` | Exclusiones desde fichero | `--excludefile=noescanear.txt` |

### Velocidad y reintentos

| Opción | Significado | Ejemplo |
|---|---|---|
| `-r`, `--retry=<n>` | Intentos por host sin respuesta (defecto ~5) | `--retry=3` |
| `-t`, `--timeout=<ms>` | Espera de la respuesta por host | `--timeout=500` |
| `-T`, `--interval=<ms>` | **Espera entre paquetes** (no es timeout) | `-T 10` |
| `--backoff=<f>` | Multiplicador de espera entre reintentos | `--backoff=1.5` |
| `--randomize` | Orden aleatorio de objetivos (menos "escaneo lineal") | `--randomize` |
| `--seed=<n>` | Semilla para reproducir el orden aleatorio | `--seed 42` |
| `--pps=<n>` | Paquetes por segundo objetivo | `--pps=1000` |
| `--gpu` | Aceleración por GPU (versiones recientes, con soporte del driver) | `--gpu` |

### Salida

| Opción | Significado | Ejemplo |
|---|---|---|
| `-v`, `--verbose` | Salida detallada (paquetes, interfaces, descartes) | `-v` |
| `--rtt` | Muestra el tiempo de ida y vuelta por host | `--rtt` |
| `--resolve` | Resuelve y muestra nombres DNS de los hosts | `--resolve` |
| `--timestamp` | Prefija cada línea con marca temporal | `--timestamp` |
| `--arping` | Tras encontrar un host, lo verifica además con arping | `--arping` |
| `-V`, `--version` | Versión | `arp-scan -V` |
| `-h`, `--help` | Ayuda completa | `arp-scan -h` |

> [!warning] Corrección respecto a notas previas
> - `-x` **no** "muestra respuestas ARP no esperadas": `-x/--exclude` **excluye** hosts. Las respuestas no solicitadas se ven con `-v`.
> - `-T` es el **intervalo entre paquetes**, no un timeout.
> - `-g` (MACs duplicadas) **no figura en el man de arp-scan 1.9.x**; verificar siempre con `arp-scan --help` en tu versión antes de usar un flag poco común.

## Interpretación de la salida

```text
Interface: eth0, type: EN10MB, MAC: 08:00:27:ab:cd:ef, own IP: 192.168.1.10
Starting arp-scan 1.9.2: 256 hosts on subnet 192.168.1.0/24 (192.168.1.1 → 192.168.1.255)

192.168.1.1	00:1e:8f:12:34:56	TP-LINK Technologies
192.168.1.20	3c:22:fb:aa:bb:cc	Apple, Inc.
192.168.1.50	b8:27:eb:11:22:33	Raspberry Pi Foundation
192.168.1.66	08:00:27:44:55:66	CAD Computer Classroom

7 packets received, 0 packets dropped
```

| Elemento | Qué significa |
|---|---|
| Cabecera `Interface: ... own IP` | Interfaz elegida y tu IP — **verifícala**: si dice la interfaz equivocada, usa `-I` |
| `192.168.1.1  00:1e:8f...  TP-LINK` | IP viva + MAC + fabricante (base OUI) |
| Última columna `(unknown)` | MAC sin entrada OUI (vanidad/privada o base desactualizada) |
| `(duplicate)` | Misma MAC contestando a varias IPs (alias/virtualización, o spoof) |
| `N packets received, N dropped` | Dropped > 0 ⇒ paquetes perdidos: sube `--timeout`/`--retry` |
| Línea `... not found` / sin respuesta | Host apagado, filtrando, o fuera de tu L2 |

> [!tip] Extraer solo las IPs (para encadenar a Nmap)
> ```bash
> sudo arp-scan -l | awk '/^[0-9]+\./ {print $1}' > hosts.txt
> nmap -sS -sV -iL hosts.txt -oA descubiertos
> ```
> La primera columna es la IP; el `awk` evita las cabeceras.

## arp-scan vs nmap vs ping vs netdiscover

| Criterio | **arp-scan** | `nmap -sn` | `ping -b` | [[netdiscover]] |
|---|---|---|---|---|
| Capa | 2 (ARP) | 3/2 (ICMP/TCP/ARP) | 3 (ICMP) | 2 (ARP, passivo+activo) |
| Cruza routers | **No** | Sí | Sí | No |
| Ve hosts con firewall ICMP | **Sí** | Parcial (puede usar ARP si hay privilegios) | No | Sí |
| Devuelve MAC + fabricante | **Sí** | Solo en red local con privilegios | No | Sí |
| Velocidad en /24 | < 1 s | 2-10 s | lento (serial) | media |
| Modo pasivo | No | No (`-sn` es activo) | No | **Sí** (sniffing ARP) |
| Requiere root | Sí | Para ARP/ICMP raw: sí | No | Sí |
| Uso típico | **Primer vistazo de la LAN** | Descubrimiento multi-red + puertos | ¿responde este host? | Pentest Wi-Fi / rangos grandes |

**Regla práctica:** estás en la misma subred → `arp-scan -l`; necesitas varias redes o detalle → [[Nmap Guía Completa]]; ya hay que mirar qué habla → `nmap -sS -sV` sobre los resultados.

## Recetas: los 10 comandos reales

```bash
# 1. Descubrimiento rápido de toda la red local
sudo arp-scan -l

# 2. Elegir interfaz explícitamente (máquinas con VPN/docker)
sudo arp-scan -I eth0 --localnet

# 3. Escanear un rango concreto con menos ruido y más rápido
sudo arp-scan 192.168.1.0/24 --retry=2 --timeout=300

# 4. Excluir el gateway y hosts conocidos (menos logueo)
sudo arp-scan -l --exclude=192.168.1.1 --excludefile=no.txt

# 5. Salida con RTT y verbose (diagnóstico de latencia/estabilidad)
sudo arp-scan -l -v --rtt

# 6. Desde fichero de objetivos generado por otra fase
sudo arp-scan -F objetivos.txt -I eth0

# 7. Orden aleatorio (evita el patrón "barre 1→254" de los IDS)
sudo arp-scan -l --randomize --seed 1234

# 8. Pipeline a Nmap
sudo arp-scan -l | awk '/^[0-9]+\./ {print $1}' | xargs nmap -sV --top-ports 100

# 9. Barrido rápido de muchas hosts (ajustar tasa)
sudo arp-scan 10.0.0.0/24 --pps=1000 -T 1

# 10. Verificar versión/soporte (antes de confiar en un flag)
arp-scan --version && arp-scan --help
```

**Flag por flag de la receta 3:** `--retry=2` baja de 5 a 2 los intentos por host muerto (menos espera total), `--timeout=300` acepta respuesta en 300 ms en vez de 1000 → el /24 entero termina en ~1-2 s sin perder hosts LAN.

## Detección y contramedidas (defensor)

| Señal | Cómo la ven los defensores | Herramienta típica |
|---|---|---|
| Ráfaga de ARP requests a todo el subnet desde una MAC | Patrón de **ARP sweep** (todos los `who-has` en <1 s) | `arpwatch`, Snort/Suricata (`ET ARP Scan`), Zeek `arp.log` |
| Tabla CAM del switch llenándose | El switch ve miles de IPs asociadas a tu puerto | Port-security / DHCP snooping |
| Solicitudes a IPs sin actividad | Correlación con baselines de tráfico ARP | SIEM (ARP request rate por host) |
| Respuestas con MAC desconocida | Nueva MAC en la red → alerta de rogue device | `arpwatch`, WIPS/NAC |
| MAC aleatorizada | Bit U/L (locally administered) activo en el primer octeto | Ver [[Macchanger Herramienta de Modificación MAC]] |

> [!tip] Mitigaciones reales
> - **ARP estático / dynamic ARP inspection (DAI)** en switches Cisco: invalida sweep y spoofing.
> - **Segmentación por VLAN/VRF**: acota el dominio de broadcast y por tanto el alcance de la herramienta.
> - **Client isolation en Wi-Fi**: los clientes no pueden hacer ARP entre sí → arp-scan solo verá el AP.
> - **802.1X/NAC**: identifica el dispositivo real aunque la MAC esté randomizada.

## Errores comunes (mensaje → arreglo)

1. **`pcap_lookupdev: no suitable device found`** — no hay interfaz IPv4 utilizable o falta privilegio.
   → `sudo arp-scan -l` y, si hay varias, `-I eth0`.
2. **`Permission denied` / `You need to be root`** — sockets crudos.
   → Ejecutar con `sudo`, o `sudo setcap cap_net_raw+ep $(readlink -f $(command -v arp-scan))`.
3. **`Can't open device eth0`** — nombre de interfaz erróneo.
   → `ip -br link` para ver nombres reales (`eth0`, `enp3s0`, `wlan0`).
4. **Escanéa la interfaz equivocada** (sale tu IP de la VPN/docker en la cabecera).
   → Forzar: `sudo arp-scan -I eth0 -l`.
5. **Cero hosts en Wi-Fi** — aislamiento de cliente, o estás en una subred que no es L2 tuya.
   → Probar `-I wlan0 -l` y comprobar IP/máscara con `ip -br addr`.
6. **`(unknown)` en el fabricante** — la OUI no está en la base instalada.
   → Actualizar el paquete (`apt update && apt install --reinstall arp-scan`) o cargar base propia (ver `--help`).
7. **Hosts "fantasma" con MAC repetida** — virtualización (VMs con MAC virtual) o ARP spoofing previo.
   → Cruzar con `ip neigh show` y con la MAC real del switch/`nmap --append-trace`.
8. **Muy lento en rangos grandes** — demasiados reintentos/timeout.
   → `--retry=1 --timeout=200` o `--pps`/`--gpu`.

## Cheatsheet — los 10 comandos del día a día

```bash
sudo arp-scan -l
sudo arp-scan -I eth0 --localnet
sudo arp-scan 192.168.1.0/24
sudo arp-scan -l --exclude=192.168.1.1
sudo arp-scan -l -v --rtt
sudo arp-scan -F hosts.txt
sudo arp-scan -l --randomize
sudo arp-scan -l --retry=2 --timeout=300
sudo arp-scan -l | awk '/^[0-9]+\./ {print $1}'
arp-scan --help
```

| Tarea | Comando |
|---|---|
| Vistazo inicial de la LAN | `sudo arp-scan -l` |
| Rango específico | `sudo arp-scan 192.168.1.0/24` |
| Excluir gateway | `sudo arp-scan -l --exclude=192.168.1.1` |
| Interfaz concreta | `sudo arp-scan -I eth0 -l` |
| Entrada desde fichero | `sudo arp-scan -F objetivos.txt` |
| Salida con latencia | `sudo arp-scan -l --rtt` |
| Rápido | `sudo arp-scan -l --retry=2 --timeout=300` |
| Anti-patrón | `sudo arp-scan -l --randomize` |
| IPs limpias a Nmap | `... \| awk '/^[0-9]+\./ {print $1}'` |
| Ayuda | `arp-scan --help` |

## Referencias

- [Repositorio oficial (royhills/arp-scan)](https://github.com/royhills/arp-scan) — README, man pages y CHANGELOG
- Man page: `man arp-scan` en Kali/Debian
- HackTricks — *Network Enumeration* (arp-scan como primer paso)
- [[Reconocimiento y Escaneo de Redes y Web]] — dónde encaja en la metodología

## Relacionadas

- [[Nmap Guía Completa]] — siguiente paso: puertos, versiones y scripts
- [[netdiscover]] — alternativa ARP con modo pasivo (ideal en Wi-Fi)
- [[Conceptos Fundamentales de Redes]] / [[Tipos de IP y Dirección MAC]] — teoría de capas y OUI
- [[Macchanger Herramienta de Modificación MAC]] — cómo se ve una MAC randomizada al otro lado
- [[Comandos Básicos de Sistema y Red en Linux]] — `ip`, `arp`, `ip neigh` para verificar a mano
