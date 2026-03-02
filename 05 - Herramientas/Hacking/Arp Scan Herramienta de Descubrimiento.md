---
aliases:
  - arp-scan
tags:
  - hacking/herramientas
  - redes/comandos
estado: 🟢 Terminado
---
# arp-scan: Herramienta de Descubrimiento de Hosts y Escaneo de Red

**arp-scan** es una herramienta de descubrimiento de hosts y escaneo de red que utiliza paquetes ARP (Address Resolution Protocol). A diferencia de herramientas como `ping`, `arp-scan` puede detectar hosts incluso cuando tienen filtros de firewall activos, ya que ARP es fundamental para el funcionamiento de las redes LAN.

> [!info] ¿Por qué arp-scan es efectivo contra firewalls locales?
> `arp-scan` opera en la Capa 2 (Enlace de Datos) del modelo OSI, utilizando el protocolo ARP para resolver direcciones IP a direcciones MAC dentro de un segmento de red local. Los firewalls a menudo filtran el tráfico de Capa 3 (como ICMP utilizado por `ping`), pero rara vez bloquean el tráfico ARP esencial para la comunicación dentro de la LAN. Esto permite a `arp-scan` identificar hosts activos incluso si están configurados para ignorar solicitudes `ping`.

## Tabla de Opciones Principales de arp-scan

| Opción | Descripción | Ejemplo |
|---|---|---|
| `-l` | Escanea la red local automáticamente. | `arp-scan -l` |
| `--localnet` | Equivalente a `-l`, escanea la red local. | `arp-scan --localnet` |
| `-I` | Especifica la interfaz de red a usar. | `arp-scan -I eth0 192.168.1.0/24` |
| `-g` | No muestra las direcciones MAC duplicadas. | `arp-scan -g 192.168.1.0/24` |
| `-r` | Número de reintentos (retries) por host. | `arp-scan -r 5 192.168.1.0/24` |
| `-t` | Tiempo de espera (timeout) en milisegundos para cada host. | `arp-scan -t 500 192.168.1.0/24` |
| `-T` | Tiempo de espera (timeout) entre paquetes en milisegundos. | `arp-scan -T 10 192.168.1.0/24` |
| `--retry` | Número total de intentos (por defecto: 5). | `arp-scan --retry=3 192.168.1.0/24` |
| `--timeout` | Tiempo de espera (timeout) total en milisegundos. | `arp-scan --timeout=10000 192.168.1.0/24` |
| `--backoff` | Factor de retroceso (backoff) para el tiempo de espera. | `arp-scan --backoff=1.5 192.168.1.0/24` |
| `-V` | Muestra la versión del programa. | `arp-scan -V` |
| `-h` o `--help` | Muestra la ayuda de uso. | `arp-scan --help` |
| `-v` | Modo verboso (muestra más información). | `arp-scan -v 192.168.1.0/24` |
| `-F` | Lee hosts desde un archivo. | `arp-scan -F hosts.txt` |
| `-x` | Muestra respuestas ARP no esperadas. | `arp-scan -x 192.168.1.0/24` |
