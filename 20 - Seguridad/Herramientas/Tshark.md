---
tags:
  - ciberseguridad
  - herramientas
  - tshark
tipo: herramienta
estado: terminado
---

# Tshark: Analizador de Protocolos en Línea de Comandos

**Tshark** es Wireshark sin interfaz gráfica: mismo motor de captura (libpcap/Npcap vía `dumpcap`) y **mismos dissectores de protocolo**, pero con salida de texto, filtros, extracción de campos y estadísticas invocables desde un script. Es la herramienta de análisis de capturas por defecto en servidores, en pipelines de automatización y en cualquier PCAP demasiado grande para abrir en la GUI.

> [!info] Por qué Tshark y no tcpdump
> `tcpdump` imprime resúmenes de paquetes; Tshark **entiende la capa de aplicación**: reconstruye streams, extrae `http.user`, descompone handshakes TLS, sigue autenticaciones FTP/NTLM y exporta objetos HTTP. Regla práctica: **tcpdump para capturar rápido en vivo, Tshark para analizar después**. En la GUI (filtros de display, coloring, export objects) ver [[Wireshark y Tshark]] — este note es la referencia completa del CLI.

## Instalación

```bash
# Kali / Debian (Wireshark sin GUI recomendada)
sudo apt update && sudo apt install tshark
# Pregunta "Should non-superusers be able to capture packets?" → Sí (grupo wireshark)
sudo usermod -aG wireshark $USER   # y relogin

# Verificar
tshark --version
```

Permisos de captura en vivo: o bien perteneces al grupo `wireshark` (uso recomendado, configura `dpkg-reconfigure wireshark-common`), o ejecutas con `sudo`. Analizar archivos `-r` no requiere privilegios.

## Los dos ejemplos que resuelven el 80%

```bash
# 1. Capturar 1000 paquetes del tráfico web a un archivo
sudo tshark -i eth0 -f "tcp port 80 or tcp port 443" -c 1000 -w web.pcap

# 2. Extraer campos de esa captura como tabla para scripts
tshark -r web.pcap -Y "http.request" -T fields -e frame.number -e ip.src -e http.host -e http.request.uri
```

## Opciones de captura (en vivo)

| Opción | Significado | Ejemplo |
|---|---|---|
| `-D` | Lista interfaces con índice | `tshark -D` |
| `-i, --interface` | Interfaz (nombre o índice; `-i any` en Linux, captura en "any" no soporta `-s`) | `-i eth0` |
| `-f` | **Filtro de captura BPF** (en kernel) | `-f "tcp port 443"` |
| `-s, --snapshot-length` | Snaplen: bytes por paquete (default 262144; `-s 0` = completo) | `-s 1500` |
| `-p` | No poner la interfaz en modo promiscuo | `-p` |
| `-I` | Modo monitor (802.11 inalámbrico) | `-I` |
| `-B, --buffer-size` | Buffer del kernel en MB (subirlo en redes intensas) | `-B 32` |
| `-c` | Terminar tras capturar N paquetes | `-c 5000` |
| `-a` | Condición de auto-parada: `duration:seg`, `filesize:KB`, `files:n` | `-a duration:300` |
| `-b` | Rotación de archivos: `filesize:KB`, `duration:seg`, `files:n`, `interval:seg` | `-b filesize:100000 -b files:10` |
| `-w` | Escribir captura a archivo (pcapng por defecto) | `-w out.pcap` |
| `-F` | Formato del archivo de salida (`pcap`, `pcapng`, `netxl`, `json`…) | `-F pcap -w out.pcap` |
| `-L` | Lista tipos de enlace disponibles en la interfaz | `-L` |
| `--time-stamp-type` | Tipo de timestamp del adaptador | `--time-stamp-type=adapter_unsynced` |
| `--capture-comment` | Comentario incrustado en la captura (pcapng) | `--capture-comment="lab day 3"` |

### Filtros BPF (captura) — referencias rápidas

```bash
host 192.168.1.10                 # un host (entrada o salida)
net 10.0.0.0/24                   # una red
src host 10.0.0.5                 # solo origen
dst port 443                      # puerto destino
portrange 1-1024                  # rango
tcp or udp                        # protocolos
not port 22                       # negación (ojo con precedencias)
'tcp[tcpflags] & tcp-syn != 0'    # solo SYN
'ip[8] < 64'                      # TTL < 64 (búsqueda de hops)
```

> [!tip] BPF antes, display filter después
> BPF (`-f`) se aplica **en el kernel**: descarta paquetes antes de costar CPU ni disco — úsalo para acotar la captura. El filtro de display (`-Y`) trabaja sobre paquetes ya decodificados: mucho más expresivo, pero tarde. La captura fina + el filtro grueso es la combinación correcta.

## Lectura y filtrado de archivos

| Opción | Significado | Ejemplo |
|---|---|---|
| `-r, --read` | Leer archivo de captura | `-r captura.pcap` |
| `-Y, --display-filter` | **Filtro de display** (post-decodificación) | `-Y "http.request.method == POST"` |
| `-2` | Filtrado en dos pasadas (necesario para filtros que requieren contexto de ambos flujos, p. ej. `tcp.analysis.ack_rtt`) | `-2 -Y "tcp.analysis.ack_rtt > 0.5"` |
| `-R` | Filtro de *read* (legacy — equivalente a `-Y` con `-2` desde Wireshark 1.12; se acepta pero está obsoleto) | `-R "tcp.port == 80"` (evitar) |
| `-d` | Decodificar tráfico como otro protocolo | `-d tcp.port==8888,http` |
| `-j` | Solo decodificar esta familia de protocolos (más rápido) | `-j http` |
| `-J` | Como `-j`, pero expandiendo su árbol con `-V` | `-J http -V` |
| `-O` | Con `-V`, mostrar detalles solo de estos protocolos | `-V -O http,tcp` |
| `-V` | Detalle completo de cada paquete (árbol de campos) | `-V -Y ftp` |
| `-x` | Hex dump + ASCII del payload | `-x -Y "frame.number==5"` |
| `-P` | Imprimir resumen aunque se escriba a `-w` | `-P -w out.pcap` |
| `-n` | Sin resolución de nombres (mucho más rápido) | `-n` |
| `-N` | Resolver tipos concretos: `n`(net), `m`(MAC), `s`(servicios), `t`(TMV), `ad`(fqdn) | `-N m` |
| `-l` | Line-buffered: impresión inmediata (para pipiar a `grep`/`cut`) | `-l -Y dns -T fields -e dns.qry.name` |
| `-o` | Preferencia del disector (misma sintaxis que en la GUI) | `-o "tcp.desegment_tcp_streams:TRUE"` |
| `-C` | Usar un perfil de configuración | `-C "Forensics"` |
| `-H` | Fichero hosts alternativo para resolución | `-H hosts.local` |
| `-K` | Keytab Kerberos para descifrar tráfico | `-K krb5.keytab` |
| `-U` | Modo PDU al escribir (timestamps por PDU, útil con `-w`) | `-U` |
| `-u` | Convención de unidades de tiempo en texto (`us` por defecto, `s`) | `-u s` |
| `-M` | Resolución de streams MPEG-TS (solo capturas de vídeo MPEG-TS) | `-M` |
| `-m` | Separador de salida (obsoleto; usar `-E separator=`) | `-m ";"` |

> [!warning] `-R` vs `-Y`: la trampa clásica
> Antes de Wireshark 1.12 existía `-R` con filtrado en una pasada. Hoy `-Y` es el filtro de display estándar; `-R` sigue funcionando pero exige `-2` en muchos contextos y confunde a quien viene de notas viejas. Regla actual: **usa `-Y` (y `-2` cuando el filtro necesite ver ambas direcciones del flujo)**.

## Extracción de campos: `-T fields`

| Opción | Significado | Ejemplo |
|---|---|---|
| `-T fields` | Una línea por paquete con los campos `-e` | `-T fields -e ip.src -e ip.dst` |
| `-e` | Campo a extraer (nombre exacto) | `-e frame.time_relative` |
| `-E separator=/t\|/s\|<char>` | Separador: `/t` tab, `/s` espacio, o carácter literal | `-E separator=,` |
| `-E quote=d\|s\|b\|n` | Comillas: dobles, simples, ambas, ninguna | `-E quote=d` |
| `-E occurrence=f\|l\|a` | Ocurrencia si el campo se repite: primera/última/todas | `-E occurrence=f` |
| `-E aggregator=,` | Cómo unir múltiples ocurrencias del mismo campo | `-E aggregator=" "` |
| `-E header=y` | Imprimir fila de cabecera con los nombres de campo | `-E header=y` |
| `-E occurrence` + `-E aggregator` | Clave para campos repetidos (p. ej. todos los `http.cookie`) | — |

### Cómo encontrar el nombre de un campo

```bash
# Buscar campos que contengan "user"
tshark -G fields | grep -i "http.user"
# Campos con su protocolo y descripción (grep ancho)
tshark -G fields | grep -i "ntlm"
```

`-G fields` imprime `Filtro Protocolo Nombre Descripción` — el **Nombre** (tercera columna) es lo que va tras `-e` y lo que usa el filtro de display en la GUI (con puntos).

## Formatos de salida (`-T`)

| Formato | Qué produce | Uso típico |
|---|---|---|
| `pdml` | XML completo de todos los campos | Procesamiento externo (XSLT, herramientas forenses) |
| `psml` | XML de resumen (una sección por paquete) | Índices de paquetes |
| `json` | JSON legible (un objeto por paquete) | `jq` |
| `jsonraw` | JSON con payloads en bruto | Análisis con binarios |
| `ek` | JSON para Elasticsearch (una línea por evento, campo `.layers`) | Ingesta EK (ELK) |
| `fields` | Texto tabular de campos `-e` | **Scripts, grep, pandas** |
| `text` (defecto) | Resumen tipo `tshark` de cada paquete | Lectura humana |
| `tabs` | Resumen con tabuladores | — |
| `ps` / `latex` | PostScript / tabla LaTeX | Informes |

### JSON para análisis programático

```bash
tshark -r cap.pcap -Y "dns.flags.response == 0" -T json | jq -r '.layers.dns[]."dns.qry.name"[0]'
# variante compacta para ingestar en Elasticsearch:
tshark -r cap.pcap -T ek -e ip.src -e ip.dst -e dns.qry.name > eventos.ndjson
```

## Estadísticas (`-z`) — el menú Statistics en CLI

Siempre con `-q` (no imprime paquetes, solo el informe):

| Estadística | Comando | Qué da |
|---|---|---|
| I/O por intervalo | `-z io,stat,60` | Paquetes/bytes cada 60 s — base de grafos de tráfico |
| I/O con filtro | `-z io,stat,0,"SUM(frame.len)","COUNT(tcp)"` | Series compuestas (0 = sin intervalo, total) |
| Jerarquía de protocolos | `-z io,phs` | % de tráfico por protocolo (primera vista de una captura) |
| Conversaciones | `-z conv,tcp` / `conv,udp` / `conv,ip` / `conv,eth` | Parejas con pkts/bytes/dirección a→b |
| Endpoints | `-z endpoints,ip` / `endpoints,tcp` | Totales por host |
| Árbol DNS | `-z dns,tree` | Distribución de consultas/tipos |
| HTTP | `-z http,stat`, `-z http,request`, `-z http,chat` | Requests, hosts, chat HTTP legible |
| SMB | `-z smb,srt` | Tiempos de respuesta SMB (rendimiento/forense) |
| Expert | `-z expert` / `-z expert,error` | Clasificación de anomalías (Chat/Note/Warn/Error) |
| Seguimiento de stream | `-z follow,tcp,ascii,<índice>` | Reconstruye el stream N como lo haría la GUI |
| RTP | `-z rtp,streams` | Flujos de voz (jitter/pérdidas) |

```bash
# Tráfico por minuto filtrado por longitud de payload HTTP
tshark -r cap.pcap -q -z io,stat,60,"SUM(http.content_length)"

# Stream TCP 3 en ASCII (equivale a Follow → TCP Stream en la GUI)
tshark -r cap.pcap -q -z follow,tcp,ascii,3
```

**Interpretación de `io,stat`:** columnas `| Frames | Bytes | ...` donde cada columna es una expresión; `Interval` = segundos desde el inicio. Un pico aislado de `COUNT(http)` marca el momento exacto de una petición relevante.

## Recetas de automatización (con explicación)

```bash
# 1. Extraer todas las peticiones HTTP (método, host, URI, UA)
tshark -r cap.pcap -Y "http.request" -T fields \
  -e frame.time -e ip.src -e http.request.method -e http.host -e http.request.uri -e http.user_agent \
  -E separator='|' -E header=y -E quote=d

# 2. Autorización Basic → decodificar credenciales
tshark -r cap.pcap -Y 'http.authorization contains "Basic"' -T fields -e http.authorization \
  | cut -d' ' -f2 | while read b; do echo "$b" | base64 -d; echo; done
# (equivalente gui: filtro http.authorization + Follow → HTTP Stream)

# 3. Passwords en claro de FTP
tshark -r cap.pcap -Y 'ftp.request.command == "PASS"' -T fields -e ftp.request.arg

# 4. Queries DNS únicas (recon / beaconing)
tshark -r cap.pcap -Y "dns.flags.response == 0" -T fields -e dns.qry.name -l | sort -u | head -50

# 5. Top conversaciones por bytes
tshark -r cap.pcap -q -z conv,ip | sort -k8 -rn | head -15

# 6. Contar conexiones SYN por origen (quién escanea)
tshark -r cap.pcap -Y "tcp.flags.syn==1 and tcp.flags.ack==0" -T fields -e ip.src -l | sort | uniq -c | sort -rn

# 7. Extraer objetos HTTP (archivos transferidos) sin salir de CLI
mkdir -p exportados
tshark -r cap.pcap --export-objects http,exportados/
tshark -r cap.pcap --export-objects smb,exportados/     # también smb, tftp, imap, mysql...

# 8. Datos de login POST (formularios enviados)
tshark -r cap.pcap -Y 'http.request.method == "POST"' -T fields -e http.host -e http.request.uri -e urlencoded-form.key -e urlencoded-form.value

# 9. Descifrar TLS 1.2/1.3 con keylog del cliente (si se capturó SSLKEYLOGFILE)
tshark -r cap.pcap -o tls.keylog_file:keys.log -Y "http.request" -T fields -e http.host -e http.request.uri

# 10. Vigilar en vivo: DNS + log line-buffered hacia grep
sudo tshark -i eth0 -l -n -Y "dns.flags.response==0" -T fields -e frame.time -e dns.qry.name

# 11. Convertir pcap a pcapng para anotar en la GUI
tshark -r viejo.pcap -F pcapng -w nuevo.pcapng

# 12. Detección de retransmisiones/rst (salud del flujo)
tshark -r cap.pcap -q -z expert | head -40
```

**Detalle de la receta 2:** `-Y` selecciona los paquetes con cabecera `Authorization: Basic …`; `-E` no aplica aquí porque se corta con `cut` y se decodifica con `base64 -d`. Es la versión scriptable de buscar "password" en la GUI — ver [[Wireshark y Tshark]] para el flujo interactivo.

**Detalle de la receta 9:** el keylog debe generarse **durante la captura** exportando `SSLKEYLOGFILE` en el cliente (`export SSLKEYLOGFILE=~/keys.log`). Descifra TLS 1.2 (y TLS 1.3 con los secretos de sesión modernos del keylog de Firefox/Chrome). En laboratorio propio es estándar; sobre tráfico ajeno aplica lo de la sección legal/ética.

## Rendimiento en capturas grandes

| Estrategia | Cómo | Por qué |
|---|---|---|
| Prefiltrar en captura | `-f "not port 53"` al capturar | El kernel descarta antes de escribir disco |
| `-n` siempre | `tshark -r big.pcap -n` | La resolución DNS/MAC es la lentitud #1 de procesado |
| `-T fields` en vez de `-V` | Campos concretos | `-V` decodifica y serializa todo el árbol |
| Acotar con `-Y` ligero | `frame.number < 100000`, `tcp.stream eq 5` | Evita decodificar capas innecesarias |
| `-j http` | Decodificación restringida | Omite dissectores no usados |
| Rotación al capturar | `-b filesize:100000 -b files:20` | Archivos manejables, paralelizables |
| Pipelines | `-l` + `grep`/`awk`/`jq` | Sin cargar todo en memoria |
| `capinfos` previo | `capinfos -a -e -u big.pcap` | Ver tamaño/duración/paquetes antes de atacar |

## Errores comunes (mensaje → arreglo)

1. **`tshark: "xyz" — neither a field nor a protocol name...` (filtro) / `tshark: ... is not a valid display filter`** — nombre de campo/protocolo mal escrito.
   → `tshark -G fields | grep -i <clave>` para el nombre exacto; la autocompletación de la GUI también sirve como diccionario.
2. **`tshark: -e can only be used with -T fields`** — `-e` sin `-T fields`.
   → Añadir `-T fields` (o usar `-T json` + `jq` si queres todo).
3. **`Permission denied` / `The capture session could not be initiated on device "X"`** — sin permisos de captura.
   → Grupo `wireshark` (relogin) o `sudo`; en contenedores `--cap-add=NET_RAW --cap-add=NET_ADMIN`.
4. **Filtro que "debería" funcionar y no da resultados con `-2` ausente** — filtros que requieren ambas pasadas (`ack_rtt`, correlaciones).
   → Añadir `-2`.
5. **Resultados vacíos con `-d tcp.port==8888,http`** — el decode-as solo aplica a puertos TCP; falta el `-Y http` o el puerto es UDP.
   → `-d udp.port==5353,http` para UDP (sí existe la forma), y recordar que `-d` no decodifica tráfico ya descartado por filtros previos.
6. **`tshark: The file isn't a capture file...`** — es un texto o está truncado (o se capturó con snaplen a medias y luego se corrompió).
   → `file cap.pcap`, `capinfos cap.pcap`; recapturar con snaplen suficiente si el header pcap está ilegible.
7. **Captura en "any" no admite `-s` ni modo monitor** — interfaz pseudo-dispositivo sin enlace real.
   → Capturar en la interfaz física con `-s`/`-I` cuando haga falta.
8. **`-z` no imprime nada** — falta `-q` o el nombre de la estadística está mal (`-z conv,tcp` no `-z conversations`).
   → Copiar la sintaxis exacta del man; `-q -z` siempre juntos.

## Cheatsheet

```bash
tshark -D                                        # interfaces
sudo tshark -i eth0 -f "tcp port 443" -c 1000 -w out.pcap
tshark -r out.pcap -n                            # resumen rápido (sin DNS)
tshark -r out.pcap -Y "http.request" -V          # detalle HTTP
tshark -r out.pcap -Y "tcp.stream eq 3" -T fields -e frame.time -e ip.src -e tcp.payload
tshark -r out.pcap -T fields -e ip.src -e ip.dst -e _ws.col.Info
tshark -r out.pcap -q -z io,stat,60
tshark -r out.pcap -q -z conv,tcp
tshark -r out.pcap -q -z io,phs
tshark -r out.pcap -q -z follow,tcp,ascii,0
tshark -r out.pcap --export-objects http,./out/
tshark -r out.pcap -T json | jq .
tshark -r out.pcap -F pcapng -w out.pcapng
tshark -G fields | grep -i <campo>
tshark -r out.pcap -Y "http.authorization" -T fields -e http.authorization
```

| Tarea | Comando |
|---|---|
| Capturar acotado | `-i <if> -f "<bpf>" -c/-a -w file` |
| Filtrar | `-Y "<display filter>"` (+ `-2` si hace falta contexto) |
| Extraer campos | `-T fields -e <campo> -E header=y -E separator=,` |
| JSON/jq | `-T json` (o `-T ek` para ES) |
| Stats | `-q -z io,stat,N` / `-z conv,tcp` / `-z io,phs` |
| Stream completo | `-q -z follow,tcp,ascii,<n>` |
| Objetos/archivos | `--export-objects http,dir/` |
| Buscar campo | `-G fields \| grep` |
| Rápido | `-n` + `-T fields` |

## Tshark vs tcpdump

| Criterio | **Tshark** | tcpdump |
|---|---|---|
| Capa de aplicación | **Sí** (1000+ protocolos) | Solo cabeceras + payload crudo |
| Filtros | BPF + display (miles de campos) | Solo BPF |
| Extracción estructurada | `-T fields/json/ek/pdml` | Texto + `-w` |
| Seguimiento de stream | `-z follow,...` integrado | `-A` + filtros manuales |
| Estadísticas | `-z` (conv, io, phs, expert…) | No |
| Velocidad de captura | Media (decodifica todo) | **Máxima** (cero trabajo) |
| Salida para scripts | Excelente | Regular (grep del resumen) |
| Rol | **Análisis** | **Captura rápida** |

Flujo habitual: `tcpdump -w` en caliente (o Tshark si ya sabés el filtro) → análisis profundo con Tshark → [[Wireshark y Tshark]] para la inspección interactiva.

## Consideraciones legales

Descifrar o registrar credenciales y payloads de protocolos en claro es legítimo **sobre tráfico propio o con autorización explícita** (lab, engagement firmado, red corporativa con política). Capturar tráfico ajeno sin consentimiento es interceptación ilegal en la mayoría de jurisdicciones — el poder de `-T fields` sobre `http.authorization` o `ftp.request.arg` es exactamente el motivo por el que las capturas se trazan y se custodian.

## Referencias

- [Man page oficial de tshark](https://www.wireshark.org/docs/man-pages/tshark.html) — referencia completa de opciones
- [Lista de campos/display filters](https://www.wireshark.org/docs/dfref/) — diccionario oficial de nombres
- [User's Guide — Dirty Secrets of Network Capture](https://www.wireshark.org/docs/wsug_html_chunked/) — capítulos de filtrado y estadísticas
- `tshark -G fields` / `tshark -G plugins` — la fuente viva en tu instalación

## Relacionadas

- [[Wireshark y Tshark]] — GUI: filtros de display, coloring, follow streams interactivos, export objects
- [[Tcpreplay]] — regenerar en la red las capturas analizadas aquí
- [[Nmap Guía Completa]] — correlacionar hallazgos de servicios con el tráfico observado
- [[CrackMapExec Guía de Uso]] / [[smbclient Cliente de Línea de Comandos para SMBCIFS]] — SMB/NTLM en vivo y su firma en capturas
- [[Reconocimiento y Escaneo de Redes y Web]] — contexto del análisis de tráfico en la metodología
