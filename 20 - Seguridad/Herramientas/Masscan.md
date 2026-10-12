---
aliases: ["masscan", "Mass IP Port Scanner", "Escáner de Puertos Masivo"]
tags: [tool/masscan]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Nmap Guía Completa]]"
  - "[[Arp Scan Herramienta de Descubrimiento]]"
  - "[[Wireshark y Tshark]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
  - "[[SSLScan]]"
  - "[[netdiscover]]"
fuente: "https://github.com/robertdavidgraham/masscan"
---

# Masscan: Escáner de Puertos a Altísima Velocidad

**masscan** (Mass IP Port Scanner) es un escáner TCP/UDP asíncrono capaz de mandar **más de un millón de paquetes por segundo**: escanea rangos enteros de Internet en minutos desde una máquina. No sustituye a [[Nmap Guía Completa]] — lo **precede**: masscan encuentra *qué puertos están abiertos a escala*, Nmap hace después el trabajo profundo (versiones, scripts, OS).

> [!info] Por qué es tan rápido (y qué cuesta)
> masscan no usa la pila de red del sistema operativo: construye y envía los paquetes **con sockets crudos (AF_PACKET)** y procesa las respuestas con su propia máquina de estados, todo de forma totalmente **asíncrono** (no espera a un host antes de mandar el siguiente). Nmap, en cambio, respeta la pila TCP y su tuning tiene techo. El precio: masscan es *best-effort* (en tasa muy alta pierde respuestas), no hace detección de SO ni NSE, y a tasa alta se parece a un flood — ver "Cautiones".

## Instalación

```bash
# Kali / Debian
sudo apt update && sudo apt install masscan

# Desde fuente (si necesitas compilar para tu kernel/NIC)
git clone https://github.com/robertdavidgraham/masscan.git
cd masscan && make -j"$(nproc)"
# binario en ./bin/masscan

# Requiere privilegios: sockets crudos
sudo masscan --version
sudo masscan --iflist      # interfaces disponibles
```

## Mecánica: qué hace realmente por debajo

| Aspecto | masscan | nmap |
|---|---|---|
| Envío | Sockets crudos, cola asíncrona propia | Pila TCP/ICMP del SO |
| Escaneo por defecto | **SYN half-open** (no completa el handshake) | Según `-sS/-sT/...` |
| Descubrimiento de hosts | **No** (asume que el host está ahí; un puerto abierto lo delata) | Sí (`-sn`, `-PS`…) |
| Tasa máxima práctica | 100k–1M+ pps | ~1–5k pps con `-T5` |
| Detección de SO / NSE | No | Sí |
| Banner grab | `--banners` (básico, con su propio stack TLS) | `-sV` + scripts (profundo) |
| Fiabilidad | Best-effort: con tasa alta **pierde** respuestas | Reintentos y verificación |

**Regla de oro:** *masscan para descubrir, Nmap para entender*.

## Referencia de flags por escenario

### Objetivos y puertos

| Flag | Significado | Ejemplo |
|---|---|---|
| `-p`, `--ports` | Puertos: lista, rangos, mezcla TCP/UDP | `-p 80,443,8000-8100` |
| prefijo `U:` | Puerto **UDP** (el resto es TCP) | `-p U:53,T:80,443` |
| `-iL`, `--iL` | Objetivos desde fichero (IP/CIDR por línea) | `-iL objetivos.txt` |
| `--exclude` | Excluye IP/rangos concretos | `--exclude 192.168.1.1` |
| `--excludefile` | Exclusiones desde fichero (firewall, gateways) | `--excludefile no.txt` |
| `--randomize-hosts` | Orden aleatorio de objetivos | `--randomize-hosts` |

### Rendimiento (el corazón de masscan)

| Flag | Significado | Ejemplo / valor |
|---|---|---|
| `--rate` | **Paquetes por segundo** (defecto: `100`) | `--rate 100000` |
| `--wait` | Segundos de espera final para respuestas tardías (defecto ~10) | `--wait 30` |
| `--seed` | Semilla del PRNG (reproducibilidad de orden/ports) | `--seed 42` |
| `--packet-trace` | Imprime paquetes enviados/recibidos (diagnóstico) | `--packet-trace` |
| `--connection-timeout` | Timeout de conexión/respuesta (segundos) | `--connection-timeout 10` |
| `--retries` | Reenvíos por puerto/host (según versión) | `--retries 2` |

> [!warning] El rate es una decisión operativa, no técnica
> `--rate 1000000` contra una red de cliente puede **denegar servicio** a los equipos más frágiles (IoT, impresoras, cámaras) y saturar el enlace. Conviene acordar el techo en el alcance y empezar bajo: `--rate 1000` en LAN, `10000` en segmentos grandes, reservar seis cifras para rangos públicos propios/autorizados.

### Red y binding (raw sockets)

| Flag | Significado | Ejemplo |
|---|---|---|
| `--iflist` | Lista interfaces y MACs detectadas | `--iflist` |
| `--adapter-ip` | IP(s) de origen | `--adapter-ip 192.168.1.50` |
| `--adapter-port` | Puerto(s) de origen | `--adapter-port 40000-41000` |
| `--adapter-mac` | MAC de origen | `--adapter-mac 00:11:22:33:44:55` |
| `--router-ip` | IP del gateway (redes con enrutado especial) | `--router-ip 192.168.1.1` |
| `--router-mac` | MAC del gateway | `--router-mac 00:11:22:33:44:55` |
| `--source-ip` | Alias alternativo de IP de origen (según versión) | `--source-ip 10.0.0.5` |

> [!warning] Spoofing de IP de origen
> Con la IP de origen falsificada, **masscan no recibe las respuestas** (van a otra parte): el escaneo deja de informar de puertos abiertos. Solo sirve para "fire-and-forget" con captura externa (sniffer), y en cualquier red ajena es una mala idea: el tráfico queda firmado con una IP que no es tuya.

### Salida

| Flag | Formato | Uso típico |
|---|---|---|
| `-oL` | Lista simple (`Discovered open port 80/tcp on 10.0.0.5`) | Pipelines con `awk`/`grep` |
| `-oX` | XML | Compatible con herramientas de informes/Nmap |
| `-oJ` | JSON | Programático |
| `-oG` | grepable (estilo Nmap) | Filtros rápidos |
| `-oB` | **Binario** propio | Escaneos gigantes (compacto) |
| `--readscan <bin>` | Convierte el binario a otro formato | `--readscan scan.bin -oX scan.xml` |
| `--echo` | Imprime la configuración actual (reutilizable) | `masscan --echo > config.conf` |
| `--iflist` | Interfaces | diagnóstico |
| `--packet-trace` | Paquetes a/b del wire | diagnóstico |

### Escaneo avanzado

| Flag | Significado | Notas |
|---|---|---|
| `--banners` | Captura **banner** del servicio (HTTP, SMTP, SSH, TLS…) | Añade tráfico extra: no combinar con rate altísimo |
| `--resume` | Reanuda desde `paused.conf` (se genera con Ctrl+C) | `masscan --resume paused.conf` |
| `--seed` / `--regress` | Opciones de reproducibilidad/testing | Uso en CI/laboratorio |

> [!warning] Flags de notas previas a verificar con `masscan --help`
> `--nmap` (imprimir comandos Nmap equivalentes), `--ping` (descubrimiento ICMP), `--heartbleed` (detección CVE-2014-0160) y `--status-updates` aparecen en documentación/varias versiones, pero **no todos existen en todas las compilaciones**. Antes de basar un flujo en ellos: `masscan --help | grep <flag>`. La integración con Nmap **no** depende de `--nmap` — se hace por pipeline (ver receta 8).

## Interpretación de la salida

```text
Starting masscan 1.3.2 (at 2026-10-11 12:00) with 1000000 ports-scan rate
Discovered open port 443/tcp on 10.10.10.5
Discovered open port 80/tcp  on 10.10.10.5
Discovered open port 445/tcp on 10.10.10.20
```

| Elemento | Significado real |
|---|---|
| `with 1000000 ports-scan rate` | Tasa efectiva usada — revisarla siempre |
| `Discovered open port X/tcp on IP` | El host respondió **SYN/ACK** a ese puerto |
| Ausencia de línea para un host | Cerrado **o** perdido por tasa alta → re-ejecutar más lento antes de concluir |
| (con `--banners`) `Banner: ...` | Primer intercambio leído por el stack propio de masscan |
| `paused.conf` (fichero) | Estado guardado al interrumpir: `--resume` lo reutiliza |

**Diferencia clave con Nmap:** masscan solo informa de **abiertos** (no distingue `filtered`/`closed`), y no hace reintentos elegantes: *ausencia de evidencia ≠ puerto cerrado*. Por eso, ante resultados limpios, se re-valida con Nmap sobre los hosts hallados.

## Recetas: los 10 comandos reales

```bash
# 1. Escaneo rápido de puertos clave en una LAN
sudo masscan -p80,443,445,3389,22 192.168.1.0/24 --rate=1000 -oL lan.txt

# 2. Rango de puertos completo (todos, orden aleatorio)
sudo masscan -p1-65535 10.10.10.0/24 --rate=5000 --randomize-hosts -oL full.txt

# 3. TCP + UDP combinados
sudo masscan -pU:53,U:161,T:80,T:443 10.10.10.0/24 --rate=500 -oL mixto.txt

# 4. Con banners (identificar servicios al vuelo)
sudo masscan -p80,443,22,25 10.10.10.0/24 --rate=2000 --banners -oB banners.bin

# 5. Convertir resultados binarios a XML/JSON
masscan --readscan banners.bin -oX banners.xml
masscan --readscan banners.bin -oJ banners.json

# 6. Objetivos desde fichero con exclusiones (evitar gateways)
sudo masscan -iL redes.txt --excludefile exclude.txt -p445 --rate=1000 -oL out.txt

# 7. Pausar y reanudar (Ctrl+C deja paused.conf)
sudo masscan -p1-65535 10.0.0.0/16 --rate=50000 -oL big.txt
# ... Ctrl+C ...
sudo masscan --resume paused.conf

# 8. Pipeline masscan → Nmap (el flujo real)
sudo masscan -p80,443,445,8080,8443 10.10.10.0/24 --rate=2000 -oL mc.txt
awk '/^Discovered/ {print $6}' mc.txt | sort -u > hosts.txt
nmap -iL hosts.txt -p 80,443,445,8080,8443 -sV -sC --min-rate 200 -oA profundizado

# 9. Guardar la configuración usada y reutilizarla
sudo masscan -p80,443 10.10.10.0/24 --rate=1000 --echo > lab.conf
sudo masscan -c lab.conf --rate=500

# 10. Diagnóstico: interfaces y paquetes en el aire
sudo masscan --iflist
sudo masscan -p80 10.10.10.5 --rate=10 --packet-trace
```

**Flag por flag de la receta 1:** `-p` acota puertos (sin esto masscan escanea los 65535 por defecto), `--rate=1000` fija el techo de pps (1000 es cómodo para un /24 sin riesgo), `-oL` deja salida parseable, y el CIDR es el objetivo.

## Integración con Nmap (el flujo completo)

```bash
# Fase 1 — barrido masivo (masscan)
sudo masscan -p1-65535 10.10.10.0/24 --rate=5000 --banners -oB fase1.bin
masscan --readscan fase1.bin -oL fase1.txt

# Fase 2 — traducir a targets Nmap (la IP es el campo $6 de "Discovered open port 80/tcp on <ip>")
awk '/^Discovered/ {print $6}' fase1.txt | sort -u > hosts.txt
# (con --banners, masscan ya trae versión tentativa: úsala para priorizar)

# Fase 3 — profundidad con Nmap (versiones, scripts, UDP selectivo)
nmap -iL hosts.txt -p 80,443,445,3389 -sV -sC --version-all -oA nmap_final

# Fase 4 — SSL/TLS donde aplique (ver [[SSLScan]])
nmap --script ssl-enum-ciphers -p 443 -iL hosts.txt
```

| Qué resuelve cada uno | Herramienta |
|---|---|
| Abrir/cerrado a escala (10.000 hosts) | **masscan** (minutos) |
| Versión real del servicio, scripts, OS | **Nmap** |
| Banner temprano para priorizar | `masscan --banners` |
| Cifrado TLS detallado | [[SSLScan]] / [[Wireshark y Tshark]] |

## Precauciones: no aplastar al objetivo

1. **Riesgo de DoS colateral**: a `--rate` alto, los SYNs llenan la tabla de conexiones del objetivo (y el `conntrack` de tu propio gateway). Equipos viejos (IoT, cámaras, switches con mgmt) se caen con tasas que un servidor moderno aguanta.
2. **Enlace saturado**: a 1M pps ≈ 1 Gbit+ según tamaño; puedes saturar tu propio uplink o el del cliente.
3. **NAT/routers**: sin `--router-ip/--router-mac` bien configurados, los paquetes no vuelven y "no ves nada" aunque haya hosts.
4. **Firewalls con rate-limit**: resultados incompletos → no interpretar "cero abiertos" como red limpia.
5. **Alcance legal**: escanear rangos que no son tuyos (p. ej. `0.0.0.0/0`) es escaneo no autorizado, aunque sea "solo SYN". Todo lo anterior documentado, **nunca ejecutado contra terceros sin permiso**.

## Detección y contramedidas (defensor)

| Señal | Dónde se ve | Mitigación |
|---|---|---|
| Ráfaga de SYN a muchos puertos/hosts desde un origen | IDS/IPS (Snort/Suricata: reglas de *rapid SYN scan*), Zeek `conn.log` con muchas `S0` | Fail2ban, tasa de SYN, IDS con umbrales |
| Miles de conexiones **a medias** (`S0`, sin handshake) | NetFlow/conntrack: ratio SYN→SYN-ACK alto sin `ESTABLISHED` | `syncookies`, límite de backlog, `nf_conntrack_max` |
| Patrón de sondeo lineal/aleatorio de un solo MAC/IP | SIEM: "host escanea muchos destinos" | Segmentación, ACLs de origen |
| TTL/IP-ID/constants fijos | Fingerprinting del escáner (paquetes "demasiado perfectos" o TTL inusual) | Honeypot/tees para identificar origen |
| `--banners` añade datos extra por conexión | Payload después del handshake → alerta de aplicación | Inspección TLS/HTTP stateful |
| Log de aplicación con decenas de peticiones de un cliente que nunca hace handshake completo | Logs web + proxy | WAF con validación de handshake |

> [!tip] Detección práctica
> La firma estadística inconfundible de masscan es **`S0` masivo**: cientos/miles de conexiones que nunca se completan desde un mismo origen en segundos. Con `--rate` bajo se confunde con tráfico normal, por eso los atacantes serios bajan la tasa y la suben poco a poco.

## Errores comunes (mensaje → arreglo)

1. **`ERROR: fail to open raw socket` / `Permission denied`** — sin root.
   → Ejecutar con `sudo` (o `setcap cap_net_raw+ep` sobre el binario).
2. **`ERROR: no network interface...` / no encuentra adaptador** — varias interfaces o sin IP.
   → `sudo masscan --iflist` y fijar `--adapter-ip`/`--router-ip`.
3. **Cero resultados y sin error** — interfaz equivocada, gateway mal, o tasa tan alta que perdieron las respuestas.
   → `--packet-trace` (¿salen paquetes?), bajar `--rate`, revisar `--router-ip`, comprobar con `tcpdump` en el mismo segmento (ver [[Wireshark y Tshark]]).
4. **Resultados incompletos tras `--rate` muy alto** — pérdida esperada.
   → Re-ejecutar sobre los hosts de interés con tasa baja o pasar a Nmap (receta 8).
5. **`--resume` no encuentra estado** — `paused.conf` se generó en otro directorio o fue sobrescrito.
   → Ejecutar en la misma ruta donde se pausó; no lanzar otro masscan encima antes de reanudar.
6. **`--readscan` sobre un fichero que no es `-oB`** — conversión imposible.
   → Solo ficheros binarios de masscan; el resto (`-oL/-oX`) ya son texto.
7. **Objetivos remotos sin respuesta en redes con NAT** — falta enrutado.
   → Configurar `--router-ip/--router-mac`; en el laboratorio, validar con un host de prueba antes del rango completo.
8. **UDP "no devuelve nada"** — UDP necesita que el servicio responda; muchos puertos UDP dan timeout silencioso.
   → Contrastar con `nmap -sU` sobre una muestra, y no inferir "cerrado" sin verificación.

## Cheatsheet — los 10 comandos del día a día

```bash
sudo masscan --iflist
sudo masscan -p80,443,445 10.10.10.0/24 --rate=1000 -oL out.txt
sudo masscan -p1-65535 10.10.10.0/24 --rate=5000 --randomize-hosts -oL full.txt
sudo masscan -pU:53,T:80 10.10.10.0/24 --rate=500 -oL udp.txt
sudo masscan -p22,80,443 10.10.10.0/24 --rate=2000 --banners -oB b.bin
masscan --readscan b.bin -oX b.xml
sudo masscan --resume paused.conf
sudo masscan -c lab.conf --rate=500
masscan --echo > lab.conf
masscan --help
```

| Tarea | Comando |
|---|---|
| Barrido rápido LAN | `-p80,443,445 <cidr> --rate=1000 -oL out.txt` |
| Todos los puertos | `-p1-65535 <cidr> --rate=5000` |
| UDP | `-pU:53,T:80 ...` |
| Banners | `--banners -oB` |
| Convertir binario | `--readscan x.bin -oX x.xml` |
| Reanudar | `--resume paused.conf` |
| Excluir gateways | `--excludefile exclude.txt` |
| Config reutilizable | `--echo > conf` / `-c conf` |
| Diagnóstico | `--packet-trace` / `--iflist` |
| Mano a Nmap | `awk '/^Discovered/ {print $6}' out.txt \| sort -u > hosts.txt` |

## masscan vs nmap (velocidad y alcance)

| Métrica | **masscan** | **Nmap** |
|---|---|---|
| Tasa máxima práctica | 100k–1M+ pps | ~1–5k pps (`-T5`) |
| /24, todos los puertos | < 1 s | ~1–10 min |
| Rango /8 completo | Minutos (a 100k pps) | Días/semanas |
| Descubrimiento de hosts | No explícito | Sí (multimétodo) |
| Versión de servicio | `--banners` (aproximado) | `-sV` (preciso) |
| Scripts/vulns (NSE) | No | Sí |
| Detección de SO | No | Sí (`-O`) |
| Fiabilidad | Best-effort según tasa | Alta (reintentos, `-PA/-PS`) |
| Salida binaria propia | Sí (`-oB` + `--readscan`) | No |

## Referencias

- [Repositorio oficial (robertdavidgraham/masscan)](https://github.com/robertdavidgraham/masscan) — README con ejemplos de tasa y configuración
- `man masscan` / `masscan --help` tras instalar
- [Masscan: lo rápido y lo seguro](https://www.robertdavidgraham.com/) — artículos del autor sobre tuning y estructura interna
- HackTricks — *Network enumeration* para el flujo masscan→nmap

## Relacionadas

- [[Nmap Guía Completa]] — la fase profunda posterior (versiones, NSE, OS)
- [[Arp Scan Herramienta de Descubrimiento]] / [[netdiscover]] — descubrimiento L2 cuando el objetivo es la LAN
- [[Wireshark y Tshark]] / [[Tshark]] — verificar qué está pasando en el wire
- [[SSLScan]] — análisis TLS de los 443 hallados
- [[Reconocimiento y Escaneo de Redes y Web]] — metodología en la que encaja
