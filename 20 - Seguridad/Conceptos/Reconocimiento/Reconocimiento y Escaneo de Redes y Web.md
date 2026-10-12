---
tags:
  - ciberseguridad
  - reconocimiento
  - reconocimiento-web
tipo: concepto
estado: terminado
---

# Reconocimiento y Escaneo de Redes y Web

El reconocimiento es la fase que **más información y menos riesgo** aporta: se divide en **pasivo** (no tocas al objetivo; solo consultas fuentes de terceros) y **activo** (le mandas paquetes y deja rastro). Esta nota recorre las dos mitades —OSINT, descubrimiento de hosts, escaneo de puertos, evasión, recon web— y cierra con la disciplina de documentación y con las defensas que el lado contrario despliega contra todo esto.

> [!warning] Alcance y autorización
> Todo escaneo activo genera tráfico hacia el objetivo y **queda registrado** en sus IDS/logs. Sin autorización escrita (o fuera del *scope* de un bug bounty) constituye un acceso no autorizado. En programas tipo HackerOne, el *scope* define dominios, IPs y tipos de vulnerabilidad elegibles: salirse de él anula la recompensa y puede acarrear consecuencias legales — <https://www.hackerone.com/>.

## 1. Fases: pasivo vs. activo

| | **Pasivo (OSINT)** | **Activo** |
|---|---|---|
| Toca el objetivo | No (usa terceros: DNS, CDNs, índices) | Sí — paquetes, logs, alertas |
| Rastro | Indirecto (consultas DNS salientes) | Directo y correlacionable |
| Fuentes | whois, DNS, CT logs, motores de búsqueda, breach data, ASN | Ping, SYN, UDP, ARP, HTTP |
| Cuándo | Siempre primero | Tras delimitar alcance |
| Ejemplo | `subfinder`, `crt.sh`, Google dorks | `nmap -p- --open` |

### Fuentes pasivas de mayor a menor rendimiento

| Fuente | Qué revela | Herramienta |
|---|---|---|
| **Whois / IRR** | Propietario del bloque, contactos, fechas | `whois dominio.com`, RDAP |
| **DNS (consultas públicas)** | Subdominios, MX/TXT (SPF, verify tokens), NS | `dig`, `host`, `subfinder`, `amass` |
| **CT logs** (certificados) | Subdominios históricos y actuales en claro | `crt.sh`, `ctfr`, Censys |
| **ASN / BLOQUES** | Rangos IP reales de la organización | `bgp.he.net`, `asnmap` |
| **Google/Bing dorks** | Archivos indexados, paneles, backups | ver §2.2 |
| **Breach data** | Credenciales comprometidas del dominio | DeHashed, haveibeenpwned (§2.5) |
| **GitHub / pastes** | Tokens, `id_rsa`, `.env` filtrados | Búsqueda de código |
| **Reversa de imágenes** | Personas, ubicaciones, equipos | PimEyes, Google Lens (§2.4) |

## 2. Reconocimiento pasivo (OSINT)

### 2.1 DNS y subdominios

Los subdominios suelen albergar lo peor protegido: entornos de staging, paneles de terceros, servicios internos mal cerrados. Dos enfoques:

| Pasivo | Activo |
|---|---|
| Motores de búsqueda (`site:`), CT logs (`crt.sh`, **CTFR**), PassiveTotal, Censys, Phonebook.cz, IntelX, sublist3r | Fuzzing de fuerza bruta contra el dominio: [[Gobuster]] (`gobuster dns -d`), [[WFuzz]], `puredns`, `massdns` |
| Cero tráfico al objetivo | Rápido y exhaustivo, pero ruidoso (consultas al DNS autoritativo) |

```bash
# Pasivo: subdominios en certificados (gratis y sorprendentemente completo)
curl -s "https://crt.sh/?q=%25.empresa.com&output=json" | jq -r '.[].name_value' | sort -u
# CTFR:  https://github.com/UnaPibaGeek/ctfr
# Phonebook / IntelX: paneles de datos OSINT
```

> [!info] Por qué los CT logs son tan útiles
> Cualquier certificado emitido (Let's Encrypt incluido) queda en un **registro de transparencia público e inmutable**. Aunque el subdominio se haya dado de baja, el certificado delata que existió — y suelen aparecer staging, vpn o pruebas que nadie pensó que fueran visibles.

### 2.2 Google Dorks (Google Hacking)

Búsqueda avanzada con operadores para encontrar lo que no sale en los resultados normales: ficheros sensibles, paneles indexados, backups, listados de directorio. **No es ilegal en sí misma**, pero acceder a lo encontrado sin autorización sí lo puede ser.

**Operadores básicos**

| Operador | Descripción | Ejemplo |
|---|---|---|
| `""` | Búsqueda exacta | `"palabra exacta"` |
| `-` | Excluir término | `wordpress -wp-content` |
| `OR` | Uno u otro | `wordpress OR joomla` |
| `AND` | Ambos (implícito) | `wordpress vulnerability` |
| `*` | Comodín | `admin * login` |
| `()` | Agrupación | `(wordpress OR drupal) login` |

**Operadores avanzados**

| Operador | Descripción | Ejemplo |
|---|---|---|
| `site:` | Limita al sitio | `site:example.com` |
| `filetype:` | Tipo de fichero | `filetype:pdf` |
| `ext:` | Extensión | `ext:php` |
| `inurl:` | Término en la URL | `inurl:admin` |
| `intitle:` | Término en el título | `intitle:"index of"` |
| `intext:` | Término en el cuerpo | `intext:password` |
| `cache:` | Versión en caché | `cache:example.com` |
| `link:` / `related:` / `info:` | Enlaces, sitios similares, info | `link:example.com` |

**Fechas y rangos**

| Operador | Descripción | Ejemplo |
|---|---|---|
| `..` | Rango numérico | `price 100..500` |
| `before:` / `after:` | Fecha | `before:2020-01-01` |
| `daterange:` | Fecha juliana | `daterange:2458150-2458155` |

**Dorks de oro para reconocimiento web**

```text
site:example.com filetype:pdf                 # documentos expuestos
site:example.com inurl:admin                  # paneles
site:example.com "index of"                   # listados de directorio
site:example.com ext:env OR ext:bak OR ext:sql # backups y configs
intitle:"index of" "wp-config.php"            # configuraciones expuestas
inurl:php?id=                                  # parámetros candidatos a SQLi
site:pastebin.com example.com                 # fugas en pastes
```

### 2.3 Correos electrónicos y personas

| Herramienta | Qué hace | Nota |
|---|---|---|
| **hunter.io** | Correos asociados a un dominio + patrón de nomenclatura | Base para phishing/credenciales |
| **intelx.io** | Búsqueda en datasets de filtraciones (mail, usuarios, docs) | También subdominios |
| **phonebook.cz** | Correos y contactos por empresa/dominio | Pasiva, gratuita |
| **Clearbit Connect** | Contactos en tiempo real dentro de Gmail | OSINT ligera |
| **theHarvester / recon-ng** | Agrega buscadores, PGP, SHODAN en un solo run | Automatización |

> [!tip] El patrón de nombres es un hallazgo
> Si de `juan.perez@empresa.com` y `ana.lopez@empresa.com` se deduce `nombre.apellido`, la enumeración de usuarios para fuerza bruta o phishing queda resuelta sin tocar el objetivo. Ese patrón también alimenta wordlists dirigidas (`cewl`, `username-anarchy`).

### 2.4 Reconocimiento de imágenes

**PimEyes** (<https://pimeyes.com/en>) busca rostros por fotografía en índices de imágenes de Internet: localiza personas, perfiles sociales y datos asociados. Potente para verificación de identidad e investigación; su uso masivo sin consentimiento plantea problemas serios de privacidad.

> [!warning] Ética del reconocimiento facial
> La búsqueda inversa facial puede revelar datos personales de terceros ajenos al alcance. Úsala solo con fines legales y proporcionales (investigación con consentimiento, verificación propia); el uso indiscriminado puede vulnerar normativas de protección de datos (GDPR).

### 2.5 Credenciales y brechas

Los leaks de bases de datos (brechas públicas o en venta) permiten probar credenciales reales en lugar de diccionarios genéricos. **DeHashed** (<https://www.dehashed.com/>) y similares indexan emails + passwords filtrados.

> [!warning] Uso responsable
> Consultar credenciales filtradas solo se justifica en pruebas autorizadas (¿están comprometidas las cuentas de la organización?) o en investigación forense. Usarlas contra sistemas ajenos es delito.

### 2.6 Identificación de tecnologías

| Herramienta | Tipo | Aporte |
|---|---|---|
| **[[WhatWeb Herramienta de Fingerprinting\|WhatWeb]]** | CLI | 180+ plugins: CMS, libs, banners; `-v` para detalle |
| **Wappalyzer** | Extensión de navegador | Tecnologías de la página visitada sin escaneo |
| **BuiltWith** | Web | Tecnologías + estadísticas del sitio |
| **Wappalyzer CLI / httpx** | CLI | Integrable en pipelines |

Conocer el stack (CMS, servidor, lenguaje, librerías JS) orienta el esfuerzo a vulnerabilidades **concretas** de esa tecnología — continuación directa en [[Enumeración de Servicios y CMS]].

### 2.7 Fingerprint de servidor y TLS (pasivo→semipasivo)

```bash
curl -sI https://IP | grep -iE 'server|x-powered-by|set-cookie'
openssl s_client -connect IP:443 </dev/null 2>/dev/null | openssl x509 -noout -subject -dates
```

Ver [[SSLScan]], [[OpenSSL Comandos Principales]] y §3.3 de [[Enumeración de Servicios y CMS]].

## 3. Reconocimiento activo: descubrimiento de hosts

### 3.1 Qué protocolo usar para descubrir

| Protocolo | Ventaja | Límite |
|---|---|---|
| **ARP** (red local) | El más fiable: obligatorio en L2, no se suele filtrar | Solo en tu segmento |
| **ICMP echo** | Rápido y barato | Windows lo bloquea por defecto; filtrado frecuente |
| **TCP SYN/ACK** a puertos comunes | Descubre hosts que filtran ICMP | Más tráfico, hay que elegir puertos |
| **UDP** breve (53, 161, 123, 445) | Sorprende a firewalls "solo pasivos" | Lento (timeouts) y con falsos negativos |

### 3.2 Herramientas

| Herramienta | Protocolo | Ejemplo |
|---|---|---|
| [[Nmap Guía Completa\|Nmap]] | ICMP + TCP + ARP | `nmap -sn 192.168.0.0/24` |
| [[Arp Scan Herramienta de Descubrimiento\|arp-scan]] | ARP puro | `arp-scan -I eth0 --localnet` |
| [[netdiscover]] | ARP pasivo/activo | `netdiscover -i eth0` |
| [[Masscan]] | TCP a velocidad de datacenter | `masscan -p21,22,139,445 -Pn 192.168.0.0/16 --rate=10000` |

> [!tip] `-sn` de Nmap (antes `-sP`)
> No escanea puertos: combina ICMP echo, TCP SYN a puertos comunes (443) y **ARP en redes locales** para decidir si un host está activo. Es el ping sweep rápido por defecto.

**Sweep con script** (bash puro, en paralelo):

```bash
#!/bin/bash
function ctrl_c() { echo -e "\n\n[!] Saliendo.....\n\n"; tput cnorm; exit 1; }
trap ctrl_c SIGINT
tput civis
for i in $(seq 1 254); do
  timeout 1 bash -c "ping -c 1 192.168.1.$i" &>/dev/null && echo -e "Host - 192.168.1.$i - Active" &
done
wait; tput cnorm
```

**Cuando ICMP está bloqueado** (típico en Windows): sondear puertos indicativos:

```bash
#!/bin/bash
function ctrl_c() { echo -e "\n\n[!] Saliendo.....\n\n"; tput cnorm; exit 1; }
trap ctrl_c SIGINT
tput civis
for i in $(seq 1 254); do
  for port in 21 22 23 25 139 445 80 8080; do
    timeout 1 bash -c "echo '' > /dev/tcp/192.168.1.$i/$port" 2>/dev/null && \
      echo -e "Host - 192.168.1.$i - Active" &
  done
done
wait; tput cnorm
```

> [!info] ¿Por qué Windows no responde al ping?
> El firewall de Windows bloquea de fábrica los ICMP echo entrantes para reducir la superficie de descubrimiento. Por eso el descubrimiento profesional combina siempre ARP + ICMP + TCP SYN: cada capa cubre los agujeros de la anterior.

## 4. Escaneo de puertos: mecánica y decisión

### 4.1 Qué significa cada combinación de tramas

| Tipo de escaneo | Envía | Respuesta esperada | Puerto… | Flags Nmap |
|---|---|---|---|---|
| **SYN ("stealth")** | SYN | SYN/ACK → RST (no completa el handshake) | Abierto | `-sS` |
| **Connect** | SYN completo vía `connect()` | Handshake completo → RST/FIN | Abierto | `-sT` |
| **ACK** | ACK | RST sin estado previo | Filtrado vs no filtrado | `-sA` |
| **NULL/FIN/Xmas** | sin flags / FIN / PSH+URG+FIN | RST (cerrado) o nada (abierto, Unix) | Ocultación | `-sN`, `-sF`, `-sX` |
| **UDP** | datagrama UDP | `ICMP port unreachable` = cerrado; payload = abierto | Abierto/filtered | `-sU` |
| **Window / Maimon** | variantes de ACK/FIN | Firma del window | Windows | `-sW`, `-sM` |

| Clasificación | Significado práctico |
|---|---|
| **Abierto** | Alguien escucha: servicio candidato |
| **Cerrado** | Host vivo, puerto no escucha (RST) — confirma que el host está up |
| **Filtrado** | Sin respuesta / ICMP unreachable: firewall en medio |
| **Open\|Filtered** | Ambiguo (típico de UDP sin respuesta) |

### 4.2 Dos escaneos de referencia (los de siempre)

```bash
# 1) Barrido completo de puertos, rápido, solo abiertos
nmap -p- --open -sS --min-rate 5000 -v -n -Pn <IP> -oG allPorts
# -p-            1–65535
# --open         solo puertos abiertos (menos ruido en la salida)
# -sS            SYN scan (mitad de handshake)
# --min-rate 5000  ≥5000 paquetes/s: termina antes, es más ruidoso
# -v             progreso en vivo     -n   sin DNS (más rápido)
# -Pn            no ping previo (hosts que filtran ICMP)
# -oG allPorts   salida grepable para parsear con awk/grep

# 2) Versiones + scripts en los puertos abiertos
nmap -p<puertos> -sCV <IP> -oN target
# -p80,443,22   puertos concretos
# -sC           scripts NSE default    -sV   detección de versión
# -oN target    salida normal legible
```

### 4.3 Tabla de decisión: qué escaneo elegir

| Situación | Elección | Por qué |
|---|---|---|
| Red local desconocida, prioridad velocidad | `nmap -sn`, arp-scan, netdiscover | ARP es inapelable en L2 |
| Barrido inicial, hosts que filtran ICMP | `-Pn -sS -p-` | No depende del ping |
| Producción sensible, bajo ruido | `-sT` con `--max-rate 200`, horario acordado | Connect + ritmo lento = menos anomalías |
| Firewalls con inspección de estado | `-f`, `--data-length`, decoys, `-sA` | Ver §5 |
| Hosts Unix y quieres pasar desapercibido | `-sN`/`-sF` (null/FIN) | Muchos firewalls los dejan pasar |
| Servicios UDP críticos (DNS, SNMP, NTP) | `-sU --top-ports 20` | TCP no los ve |
| Redes grandes (millones de hosts) | [[Masscan]] (`--rate`) | Nmap no llega a esa escala |
| Solo descubrir puertos comunes rápido | `--top-ports 1000` | 80% del valor, 2% del tiempo |

### 4.4 Flags de temporización (los que controlan ruido vs velocidad)

| Flag | Efecto | Uso |
|---|---|---|
| `--min-rate N` / `--max-rate N` | Fuerza un mínimo/máximo de paquetes/s | `--min-rate 5000` para acabar rápido; `--max-rate 50` para pasar invisible |
| `--min-parallelism` / `--max-parallelism` | Exploraciones concurrentes | Más paralelismo = más ruido |
| `-T0…-T5` (paranoid…insane) | Atajos de tempo | `-T2` conservador, `-T4` agresivo, `≥T4` en producción = mala idea |
| `--host-timeout 30m` | Abandona hosts lentos | Redes con hosts muertos que tardan |
| `--initial-rtt-timeout 100ms` | Timeout del primer SYN | Bajarlo acelera hosts caídos |
| `-PE -PS80,443 -PA` | Vectores de host discovery | Ajustar según filtros |

> [!tip] La regla de oro del timing
> En laboratorio: `--min-rate 5000` y a correr. En un objetivo real: **rate bajo, horario acordado y `-sT` si hay duda** — un escaneo a 5 000 pps dispara IDS en segundos y puede tumbar equipos viejos.

### 4.5 Escaneo sin herramientas: `/dev/tcp` de Bash

`/dev/tcp/HOST/PUERTO` (y `/dev/udp/...`) son pseudo-ficheros que **Bash** interpreta como conexiones de red: redirigir hacia ellos ejecuta el handshake. Ideal cuando no hay nmap instalado (singles, contenedores mínimos):

```bash
echo > /dev/tcp/10.10.10.5/80 2>/dev/null && echo "abierto"
```

Barrido completo en paralelo:

```bash
#!/bin/bash
function ctrl_c() { echo -e "\n\n[!] Saliendo...\n\n"; exec 3<&-; exec 3>&-; tput cnorm; exit 1; }
trap ctrl_c SIGINT
declare -a ports=($(seq 1 65535))

function checkPort() {
  tput civis
  bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null
  [ $? -eq 0 ] && echo "[+] Host $1 - Port $2 - Active"
  exec 3<&-; exec 3>&-
}

if [ $1 ]; then
  for port in ${ports[@]}; do checkPort $1 $port & done
else
  echo -e "\n[!] Uso: $0 <ip-address>\n"
fi
wait
```

> [!info] Mecánica del descriptor
> `exec 3<>/dev/tcp/host/port` pide a Bash abrir un fd 3 como socket TCP: si el kernel completa el SYN/ACK, el descriptor se abre (código 0) y se puede leer/escribir; si el puerto está cerrado, recibe RST y Bash devuelve error. Es exactamente lo que hace `nc` o `nmap -sT`, en 40 bytes de script.

## 5. Técnicas de evasión de firewall/IDS

| Técnica | Flag | Mecánica | Contra qué funciona / límite |
|---|---|---|---|
| **MTU / fragmentación** | `--mtu 8` / `-f` | Divide la cabecera TCP en fragmentos; algunos firewall no reensamblan bien | IDS que no reensamblan; añade tráfico y lo detectan los modernos |
| **Data length** | `--data-length N` | Añade bytes aleatorios al payload → altera la firma del paquete | Filtros por tamaño/shape de paquete |
| **Puerto de origen** | `--source-port 53` | Finge ser respuesta DNS/HTTP legítimo | Reglas laxas que confían en puertos origen (poco común hoy) |
| **Decoys** | `-D SPR2,ME` (o `-D RND,ME`) | Envía paquetes desde IPs falsas intercalados: el objetivo no sabe cuál eres | Logs/IDS que atribuyen el origen; exige decoys vivos |
| **SYN stealth** | `-sS` | No completa el handshake → muchas apps no registran la conexión | Logs de aplicación (no de firewall) |
| **Bajar el ritmo** | `--max-rate`, `-T2` | Se mezcla con el tráfico normal | Detección por umbral/volumen |
| **MAC spoof** | `--spoof-mac 00:11:22...` | Cambia la MAC del paquete emitido | Filtros/Firewall por MAC en LAN |
| **Nulo/FIN/Xmas** | `-sN/-sF/-sX` | Flags "imposibles" que los firewalls mal configurados dejan pasar | Solo Unix; muchos IDS los marcan igual |
| **Evasión por fragmentos de datos** | `--data-length` + `-f` combinados | Rompe la reconstitución de payload | Deep inspection con reensamblado |

```bash
# Ejemplo combinado (laboratorio)
nmap -sS -f --mtu 8 --data-length 25 --source-port 53 -D RND,ME --max-rate 200 -p80,443 <IP>
```

> [!warning] Evasión ≠ invisibilidad
> Todo lo anterior genera tráfico y patrones: los IDS modernos reensamblan fragmentos, correlacionan timing y ven el *ruido extra* que produce la propia evasión. La evasión sirve para **atravesar firewalls mal configurados**, no para no ser observado — en engagements, el evasión agresivo suele estar prohibido por el contrato.

## 6. Reconocimiento web

### 6.1 Descubrimiento de contenido (dirbusting)

La técnica: miles de peticiones con un diccionario midiendo **diferencias** respecto a una baseline (status, tamaño, hash, tiempo), porque muchas apps devuelven 200 con página de error.

| Herramienta | Fortaleza | Debilidad | Ejemplo |
|---|---|---|---|
| **[[Gobuster]]** | Muy rápido, multi-modo (dir/dns/vhost/fuzz), Go | Menos personalizable | `gobuster dir -u http://IP -w common.txt -x php,html -t 50` |
| **[[WFuzz]]** | Fuzzing general: parámetros, headers, payloads, `FUZZ` multicapa | Sintaxis más compleja, más lenta | `wfuzz -w wordlist -u "http://IP/FUZZ" -H "Host: FUZZ"` |
| **ffuf** | Rápida y con filtros por tamaño/regex | — | `ffuf -u http://IP/FUZZ -w list.txt -mc 200,301 -fs 4242` |
| **feroxbuster** | Recursiva, sigue links | Ruidosa | `feroxbuster -u http://IP -w list.txt` |
| **dirb / dirbuster** | Clásicas | Lentas, sin mantenimiento | `dirb http://IP /usr/share/wordlists/dirb/common.txt` |
| **Burp Suite** (Intruder/Discover content) | Integrada con el proxy y el auditor | Community limitada (throttle) | — |

```bash
# Gobuster con extensiones y filtro de tamaño
gobuster dir -u http://target.lab -w /usr/share/wordlists/dirb/common.txt \
  -x php,bak,txt,zip -t 40 -s 200,301,403 -o gobuster.txt

# WFuzz: FUZZ es el marcador, y se puede negar con -b (baseline)
wfuzz -w wordlist.txt -u "http://target.lab/FUZZ" -b 1234 --hc 404
```

### 6.2 Fuzzing de vhosts y sitemap

El *virtual hosting* hace que el **header `Host`** decida qué sitio sirve el mismo IP: muchos setups alojan `app.empresa.com`, `staging.empresa.com` y un panel interno en el mismo backend — inaccesibles por URL pero descubiertos por `Host`.

```bash
# Vhost fuzzing con Gobuster
gobuster vhost -u http://target.lab -w subdomains.txt --append-domain -t 40

# WFuzz con cabecera Host
wfuzz -w subdomains.txt -H "Host: FUZZ.target.lab" -u http://target.lab -t 30 --hc 443

# Comprobación manual
curl -s -H "Host: staging.target.lab" http://target.lab/ | head -40
```

| Señal de vhost interesante | Qué mirar |
|---|---|
| Status/longidad distinta al baseline | `200` con contenido nuevo, `403` (existe pero restringido) |
| Redirección a otro host | El backend conoce ese nombre |
| `Server`/`X-Powered-By` diferente | Otro stack detrás del mismo IP |

**Sitemap y superficie indexable**:

```bash
curl -s https://target.lab/robots.txt
curl -s https://target.lab/sitemap.xml | grep -oE '<loc>[^<]+' | sort -u
curl -s https://target.lab/sitemap_index.xml
# También: rutas de framework — /actuator, /graphql, /.well-known/, /server-status
```

> [!tip] `robots.txt` y `sitemap.xml` son recon pasivo real
> Suelen listar justo lo que no quieres ver: `/admin/`, `/backup/`, `/staging/`. Y su ausencia también informa: un sitio sin sitemap con robots agresivo (`Disallow: /`) está ocultando superficie — prioriza fuzzearlo.

### 6.3 Burp Suite: el proxy del auditor

| | **Community Edition** | **Professional** |
|---|---|---|
| Precio | Gratuito (incluido en Kali) | De pago (PortSwigger) |
| **Proxy HTTP/HTTPS** | ✅ interceptar, modificar, reenviar | ✅ |
| Repeater / Sequencer / Decoder | ✅ | ✅ |
| **Escáner automatizado** | ❌ | ✅ |
| Crawling/Discovery asistido | Limitado | ✅ |
| Intruder con payloads ilimitados | Throttle (lento) | Sin límite |
| Informes personalizados | ❌ | ✅ |
| Extensiones (BApp Store) | ✅ | ✅ |

El proxy intercepta todo el tráfico del navegador (instalando su CA para HTTPS): permite inspeccionar peticiones/responder, repetir con modificaciones y descubrir contenido con *link discovery*. La mayoría de flujos de este vault (XSS, SQLi, IDOR) se documentan con Burp en medio — ver [[Abuso de APIs]] y las notas de la carpeta Web.

## 7. Identificación de sistema operativo por TTL

El **TTL** (*time to live*) es el contador de saltos que lleva cada paquete IP: cada router resta 1; si llega a 0 se descarta y se devuelve un ICMP *time exceeded*. Como cada SO tiene un valor por defecto al emitir, la primera respuesta `ping` da una pista:

| TTL recibido (aprox.) | SO probable |
|---|---|
| **64** | Linux / Unix / Android |
| **128** | Windows |
| **255** | Equipos de red (routers, impresoras) / algunos Solaris |

> [!info] Advertencias del TTL
> - Los routers intermedios **decrementan** el valor: con 3 saltos, un Windows 128 se ve como 125 (sigue en rango 65–128).
> - Muchos administradores **cambian** el TTL a propósito.
> - Es una *pista*, no una prueba: confirma con `nmap -O`, p0f o post-explotación.

```python
#!/usr/bin/python3
# whichSystem.py 10.10.10.188  → estima SO por TTL
import re, sys, subprocess

if len(sys.argv) != 2:
    print("\n[!] Uso: python3 " + sys.argv[0] + " <direccion-ip>\n"); sys.exit(1)

def get_ttl(ip_address):
    proc = subprocess.Popen(["/usr/bin/ping -c 1 %s" % ip_address, ""],
                            stdout=subprocess.PIPE, shell=True)
    (out, err) = proc.communicate()
    out = out.split()[12].decode('utf-8')
    return re.findall(r"\d{1,3}", out)[0]

def get_os(ttl):
    ttl = int(ttl)
    if ttl <= 64:   return "Linux"
    if ttl <= 128:  return "Windows"
    return "Not Found"

if __name__ == '__main__':
    ip_address = sys.argv[1]
    ttl = get_ttl(ip_address)
    print("\n%s (ttl -> %s): %s\n" % (ip_address, ttl, get_os(ttl)))
```

Tablas de referencia de valores TTL: <https://subinsb.com/default-device-ttl-values/>.

## 8. Disciplina de documentación

El reconocimiento genera más datos de los que la memoria aguanta; sin estructura, el informe final se reconstruye a ciegas.

| Regla | Práctica |
|---|---|
| **Una fuente de verdad** | Un único `.md`/plantilla por objetivo con secciones: hosts, puertos, servicios, web, OSINT |
| **Salidas de herramienta en disco** | `-oN/-oG` de nmap, `-o` de gobuster, capturas con timestamp |
| **Timestamps** | `date` al inicio de cada bloque; los hallazgos deben poder situarse en el tiempo |
| **Evidencia = comando + salida** | Guardar el comando exacto que produjo cada hallazgo (reproducibilidad) |
| **Hashes de artefactos** | `sha256sum` de captures/ficheros clave (cadena de custodia) |
| **Sin duplicados** | Marcar lo ya cubierto para no re-escanear objetivos |

La plantilla de salida para clientes vive en [[Creación de Informes en LaTeX]] (resumen ejecutivo → hallazgos → evidencia).

## 9. Defensas: lo que el objetivo pone en tu contra

| Mecanismo | Qué detecta/bloquea | Contramedida operativa (dentro del contrato) |
|---|---|---|
| **Rate limiting / connlimit** | Fuzzing y barridos rápidos | Bajar `--rate`, dispersar en el tiempo |
| **WAF** (ModSecurity, Cloudflare) | Patrones de inyección, scanners conocidos | Headers limpios, payloads variados; el WAF es un *hallazgo* que se reporta |
| **IDS/IPS** (Snort, Suricata, Zeek) | Firmas de nmap, deauth, exploits | `-sT`, timing lento, fragmentación (§5) |
| **Firewall con inspección de estado** | SYN half-open, flags raros | Vectores de discovery múltiples |
| **Honeypots / tarpits** | Todo lo que toca puertos viejos | Verificar qué es real antes de gastar tiempo |
| **Logs y correlación (SIEM)** | Patrones de escaneo multi-puerto | Un único objetivo acordado, ventanas definidas |
| **Fail2ban / lockout** | Fuerza bruta | Límites de intentos pactados |

> [!info] Perspectiva defensiva
> Casi todo lo de esta nota tiene un espejo: los mismos logs que te delatan permiten al defensor **detectar el reconocimiento en fase 0** (mucho antes de la explotación). Por eso en el informe se documenta también el ruido generado: el cliente necesita saber qué vio su IDS.

## 10. Errores comunes

1. **Empezar por el activo**: hacer `nmap -p-` antes de agotar OSINT = gastar la sorpresa y sembrar logs sin necesidad.
2. **`--min-rate 5000` en producción**: velocidad de laboratorio en un entorno que no lo tolera.
3. **Omitir `-Pn`** cuando el host filtra ICMP y concluir "está caído".
4. **No fijar alcance/scope** antes de tocar nada (bug bounty incluido).
5. **Olvidar UDP y IPv6**: `-sU` y `nmap -6` descubren servicios que TCP no verá nunca.
6. **Confiar en un solo vecto de descubrimiento**: ICMP solo → falsos negativos con Windows.
7. **No guardar las salidas** (`-oN`): el informe queda sin evidencias reproducibles.
8. **Fuzzing sin baseline**: contar como hallazgo todo 200 con página de error.
9. **Descartar el TTL/OS guess como cierto**: es una heurística, no un hecho.

## 11. Cheatsheet

```bash
# ── Pasivo ──
whois dominio.com ; dig +short NS/MX/TXT dominio.com
curl -s "https://crt.sh/?q=%25.empresa.com&output=json" | jq -r '.[].name_value' | sort -u
site:empresa.com filetype:pdf ; site:empresa.com inurl:admin        # dorks

# ── Descubrimiento ──
nmap -sn 192.168.1.0/24
arp-scan -I eth0 --localnet ; netdiscover -i eth0
masscan -p21,22,139,445 -Pn 192.168.0.0/16 --rate=10000

# ── Escaneo de puertos ──
nmap -p- --open -sS --min-rate 5000 -v -n -Pn <IP> -oG allPorts
nmap -p80,443,22 -sCV <IP> -oN target
nmap -sU --top-ports 20 <IP>                      # UDP
echo > /dev/tcp/HOST/22 && echo abierto           # sin nmap

# ── Evasión (solo con autorización) ──
nmap -sS -f --data-length 25 -D RND,ME --max-rate 200 -p80 <IP>

# ── Web ──
gobuster dir -u http://IP -w common.txt -x php,bak -t 40
gobuster vhost -u http://IP -w subdomains.txt --append-domain
wfuzz -w list -u http://IP/FUZZ -b 4242 --hc 404
curl -sI https://IP | grep -iE 'server|x-powered'
whatweb -v https://IP
```

## 12. Referencias

- Nmap manual: <https://nmap.org/book/man.html> · NSE: <https://nmap.org/book/nse.html>
- Google Hacking Database: <https://www.exploit-db.com/google-hacking-database>
- crt.sh (CT logs): <https://crt.sh/> · CTFR: <https://github.com/UnaPibaGeek/ctfr>
- Burp Suite: <https://portswigger.net/burp> · PortSwigger Web Security Academy: <https://portswigger.net/web-security>
- HackerOne (scope/programas): <https://www.hackerone.com/>
- Shodan/Censys (búsqueda de servicios): <https://www.shodan.io/> · <https://censys.io/>
- HackTricks — recon: <https://book.hacktricks.wiki/>

## Relacionadas

- [[Nmap Guía Completa]] — referencia completa de flags y NSE
- [[Enumeración de Servicios y CMS]] — el paso siguiente: fingerprint y vulnerabilidades
- [[Gobuster]] · [[WFuzz]] · [[Masscan]] · [[netdiscover]] · [[Arp Scan Herramienta de Descubrimiento|arp-scan]] — herramientas citadas
- [[WhatWeb Herramienta de Fingerprinting]] — tecnología web
- [[Wireshark y Tshark]] / [[Tshark]] — análisis de las capturas
- [[Python Ofensivo]] — escáneres propios en Python
- [[Bash Scripting - Fundamentos Básicos]] — los scripts de esta nota
- [[Creación de Informes en LaTeX]] — documentar los hallazgos
