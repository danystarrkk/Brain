---
aliases: ["MDK3", "MDK3 WiFi", "mdk3"]
tags:
  - tool/mdk3
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Hacking WiFi Fundamentos y Ataques]]"
  - "[[Aircrack-ng Suite de Herramientas]]"
  - "[[Nmap Guía Completa]]"
  - "[[Reconocimiento y Escaneo de Redes y Web]]"
fuente: "https://www.aircrack-ng.org/"
---

# MDK3

**MDK3** (*Mauler's Denial of Kit v3*) es una herramienta de **auditoría inalámbrica** que implementa ataques de capa 2 sobre redes Wi‑Fi: inundación de beacons, denegación de servicio por autenticación/deautenticación, confusión de sistemas WIDS y monitoreo de probe requests. Es el sucesor espiritual de las técnicas de inyección de tramas 802.11 de [[Aircrack-ng Suite de Herramientas]] pero enfocado exclusivamente en **la parte ofensiva de la capa MAC**: no descifra, no crackea — satura, desconecta y confunde.

> [!warning] Uso legal — obligatorio antes de tocar un canal
> MDK3 genera **tramas de gestión falsificadas** (deauth, disassoc, beacons, autenticaciones) contra APs y clientes reales. Cualquiera de estos modos contra una red sin autorización escrita es **ilegal** en prácticamente toda jurisdicción (en España: arts. 197 y 278 bis del Código Penal; en EE. UU.: CFAA). Los DoS contra infraestructura crítica agravan el tipo. El marco legítimo es: **laboratorio propio, CTF, o contrato de pentest con cláusula explícita de denegación de servicio**. Un único paquete de deauth hacia un AP ajeno ya es un evento registrado en WIDS corporativos.

> [!info] Requisitos previos
> 1. **Modo monitor**: la interfaz debe estar en monitor mode (`airmon-ng start wlan0` — ver [[Aircrack-ng Suite de Herramientas]]).
> 2. **Inyección**: la tarjeta debe soportar inyección de paquetes (`aireplay-ng --test wlan0mon` lo verifica).
> 3. **Canal fijo**: la interfaz debe estar en el canal del objetivo (`iwconfig wlan0mon channel 6`). MDK3 no salta de canal por ti en los modos dirigidos — si el canal es incorrecto, simplemente no pasa nada.

### Canales 2.4/5 GHz de referencia (para fijar la interfaz)

| Canal | Freq. (MHz) | Nota |
|---|---|---|
| 1 / 6 / 11 | 2412 / 2437 / 2462 | Los tres **no solapados** en EE. UU. — los que usan casi todos los clientes al escanear |
| 2–5, 7–10 | 2417–2457 | Solo Europa/APAC; se solapan con 1/6/11 |
| 13 | 2472 | Europa |
| 14 | 2484 | Solo Japón (802.11b) |
| 36–48 | 5180–5240 | UNII-1, los comunes en 5 GHz |
| 149–165 | 5745–5825 | UNII-3 |

La regla práctica: fijar siempre el canal exacto del AP objetivo antes de inyectar — saltar de canal hace perder la sincronía con clientes y los ataques dirigidos fallan en silencio.

## MDK3 vs mdk4 vs aireplay-ng

| Herramienta | Estado | Fortaleza | Cuándo usarla |
|---|---|---|---|
| **MDK3** | Legacy (2010, sin mantenimiento) | Estable, en todos los Kali antiguos | Reproducir guías/lab clásicos, CTFs |
| **mdk4** | Activo (por Mocking) | Reescritura: menos falsos positivos, más modos (EAPOL flood, PMKID), mejor WIDS | Auditorías nuevas — **herramienta recomendada** |
| **aireplay-ng** | Activo (aircrack-ng) | Inyección dirigida: deauth, fakeauth, injection test | Deauth controlado para capturar handshake WPA — ver [[Hacking WiFi Fundamentos y Ataques]] |

> [!tip] Diferencia práctica clave con aireplay-ng
> `aireplay-ng --deauth` deautentica **un** objetivo concreto. `mdk3 d -m` masacra **todo** lo que lea de una captura. Para el flujo normal de pentest (capturar el 4-way handshake) basta aireplay-ng; MDK3 se usa cuando necesitas saturar o confundir deliberadamente.

## Sintaxis y estructura de modos

```bash
mdk3 <interfaz> <modo> [opciones]
```

Sin opciones, cada modo imprime **su propia ayuda** — es la forma correcta de consultar los flags en lugar de memorizarlos:

```bash
mdk3 wlan0mon d        # ayuda del modo deautenticación
```

### Tabla maestra de modos

| Modo | Nombre | Qué hace | Ruido / detección |
|---|---|---|---|
| `b` | **Beacon Flood** | Inunda de beacons falsos (APs ficticios con SSID/BSSID/flags a elección) | Alto — cualquier WIDS lista cientos de "redes nuevas" |
| `a` | **Authentication DoS** | Envía floods de frames de autenticación (y asociación) al AP | Medio-alto — satura la tabla de estados del AP |
| `d` | **Deauth / Disassoc** | Inyecta tramas de deautenticación o desasociación | Bajo en volumen, **muy** visible en logs WIDS |
| `p` | **Probe Request Monitor** | Monitorea/filtra probe requests (modos `-f`, `-t`, `-r`, `-d`, `-c`, `-e`, `-q`) | Pasivo — solo escucha |
| `s` | **VoIP Harvesting** | Captura SSIDs y claves WPA de clientes VoIP configurados incorrectamente | Pasivo |
| `w` | **WIDS Confusion** | Genera tráfico que confunde a sistemas de detección WIDS/WIPS (tramas 1024, EAPOL, redirecciones) | Diseñado para engañar al defensor |

> [!warning] Corrección frecuente: el modo `w` NO es WPS
> Es un error muy repetido en guías antiguas: `w` = **WIDS Confusion Mode**, no "WPS attack" ni "Reaver-like". Los ataques WPS los hace **Reaver/Bully/wifite**; MDK3 no implementa WPS. Ejemplos tipo `mdk3 wlan0mon w -r` que circulan por ahí se refieren a las opciones internas del modo de confusión, no a Reaver.

## Modo `d` — Deautenticación / Desasociación (el más usado)

Explota la debilidad de diseño del estándar 802.11: **las tramas de gestión de deauth no llevan cifrado ni autenticación** (IEEE 802.11‑2016, ahora mitigable con 802.11w/PMF). Cualquier estación puede emitir una deauth falsificada aparentando ser el AP o el cliente.

### Opciones principales del modo `d`

| Flag | Significado | Ejemplo |
|---|---|---|
| `-a <BSSID>` | Deautentica **todos** los clientes asociados a ese AP | `-a 00:14:6C:7E:40:80` |
| `-c <MAC_cliente>` | Deautentica **solo** ese cliente (combinar con `-a` = par dirigido) | `-a 00:14:6C:7E:40:80 -c 00:0F:B5:32:31:31` |
| `-s <ppk>` | Modo dirigido vía AP (*targeted deauth*): el AP reenvía la deauth al cliente | `-s 3` (vease ayuda del modo) |
| `-m` | Lee BSSIDs **y clientes** de una captura `.cap` de Wireshark/airodump y ataca a todos | `-m captura.cap` |
| `-E <SSID>` | Detecta clientes por ESSID en lugar de BSSID | `-E "MiRed"` |
| `-B` | Detecta clientes por BSSID a partir de la captura | `-B captura.cap` |

### Recetas del modo `d`

```bash
# 1. Deauth total de un AP (todos sus clientes) — el "clásico"
mdk3 wlan0mon d -a 00:14:6C:7E:40:80

# 2. Deauth dirigida: un solo cliente, forzando reconexión (para capturar el handshake)
mdk3 wlan0mon d -a 00:14:6C:7E:40:80 -c 00:0F:B5:32:31:31

# 3. Deauth masiva desde una captura de airodump (¡esto es un DoS de área!)
mdk3 wlan0mon d -m captura.cap
```

> [!tip] Para qué sirve en un pentest legítimo
> El objetivo casi nunca es "dejar la red caída": es **forzar la reconexión de un cliente** para capturar el 4‑way handshake de WPA2 y llevarlo a crackeo offline con [[Aircrack-ng Suite de Herramientas]]. Por eso la receta nº 2 (dirigida, un cliente, unos segundos) es la de uso real; la nº 3 es material de laboratorio controlado.

## Modo `b` — Beacon Flood

Genera beacons con SSIDs/BSSIDs aleatorios o de un archivo, con control de flags de seguridad (WEP/WPA/WPA2/WPS). Sirve para auditar **cómo reaccionan clientes y WIDS** ante un entorno saturado de redes ficticias.

| Flag | Significado | Ejemplo |
|---|---|---|
| `-n <SSID>` | SSID fijo a anunciar | `-n "FreeWiFi"` |
| `-F <archivo>` | SSIDs tomados de un archivo (una por línea) | `-F ssids.txt` |
| `-c <canal>` | Canal del beacon | `-c 6` |
| `-w` | Activa el **bit WEP** en los beacons | `-w` |
| `-s` | Activa WPA/WPA2 (bit de privacidad + RSN) | `-s` |
| `-B` | Copia beacons de APs reales vistos en una captura | `-B captura.cap` |
| `-m` | Copia BSSIDs de la captura | `-m captura.cap` |
| `-h <MAC>` | MAC de origen falsificada | `-h 00:11:22:33:44:55` |
| `-i` | Genera SSIDs inválidos (caráteres no permitidos) para probar robustez | `-i` |
| `-z` | Padding — paquetes hasta el tamaño máximo | `-z` |

```bash
# Redes aleatorias con WPA2 en canal 1 — prueba de resiliencia del WIDS
mdk3 wlan0mon b -c 1 -s

# Flood controlado con lista propia de SSIDs
mdk3 wlan0mon b -n "CorpNet" -F ssids.txt -c 6 -s
```

## Modo `a` — Authentication DoS

El AP mantiene una **tabla de autenticaciones pendientes** de tamaño finito. Inundarlo con autenticaciones falsas (MACs aleatorios) llena la tabla y el AP deja de aceptar clientes legítimos — DoS sin tocar el medio inalámbrico de datos.

| Flag | Significado | Ejemplo |
|---|---|---|
| `-a <BSSID>` | AP objetivo | `-a 00:14:6C:7E:40:80` |
| `-c` | No comprobar si el canal es válido (**peligroso**: salta canales) | `-c` |
| `-h` | No comprobar si la MAC origen ya existe (permite flood rápido) | `-h` |
| `-m` | Incluye EAPOL en el flood | `-m` |

```bash
mdk3 wlan0mon a -a 00:14:6C:7E:40:80
```

> [!warning] Este modo es la definición de DoS
> A diferencia de la deauth (que es molesta pero reversible al instante), el auth flood **deja el AP inutilizable** hasta que se reinicie o expiren las entradas. Es el modo con más consecuencias operativas y el que más rápido dispara alertas.

## Modo `p` — Probe Request Monitor (pasivo)

Modo de **escucha**: filtra los probe requests que los clientes emiten buscando redes recordadas — una de las mejores fuentes de inteligencia del reconocimiento inalámbrico (descubre clientes "off-network", redes que el cliente visita, dispositivos IoT).

| Flag | Significado |
|---|---|
| `-f <archivo>` | Muestra solo los SSIDs listados en el archivo |
| `-t <MAC>` | Filtra por MAC del cliente |
| `-r` | Muestra solo las respuestas (*replies*) |
| `-d` | Muestra solo probes hacia **redes inexistentes** (huella de clientes sigilosa) |
| `-c` | Clientes que sondean múltiples canales (posible punto de acceso móvil) |
| `-e` | Probes fallidos |
| `-q` | Modo silencioso — solo lo relevante |

```bash
mdk3 wlan0mon p -q
```

## Modo `w` — WIDS Confusion

Genera tráfico deliberadamente anómalo (tramas de 1024 bytes, EAPOL falsos, redirecciones, respuestas de broadcast) para evaluar si el **sistema de detección de intrusiones inalámbrico** del cliente genera alertas útiles o se ahoga en ruido. Se audita así la capacidad del defensor, no la del AP.

```bash
mdk3 wlan0mon w          # imprime las opciones del modo de confusión
```

## Modo `s` — VoIP Harvesting

Captura SSIDs y, cuando el cliente VoIP los transmite en claro por mala configuración (protocolos de provisioning como UPnP/WRQ), **claves WPA**. Es un escenario de auditoría muy concreto: teléfonos IP mal configurados en la red corporativa.

## Flujo de trabajo completo (laboratorio autorizado)

1. **Preparar**: `airmon-ng check kill` → `airmon-ng start wlan0` → verificar inyección con `aireplay-ng --test wlan0mon` (ver [[Aircrack-ng Suite de Herramientas]]).
2. **Reconocer**: `airodump-ng wlan0mon` — anotar BSSID, canal y MAC del cliente objetivo. Complementar con el modo `p` de MDK3 para mapear clientes.
3. **Atacar de forma mínima**: deauth dirigida (`d -a ... -c ...`) durante unos segundos mientras airodump captura.
4. **Verificar**: en la consola de airodump debe aparecer `WPA handshake: <BSSID>` junto a la MAC del cliente.
5. **Parar**: Ctrl+C. El handshake va a crackeo offline (o a [[THC Hydra]]/hashcat si hay portal autenticado).
6. **Documentar**: hora, canal, número de clientes afectados y duración de la indisponibilidad — el informe lo pide [[Creación de Informes en LaTeX]].

## Detección y contramedidas (la vista del defensor)

| Ataque | Cómo lo ve el defensor | Contramedida |
|---|---|---|
| Deauth/disassoc | Conteo de tramas de gestión por segundo por STA/AP; firmas "excess deauth" en WIDS (Kismet, AirMagnet) | **802.11w (PMF)** — firma las tramas de gestión; sin PMF, imposible distinguir deauth real de falsa |
| Beacon flood | Cientos de BSSIDs nuevos en minutos, SSIDs aleatorios o caracteres inválidos | Alerta de "evil twin / rogue AP", listas blancas de BSSIDs corporativos |
| Auth flood | Tabla de autenticación llena, autenticaciones desde MACs inexistentes | Limitación de autenticaciones por radio, control de admisión en el AP |
| WIDS confusion | Alertas incoherentes, tramas de tamaño anómalo | Correlación entre sensores, tuning de umbrales |
| Probe monitor | **Nada** — es pasivo | El único detector: tráfico ARP/MDNS anómalo no aplica; reducir probes de clientes (configurar solo red corporativa) |

> [!info] 802.11w (Protected Management Frames)
> La raíz del problema de la deauth es histórica: las tramas de gestión viajaban en claro sin autenticación. PMF (obligatorio en WPA3, opcional en WPA2‑Enterprise) firma las tramas de gestión, invalidando los modos `d` y `a` por completo. Medir cuántos APs de un cliente **no** tienen PMF activado es un hallazgo de informe de alto valor.

## Errores comunes

1. **"No pasa nada" al atacar**: casi siempre es **canal incorrecto** o interfaz sin modo monitor. Verificar con `iwconfig` que la interfaz está en monitor y en el canal del AP.
2. **`ioctl(SIOCSIWMODE) failed: Operation not permitido`**: falta de privilegios o interfaz no soporta monitor — ejecutar como root y probar con otra tarjeta o controlador.
3. **La deauth sale pero el cliente no se reconecta**: el cliente tiene PMF activo, o el AP filtra clientes con *roaming protection*. Es señal de que la contramedida funciona.
4. **Beacon flood sin efecto**: el canal (`-c`) no coincide con el del escáner del cliente; los clientes solo escanean canales 1/6/11.
5. **Confundir `-a` entre modos**: en el modo `d`, `-a` es el BSSID del AP; en el modo `b`, `-a` es el BSSID del beacon falso. Leer la ayuda del modo (`mdk3 wlan0mon <modo>`) antes de copiar flags de otra guía.
6. **Usar MDK3 para "hackear WiFi"**: no descifra nada. Su outputs son desconexiones, beacons y confusión. El material de ataque real (handshake, PMKID) se captura con airodump/[[Wireshark y Tshark]] y se crackea aparte.

## Cheatsheet

```bash
# Preparar interfaz (ver Aircrack-ng)
airmon-ng start wlan0 && aireplay-ng --test wlan0mon

# Ayuda de un modo
mdk3 wlan0mon <b|a|d|p|s|w>

# Deauth: todos los clientes de un AP
mdk3 wlan0mon d -a <BSSID>

# Deauth: un cliente concreto (captura de handshake)
mdk3 wlan0mon d -a <BSSID> -c <MAC_cliente>

# Deauth masiva desde captura (DoS de área)
mdk3 wlan0mon d -m captura.cap

# Beacon flood WPA2 en canal 6
mdk3 wlan0mon b -c 6 -s -n "SSID_falso"

# Auth flood contra un AP (DoS)
mdk3 wlan0mon a -a <BSSID>

# Monitoreo pasivo de probes
mdk3 wlan0mon p -q
```

## Referencias

- Sitio oficial de la familia: <https://www.aircrack-ng.org/> (aircrack-ng incluye mdk4)
- Repositorio de mdk4 (sucesor): <https://github.com/wiire-a/mdk4>
- IEEE 802.11‑2016 §11.3 — gestión de autenticación/deautenticación (la debilidad explotada)
- 802.11w Protected Management Frames — <https://en.wikipedia.org/wiki/IEEE_802.11w-2009>
- Conceptos previos: [[Hacking WiFi Fundamentos y Ataques]]
- Suite de captura y crackeo: [[Aircrack-ng Suite de Herramientas]]
- Análisis de tramas capturadas: [[Wireshark y Tshark]]

## Relacionadas

- [[Hacking WiFi Fundamentos y Ataques]] — marco completo del ataque inalámbrico
- [[Aircrack-ng Suite de Herramientas]] — aireplay-ng/airodump-ng, el flujo de handshake
- [[Wireshark y Tshark]] — verificar las tramas inyectadas
- [[Reconocimiento y Escaneo de Redes y Web]] — fase en la que vivió esta herramienta
