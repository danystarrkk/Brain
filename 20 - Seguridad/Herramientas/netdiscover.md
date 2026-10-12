---
aliases: ["netdiscover", "Netdiscover", "Descubrimiento ARP"]
tags:
  - tool/netdiscover
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Arp Scan Herramienta de Descubrimiento]]"
  - "[[Nmap Guía Completa]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Wireshark y Tshark]]"
  - "[[Macchanger Herramienta de Modificación MAC]]"
fuente: "https://github.com/alexxy/netdiscover"
---

# Netdiscover

**Netdiscover** es un descubrimiento de hosts basado en **ARP** (Address Resolution Protocol): sonda o escucha el tráfico de capa 2 para mapear qué IPs de la subred responden, con su MAC y fabricante (OUI). Es la herramienta de descubrimiento por defecto cuando se trabaja **desde dentro de una red local** — por ejemplo, con una tarjeta inalámbrica en modo monitor conectada a un WiFi — porque no necesita ruta IP ni que el host responda a ICMP.

> [!info] Por qué ARP y no ping
> ARP opera en la capa 2 (enlace de datos) del modelo OSI: solo funciona **dentro de la misma subred** (broadcast de dominio de broadcast). Ventaja decisiva para el reconocimiento: un host con firewall que bloquea **todo** ICMP y todos los puertos TCP/UDP **no puede dejar de responder ARP** — es requisito para comunicarse en su propia LAN. Por eso, donde un ping scan de [[Nmap Guía Completa]] da cero hosts, un ARP scan da la lista completa. Limitación: inútil fuera de tu subred (para eso está ICMP/TCP de Nmap).

## Netdiscover vs sus alternativas

| Herramienta | Mecanismo | Modo | Mejor para |
|---|---|---|---|
| **netdiscover** | ARP activo o pasivo | Interactivo (tabla que crece en pantalla) | Redes inalámbricas en modo monitor; vigilancia pasiva prolongada |
| **arp-scan** (`Arp Scan Herramienta de Descubrimiento`) | ARP activo | Batch (salida final) | Rendimiento y scripting; escaneo rápido de rangos grandes |
| `nmap -sn` (ARP en LAN) | ARP automático en subred local | Batch | Integrarse en el flujo de Nmap |
| `ip neigh` / `arp -a` | Tabla ARP del sistema | Pasivo (lo que ya se ha visto) | Consulta instantánea sin generar tráfico |

> [!tip] La elección en la práctica
> ¿Estás en modo monitor inalámbrico (sin IP asignada)? → **netdiscover** (es el único que funciona sin IP). ¿Tienes IP y quieres una lista para piping? → **arp-scan**. ¿Vas a escanear puertos después? → directamente `nmap -sn` y sigue con Nmap sobre la lista.

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux** | Preinstalado (`netdiscover -V`) |
| Debian/Ubuntu/Parrot | `sudo apt install netdiscover` |
| Compilar | `git clone https://github.com/alexxy/netdiscover && ./configure && make && sudo make install` |

Requiere **privilegios root** (necesita enviar/recibir paquetes ARP crudos).

## Los tres modos de descubrimiento

| Modo | Bandera | Qué hace | Genera tráfico propio |
|---|---|---|---|
| **Rango activo** | `-r <CIDR>` | Sonda secuencialmente cada IP del rango con ARP who-has | Sí — ARP requests |
| **Pasivo** | `-p` | Solo escucha el tráfico ARP existente (requests y replies ajenos) | **No** |
| **Lista** | `-l <archivo>` | Lee rangos/hosts de un archivo | Sí |

### Modo activo por rango (el uso estándar)

```bash
sudo netdiscover -r 192.168.1.0/24
```

Salida interactiva — una fila que crece en vivo:

```
 Current MAC: 00:0c:29:aa:bb:cc (VMware, Inc.)
   IP            Count  MAC          Len   Hostname
 192.168.1.1       1   00:1a:2b:3c:4d:5e   60    router.local
 192.168.1.23      1   00:50:56:aa:bb:cc   60    win10-pc
```

| Campo | Significado |
|---|---|
| `Current MAC` | Última MAC vista con el fabricante resuelto (OUI) |
| `IP` | Dirección que respondió |
| `Count` | Número de respuestas ARP recibidas de ese host (↑ = host activo o ruidoso) |
| `MAC` | Dirección física del host |
| `Len` | Tamaño del paquete ARP (60 = Ethernet estándar; valores extraños pueden indicar VLAN/tunnels) |
| `Hostname` | Nombre si el host lo anunció (Nbns/LLMNR o respuesta con nombre) |

### Interpretar el fabricante (OUI) — inteligencia del mapa

| Fabricante visto | Lectura |
|---|---|
| VMware / VirtualBox / QEMU / Xen | Hipervisores — hosts de laboratorio o infraestructura; contar VMs ≠ contar hosts físicos |
| Cisco / Ubiquiti / MikroTik / Aruba | Infraestructura de red — el `1` de la subred suele ser el gateway |
| Apple / Samsung / Intel | Dispositivos de usuario (móviles, portátiles) — confirma que es una red "viva" |
| Realtek / Intel Corporate (adaptadores) | Máquinas físicas — objetivo clásico de post-explotación |
| MAC local administrada (`x2`, `x6`, `xA`, `xE` en el primer byte) | Virtualización, Docker, MAC randomizada (Android/iOS/Windows "private MAC") — marca hosts que evitan tracking |

El fabricante no es identidad — es contexto: permite separar infraestructura, hosts de usuario y ruido virtual antes de priorizar objetivos.

### Modo pasivo

```bash
sudo netdiscover -p
```

No envía **ni un paquete**: solo registra lo que la red ya está diciendo (los who-has del router hacia clientes, mDNS, etc.). Cero exposición como escáner, pero descubre **solo** hosts activos en el momento — en redes dormidas (noches, oficinas vacías) la lista puede ser mínima. Es la opción para reconocimiento sigiloso en redes cableadas o WiFi con mucho tráfico.

**Qué tráfico ve el modo pasivo** (y por qué la lista crece sola):

| Tráfico ARP | Ejemplo práctico |
|---|---|
| Who-has de router→cliente | Casi toda actividad saliente del cliente dispara la resolución de su MAC |
| Who-has de cliente→router | El cliente busca su gateway al despertar/conectarse |
| Gratuitis (respuestas sin petición) | Gratuituitous ARP de alta disponibilidad (VRRP), DHCP servers, algunos VoIP |

### Modo lista

```bash
sudo netdiscover -l rangos.txt
```

Un host/rango por línea (`192.168.1.0/24`, `10.0.0.5`) — útil para combinar varios segmentos en una ejecución.

## Referencia completa de flags

### Escaneo

| Flag | Significado | Ejemplo |
|---|---|---|
| `-i, --interface` | Interfaz a usar (por defecto detecta la primera) | `-i wlan0mon` |
| `-r, --range` | Rango CIDR a escanear (modo activo) | `-r 192.168.0.0/16` |
| `-l, --file` | Archivo con rangos/hosts | `-l targets.txt` |
| `-p, --passive` | Modo pasivo (solo escucha) | `-p` |
| `-f, --fast` | Modo rápido: scan intensivo con paquetes más densos | `-f` |
| `-s, --sleep` | Milisegundos de espera entre sondeos (mayor = más sigiloso) | `-s 2000` |
| `-c, --count` | Número de paquetes a enviar por IP | `-c 3` |
| `-L, --listen` | Modo loop: continúa escuchando/sondeando indefinidamente | `-L` |

### Red y fabricantes

| Flag | Significado | Ejemplo |
|---|---|---|
| `-d, --db` | Archivo de base de datos de fabricantes (reemplaza al OUI interno) | `-d oui.txt` |
| `-g, --gateway` | IP del gateway — necesario en modo pasivo para identificarlo | `-g 192.168.1.1` |
| `-m, --mask` | Archivo de máscaras personalizado | `-m masks.txt` |
| `-F, --regexp` | Filtra fabricantes por expresión regular | `-F "Cisco|Ubiquiti"` |

### Salida

| Flag | Significado | Ejemplo |
|---|---|---|
| `-P, --parseable` | Salida parseable: IP TAB MAC TAB fabricante (una línea por host) | `-P > hosts.tsv` |
| `-N, --noheader` | Sin cabecera en la salida | `-PN` |
| `-v, --verbose` | Modo verboso | `-v` |
| `-h / -V` | Ayuda / versión | |

## Flujos de trabajo reales

### 1. Reconocimiento inicial desde WiFi (modo monitor sin IP)

```bash
# Tarjeta en modo monitor (ver Aircrack-ng), canal fijo, SIN IP:
sudo netdiscover -i wlan0mon -r 192.168.50.0/24 -c 2 -s 500
```

Antes de lanzarlo hay que averiguar la subred — no hay DHCP en modo monitor. La subred se deduce así:

| Técnica | Cómo |
|---|---|
| Airodump + clientes | Ver las IPs de los clientes en las columnas cuando obtienen DHCP |
| Tramas del beacon | Algunos APs anuncian el gateway en DHCP offers visibles en la captura |
| DNS/NetBIOS escuchados | Los nombres de hosts resueltos delatan rangos habituales |
| Rangos por defecto | Probar `192.168.1.0/24`, `192.168.0.0/24`, `10.0.0.0/24`, `172.16.0.0/24` con `-c 1` |

### 2. Vigilancia pasiva prolongada

```bash
sudo netdiscover -p -g 192.168.1.1 -L
```

Dejarlo corriendo en una ventana durante la fase de reconocimiento: cada host que hable queda registrado con fabricante — un mapa de la red que no deja rastro de escáner.

### 3. De netdiscover a Nmap (pipeline típico)

```bash
sudo netdiscover -r 192.168.1.0/24 -P -N | awk '{print $1}' > hosts.txt
nmap -sV --top-ports 100 -iL hosts.txt -oA reconocimiento
```

La columna 1 del formato parseable es la IP — con `awk` sale la lista limpia para Nmap (o para [[THC Hydra]] o [[CrackMapExec Guía de Uso]]).

## Cuándo NO usar ARP (y qué usar)

| Escenario | Problema | Alternativa |
|---|---|---|
| Objetivo fuera de tu subred | ARP no cruza routers | `nmap -sn <rango>` con ICMP/TCP ping |
| Redes IPv6 puras | ARP no existe (NDP lo reemplaza) | `nmap -sn -6` o `ndisc6` |
| Solo necesitas hosts que respondan TCP | ARP responde hosts sin servicios TCP útiles | `nmap -sn -PS80,443` |
| Escaneo de rango enorme (routers, /8) | ARP a millones de IPs = inundación de broadcast | [[Masscan]] (con precaución) / segmentar |

## Detección y contramedidas (visión del defensor)

| Señal | Cómo se ve | Nota |
|---|---|---|
| ARP scan activo | Ráfaga de who-has a decenas/hundreds de IPs desde una sola MAC en poco tiempo | Detección trivial con cualquier sensor de capa 2; los switches con DAI (*Dynamic ARP Inspection*) pueden bloquear o alertar |
| MAC inexistente sondeando | ARP requests desde una MAC que no tiene IP asignada (típico en modo monitor) | Firma clásica de herramienta de auditoría/ataque |
| ARP spoofing asociado | Respuestas gratuitis (unsolicited ARP replies) | netdiscover **no** spoofea, pero coexiste en el mismo repertorio de ataques — correlacionar |
| Modo pasivo | **Ninguna** señal externa | El único riesgo es el tráfico ARP que el host ya genera por su actividad normal |

> [!warning] ARP no se autentica (salvo con DAI/DHCP snooping)
> El protocolo confía en lo que escucha: cualquier host puede responder "esa IP soy yo" (ARP spoofing — ver [[Wireshark y Tshark]] para analizarlo). Netdiscover solo lee y sondea, pero el mismo canal permite ataques; las defensas son DAI en el switch + estáticos en hosts críticos.

## Errores comunes

| Mensaje / síntoma | Causa | Fix |
|---|---|---|
| `You must be root to use this option` | Sin privilegios | `sudo netdiscover ...` |
| `Could not find a valid interface` / no detecta interfaz | Interfaz inexistente o sin carrier | `-i <interfaz>` con nombre exacto (`ip link`) |
| Modo pasivo no descubre nada | Red muy silenciosa o interfaz equivocada | Esperar actividad, usar `-g` para marcar el gateway, o caer a modo activo |
| El escaneo activo no ve hosts en WiFi | Interfaz en modo monitor **sin IP** y fuera de la subred anunciada, o canal equivocado | Fijar el canal correcto y confirmar la subred del gateway |
| Salida gigante en red corporativa | ARP a toda una /16 | Dividir el rango; `-c 1` y `-s` alto para reducir ruido |
| `--json`/`-t`/`-T`/`-A` no existen | Flags que circulan en guías antiguas: netdiscover **no** tiene `-j`/`--json`, ni timeout (`-t`/`-T`), ni modo adaptativo (`-A`) | Usar `-P` (parseable) + `awk`, y regular la velocidad con `-s` y `-c` |

## Cheatsheet

```bash
sudo netdiscover                          # interfaz por defecto, modo interactivo
sudo netdiscover -r 192.168.1.0/24        # escaneo activo del rango
sudo netdiscover -r 10.0.0.0/24 -c 2 -s 500   # menos agresivo
sudo netdiscover -p                       # pasivo (solo escucha)
sudo netdiscover -p -g 192.168.1.1 -L     # pasivo continuo con gateway marcado
sudo netdiscover -r 192.168.1.0/24 -P -N | awk '{print $1}'   # lista de IPs para Nmap
sudo netdiscover -l targets.txt           # escanear rangos de archivo
```

## Referencias

- Repositorio oficial: <https://github.com/alexxy/netdiscover>
- Man page: `man netdiscover`
- Espejo del repositorio original en BackTrack: jcameron/netdiscover
- Conceptos de reconocimiento: [[Reconocimiento y Escaneo de Redes y Web]]
- Tráfico ARP para análisis: [[Wireshark y Tshark]]

## Relacionadas

- [[Arp Scan Herramienta de Descubrimiento]] — el hermano batch, más rápido y scriptable
- [[Nmap Guía Completa]] — el escaneo de puertos que sigue al descubrimiento
- [[Macchanger Herramienta de Modificación MAC]] — la otra cara del ARP (cambiar la MAC propia)
- [[Masscan]] — para rangos demasiado grandes para ARP
- [[THC Hydra]] / [[CrackMapExec Guía de Uso]] — qué hacer con la lista de hosts viva
