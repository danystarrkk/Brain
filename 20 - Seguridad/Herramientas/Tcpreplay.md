---
tags:
  - ciberseguridad
  - herramientas
  - tcpreplay
tipo: herramienta
estado: terminado
---

# Tcpreplay: Suite de Reproducción y Reescritura de Tráfico

**Tcpreplay** es una suite de cuatro herramientas que **reinyectan capturas PCAP en una interfaz de red**: `tcpreplay` (envía el archivo tal cual o reescrito), `tcpreplay-edit` (envía aplicando reescrituras al vuelo), `tcpprep` (marca qué paquetes son de cliente y de servidor) y `tcprewrite` (reescribe paquetes a un archivo nuevo sin tocar la red). Es la forma estándar de regenerar tráfico de laboratorio: reproducir una sesión capturada contra un IDS/IPS para verificar detecciones, poblar un entorno con tráfico realista o probar reglas de firewall con paquetes bit-exactos.

> [!info] Por qué replay y no "generar tráfico con un script"
> Un generador (Scapy, scripts python) **resintetiza** paquetes: nunca reproducirá con exactitud los TCP timestamps, tamaños de ventana, patrones de ACK y payloads reales de una sesión legítima. Tcpreplay inyecta los bytes **tal como se capturaron** (con sus sumas de verificación, sus flags y su timing), lo que es exactamente lo que necesita un detector de firmas para dar un veredicto útil. La contrapartida: no puedes cambiar el contenido sin `tcprewrite`/`tcpreplay-edit`, y reinyectar MACs/IPs ajenas en una red real es una operación delicada (ver "Detección y contramedidas").

## La suite: qué hace cada binario

| Binario | Función | Entrada → Salida |
|---|---|---|
| `tcpreplay` | Envía un PCAP por una interfaz, con control de velocidad/timing | PCAP → interfaz de red |
| `tcpreplay-edit` | Igual que tcpreplay **pero aplicando opciones de tcprewrite** durante el envío (sin generar archivo intermedio) | PCAP → interfaz (con reescritura al vuelo) |
| `tcpprep` | Genera un **archivo de caché** que clasifica paquetes en cliente/servidor (necesario para replay bidireccional con `-I/--intf2` y para `-e/--endpoints`) | PCAP → caché binario |
| `tcprewrite` | Reescribe direcciones, puertos, checksums, longitudes… a un PCAP de salida, sin enviar nada | PCAP → PCAP nuevo |

> [!tip] Flujo de trabajo clásico
> `tcpprep` (crear caché) → `tcprewrite -e cliente:servidor` (cambiar endpoints al pcap) → `tcpreplay -i eth-lab` (inyectar). O en una sola pasada: `tcpreplay-edit -c cacho.pcap -e 10.0.0.1:10.0.0.2 -i eth0 cap.pcap`. Esta cadena es la que permite, por ejemplo, capturar un ataque real contra tu máquina y **reproducirlo contra un Snort/Suricata aislado** desde otra IP.

## Instalación

```bash
# Kali / Debian / Ubuntu (incluye los 4 binarios)
sudo apt update && sudo apt install tcpreplay

# Verificar versión — los flags cambiaron entre la serie 3.x y la 4.x
tcpreplay --version
tcpreplay -h

# Compilar desde fuente (últimas funciones, p. ej. 4.1+)
sudo apt install build-essential libpcap-dev libtool autoconf automake
git clone https://github.com/sduffy/tcpreplay.git && cd tcpreplay
./autogen.sh && ./configure && make && sudo make install
```

> [!warning] Deriva 3.x → 4.x (afecta a notas y scripts viejos)
> En la serie **4.x**: `-D` es `--topspeed`, `-C` es `--crcfix`, `-p` es `--pps` (paquetes/segundo), `-M` es `--multiplier`, `-l` es `--loop`. En notas antiguas (y en mucha documentación circulante) estos mismos símbolos se asignaban a funciones distintas ("`-C` veces de repetición", "`-M` pps"…). **Confirma siempre con `tcpreplay -h` en tu instalación.** Las funciones de repetición se hacen con `-l` (loop) y de velocidad con `-D`/`-p`/`-r`/`-M`.

## Referencia de flags

### `tcpreplay` — envío

| Flag | Significado | Ejemplo |
|---|---|---|
| `-i, --intf=<nombre>` | Interfaz de salida (obligatoria) | `-i eth0` |
| `-I, --intf2=<nombre>` | Segunda interfaz para replay **bidireccional** (requiere caché `-c`) | `-I eth1` |
| `-c, --cachefile=<archivo>` | Archivo de caché generado por `tcpprep` | `-c cache.pcap` |
| `-l, --loop=<n>` | Repetir el archivo n veces (`0` o `--loop` sin valor = infinito hasta Ctrl+C) | `-l 10` |
| `-L, --limit=<n>` | Enviar como máximo n paquetes | `-L 500` |
| `-X, --skip=<n>` | Saltar los primeros n paquetes | `-X 100` |
| `-D, --topspeed` | Lo más rápido posible (sin esperas) | `-D` |
| `-p, --pps=<n>` | Limitar a n **paquetes por segundo** | `-p 1000` |
| `-r, --rate=<mbps>` | Limitar a n **Mbit/s** | `-r 100` |
| `-M, --multiplier=<x>` | Multiplicar la velocidad original por x (`2.0` = doble rápido, `0.5` = mitad) | `-M 0.5` |
| `-e, --sleep=<tipo>` | Método de espera entre paquetes (`select`, `nanosleep`, `ioport`, `rdtsc`, `gettimeofday`…). `nanosleep` = más preciso, `rdtsc`/`--preload-pcap` = más rápido | `-e nanosleep` |
| `-T, --timer=<tipo>` | Método de medición del tiempo de envío (`select`, `gettimeofday`, `rdtsc`, `gtod-count`, `tsc-count`) | `-T rdtsc` |
| `-t, --enable-mulithread` | Envío multihilo (nota: la opción se escribe así, sin la "i" de "multi", en el propio binario) | `-t` |
| `--singlethread` | Fuerza envío de un solo hilo | `--singlethread` |
| `--preload-pcap` | Carga el PCAP completo en RAM antes de enviar (requisito para rates máximos; en 3.x era `-F`) | `--preload-pcap` |
| `-C, --crcfix` | Verificar y **recalcular** checksums IP/TCP/UDP antes de enviar (imprescindible si reescribiste el pcap) | `-C` |
| `-s, --stats=<seg>` | Imprimir estadísticas cada n segundos durante el replay | `-s 5` |
| `-d, --dbstats` | Estadísticas del archivo de captura | `-d` |
| `-v, --verbose` | Salida detallada (imprime cada paquete) | `-v` |
| `-q, --quiet` | Silencioso (solo errores y resumen final) | `-q` |
| `-h, --help` / `-V, --version` | Ayuda / versión | — |

### `tcpprep` — clasificación cliente/servidor para el caché

| Flag | Significado | Ejemplo |
|---|---|---|
| `-i, --pcap-file=<entrada>` | PCAP de entrada | `-i cap.pcap` |
| `-o, --cache-file=<salida>` | Archivo de caché a crear | `-o cache.pcap` |
| `-a, --auto=<tipo>` | Decisión automática: `mac`, `cidr`, `port`, `seed` | `-a cidr` |
| `-c, --cidr=<lista>` | CIDRs de **cliente** (el resto = servidor): `10.0.0.0/8,172.16.0.0/12` | `-c "10.0.0.0/8"` |
| `-p, --port=<lista>[:lista2]` | Puertos que marcan al cliente/servidor | `-p 1024-65535:80` |
| `-r, --regex=<regex>` | Regex sobre el payload del paquete | `-r "^GET /"` |
| `-s, --mac=<mac>` | MAC considerada cliente | `-s 00:11:22:33:44:55` |
| `-F, --fixcsum` | Corregir checksums al escribir | `-F` |
| `-v, --verbose` | Detalle | `-v` |

### `tcprewrite` — reescritura (sin enviar)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-i, --infile=<pcap>` | Entrada | `-i cap.pcap` |
| `-o, --outfile=<pcap>` | Salida | `-o nuevo.pcap` |
| `-c, --cachefile=<caché>` | Caché de tcpprep (requerido por `--endpoints`) | `-c cache.pcap` |
| `-e, --endpoints=<origen:destino>` | Reescribe IPs de origen y destino de todos los paquetes | `-e 10.1.1.10:10.2.2.20` |
| `--portmap=<lista>` | Remapea puertos (`80:8888`, rangos `2000-2999:80`) | `--portmap=22:2222` |
| `--mac=<mac>` | Fija MAC origen+destino | `--mac=00:de:ad:be:ef:00` |
| `--seed=<n>` | Randomiza direcciones IP de forma **determinística** (mismo seed = mismo resultado) | `--seed=42` |
| `--srcipmap / --dstipmap=<mapa>` | Mapeo de rangos de IP origen/destino | `--srcipmap=10.0.0.0/8:192.168.0.0/16` |
| `--fixcsum` | Recalcula checksums tras la reescritura | `--fixcsum` |
| `--fixlen=<pad\|trunc\|none>` | Normaliza longitud del paquete (rellena o trunca) | `--fixlen=pad` |
| `--skipbadsum` | Envía igual los paquetes con checksum inválido (para probar receptores) | `--skipbadsum` |
| `--mtu=<bytes>` / `--caplen=<bytes>` | Cambia MTU / longitud capturada | `--mtu=1500` |

## Workflow: regenerar tráfico de laboratorio (paso a paso)

**Caso típico:** capturaste en producción (o en un CTF) el tráfico de un exploit; quieres reproducirlo contra tu IDS aislado desde IPs del rango de laboratorio.

### Paso 1 — Captura (con filtros, no la vida entera)

```bash
sudo tcpdump -i eth0 -w cap.pcap host 10.10.10.50 and port 80   # o desde [[Wireshark y Tshark]]
```

### Paso 2 — Marcar cliente/servidor

```bash
tcpprep -a cidr -c "10.0.0.0/8" -i cap.pcap -o cache.pcap
# o explícito: cliente = puertos efímeros, servidor = 80
tcpprep -p 1024-65535:80 -i cap.pcap -o cache.pcap
```

### Paso 3 — Reescribir endpoints a tu rango de lab

```bash
tcprewrite -c cache.pcap -e 10.9.9.10:10.9.9.20 --fixcsum -i cap.pcap -o lab.pcap
```

### Paso 4 — Inyectar

```bash
sudo tcpreplay -i eth-lab -M 1 --stats 5 lab.pcap
```

### Paso 5 — Interpretar la salida

```
Running in bidirectional mode.      ← (si usaste -I; si no, "Running in top-speed mode" etc.)
Actual: 1524 packets sent in 2.5s.  Rated: 4.9 Mbps...  (381.0 pps, 16.4 us/byte)
```

| Campo | Qué significa |
|---|---|
| `Actual: N packets sent` | Paquetes **realmente** transmitidos. Si es menor que el total del pcap, hubo drops de la interfaz (sube buffers, baja rate, o usa `--preload-pcap`) |
| `Rated: X Mbps / pps` | Velocidad efectiva de salida — compárala con la pedida (`-p`, `-r`) para detectar cuellos de botella de la NIC |
| `us/byte` | Coste por byte: útil para benchmark de la ruta de envío |
| `Warning: ... packet size ...` | Paquetes truncados vs snaplen original (normal si la captura tenía snaplen corto) |

## Recetas (copy-paste)

```bash
# 1. Replay "tal cual", al ritmo original de captura
sudo tcpreplay -i eth0 cap.pcap

# 2. Lo más rápido posible (benchmark / poblar logs de golpe)
sudo tcpreplay -i eth0 -D cap.pcap

# 3. Preciso con la pausa original entre paquetes (timings relativos)
sudo tcpreplay -i eth0 -e nanosleep cap.pcap

# 4. Limitar a 1000 pps (evitar saturar el lab)
sudo tcpreplay -i eth0 -p 1000 cap.pcap

# 5. Repetir el flujo 5 veces (simular patrón de scanning)
sudo tcpreplay -i eth0 -l 5 cap.pcap

# 6. Máxima velocidad multihilo con pcap en RAM
sudo tcpreplay -i eth0 -t --preload-pcap -D cap.pcap

# 7. Cambiar endpoints y enviar (requiere tcpprep)
tcpprep -a cidr -c "192.168.0.0/16" -i cap.pcap -o cache.pcap
sudo tcpreplay-edit -c cache.pcap -e 172.16.0.10:172.16.0.20 --fixcsum -i eth0 cap.pcap

# 8. Replay bidireccional (dos interfaces, cliente y servidor)
tcpprep -a port -p 80 -i cap.pcap -o cache.pcap
sudo tcpreplay -i eth0 -I eth1 -c cache.pcap cap.pcap

# 9. Solo los paquetes 100 a 149 (analizar un episodio concreto)
sudo tcpreplay -i eth0 -X 99 -L 50 cap.pcap

# 10. Probar un IDS con checksums manipulados a propósito
tcprewrite --skipbadsum -i cap.pcap -o badsum.pcap
sudo tcpreplay -i eth0 badsum.pcap
```

**Detalle de la receta 3:** `-e nanosleep` reemplaza el selector de espera por defecto por un sleep de kernel más preciso; combinar con `-M` permite reordenar la velocidad manteniendo la proporción entre paquetes — "relativo" no significa "en tiempo real absoluto".

## Detección y contramedidas (defensor)

Reproducir tráfico se nota. Señales típicas de replay en la red:

| Señal | Por qué aparece | Dónde se ve |
|---|---|---|
| **Paquetes TCP duplicados** (mismos seq/ack bytes repetidos en flujos nuevos) | El replay clona sesiones completas incluido el estado TCP | `tcp.analysis.retransmission`, `tcp.analysis.duplicate_ack` en [[Tshark]] |
| **TCP timestamps inconsistentes** (TSval se repite o retrocede, o no guarda relación con el uptime del emisor) | El replay no negocia TSval; el receiver detecta "timestamp no monotónico" (RFC 7323 lo trata como ataque de reutilización) | tcpdump/Wireshark, o drops silenciosos en el destino |
| **Falta de ARP/ICMP previo** | La sesión "empieza a mitad de conversación" sin resolución de vecinos | Tabla ARP vacía + tráfico TCP que responde a hosts desconocidos |
| **MAC origen = MAC del host emisor** en tráfico que "dice" venir de otra IP (si no reescribiste) | Envío en capa 2 | Switch/IDS: mismatch IP↔MAC vs tabla ARP |
| **TTL/hop-limit inconsistente** con el salto esperado | El pcap conserva el TTL original | Análisis de hop count en IDS |
| **Ráfagas idénticas en intervalos exactos** (`-l` loop con `-D`) | Timing demasiado perfecto / demasiado rápido | Grafos de flujo, detección de beaconing |
| **Checksums inválidos** (si se omite `--fixcsum`) | Reescritura sin recálculo | Descartes silenciosos por NICs con validación estricta |

> [!warning] Legal y ética
> Reinyectar tráfico con direcciones MAC/IP ajenas solo tiene sentido en **laboratorios aislados o redes con autorización escrita**. En una red real, el replay bidireccional puede secuestrar sesiones vivas (si reproduces un flujo activo, los endpoints reales reciben paquetes duplicados con secuencias viejas) y el cambio de MAC (`--mac`, `[[Macchanger Herramienta de Modificación MAC]]` a nivel de interfaz) suele violar políticas de red.

## Errores comunes (mensaje → arreglo)

1. **`You must be superuser to use this utility` / `socket (PF_PACKET): Permission denied`** — falta privilegio de capa 2.
   → `sudo`, o dar capacidad al binario: `sudo setcap cap_net_raw,cap_net_admin=eip $(which tcpreplay)`.
2. **`Please run tcpreplay with root privileges`** (variantes según versión/libpcap) — mismo caso.
3. **Error indicando que `-I/--intf2` requiere caché** — el replay bidireccional necesita saber qué paquetes van por cada interfaz.
   → Generar caché con `tcpprep` y pasar `-c cache.pcap` (recetas 7/8).
4. **`tcpreplay: could not determine link layer type` / paquetes "sent" pero nada en la red** — interfaz equivocada, captura SLL (hecha con cooked capture de tcpdump) o VLAN sin etiquetar.
   → Capturar con `tcpdump -e`/Wireshark en modo normal, o usar `tcprewrite --mtu`/etiquetado según el escenario. Las capturas *Linux cooked* (SLL) no se pueden reinyectar tal cual.
5. **Muchos paquetes perdidos (`Actual:` mucho menor que el total)** — la interfaz no da abasto.
   → `--preload-pcap -t -D`, subir `net.core.wmem_max`, o bajar el rate (`-p`/`-r`).
6. **Paquetes rechazados por checksum tras `tcprewrite`** — se reescribieron direcciones sin `--fixcsum`.
   → Añadir `--fixcsum` en tcprewrite o `-C` en tcpreplay (o directamente `tcpreplay-edit`).
7. **`-M 1000` no limita a 1000 pps** — confusión de serie: en 4.x `-M` es un **multiplicador** (1000× la velocidad original).
   → Para pps usar `-p 1000`; para Mbps `-r 100`; para "todo lo que dé" `-D`.
8. **Nada de tráfico pasa con `-l 0`** — loop infinito sin Ctrl+C es intencionado, no es un cuelgue.
   → Combinar con `-L`/`--stats` para trabajos acotados en scripts.

## Cheatsheet

```bash
sudo tcpreplay -i eth0 cap.pcap                # ritmo original
sudo tcpreplay -i eth0 -D cap.pcap             # top speed
sudo tcpreplay -i eth0 -p 1000 cap.pcap        # 1000 pps
sudo tcpreplay -i eth0 -r 100 cap.pcap         # 100 Mbps
sudo tcpreplay -i eth0 -l 10 cap.pcap          # 10 bucles
sudo tcpreplay -i eth0 -X 99 -L 50 cap.pcap    # paquetes 100..149
sudo tcpreplay -i eth0 -C cap.pcap             # fix checksums al enviar
tcpprep -a cidr -c "10.0.0.0/8" -i cap.pcap -o cache.pcap
tcprewrite -c cache.pcap -e 10.0.0.1:10.0.0.2 --fixcsum -i cap.pcap -o out.pcap
sudo tcpreplay-edit -c cache.pcap -e 10.0.0.1:10.0.0.2 -i eth0 cap.pcap
```

| Tarea | Comando |
|---|---|
| Replay básico | `sudo tcpreplay -i <if> <pcap>` |
| Velocidad máxima | `-D --preload-pcap -t` |
| Limitar tasa | `-p <pps>` o `-r <mbps>` o `-M <factor>` |
| Repetir | `-l <n>` |
| Cambiar IPs | `tcpprep … && tcpreplay-edit -c cache -e orig:dest` |
| Cambiar puertos | `tcprewrite --portmap=80:8888` |
| Ver stats en vivo | `-s 5` |
| Ayuda real | `tcpreplay -h` |

## Tcpreplay vs otras formas de "reenviar" tráfico

| Criterio | **Tcpreplay** | Scapy/Python (`sendp`) | Reproxy (mitmproxy/nginx mirror) | `tcpdump -r \| ssh …` |
|---|---|---|---|---|
| Fidelidad del paquete | **Bit-exacta** (salvo lo reescrito) | Resintetizado: headers nuevos | Sólo capa 7 | Bit-exacta pero por stream |
| Velocidad | Mpps con `--preload-pcap` | Cientos–miles pps (overhead Python) | Limitado por app | Limitada por pipe |
| Timing original | Sí (`-e nanosleep`, `-M`) | Manual | No | No |
| Reescritura de campos | `tcprewrite`/`tcpreplay-edit` | Total (pero manual) | No aplica | No aplica |
| Bidireccional | Sí (con `-c` e `-I`) | Manual | Sí | No |
| Ideal para | Validar IDS/IPS, regenerar labs, benchmark | Fuzzing de paquetes, PoCs | Reaprovechar apps | Diagnóstico ad-hoc |

## Referencias

- [Documentación oficial (tcpreplay.appneta.com)](https://tcpreplay.appneta.com/) — man pages de los 4 binarios, guías de velocidad y de endpoints
- [Repositorio (sduffy/tcpreplay)](https://github.com/sduffy/tcpreplay) — código, issues y changelog de flags
- `man tcpreplay`, `man tcpprep`, `man tcprewrite` — referencia definitive instalada en Kali

## Relacionadas

- [[Tshark]] / [[Wireshark y Tshark]] — capturar los PCAP que luego se reinyectan y verificar el resultado
- [[Aircrack-ng Suite de Herramientas]] — otro usuario habitual de PCAP/IVs en el laboratorio
- [[Masscan]] — generar grandes volúmenes de tráfico de scanning comparable
- [[Nmap Guía Completa]] — el tráfico de scanning que a menudo se termina regenerando con tcpreplay
- [[Macchanger Herramienta de Modificación MAC]] — control de la MAC a nivel de interfaz (no de paquete)
- [[Reconocimiento y Escaneo de Redes y Web]] — contexto del replay en la fase de validación
