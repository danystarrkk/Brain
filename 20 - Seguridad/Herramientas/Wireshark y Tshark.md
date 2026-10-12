---
aliases:
  - Wireshark
  - Tshark
  - Analizador de protocolos
  - Captura de red
  - Análisis de tráfico
tags: [tool/wireshark]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Tshark]]"
  - "[[Tcpreplay]]"
  - "[[Nmap Guía Completa]]"
  - "[[CrackMapExec Guía de Uso]]"
  - "[[smbclient Cliente de Línea de Comandos para SMBCIFS]]"
  - "[[THC Hydra]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
fuente: "https://www.wireshark.org/docs/"
---

# Wireshark y Tshark: Dominio del Análisis de Tráfico

**Wireshark** (GUI) y **Tshark** (CLI) son la misma bestia: el analizador de protocolos de referencia, con el mismo motor de captura (libpcap/Npcap vía `dumpcap`) y los mismos 1000+ dissectores. Este note es la **guía maestra del análisis**: filtros de display, filtros de captura (BPF), coloring, follow streams, estadísticas, dissectores clave para pentest y rendimiento con capturas grandes. La referencia completa del CLI (flags, `-T fields`, `-z`, scripting) vive en [[Tshark]] — aquí la esencia GUI + los conceptos que comparten ambos.

> [!info] Elegir superficie de trabajo
> **GUI** para explorar, preguntar en caliente y mostrar evidencias en un informe. **Tshark** para repetir, automatizar y trabajar en servidores. La mayoría de flujos empieza en GUI (hipótesis) y termina en Tshark (pipeline). La sintaxis de filtros de display es idéntica en ambos.

## Instalación y permisos

```bash
# Kali trae ambas; en Debian/Ubuntu:
sudo apt install wireshark tshark
# Pregunta clave: "Allow non-superusers to capture packets?" → Sí
sudo dpkg-reconfigure wireshark-common
sudo usermod -aG wireshark $USER      # cerrar y abrir sesión

# Verificar
wireshark --version; tshark --version
```

> [!warning] El error de permisos #1
> `The capture session could not be initiated on device "eth0" (Permission denied).` — tu usuario no está en el grupo `wireshark` o no relogueaste. `sudo` funciona pero es mala práctica para una GUI. En Wayland/Xorg, ejecutar la GUI con `sudo` puede fallar por el display: usa el grupo.

## Anatomía de la GUI (lo que importa)

| Zona / menú | Función | Atajo típico |
|---|---|---|
| Barra de interfaces | Iniciar/detener captura en una interfaz | — |
| **Barra de filtro de display** | Filtra paquetes decodificados (sintaxis Wireshark) | `/` para enfocar, Ctrl+/ autocompleta |
| Lista de paquetes | Resumen por línea (nº, tiempo, origen, destino, info) | flechas, Enter = detalle |
| Panel de detalle | Árbol de campos por capa | Expandir; clic derecho → *Apply as Filter* |
| Hex view | Bytes del paquete resaltando el campo seleccionado | — |
| **Colorización** | Paleta por tipo de evento (ver más abajo) | *View → Coloring Rules* |
| **Statistics** | Conversations, Endpoints, Protocol Hierarchy, Expert… | — |
| **Analyze → Display Filter** | Gestión de filtros guardados | — |
| *File → Export Objects* | Extraer archivos transferidos (HTTP/SMB/TFTP/IMF…) | — |
| *File → Export Packet Dissections* | Exportar árbol/CSV/JSON del paquete | — |
| *View → Name Resolution* | Resolver IP/MAC/puertos en la vista | Ctrl+Alt+M/N/P |
| *Analyze → Expert Information* | Anomalías clasificadas (Chat/Note/Warn/Error) | — |

## Filtros de display: el lenguaje de verdad

### Operadores

| Operador | Ejemplo | Notas |
|---|---|---|
| `== != > < >= <=` | `tcp.port == 443`, `frame.len > 1000` | Comparación numérica/cadenas |
| `and or not` / `&& \|\| !` | `ip.src==10.0.0.5 and not dns` | Paréntesis para agrupar |
| `contains` | `http.host contains "admin"` | Subcadena (case-sensitive) |
| `matches` | `http.host matches "\.gov$"` | Regex |
| `in` (rangos/set) | `tcp.port in {80, 443, 8080}`, `ip.addr in 10.0.0.0/8` | Set y CIDR |
| `[]` de bytes | `tcp[13] == 0x02` (sintaxis BPF-like disponible también) | — |
| `..` rangos | `frame.time_relative >= 10 and frame.time_relative <= 20` | — |

### Los 12 filtros de pentest que hay que saber de memoria

```text
http.request.method == "POST"              # posts (logins, uploads, API calls)
http.host == "target.local"                # virtual host concreto
http.authorization contains "Basic"        # Basic auth en vuelo (credenciales base64)
http.response.code >= 400                  # respuestas de error
dns.flags.response == 0                    # solo queries DNS
tls.handshake.extensions_server_name == "target.local"  # SNI (sin descifrar)
ftp.request.command == "PASS"              # password FTP en claro
tcp.flags.syn == 1 and tcp.flags.ack == 0  # inicios de conexión (scanning)
tcp.analysis.retransmission                # retransmisiones
tcp.analysis.zero_window                   # receptor saturado
ntlmssp                                    # autenticaciones NTLM (hashes NTLMv2)
frame contains "password"                  # búsqueda literal en cualquier byte
```

> [!tip] Preparar, aplicar y seguir
> Clic derecho sobre un campo → *Prepare a Display Filter* (construye sin aplicar), *Apply as Filter* (aplica ya), *Follow → TCP Stream* (reconstruye). El campo seleccionado en el detalle define el contexto: es la forma más rápida de armar filtros sin memorizar nombres.

### Búsqueda (Find) ≠ filtro

Ctrl+F busca **texto/payload** en la captura (Packet bytes / Packet list / Details); el filtro de display selecciona por **campos**. Para cazar "password" en claro: primero `frame contains "pass"` (filtro), luego *Follow Stream* para contexto.

## Filtros de captura (BPF) vs filtros de display

| | **Capture filter (BPF)** | **Display filter** |
|---|---|---|
| Dónde actúa | Kernel, antes de capturar | Después, sobre el archivo |
| Lenguaje | BPF clásico | Sintaxis Wireshark (rica) |
| Coste | Cero (descarta antes) | Decodificación completa |
| Reversible | **No** (el paquete no existe) | Sí (si se guardó) |
| Ejemplos | `host 10.0.0.5`, `port 443`, `tcp port 80 or udp port 53` | `http.request`, `tcp.port==443 && ip.src==10.0.0.5` |

BPF útil para pentest:

```text
tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn   # solo SYN (scanning)
not arp and not port 53                        # sin ruido local
src net 10.0.0.0/24 and dst portrange 1-1024   # barrido entrante
```

> [!warning] "Capturé con filtro y no hay nada que filtrar"
> Si pusiste un filtro de **captura** estrecho y el evento quedó fuera, ningún filtro de display lo recupera: se perdió en el kernel. En sesiones forenses, captura **ancha** (o sin filtro) y filtra después; el filtro de captura se usa cuando el volumen haría inviable grabar todo.

## Coloring rules: leer la lista de paquetes a golpe de vista

La paleta por defecto codifica eventos de análisis, no protocolos:

| Color (default) | Regla | Qué te dice |
|---|---|---|
| Rojo claro | *Bad TCP* (checksums, paquetes malformados) | Corrupción, offload, o manipulación maliciosa de paquetes |
| Negro | TCP error (RST, segmentos ilegales) | Corte de sesión, firewall que resetea |
| Lila claro | TCP SYN/FIN/RST | Inicios/fines de conexión — patrones de scan se ven aquí |
| Amarillo | *DNS* | Consultas/respuestas |
| Verde claro | *HTTP* | Tráfico web (resaltado por puerto/contenido) |
| Azul claro | *SMB* | Tráfico de archivos Windows |
| Naranja | *Checksum* | Bad/unchecked checksums |

*View → Coloring Rules* para reglas propias — ejemplos potentes: `http.request.method == "POST"` (todos los formularios en un color), `ntlmssp || kerberos` (momentos de autenticación), `tcp.analysis.retransmission` (problemas de red).

## Follow Streams: la reconstrucción de sesiones

Clic derecho → *Follow* → **TCP / UDP / TLS / HTTP / QUIC… Stream** (o `-z follow,tcp,ascii,N` en [[Tshark]]).

- Salida ASCII con el cliente en **rojo** y el servidor en **azul**; se puede reconvertir a hex/raw.
- Reconstruye el payload **reensamblado** (desagregación TCP): lo que ves es el mensaje de aplicación real, en orden, sin duplicados.
- El filtro de stream (`tcp.stream eq 5`) es la forma correcta de aislar una sesión: sobrevive a retransmisiones y ACKs.
- Para HTTP: *Follow → HTTP Stream* muestra request y response con cabeceras — el lugar donde se copian credenciales Basic y tokens.

## Exportar objetos: extraer archivos transferidos

*File → Export Objects → HTTP / SMB / TFTP / IMF (email) / DICOM…* o en CLI `tshark -r cap.pcap --export-objects http,./out/`.

- Lista todos los objetos transferidos con nombre, tamaño, tipo y **el paquete de origen** (clic = ver bytes exactos).
- Flujo forense estándar: exportar → `file *` → antivirus/hash (sha256) → carving con binwalk si el archivo está roto.
- En SMB (exfiltración, lateral movement) la exportación desde `--export-objects smb` recupera los ficheros movidos entre hosts — el clásico hallazgo cuando se revisa una captura de compromiso.

## Estadísticas del menú Statistics

| Menú | Qué aporta en pentest |
|---|---|
| **Conversations** | Parejas de hosts con bytes/paquetes/duración; ordenar por bytes → mover el foco a lo relevante |
| **Endpoints** | Totales por host: quién habla más, quién solo escuchó (recon) |
| **Protocol Hierarchy** | % por protocolo: un 3% de SMB en red corporativa "web" es anómalo |
| **Expert Information** | Anomalías clasificadas (Chat/Note/Warn/Error): retransmisiones, resets, respuestas raras — la primera parada tras abrir una captura ajena |
| **HTTP → Request/Response/Packet Counter** | Volumen y tiempos de web |
| **DNS → Queries** | Beaconing y dga (consultas largas, NXDOMAIN en ráfaga) |
| **VoIP → RTP → Streams** | Calidad y presencia de llamadas (si aplica) |
| **I/O Graph** | Serie temporal con filtros como series — la mejor herramienta para detectar beaconing (picos periódicos) y volúmenes anómalos |

## Dissectores de interés para pentest

| Protocolo | Qué buscar | Filtro de entrada |
|---|---|---|
| HTTP | Credenciales Basic, cookies de sesión, form POST en claro, tokens JWT | `http.authorization \|\| http.request.method=="POST"` |
| TLS | SNI (visibilidad parcial), certificados autofirmados/antiguos, JA3 (si el disector lo soporta) | `tls.handshake.type == 1` |
| DNS | Exfiltración (TXT grandes, subdominios aleatorios), tunelado, DGA | `dns.qry.type == 16` (TXT) |
| FTP/POP/IMAP/SMTP | Contraseñas y datos en claro (`PASS`, `USER`, AUTH) | `ftp.request \|\| pop.request \|\| imap.request` |
| SMB/NTLM | Movimiento lateral, hashes NTLMv2 capturados (relay/offline crack), ficheros exportables | `ntlmssp` / `smb` |
| Kerberos | AS-REP roasting (`KDC_ERR_PREAUTH_FAILED`), usernames visibles | `kerberos` |
| RDP/VNC | Handshakes, login VNC (reto-respuesta, no claro) | `rdp \|\| vnc` |
| LDAP | Bind simples en claro, enumeración AD | `ldap` |
| SNMP | Comunidades (`public`) — trampa clásica | `snmp` |

> [!warning] Lo que Tshark/Wireshark **no** ve sin ayuda
> TLS 1.2/1.3 moderno oculta el contenido: solo SNI, certificado y metadatos. Descifrar requiere el **keylog del cliente** (`SSLKEYLOGFILE` exportado durante la captura, luego `-o tls.keylog_file:` en Tshark o *Edit → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename* en la GUI). Sin keylog, el análisis TLS se limita a metadatos — y el "find password" no aplica a HTTPS. Ver [[Tshark]] receta 9.

## TCP Analysis: interpretar el estado de una sesión

| Campo (aparece en el detalle) | Significado |
|---|---|
| `tcp.analysis.retransmission` | El emisor reenvió datos no confirmados: pérdida o atasco |
| `tcp.analysis.fast_retransmission` | Retransmisión rápida (3 dupACKs): pérdida moderada |
| `tcp.analysis.duplicate_ack` | ACKs repetidos: señal temprana de pérdida |
| `tcp.analysis.lost_segment` | Hueco en la secuencia (paquete perdido) |
| `tcp.analysis.ack_lost_segment` | ACK perdido |
| `tcp.analysis.window_full` / `zero_window` | Emisor saturado / receptor saturado (flujo detenido) |
| `tcp.analysis.out_of_order` | Reordenamiento (o replay/IDS inline) |
| `tcp.analysis.spurious_retransmission` | Retransmisión innecesaria |

**Lectura pentest:** muchas retransmisiones + resets desde un solo host pueden ser scaneo con drops; `out_of_order` masivo en un flujo es sospechoso de replay (ver [[Tcpreplay]] y su sección de detección).

## Rendimiento con capturas grandes

| Situación | Estrategia |
|---|---|
| PCAP de varios GB | Primero `capinfos`/`tshark -q -z io,phs` (visión sin cargar); GUI solo si el host tiene RAM de sobra (Wireshark carga el archivo completo) |
| Captura continua | Rotación: `-b filesize:100000 -b files:50` (Tshark/dumpcap) o en *Capture Options* |
| Resolución de nombres lenta | Desactivar *Name Resolution* (o `-n`): cada IP esperando DNS multiplica el tiempo |
| Solo te interesa un flujo | Guardar el stream: filtro `tcp.stream eq N` → *Export Specified Packets* a archivo nuevo → reabrir liviano |
| Filtros repetitivos | Guardar en *Display Filter* con nombre; en Tshark, alias/script |
| Muchos protocolos | *Analyze → Enabled Protocols*: desactivar los que ensucian (p. ej. en tráfico cifrado, desactivar HTTP misidentified) |

## Recetas (los flujos del día a día)

```bash
# 1. Encontrar credenciales en claro en una captura (GUI: el mismo filtro + Follow)
tshark -r cap.pcap -Y 'ftp.request.command=="PASS" || http.authorization || pop.request.command=="PASS"' \
  -T fields -e frame.time -e ip.src -e ip.dst -e _ws.col.Info

# 2. Decodificar los Basic capturados
tshark -r cap.pcap -Y 'http.authorization contains "Basic"' -T fields -e http.authorization \
  | cut -d' ' -f2 | while read c; do echo "$c" | base64 -d; echo; done

# 3. Ver los POST completos (qué se envió a qué login)
tshark -r cap.pcap -Y 'http.request.method=="POST"' -T fields \
  -e frame.number -e http.host -e http.request.uri -e urlencoded-form.key -e urlencoded-form.value

# 4. Extraer ficheros movidos por SMB (exfil/lateral)
mkdir -p smb_export && tshark -r cap.pcap --export-objects smb,smb_export/

# 5. Detectar beaconing: consultas DNS periódicas
tshark -r cap.pcap -q -z io,stat,60,"COUNT(dns)"     # picos regulares = sospecha
tshark -r cap.pcap -Y 'dns.flags.response==0' -T fields -e frame.time_relative -e dns.qry.name | \
  awk '{print int($1/60), $2}' | sort | uniq -c | sort -rn | head

# 6. Seguir un stream por índice (GUI: Follow → TCP Stream)
tshark -r cap.pcap -q -z follow,tcp,ascii,7 | less

# 7. Quitar la captura para análisis (guardar solo lo relevante)
tshark -r cap.pcap -Y 'http or tcp.port==443' -w recorte.pcap
```

## Errores comunes (mensaje → arreglo)

1. **`The capture session could not be initiated ... (Permission denied)`** — permisos (ver Instalación). `sudo usermod -aG wireshark $USER` + relogin.
2. **Filtro de display que no matchea nada** — error de sintaxis (la barra se pone roja) o nombre de campo inexistente.
   → Autocomplete (Ctrl+Space), o `tshark -G fields | grep -i <clave>`; mayúsculas importan en valores (`"POST"` ≠ `"post"`).
3. **Capturaste con BPF estrecho y no está el evento** — se descartó en el kernel.
   → No hay arreglo: recapturar anchando o sin filtro de captura.
4. **La GUI "no va" con capturas de 5 GB** — carga todo en RAM.
   → `tshark` para análisis por lotes, o recortar el pcap y reabrirlo.
5. **HTTP vacío tras `-s 96` (snaplen corto)** — solo cabeceras guardadas, sin body.
   → Snaplen ≥ 1500 para inspección web; `-s 0` para forense completo.
6. **`Expert: [Checksum incorrect]` en todas las líneas** — offload de checksums en la máquina capturadora (lo "incorrecto" se arregla en la NIC al salir).
   → Normal en capturas locales; no es tráfico malicioso. Para regenerar el tráfico correcto usar `-C/--fixcsum` de [[Tcpreplay]].
7. **TLS "sólo chino" (hex)** — falta el keylog (ver arriba) o es TLS 1.3 sin secretos de sesión.
   → Capturar con `SSLKEYLOGFILE` desde el cliente legítimo.

## Referencias

- [wireshark.org/docs](https://www.wireshark.org/docs/) — User's Guide, Book of NetShark
- [Display Filter Reference](https://www.wireshark.org/docs/dfref/) — diccionario oficial de campos por protocolo
- [Man page de tshark](https://www.wireshark.org/docs/man-pages/tshark.html) — CLI completa (ver [[Tshark]])
- `tshark -G fields` — inventario vivo de campos en tu instalación

## Relacionadas

- [[Tshark]] — referencia CLI completa: `-T fields`, `-z` stats, scripting, rendimiento
- [[Tcpreplay]] — regenerar capturas en la red (y detectar replays con los campos `tcp.analysis.*`)
- [[Nmap Guía Completa]] — correlacionar servicios descubiertos con tráfico observado
- [[THC Hydra]] — la firma de un ataque de credenciales en logs de captura
- [[CrackMapExec Guía de Uso]] / [[smbclient Cliente de Línea de Comandos para SMBCIFS]] — SMB/NTLM: en vivo (herramientas) y en captura (aquí)
- [[Reconocimiento y Escaneo de Redes y Web]] — el análisis de tráfico dentro de la metodología
