---
aliases: ["macchanger", "cambiar mac", "MAC Randomizer", "Spoofing MAC", "mac changer"]
tags: [tool/macchanger]
tipo: herramienta
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Tipos de IP y Dirección MAC]]"
  - "[[Conceptos Fundamentales de Redes]]"
  - "[[Hacking WiFi Fundamentos y Ataques]]"
  - "[[Hacking Wi-Fi Tarjetas de Red]]"
  - "[[Arp Scan Herramienta de Descubrimiento]]"
  - "[[Comandos Básicos de Sistema y Red en Linux]]"
fuente: "https://github.com/alobbs/macchanger"
---

# Macchanger: Herramienta para Modificación de Dirección MAC

**macchanger** cambia (o restablece) la dirección MAC de una interfaz de red en Linux: aleatoriza, clona una MAC concreta, conserva el fabricante o restaura la de fábrica. Se usa antes de subir una interfaz en pentests (evitar filtrados por MAC, saltar filtros de laboratorio, anonimizar ruido en Wi-Fi) y en forense/redes para reproducir escenarios.

> [!info] ¿Qué es una dirección MAC?
> Identificador de 48 bits (6 octetos) de la capa 2. Los **primeros 24 bits (3 octetos) son el OUI** del fabricante; los últimos 24, el identificador de la placa. Dos bits del primer octeto importan para auditoría: **I/G** (bit 0: multicast) y **U/L** (bit 1: *locally administered*). Si el bit U/L está activo, la MAC es "localmente administrada" — es decir, **aleatoria o spoofeda**. Ver [[Tipos de IP y Dirección MAC]].

## Instalación

```bash
# Kali / Debian
sudo apt update && sudo apt install macchanger

# Fedora / RHEL
sudo dnf install macchanger

# Ver ayuda y base OUI incluida
macchanger --help
```

## Tabla de parámetros

| Parámetro | Significado | Ejemplo |
|---|---|---|
| `-s`, `--show` | Muestra la MAC **actual** y la **permanente** (de fábrica) | `macchanger -s eth0` |
| `-r`, `--random` | MAC **enteramente aleatoria** (fabricante aleatorio también) | `macchanger -r eth0` |
| `-a`, `--another` | MAC aleatoria **de otro fabricante** (cambia el OUI, no el tuyo) | `macchanger -a eth0` |
| `-e`, `--ending` | **Conserva los 3 octetos del fabricante** y aleatoriza solo los 3 finales | `macchanger -e eth0` |
| `-m`, `--mac <mac>` | Fija una MAC **concreta** (clonar/establecer) | `macchanger -m 00:11:22:33:44:55 eth0` |
| `-p`, `--permanent` | Restaura la MAC **original de fábrica** | `macchanger -p eth0` |
| `-l`, `--list` | Imprime la **base de vendors/OUI** conocidos (no las interfaces) | `macchanger -l \| grep -i apple` |
| `-q`, `--quiet` | Sin salida informativa (para scripts) | `macchanger -q -r eth0` |
| `-b`, `--brief` | Salida resumida | — |
| `-h`, `--help` / `-V` | Ayuda / versión | `macchanger --help` |

> [!warning] Correcciones respecto a notas previas
> - `-e/--ending` **no** genera "una MAC que no coincide con ninguna OUI": hace exactamente lo contrario, **mantiene el vendor** y solo cambia la parte final. La aleatorización total es `-r`.
> - `-l` **no** lista interfaces: imprime los vendors conocidos. Las interfaces se listan con `ip -br link`.
> - La MAC **permanente** no cambia en el firmware: `macchanger -s` la muestra como `Permanent MAC` para que puedas volver atrás con `-p`.

## Flujo correcto: la interfaz antes/después de `up`

Los drivers rechazan cambiar la MAC con la interfaz activa, y NetworkManager/wpa_supplicant la re-escriben al subirla. Orden correcto:

```bash
# 1. Bajar la interfaz
sudo ip link set eth0 down

# 2. Cambiar la MAC
sudo macchanger -r eth0

# 3. Subirla de nuevo
sudo ip link set eth0 up

# 4. Verificar
ip -br link show eth0
cat /sys/class/net/eth0/address
```

```text
Current MAC:   00:1a:2b:3c:4d:5e (Asustek)
Permanent MAC: 00:1a:2b:3c:4d:5e (Asustek)
New MAC:       7e:3f:9a:11:22:33 (unknown)
```

| Campo | Significado |
|---|---|
| `Current MAC` | La que tenía antes del cambio |
| `Permanent MAC` | La de fábrica (recuperable con `-p`) |
| `New MAC` | La resultante; `(unknown)` = OUI no está en la base local |
| `x2/x6/xA/xF` en el 1º octeto | Bit U/L activo → **MAC localmente administrada** (esto es lo que ven los defensores) |

### Truco: ¿es esta MAC aleatoria?

```bash
# Segundo dígito hex del primer octeto ∈ {2,3,6,7,A,B,E,F} → bit U/L = 1 → randomizada
mac=$(cat /sys/class/net/wlan0/address); n=${mac:1:1}
case "$n" in 2|3|6|7|a|b|e|f) echo "localmente administrada (random)";; *) echo "OUI de fabricante";; esac
```

## Recetas: los 10 comandos reales

```bash
# 1. Ver la MAC actual y la de fábrica
macchanger -s eth0

# 2. Aleatorización completa (el uso clásico antes de empezar)
sudo ip link set eth0 down && sudo macchanger -r eth0 && sudo ip link set eth0 up

# 3. Aleatorizar "creíble" (mismo fabricante que tu NIC)
sudo ip link set eth0 down && sudo macchanger -e eth0 && sudo ip link set eth0 up

# 4. Otro fabricante (para pasar filtros por vendor en el laboratorio)
sudo macchanger -a eth0

# 5. Clonar una MAC concreta (duplicar acceso en LAN autorizada)
sudo ip link set eth0 down && sudo macchanger -m 00:11:22:33:44:55 eth0 && sudo ip link set eth0 up

# 6. Restaurar la original al terminar
sudo macchanger -p eth0

# 7. Buscar un OUI concreto en la base
macchanger -l | grep -i -A1 "apple"

# 8. En scripts (silencioso, sin salir del flujo)
sudo macchanger -q -r wlan0 && ip -br link show wlan0

# 9. Wi-Fi: quitar wpa_supplicant del medio antes del cambio
sudo systemctl stop wpa_supplicant && sudo ip link set wlan0 down && sudo macchanger -r wlan0 && sudo ip link set wlan0 up

# 10. Verificación doble (sysfs + iproute2)
ip -br link show eth0; cat /sys/class/net/eth0/address
```

**Regla operativa:** *cambio de MAC siempre entre `down` y `up`*. Si la cambias con la interfaz arriba, o bien el driver falla, o bien el gestor de red (NetworkManager) la restaura en cuanto renegocie.

## Persistencia

| Método | Cómo | Ámbito |
|---|---|---|
| `/etc/network/interfaces` | `pre-up macchanger -r eth0` bajo la interfaz | Debian/ifupdown |
| NetworkManager (por conexión) | `nmcli connection modify <name> cloned-mac-address random` (o una MAC fija) | Conexión concreta |
| NetworkManager (Wi-Fi por defecto) | `wifi.cloned-mac-address=random` en `/etc/NetworkManager/conf.d/` | Todas las Wi-Fi |
| wpa_supplicant | Opciones `mac_addr=1` / `preassoc_mac_addr=1` en el bloque de red | Asociación temprana |
| Reglas udev / systemd.link | Asignar MAC al detectar el hardware | Arranque temprano |
| Directo (sin persistir) | `ip link set dev eth0 address ...` | Cambio puntual |

> [!warning] NetworkManager gana
> Si cambias la MAC a mano con NetworkManager activo, es muy probable que la vuelva a sobreescribir. O lo gestionas con `nmcli`/`cloned-mac-address`, o pones la interfaz en `managed=no`, o usas `macchanger` entre `down`/`up` y compruebas con `ip -br link` inmediatamente.

## Casos de uso reales (y su límite)

| Caso | Comando base | Nota |
|---|---|---|
| Evitar que tu NIC real quede registrada en logs | `macchanger -r` | El cambio es local; el resto de la sesión sigue identificable |
| Saltar filtrado MAC de un switch/router **del laboratorio** | `-m <mac>permitida>` | Solo en infraestructura autorizada |
| Prueba de NAC/802.1X | `-r` + reconectar | Esperable que el NAC lo detecte y bloquee (eso es lo que pruebas) |
| Evitar correlación entre redes Wi-Fi | `-r` por sesión | Ver caveat de detección abajo |
| Recuperar tras un experimento | `-p` | Siempre posible con `Permanent MAC` |

## Detección y contramedidas (el caveat importante)

> [!warning] Cambiar la MAC no te hace anónimo
> Un filtro de MAC es un control de acceso débil, no privacidad. La randomización **rompe correlación por MAC**, pero deja otras señales:

| Señal | Qué revela | Quién la ve |
|---|---|---|
| **Bit U/L activo** (`02:`, `6e:`, `a4:`…) | "MAC randomizada" de un vistazo | arpwatch, WIPS, forense de logs |
| **Cambio de MAC en la misma IP/puerto** | Mismo dispositivo cambiando de identidad | arpwatch (`mac change` alerts), switch logs, SIEM |
| **Port security / sticky MAC** | Más de N MACs por puerto | Switch Cisco: `err-disabled` |
| **802.1X / NAC** | Identidad real dentro del EAP (PEAP expone el usuario) | Control de acceso corporativo |
| **DHCP fingerprint** (opciones 55/61), TTL, MTU, TCP/IP stack | Sigue siendo "la misma clase de equipo" | DHCP logs, fingerprinting pasivo |
| **Wi-Fi probe requests / 802.11 sequence numbers** | Correlación entre sesiones con la misma radio | WIDS |
| **Cookies, TLS fingerprint (JA3), horarios** | Identidad de aplicación, no de red | Servicios web |

**Defensa recomendada (equipo red):** MAC filtering + 802.1X, DAI (Dynamic ARP Inspection), DHCP snooping, y detección de cambios de MAC con arpwatch/SIEM. **Del lado del pentester:** documenta siempre en el alcance que vas a randomizar MACs (afecta a los logs del cliente) y restaura con `-p` al terminar.

## Errores comunes (mensaje → arreglo)

1. **`SIOCSIFHWADDR: Device or resource busy`** (o `Cannot assign requested address`) — interfaz arriba.
   → `sudo ip link set eth0 down` → `macchanger` → `up`.
2. **`Permission denied` / `Cannot open device`** — falta root o nombre de interfaz mal escrito.
   → `sudo` + `ip -br link` para el nombre exacto (`enp3s0`, no `eth0`).
3. **La MAC "cambia" pero vuelve a la original** — NetworkManager/wpa_supplicant la reasigna.
   → Usar `nmcli ... cloned-mac-address` o parar el gestor durante el cambio.
4. **Sin conectividad tras el cambio** — el equipo de red filtra por MAC o el DHCP no renueva.
   → Liberar/renew (`dhclient -r eth0 && dhclient eth0`), o volver a `-p` si estaba filtrada.
5. **`New MAC: ... (unknown)`** — el OUI generado no está en la base local: es **normal** con `-r`.
   → No es error; si necesitas vendor conocido, usa `-a` (otro vendor real) o `-e`.
6. **Wi-Fi no cambia** — driver rtl/ath con restricciones.
   → Alternativa directa: `sudo ip link set wlan0 address XX:XX:XX:XX:XX:XX` con la interfaz abajo; si falla, consultar `iw dev` y el soporte del chipset (ver [[Hacking Wi-Fi Tarjetas de Red]]).
7. **Se perdió el MAC permanente** — no se puede: `Permanent MAC` está en el driver/hardware.
   → `macchanger -p` lo restaura siempre; si no, reinicia el equipo (en VMs puede requerir regenerar la interfaz).
8. **Cambio hecho en la interfaz equivocada** (p. ej. la de la VPN) — doble confusión de red.
   → Verificar con `ip -br addr` **antes**; listar con `ip -br link`.

## Cheatsheet — los 10 comandos del día a día

```bash
macchanger -s eth0
macchanger -l | grep -i intel
sudo ip link set eth0 down
sudo macchanger -r eth0
sudo macchanger -e eth0
sudo macchanger -a eth0
sudo macchanger -m 00:11:22:33:44:55 eth0
sudo macchanger -p eth0
sudo ip link set eth0 up
cat /sys/class/net/eth0/address
```

| Tarea | Comando |
|---|---|
| Ver MAC actual/permanente | `macchanger -s <if>` |
| Random total | `ip link set <if> down; macchanger -r <if>; ip link set <if> up` |
| Random manteniendo vendor | `macchanger -e <if>` |
| Otro vendor | `macchanger -a <if>` |
| Fijar/clonar | `macchanger -m <mac> <if>` |
| Restaurar | `macchanger -p <if>` |
| Buscar vendor | `macchanger -l \| grep -i <marca>` |
| Verificar | `cat /sys/class/net/<if>/address` |
| Random persistente (NM) | `nmcli con mod <c> cloned-mac-address random` |
| Ayuda | `macchanger --help` |

## Comparativa de métodos

| Método | Aleatorización inteligente | Base OUI | Muestra permanente | Riesgo |
|---|---|---|---|---|
| **macchanger** | Sí (`-e`, `-a`) | Sí | Sí (`-s`) | Bajo (restaurable) |
| `ip link set address` | No | No | No | Medio (sin registro) |
| NetworkManager `cloned-mac-address` | Sí (`random`) | Parcial | No | Bajo, pero NM manda |
| wpa_supplicant `mac_addr` | Sí | No | No | Solo Wi-Fi |

## Referencias

- [Repositorio oficial (alobbs/macchanger)](https://github.com/alobbs/macchanger) — README y man page
- IEEE OUI registry — base oficial de fabricantes (mapeada por `macchanger -l`)
- HackTricks — *Wireless/Network attacks* (contexto de uso autorizado)
- [[Arp Scan Herramienta de Descubrimiento]] — cómo se ve tu MAC randomizada desde el lado del escáner

## Relacionadas

- [[Tipos de IP y Dirección MAC]] — teoría de la MAC, OUI y bits U/L
- [[Conceptos Fundamentales de Redes]] — capa 2 y ARP
- [[Hacking WiFi Fundamentos y Ataques]] — Wi-Fi: randomización y ataques relacionados
- [[Hacking Wi-Fi Tarjetas de Red]] — chipsets que permiten cambiar MAC en modo monitor
- [[Arp Scan Herramienta de Descubrimiento]] / [[netdiscover]] — qué ve el otro extremo
- [[Comandos Básicos de Sistema y Red en Linux]] — `ip link` como base de todo el flujo
