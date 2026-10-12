---
aliases: ["Nmap", "Network Mapper", "Escaneo de Redes", "nmap"]
tags:
  - tool/nmap
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[Masscan]]"
  - "[[Enumeración de Servicios y CMS]]"
  - "[[netdiscover]]"
  - "[[Arp Scan Herramienta de Descubrimiento]]"
  - "[[Wireshark y Tshark]]"
  - "[[CrackMapExec Guía de Uso]]"
  - "[[smbclient Cliente de Línea de Comandos para SMBCIFS]]"
  - "[[SSLScan]]"
  - "[[Creación de Informes en LaTeX]]"
fuente: "https://nmap.org/book/man.html"
---

# Nmap — Guía Completa

**Nmap** (*Network Mapper*) es la herramienta de referencia para descubrir hosts, servicios y sistemas operativos en una red, y para auditar su configuración con el motor de scripts **NSE** (*Nmap Scripting Engine*, Lua). Dos décadas de desarrollo la convierten en el estándar *de facto* del reconocimiento: cualquier informe de pentest empieza con un mapa hecho con Nmap, y casi todas las demás herramientas del vault (Hydra, CrackMapExec, SMBMap, SSLScan) consumen sus resultados.

> [!info] El modelo mental correcto
> Nmap no es "un escáner de puertos": es un **motor de reconocimiento por fases**. Un escaneo típico ejecuta: (1) *host discovery* — ¿quién está vivo?; (2) *port scanning* — ¿qué puertos responde y en qué estado?; (3) *service detection* — ¿qué proceso responde y con qué versión?; (4) opcionalmente *NSE scripts* — ¿qué vulnerabilidades/malas configuraciones revela? Cada fase tiene sus propios flags y sus propios fallos; entenderlas por separado es la diferencia entre usar Nmap y solo ejecutarlo.

## Instalación

| Contexto | Comando |
|---|---|
| **Kali Linux** | Preinstalado (`nmap --version`) |
| Debian/Ubuntu/Parrot | `sudo apt install nmap ncat` |
| Extras de scripts | `sudo apt install NSE` incluido; bases de datos en `/usr/share/nmap/` |
| Windows/macOS | Instaladores oficiales en nmap.org |

Los escaneos SYN/OS/ACK requieren **privilegios root** (paquetes crudos). Sin root, Nmap degrada automáticamente al TCP connect (`-sT`).

## Tabla maestra: tipos de escaneo de puertos

| Flag | Nombre | Mecanismo | Requiere root | Deja handshake completo | Para qué |
|---|---|---|---|---|---|
| `-sS` | **SYN scan** (por defecto con root) | SYN → SYN/ACK (abierto) / RST (cerrado); RST final del escáner | Sí | **No** (*half-open*) | El estándar: rápido y razonablemente discreto |
| `-sT` | TCP connect | Completa el handshake 3 vías con `connect()` del sistema | No | Sí | Sin root; lo más logueable por el objetivo |
| `-sU` | UDP scan | Paquetes UDP vacíos/payload por protocolo; `ICMP port unreachable` = cerrado | Sí | n/a | Imprescindible: DNS/SNMP/T DHCP/NTP/SIP no viven en TCP |
| `-sA` | ACK scan | Paquete ACK → RST = **no filtrado**; sin respuesta/ICMP = **filtrado** | Sí | — | **No detecta puertos abiertos**: mapea reglas de firewall |
| `-sW` | Window scan | Igual que ACK pero analiza el **campo ventana TCP** del RST | Sí | — | Distingue abierto/cerrado detrás de algunos filtros |
| `-sN` | NULL scan | Paquete TCP **sin flags** | Sí | — | RFC 793: RST si cerrado, silencio si abierto |
| `-sF` | FIN scan | Paquete con solo FIN | Sí | — | Igual que NULL |
| `-sX` | Xmas scan | Flags FIN+PSH+URG ("árbol de Navidad") | Sí | — | Igual que NULL; algunos filtros los tratan distinto |
| `-sM` | Maimon scan | FIN+ACK | Sí | — | Funciona contra algunos BSD antiguos |
| `-sI` | Idle scan | Escaneo con zombi (spoofing con MAC de un host inactivo) | Sí | — | **Sin IP propia visible** — requiere zombi ideal (ID IP incremental) |
| `-sY` / `-sZ` | SCTP INIT / COOKIE-ECHO | Escaneo SCTP | Sí | — | Servicios telecom (raro fuera de entornos SS7/SIGGW) |
| `--scanflags <flags>` | Personalizado | Cualquier combinación (ej. `URGACKPSH`) | Sí | — | Evasión de firmas específicas |
| `-sL` | List scan | Solo **resuelve** los objetivos, no envía paquetes | No | — | Sanity check de rangos y DNS antes de escanear |

### Cómo interpretar los estados de puerto

| Estado | Significado | Evidencia |
|---|---|---|
| `open` | Un servicio **acepta** conexiones en ese puerto | SYN/ACK (TCP) o respuesta de aplicación (UDP) |
| `closed` | El host responde, pero **ningún servicio** escucha | RST / ICMP unreachable |
| `filtered` | Un firewall **no deja pasar** el probe; no hay respuesta utilizable | Sin respuesta o ICMP error |
| `unfiltered` | El puerto responde, pero ACK no permite decidir abierto/cerrado | RST al ACK scan — se aclara con `-sS` |
| `open\|filtered` | Abierto o filtrado (ambiguo) | Típico en UDP y en escaneos stealth |
| `closed\|filtered` | Cerrado o filtrado | Muy raro (SCTP, algunos proxies) |

> [!tip] `--reason`: por qué Nmap decidió eso
> `nmap --reason -sS ...` muestra la evidencia exacta (`syn-ack`, `reset`, `no-response`, `conn-refused`). Es la primera opción a activar cuando un resultado no cuadra con lo esperado.

### Tabla: qué tipo usar en cada situación

| Situación | Escaneo recomendado |
|---|---|
| Uso general, red local, raíz disponible | `-sS` (o dejar el default) |
| Sin root / Windows sin privilegios | `-sT` |
| Fase 1 de todo pentest (nunca sobresaltarse) | `-sS --top-ports 100` |
| Auditoría de protocolos no-TCP | `-sU --top-ports 20` |
| Mapear firewall entre tú y el objetivo | `-sA -p 80,443,22` |
| Escaneo silencioso extremo (lento, IDS sensible) | `-sN -T0` / idle scan `-sI` |
| Objetivos masivos (10k+ hosts) | Ver [[Masscan]] → refinar con Nmap |

## Host discovery — antes de tocar puertos

Sin indicación contraria, Nmap hace host discovery antes de escanear puertos (ping TCP SYN a 443/80/… + ICMP echo). Controlarlo explícitamente:

| Flag | Mecanismo | Ejemplo / uso |
|---|---|---|
| `-sn` | Solo discovery, **sin** escaneo de puertos (antes `-sP`) | `nmap -sn 192.168.1.0/24` — el "ping scan" de siempre |
| `-Pn` | Omite discovery: trata **todos** los hosts como vivos (antes `-P0`) | `-Pn` cuando el firewall bloquea el ping |
| `-Pn-` | Fuerza discovery aunque pases `-Pn` (nmap ≥ 7.80) | Recuperar discovery en scripts automatizados |
| `-PS<puertos>` | TCP **SYN** ping — SYN a los puertos indicados | `-PS80,443,22` |
| `-PA<puertos>` | TCP **ACK** ping — atraviesa firewalls con estado que responden RST al ACK | `-PA80` |
| `-PU<puertos>` | UDP ping | `-PU53,161` |
| `-PE` | ICMP echo (ping clásico) — el más bloqueado | `-PE` |
| `-PP` | ICMP timestamp — pasa algunos filtros que solo bloquean echo | `-PP` |
| `-PM` | ICMP netmask request | `-PM` |
| `-PO<n>` | IP protocol ping (proto n) | `-PO 1` (ICMP) |
| `-PY` | SCTP INIT ping | `-PY` |
| `-n` | **No** resuelve DNS (más rápido y silencioso) | Siempre en redes grandes |
| `-R` | Fuerza resolución DNS de todos los hosts | Cuando necesitas nombres |
| `--dns-servers <srv>` | Servidores DNS propios | `--dns-servers 8.8.8.8` |

> [!warning] El coste real de `-Pn`
> `-Pn` salva el escaneo de hosts filtrados, pero (a) vuelve cada IP en "viva" — un /16 con `-Pn` escanea 65 536 hosts; (b) los escaneos UDP y los scripts NSE tardarán lo máximo sobre objetivos muertos. Regla práctica: `-Pn` contra **pocos hosts conocidos**, nunca contra rangos enteros sin filtrar antes con discovery.

## Selección de puertos

| Flag | Significado | Ejemplo |
|---|---|---|
| `-p <rango>` | Puertos concretos; rangos, listas y `U:`/`T:` por protocolo | `-p 22,80,443`, `-p 1-1024`, `-p U:53,161 T:80` |
| `-p-` | Los 65 535 puertos TCP (con `-sU`, todos los UDP) | `nmap -p- -sS` |
| `-F` | Modo rápido: los 100 puertos más comunes | `-F` |
| `--top-ports <n>` | Los N más comunes según `nmap-services` (estadística real) | `--top-ports 1000` |
| `-r` | Orden secuencial en vez de aleatorio | Redes con IDS de "barrido" |
| `--port-ratio <p>` | Puertos con frecuencia ≥ p en nmap-services | `--port-ratio 0.1` |

> [!tip] La progresión de trabajo correcta
> 1) `--top-ports 100` en todos los hosts vivos → 2) `-p-` completo **solo** sobre hosts interesantes → 3) `-sU --top-ports 20` como mínimo UDP. Escanear `-p-` a toda una red es la forma clásica de convertir un reconocimiento en una tarde perdida.

## Timing y rendimiento

### Plantillas `-T0`…`-T5`

| Template | Nombre | Perfil |
|---|---|---|
| `-T0` | Paranoid | Un paquete cada 5 min; evasión de IDS extrema; laboratorio, no práctica |
| `-T1` | Sneaky | ~15 s entre probes |
| `-T2` | Polite | ~0.4 s — redes sensibles a carga |
| `-T3` | Normal | Por defecto; balance |
| `-T4` | Aggressive | El estándar de trabajo en LAN/lab: paralelismo alto |
| `-T5` | Insane | Máximo paralelismo; pierde resultados en redes con pérdida |

### Control fino (lo que realmente se usa en la práctica)

| Flag | Significado | Ejemplo |
|---|---|---|
| `--min-rate <pps>` | Piso de paquetes/segundo — **elimina la espera adaptativa** | `--min-rate 500` (duración predecible) |
| `--max-rate <pps>` | Techo de paquetes/segundo | `--max-rate 200` |
| `--min-parallelism` / `--max-parallelism` | Probes simultáneos por host | `--min-parallelism 10` |
| `--min-hostgroup` / `--max-hostgroup` | Tamaño del lote de hosts | `--max-hostgroup 64` |
| `--host-timeout <t>` | Máximo por host — evita que un host muerto con `-Pn` frene todo | `--host-timeout 30m` |
| `--max-retries <n>` | Reintentos de probe antes de dar por filtrado | `--max-retries 2` |

```bash
# Escaneo de puerto completo con duración predecible sobre 256 hosts
nmap -sS -p- --min-rate 1000 --max-retries 1 -T4 -n -oA full 192.168.1.0/24
```

## Detección de servicio y versión

| Flag | Significado |
|---|---|
| `-sV` | Activa *version detection*: sondea cada puerto abierto y compara con `nmap-service-probes` |
| `--version-intensity <0-9>` | Profundidad (por defecto 7). 0 = solo el probe activo; 9 = todos los probes, el más fiable y lento |
| `--version-light` | Atajo de intensidad 2 (los 2 probes principales) |
| `--version-all` | Intensidad 9 + versiones alternativas |
| `--version-trace` | Muestra cada probe/respuesta (depuración) |
| `-O` | Detección de SO por fingerprint TCP/IP (`nmap-os-db`) |
| `--osscan-limit` | Detección de SO solo contra hosts "prometedores" (≥1 puerto abierto) |
| `--osscan-guess` | Fuerza el emparejamiento con porcentaje de confianza |
| `-A` | **Atajo total**: `-sV -O -sC --traceroute` |

### Interpretar la salida de `-sV`

```
PORT     STATE SERVICE  VERSION
445/tcp  open  microsoft-ds  Windows Server 2019 microsoft-ds
8080/tcp open  http         Apache Tomcat 9.0.30
```

| Campo | Qué aporta al informe |
|---|---|
| `microsoft-ds` + versión | Base para decidir EternalBlue, SMB signing, enumeración con [[smbclient Cliente de Línea de Comandos para SMBCIFS]] |
| `Apache Tomcat 9.0.30` | Guía a CVEs concretas de esa línea y a la vía de subida WAR de [[Msfvenom - Herramienta de Generación de Payloads]] |
| `OpenSSH 7.4` | Posibles vulnerabilidades de la versión, soporte de usuarios/keys |
| `(service info: "os: ...")` | El service detection a veces identifica el SO aunque no pidas `-O` |

> [!tip] `--version-intensity 9` tiene coste
> En la práctica: intensidad por defecto (7) para el escaneo inicial; subir a 9 solo contra los hosts/ports de interés cuando la versión exacta abre una puerta concreta (p. ej. decidir un exploit).

## NSE — Nmap Scripting Engine

Los scripts NSE (Lua) viven en categorías, se seleccionan por categoría, nombre o expresión, y reciben argumentos con `--script-args`.

### Categorías

| Categoría | Contenido | Riesgo |
|---|---|---|
| `default` | Equivale a `-sC` — seguros, rápidos, siempre útiles | Bajo |
| `safe` | No intrusivos, sin impacto | Bajo |
| `discovery` | Enumeración de configuraciones | Bajo-medio |
| `vuln` | Detección de vulnerabilidades conocidas | Medio (puede ser ruidoso) |
| `auth` | Prueba de credenciales válidas | Medio |
| `brute` | Fuerza bruta | **Alto** — requiere autorización explícita |
| `intrusive` | Pueden romper servicios o generar muchísimo tráfico | Alto |
| `exploit` | **Explotan** vulnerabilidades | Crítico |
| `dos` | Denegación de servicio | Crítico — solo laboratorio |
| `malware` | Detección de backdoors | Bajo |
| `external` | Dependen de recursos externos (consultas whois/OSVDB) | Variable |
| `broadcast` | Descubrimiento por broadcast (NetBIOS, mDNS, DHCP…) | Bajo |

### Selección de scripts

| Sintaxis | Ejemplo |
|---|---|
| Nombre | `--script http-enum` |
| Lista | `--script smb-os-discovery,ssl-cert` |
| Categoría | `--script vuln` |
| Comodín | `--script "http-*"` |
| Booleanos | `--script "default and not intrusive"` |
| Ruta directa | `--script /ruta/script.nse` |

### Argumentos y traza

| Flag | Uso |
|---|---|
| `--script-args k=v,...` | `--script-args smbuser=admin,smbpass=Pass123,http.useragent="Mozilla/5.0"` |
| `--script-args-file f` | Argumentos desde archivo |
| `--script-timeout <t>` | Corte por script colgado |
| `--script-trace` | Muestra todo el tráfico que envía/recibe el script |
| `--script-updatedb` | Regenera `scripts.db` tras añadir scripts propios |

### Catálogo de recetas NSE (las que se usan de verdad)

```bash
# Enumeración web — directorios virtuales, headers, métodos, robots
nmap -sV --script http-enum,http-headers,http-methods,http-robots.txt -p80,443 10.10.10.50

# Toma de decisiones sobre SMB (¿signing? ¿qué SO? ¿shares?)
nmap -p445 --script smb-os-discovery,smb-security-mode,smb-enum-shares,smb-enum-users 10.10.10.50

# Vulnerabilidades SMB clásicas (EternalBlue, etc.)
nmap -p445 --script smb-vuln* --script-args=unsafe=1 10.10.10.50

# Auditoría TLS rápida desde Nmap (complementa a SSLScan/SSLyze)
nmap -p443 --script ssl-cert,ssl-enum-ciphers,ssl-heartbleed,sslv3 10.10.10.50

# Detección de vulnerabilidades general + versiones
nmap -sV --script vuln -Pn 10.10.10.50

# Fuerza bruta SMB autorizada (¡ruidosa!)
nmap -p445 --script smb-brute --script-args userdb=users.txt,passdb=rockyou-top.txt 10.10.10.50

# Broadcast: descubrir hosts y nombres por broadcast
nmap --script broadcast-netbios-name-discovery,broadcast-dhcp-discover 192.168.1.255

# DNS enumeration
nmap --script dns-brute,dns-zone-transfer -p53 10.10.10.53
```

| Script | Qué entrega |
|---|---|
| `http-enum` | Directorios virtuales con posible CVE asociada (vía `http-vuln-cve2017-5638` etc.) |
| `smb-os-discovery` | SO, workgroup, dominio, fecha, si es controlador de dominio |
| `smb-security-mode` | ¿Signing required/disabled? (base de relay attacks) |
| `ssl-enum-ciphers` | Calificación A-F de cada protocolo con sus cifrados |
| `vuln` | Hallazgos con CVE y confianza (`VULNERABLE`, `Likely VULNERABLE`) |

> [!warning] `unsafe=1` y categorías peligrosas
> Muchos scripts de detección de vulnerabilidades (p. ej. `smb-vuln-ms17-010`) envían payloads de prueba potencialmente crasheros y solo ejecutan la parte arriesgada con `--script-args unsafe=1`. Ejecutar siempre primero **sin** el argumento; activarlo solo con autorización y contra hosts descartables.

## Formatos de salida y análisis con línea de comandos

| Flag | Formato | Uso principal |
|---|---|---|
| `-oN archivo` | Normal (humano) | Lectura, informes |
| `-oX archivo` | **XML** | Procesamiento programático, Metasploit, herramientas de informe |
| `-oG archivo` | *Greppable* — una línea por host con `Ports:` | `grep`/`awk` rápido (legacy) |
| `-oS archivo` | "Script kiddie" | Broma del autor; no usar |
| `-oA base` | **Todos** anteriores: `base.nmap`, `base.xml`, `base.gnmap` | El estándar: nunca se arrepiente quien guarda todo |

### Extracción con `grep`/`awk` (formato greppable)

```bash
# Hosts vivos desde un -oG
grep "Status: Up" escaneo.gnmap | awk '{print $2}'

# Puertos abiertos: una lista "ip:puerto" lista para Nmap posterior o Hydra
grep "/open/" escaneo.gnmap | \
  awk -F'[ /]+' '{for(i=6;i<=NF;i+=3) print $2":"$i}' | sort -u
```

### JSON con `jq` (desde `-oX`)

```bash
# XSLT directo de Nmap a HTML para el informe
xsltproc escaneo.xml -o informe.html

# jq sobre el XML convertido (o usando el plugin nmap para jq):
# lista puertos abiertos con servicio y versión
jq -r '.nmaprun.host[] | select(.status.state=="up") |
  .address[0].addr as $ip |
  .ports.port[]? | select(.state=="open") |
  "\($ip)\t\(.portid)/\(.protocol)\t\(.service.name)\t\(.service.version // "")"' escaneo.json

# Solo hosts con 445 abierto (para CrackMapExec)
jq -r '.nmaprun.host[] | select(any(.ports.port[]?; .portid=="445" and .state=="open")) | .address[0].addr' escaneo.json > smb-hosts.txt
```

> [!tip] XML es el formato canónico
> El XML es el único formato estable para ingestión (el greppable cambió de formato entre versiones y rompió scripts famosos). Flujo recomendado: `-oA` siempre; humano desde `.nmap`, automatización desde `.xml` (+`jq`/`xsltproc`).

## Interpretar la salida de un escaneo completo

```
Nmap scan report for 10.10.10.50
Host is up (0.0042s latency).                    ← calidad de la medición; >0.5s = sospecha de tarpitting
Not shown: 996 closed tcp ports (reset)          ← "filtered" en vez de "closed" = firewall
PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.5   ← versión con parche exacto
80/tcp   open  http          Apache httpd 2.4.41
445/tcp  open  microsoft-ds  Windows 10
3306/tcp open  mysql         MySQL 8.0.28                        ← service info añade OS hints
Device type: general purpose|router
Running: Linux 4.X|5.X                                                   ← de -O
OS details: Linux 4.15 - 5.6 (conf: 92%)                        ← confianza del fingerprint
```

| Señal | Lectura correcta |
|---|---|
| Muchos `filtered` | Firewall activo — repetir con `-Pn`, `-PS`, o `-sA` para mapear reglas |
| `open\|filtered` en UDP | Normal: subir con payloads específicos (`--script` de versiones) o paciencia |
| OS details con confianza baja | No poner el SO en el informe como hecho — es una **estadística** |
| Latencia alta e inestable | Posible IDS de rate-limiting/tarpitting: bajar a `-T2` y verificar resultados repetidos |

## Evasión de firewalls e IDS

| Técnica | Flag | Mecanismo / cuándo sirve |
|---|---|---|
| Fragmentación | `-f` (8 bytes) / `--mtu 16|24|32|…` | Evita filtros que inspeccionan solo el primer fragmento (firewalls viejos, ACLs simples) |
| Decoys | `-D RND:5` o `-D IP1,ME,IP2` | Los logs del target ven múltiples orígenes; `ME` marca la posición real. Requiere ruta de retorno plausible |
| Puerto origen | `--source-port 53` | Tráfico desde puertos privilegiados (DNS) pasa filtros mal configurados que confían en el puerto |
| Longitud del payload | `--data-length 100` | Bytes aleatorios al final del paquete: evade firmas por tamaño exacto |
| TTL personalizado | `--ttl 64` | Igualar TTL de la red local — evita detección por TTL anómalo |
| MAC aleatoria | `--spoof-mac 0` | Ofusca el identificador de capa 2 |
| IP falsificada | `-S <ip>` | Requiere root, red y conocimiento del entorno; casi siempre inútil sin sniffing |
| Flags TCP | `--scanflags URGACKPSH` | Paquetes que ninguna firma estándar espera |
| Timing lento | `-T1`/`-T2`, `--max-rate` | Evita IDS por umbral de velocidad |
| Sin resolución DNS | `-n` | Elimina consultas DNS ruidosas desde tu IP |
| Alternativa total | [[Masscan]] + -e/pcap | Si el firewall bloquea Nmap por fingerprint de paquete |

> [!warning] Evasión ≠ invisibilidad
> Un firewall moderno con inspección de estado y correlación de flujo detecta casi todas estas técnicas en aggregate: fragmentos que reensamblan a un SYN, decoys sin respuesta posterior, TTL inconsistente con la ruta. La evasión realista es **menos ruido + timing correcto**, no flags exóticos.

## Rendimiento a escala

| Escala | Estrategia |
|---|---|
| < 50 hosts | Nmap directo: `-sS -T4 -F` y refinado con `-p-` |
| 50–1 000 hosts | `-sn` primero, `-sS --top-ports 1000 --min-rate` sobre vivos, `-oA` |
| 1 000+ hosts / rangos grandes | [[Masscan]] a ~1 000 pps con lista de puertos → resultados a Nmap con `-iL vivos.txt -sV` |
| Puertos UDP masivos | Solo `--top-ports 20 -sU` con `--host-timeout`; el UDP completo es un pozo de tiempo |
| Redes con pérdida | Evitar `-T5`; `--max-retries 2` y confiar en la repetición del escaneo completo |

Costes por defecto: Nmap espera ~un minuto por puerto filtrado y reintenta 2-3 veces. Un `-p-` con cientos de hosts filtrados **antes** de `--host-timeout`/`--min-rate` es el cuello de botella típico.

## Nmap vs Masscan

| Aspecto | Nmap | Masscan |
|---|---|---|
| Velocidad | ~1 000 pps práctico | > 100 000 pps (async, raw) |
| Precisión de estado | Alta (reintentos, fingerprints) | Baja en redes con pérdida — genera falsos positivos |
| Service/version detection | `-sV` completo | No |
| OS detection | `-O` | No |
| NSE | Sí (miles de scripts) | No |
| Estructura de salida | XML/greppable/normal | Propia (bucket sort) + pcap |
| Rol en un engagement | **Análisis** | **Descubrimiento bruto** |

Flujo combinado estándar:

```bash
masscan -p80,443,445,22,8080 10.10.10.0/24 --rate 1000 -oL masivo.txt
awk '{print $4}' masivo.txt | sort -u > vivos.txt
nmap -sV -sC -iL vivos.txt -oA refinado
```

## Recetas — los comandos que se escriben cada día

```bash
# 1. Reconocimiento inicial (lo primero en cualquier alcance)
nmap -sS --top-ports 100 -T4 -n -oA recon 192.168.1.0/24

# 2. Descubrimiento de hosts rápido (sin tocar puertos)
nmap -sn -n 192.168.1.0/24 -oG vivos.gnmap

# 3. Puerto completo con duración predecible
nmap -sS -p- --min-rate 1000 --max-retries 1 -T4 -n -oA full <ip>

# 4. Ping scan evitando ICMP filtrado
nmap -PS80,443,445,22 -sn -n 10.10.10.0/24

# 5. Servicios y SO de un host ya descubierto
nmap -sV -O --osscan-guess -p- -T4 -oA detalle 10.10.10.50

# 6. Escaneo agresivo completo (lab/alcance amplio)
nmap -A -T4 -Pn -oA agresivo 10.10.10.50

# 7. Host que no responde al ping
nmap -Pn -sV --top-ports 1000 10.10.10.50

# 8. Mapeo de firewall
nmap -sA -p 21,22,80,443,445,3389 -T4 10.10.10.0/24

# 9. UDP esencial
nmap -sU --top-ports 20 -T4 -oA udp 10.10.10.0/24

# 10. Enumeración SMB lista para CrackMapExec
nmap -p445 --script smb-os-discovery,smb-security-mode,smb-enum-shares 10.10.10.0/24 -oA smb

# 11. Auditoría TLS
nmap -p443,8443 --script ssl-cert,ssl-enum-ciphers,ssl-heartbleed 10.10.10.50

# 12. Silencioso para IDS sensible
nmap -sN -T1 -f --source-port 53 -n 10.10.10.50

# 13. Pipeline a Hydra (extraer host:puerto ssh/http)
grep "/open/" escaneo.gnmap | awk -F'[ /]+' '{print $2":"$6}' > targets-hydra.txt
```

## Errores comunes (con mensajes exactos)

| Mensaje / síntoma | Causa | Fix |
|---|---|---|
| `TCP/IP fingerprinting requires root privileges` | Sin privilegios | `sudo nmap ...` (o aceptar `-sT` y `-O` deshabilitado) |
| `QUITTING!` tras `Failed to resolve` | Nombre de host no resoluble y usado como target | Usar IP, o pasar DNS con `--dns-servers`, o comprobar `-n` no rompe algo |
| Todos los puertos `filtered` con `-Pn` | Firewall + discovery omitido en hosts muertos | Quitar `-Pn` o filtrar hosts vivos antes |
| `Host seems down. If it is really up...` | Firewall responde RST a SYN de discovery | Aceptar el flag sugerido: `--send-ip` / usar `-Pn` justificado |
| Escaneo UDP que tarda horas | 65 535 puertos UDP + reintentos | `--top-ports 20`, `--host-timeout`, `-T4` |
| NSE no ejecuta el script | Nombre mal escrito o DB desactualizada | `--script-updatedb`; verificar con `ls /usr/share/nmap/scripts/<s>.nse` |
| Resultados que no coinciden entre ejecuciones | Rate limiting del objetivo/IDS | Repetir con `-T2 --max-retries 3` y comparar |
| Los decoys no engañan | El target ve SYN pero no el tráfico posterior | Los decoys solo estorban en logs; para evasión real usar timing/fragmentación |

## Cheatsheet

```bash
# Discovery
nmap -sn <rango>                 # ¿quién vive?
nmap -PS80,443 -sn <rango>       # por TCP (ICMP bloqueado)

# Escaneo
nmap -sS --top-ports 100 -T4     # recon inicial
nmap -sS -p- --min-rate 1000     # completo predecible
nmap -sU --top-ports 20          # UDP esencial
nmap -sA -p 80,443               # firewall: filtrado?
nmap -Pn -sV <ip>                # host que no pingea
nmap -A -T4 <ip>                 # todo (lab)

# NSE
nmap -sC <targets>               # default scripts
nmap --script vuln -Pn <ip>      # vulnerabilidades
nmap --script "http-*" -p80 <ip>
nmap --script-args k=v --script <s> <ip>

# Salida
nmap ... -oA salida              # .nmap + .xml + .gnmap
grep "/open/" salida.gnmap       # puertos abiertos
xsltproc salida.xml -o r.html    # informe HTML

# Evasión (autorizada)
nmap -f --data-length 64 --source-port 53 -T2 <ip>
```

## Referencias

- Manual oficial: <https://nmap.org/book/man.html>
- Guía de NSE: <https://nmap.org/book/nse.html>
- Libro "Nmap Network Scanning" (Gordon Lyon, libre): <https://nmap.org/book/>
- NSE scripts search: <https://nmap.org/nsedoc/>
- Análisis de los paquetes que envía Nmap: [[Wireshark y Tshark]]
- Conceptos de reconocimiento: [[Reconocimiento y Escaneo de Redes y Web]]

## Relacionadas

- [[Masscan]] — descubrimiento bruto a velocidad imbatible
- [[netdiscover]] / [[Arp Scan Herramienta de Descubrimiento]] — descubrimiento ARP en LAN
- [[Enumeración de Servicios y CMS]] — qué hacer con cada puerto abierto
- [[smbclient Cliente de Línea de Comandos para SMBCIFS]] / [[SMBMap]] / [[CrackMapExec Guía de Uso]] — la cadena SMB
- [[SSLScan]] / [[SSLyze Analizador SSL TLS]] — auditoría TLS profunda (Nmap solo hace el primer sondeo)
- [[THC Hydra]] — fuerza bruta sobre los servicios descubiertos
- [[Creación de Informes en LaTeX]] — cómo documentar el mapa de red
