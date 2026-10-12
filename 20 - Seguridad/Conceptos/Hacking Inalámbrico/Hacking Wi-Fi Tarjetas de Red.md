---
tags:
  - ciberseguridad
  - inalambrico
  - hacking-red
tipo: concepto
estado: terminado
---

# Hacking Wi-Fi: Requisitos de Hardware — Tarjetas de Red

No se puede auditar Wi-Fi con cualquier adaptador: hacen falta dos capacidades que las tarjetas de portátil normal **no** ofrecen — **modo monitor** (capturar tramas 802.11 crudas, incluso las dirigidas a otros) e **inyección** (enviar tramas fabricadas: deauth, spoofed, beacon). La clave no es la marca ni el precio del dongle, sino el **chipset interior** y el **driver** que lo gobierna.

> [!warning] Consideraciones legales
> Auditar redes Wi-Fi sin autorización explícita es ilegal en la mayoría de jurisdicciones. Este material es estrictamente educativo y de laboratorio controlado: ten permiso escrito antes de capturar, inyectar o desautenticar en cualquier red que no sea tuya.

## 1. La pila de drivers: por qué el chipset lo es todo

```
airmon-ng / airodump-ng / aireplay-ng / wireshark
        │  (nl80211 / libpcap)
   cfg80211  ──  mac80211  ──  driver del chipset (ath9k_htc, rt2800usb, mt76…)
        │                          │
      nl80211                   firmware
        │
   kernel (Linux)
```

| Capa | Papel | Qué falla si falta |
|---|---|---|
| **nl80211 / cfg80211** | API del kernel para configurar interfaces | Herramientas antiguas con `iwconfig` (WEXT) no funcionan |
| **mac80211** | Stack 802.11 *softMAC* de Linux: frame management, monitor, inyección | Sin él: no hay modo monitor ni inyección |
| **Driver + firmware** | Traduce a hardware real; implementa canales, TX power | Chipset "soportado" pero driver incompleto = solo `Managed` |
| **HardMAC vs softMAC** | En hardMAC (p. ej. muchos Broadcom/Intel) el 802.11 vive en el firmware | El kernel **no puede** inyectar ni poner monitor fiable |

> [!info] SoftMAC vs HardMAC
> - **SoftMAC** (Atheros ath9k, Ralink rt2800usb, MediaTek mt76, Realtek rtl8xxx): la lógica 802.11 está en el driver de Linux → **monitor + inyección posibles**. Es lo que buscas.
> - **HardMAC** (Intel iwlwifi, Broadcom wiphy): el firmware gestiona el MAC → monitor a veces limitado, **inyección casi nunca**. Sirven para captura básica, no para ataques.

## 2. Tabla de compatibilidad por chipset

| Chipset / adaptadores típicos | Driver | Monitor | Inyección | Bandas | Veredicto |
|---|---|---|---|---|---|
| **Atheros AR9271** (Alfa AWUS036NHA, TL-WN722N v1) | `ath9k_htc` (en árbol) | ✅ | ✅ excelente | 2,4 GHz | **El referente**: plug & play, inyección impecable |
| **Atheros AR9300/AR9462** (PCIe) | `ath9k` (en árbol) | ✅ | ✅ | 2,4 (+5 en algunos) | Ideal en portátiles/desktop |
| **Ralink/MediaTek RT3070, RT5370** | `rt2800usb` (en árbol) | ✅ | ✅ | 2,4 GHz | Muy buena alternativa barata |
| **MediaTek MT7612U / MT7661U** | `mt76` (en árbol) | ✅ | ✅ | 2,4 + **5 GHz** | Mejor opción dual-band con driver en kernel |
| **Realtek RTL8187L** (Alfa AWUS036H clásica) | `rtl8187` (en árbol) | ✅ | ✅ | 2,4 GHz | Leyenda de Kali; 802.11g, lenta pero fiable |
| **Realtek RTL8812AU/8812BU** (Alfa ACH/ACHM) | `aircrack-ng/rtl8812au` (fuera de árbol) | ✅ | ✅ | 2,4 + 5 GHz, 802.11ac | Potente; **requiere DKMS** y compila contra el kernel en uso |
| **Realtek RTL8188EU / RTL8821CU** | `rtl8xxxu` / staging / out-of-tree | ⚠️ parcial | ❌/⚠️ | 2,4 GHz | Evitar: funciones incompletas |
| **Intel AX200 / 8265** (iwlwifi) | `iwlwifi` (en árbol) | ⚠️ limitado | ❌ | 2,4 + 5 GHz | Bueno para captura de clientes propios, **no para inyección** |
| **Broadcom BCM43xx** (brcmfmac/brcmsmac) | en árbol | ❌/⚠️ | ❌ | 2,4 + 5 GHz | Malo para auditoría; firmware propietario |

> [!tip] Reglas de compra
> 1. **Primero el chipset, después la marca**: dos "misma" Alfa pueden traer chipsets distintos según revisión (comprobar en la web del fabricante o FCC ID).
> 2. Patrón seguro: **Atheros ath9k\*** o **MediaTek mt76** con driver *en árbol* → sin DKMS, sin sorpresas tras actualizar el kernel.
> 3. Para **5 GHz / 802.11ac**: MT7612U o RTL8812AU (con su driver out-of-tree).
> 4. Antena externa + USB con **hub alimentado**: la potencia de TX y la estabilidad del USB importan tanto como el chipset.
> 5. Barato y USB grande ≠ bueno: revisar el chipset real (chipset-identify, `lsusb -v`, fotos de la placa).

## 3. Verificar capacidades antes de comprar/usar

```bash
# 1) Qué interfaces hay y qué driver usa cada una
iw dev                    # interfaces y tipo actual
ip link show
lsusb -v 2>/dev/null | grep -iE 'idVendor|idProduct|Product'   # chipset USB

# 2) Capacidades declaradas por el driver (clave)
sudo iw list | less
#   Supported interface modes:
#        * managed
#        * monitor          ← SIN ESTA LÍNEA NO HAY MODO MONITOR
#   Band 2: ... Frequencies: ... 2412 [1] ... 5180 [36] ...   ← bandas soportadas
#   Station PHY dump ... (algún driver expone capacidades de inyección)

# 3) Listado compacto
sudo iw list | grep -A 8 "Supported interface modes"
```

**Test real de inyección** (la única prueba que no miente — declara y luego verifica):

```bash
# Poner en modo monitor (dos caminos equivalentes)
sudo airmon-ng check kill          # mata NetworkManager/wpa_supplicant que pelean
sudo airmon-ng start wlan0         # → wlan0mon
# o manualmente con mac80211:
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up

# Test de inyección contra un AP conocido (laboratorio)
sudo aireplay-ng --test wlan0mon
#   0% packet injection...  → NO inyecta (driver/tarjeta malos)
#   Injection is working!   → ✅ lista para aireplay/mdk3
```

| Resultado | Significado | Acción |
|---|---|---|
| `iw list` sin `monitor` | Driver no lo permite | Cambiar de tarjeta; no hay workaround |
| Monitor OK, `--test` al 0% | TX filtrado (regulatory domain, driver, potencia) | `iw reg set US`, revisar `txpower`, driver |
| Funciona en 2,4 pero no en 5 | Canal DFS/no permitido en tu reg. domain | Elegir canal no-DFS o `iw reg set` |
| Falla tras actualizar kernel | DKMS out-of-tree descompilado | Recompilar (`dkms install`) o usar chipset en árbol |

```bash
# Utilidades de diagnóstico
sudo iw reg get                  # dominio regulatorio activo
sudo iw dev wlan0 info           # canal, tipo, dirección MAC
sudo iw dev wlan0 set txpower fixed 2000   # 20 dBm (si el driver lo permite)
dmesg | grep -iE 'ath9k|rt2800|mt76|rtl8'  # errores de firmware/driver
```

## 4. Adaptadores USB: patrones recomendados

| Perfil | Chipset ideal | Ejemplo de uso | Compromiso |
|---|---|---|---|
| **Estándar CTF/curso** | AR9271 (ath9k_htc) | 2,4 GHz, WPA2 handshake, deauth | Sin instalación de drivers en Kali |
| **Dual-band / 5 GHz** | MT7612U (mt76) | Auditar 5 GHz, canales DFS | Driver en árbol, estable |
| **Potencia 802.11ac** | RTL8812AU (out-of-tree) | Alto throughput, 5 GHz, hackeo a distancia | DKMS; rompe con updates de kernel |
| **Portátil sin USB libre** | AR9300 PCIe / m.2 | Monitor permanente | Revisar compatibilidad concreta del portátil |

```bash
# Antes de cada sesión de auditoría
sudo airmon-ng check kill        # stop NetworkManager, wpa_supplicant, dhclient
sudo airmon-ng start wlan0
sudo iw dev                      # confirmar wlan0mon y canal
ip link show wlan0mon            # debe estar UP
macchanger -s wlan0mon           # (opcional) spoof de MAC antes de capturar
# Al terminar: airmon-ng stop wlan0mon && systemctl restart NetworkManager
```

> [!tip] Doble interfaz
> Lo ideal es **dos radios**: la integrada del portátil para gestionar/consultar (NetworkManager activo) y una **USB dedicada** en modo monitor para capturar. Así no pierdes Internet ni la consola mientras auditas — y evitas el error clásico de quedarte aislado al matar el servicio de red.

## 5. Límites de doble banda, antenas y potencia

| Aspecto | 2,4 GHz | 5 GHz |
|---|---|---|
| Canales | 1–14 (1–11 US, 1–13 EU, 14 JP) | 36–165 (incl. DFS 52–144) |
| Ancho de canal | 20/40 MHz | 20/40/**80** MHz |
| Alcance / penetración | Mayor alcance, más congestión | Menor alcance, menos interferencia |
| Problemas de captura | Solapamiento (solo 3 canales no solapados: 1, 6, 11) | DFS: el AP puede saltar de canal (radar); capturas inestables |
| Requisitos de tarjeta | Todas | Chipset con soporte 5 GHz + reg. domain que lo permita |

> [!warning] Anchura de canal y modo monitor
> En canales de 40/80 MHz, la captura en un solo canal puede perder tramas de la otra mitad del espectro. Para ataques de handshake: fija **20 MHz** y el canal exacto del AP (`airodump-ng -c <ch> --bssid <AP>`, ver [[Hacking WiFi Fundamentos y Ataques]]).

### Antenas y potencia (lo que sí cambia el alcance)

| Concepto | Qué es | Efecto práctico |
|---|---|---|
| **dBi** | Ganancia direccional de la antena (logarítmica) | +3 dBi ≈ doble alcance efectivo en un eje; un alambrita de 2 dBi vs. 7 dBi cambia el rango real de 10 a 40 m |
| **TX power** | Potencia de transmisión (dBm); legal típico 20 dBm (100 mW) EU | `iw dev wlan0 set txpower fixed 2000` (20 dBm); el inyección depende más del driver que del vatios |
| **Tipo de antena** | Omnidireccional (sector amplio) vs. direccional (Yagi/panel) | Para captura de un objetivo concreto, direccional gana SNR sin ilegalidad de potencia |
| **Sensibilidad RX** | dBm mínimo que el radio decodifica | Chipsets buenos reciben a -85/-90 dBm; los malos se cortan a -75 |
| **Hub USB alimentado** | Alimentación estable del dongle | Sin él, los TX potentes desconectan a mitad de captura |

> [!info] Legalidad de la potencia
> Exceder la potencia legal o añadir amplificadores es **ilegal** en la mayoría de jurisdicciones, independientemente de que la tarjeta lo soporte. El alcance en auditoría se gana con **ganancia de antena y posición**, no con vatios.

## 6. Cómo identificar el chipset real de tu adaptador

Las cajas y las descripciones de Amazon mienten: el chipset solo se ve en el hardware o en los identificadores USB.

```bash
lsusb                             # bus → device → descripción
lsusb -v 2>/dev/null | grep -iE 'idVendor|idProduct|iManufacturer|iProduct'
# Ejemplos típicos:
# 0cf3:9271  Atheros AR9271  (ath9k_htc)   → estrella de la auditoría
# 2357:010c  Realtek RTL8188EU (rtl8xxxu)  → evadir
# 0e8d:7612  MediaTek MT7612U (mt76)      → dual-band en kernel
# 0bda:8812  Realtek RTL8812AU (out-of-tree)
lsmod | grep -iE 'ath9k|rt28|mt76|rtl8|iwlwifi'   # driver realmente cargado
dmesg | tail -30                                # errores de firmware al conectar
```

| Dato | Dónde mirarlo | Cuidado |
|---|---|---|
| **VID:PID** | `lsusb` | El mismo PID puede usarse con chipsets distintos en revisiones posteriores |
| **OUI de la MAC** | `ip link` / `macchanger -s` | Indica el fabricante real del adaptador |
| **FCC ID** | Foto de la etiqueta o base de la FCC | La base de la FCC muestra la foto de la placa y el chipset exacto |
| **Driver cargado** | `lsmod`, `dmesg` | Confirma qué stack gobierna la tarjeta (§1) |

> [!tip] Antes de comprar en segunda mano
> Busca "<modelo> FCC ID" + "teardown": foros y la base de la FCC muestran el chip real. Muchos dongles "compatibles Alfa" traen chipsets de gama baja con el mismo framebuffer plástico.

### Escenarios: qué elegir

| Escenario | Recomendación |
|---|---|
| Curso/CTF en 2,4 GHz con Kali | **AR9271** (AWUS036NHA o equivalente): sin drivers extra |
| Auditoría con 5 GHz | **MT7612U**: driver en árbol, dual-band |
| Portátil con Linux actualizado a menudo | Chipset **en árbol** (ath9k/rt2800usb/mt76): nada de DKMS que se rompa |
| Máxima potencia 802.11ac y aguantas DKMS | RTL8812AU con el driver de aircrack-ng |
| Solo captura pasiva de un entorno propio | Intel iwlwifi integrada sirve; para inyección, USB dedicada |

## 7. Fallos comunes y solución

| Síntoma | Causa probable | Solución |
|---|---|---|
| `airmon-ng start` no crea `wlan0mon` | NetworkManager retiene la interfaz | `airmon-ng check kill` y reintentar como root |
| `ioctl(SIOCSIWMODE) failed` | Driver sin mac80211 / modo inválido | Usar `iw dev … set type monitor`; si falla → tarjeta incompatible |
| La Wi-Fi "desaparece" tras activar monitor | Normal: la misma radio no asocia y captura a la vez | USB dedicado + interface interna para gestión (`wlan0` gestión / `wlan1mon` captura) |
| `aireplay-ng --test` → 0% | Inyección no soportada o TX bloqueado | `iw reg set`, probar otro canal, probar otra tarjeta |
| Capturas con `-1` en PWR | Driver no informa RSSI | Chipset/driver pobre; cambia la tarjeta |
| USB desconectado en carga alto | Fallo de alimentación | Hub USB alimentado |
| Tras `apt upgrade` deja de compilar DKMS | Driver out-of-tree incompatible con kernel nuevo | `dkms status` → reinstalar o migrar a chipset en árbol |
| Errores de firmware en dmesg (`ath9k: firmware missing`) | Paquete `firmware-atheros` ausente | `apt install firmware-atheros` (o `firmware-realtek`, `firmware-misc-nonfree`) |

## 8. Cheatsheet

```bash
iw list | grep -A 8 "Supported interface modes"   # ¿soporta monitor?
lsusb -v | grep -iE 'idVendor|idProduct'          # ¿qué chipset hay dentro?
sudo airmon-ng check kill
sudo airmon-ng start wlan0 → wlan0mon
sudo iw dev wlan0 set type monitor                # alternativa manual
sudo aireplay-ng --test wlan0mon                  # ¿inyecta?
sudo iw reg get / iw reg set US                   # dominio regulatorio
dmesg | grep -i ath9k                             # salud del driver
# cierre:
sudo airmon-ng stop wlan0mon && sudo systemctl start NetworkManager
```

## 9. Referencias

- Aircrack-ng — guía de drivers/mac80211: <https://www.aircrack-ng.org/driver-mac80211.html>
- Compatibilidad de tarjetas: <https://www.aircrack-ng.org/dump.html>
- iw (nl80211): <https://wireless.wiki.kernel.org/documentation/usage/iw>
- mac80211: <https://wireless.wiki.kernel.org/documentation/driver-mac80211>
- Alfa/foros de hardware: revisar siempre el chipset real antes de comprar

## 10. Relacionadas

- [[Hacking WiFi Fundamentos y Ataques]] — protocolo 802.11, handshake y flujos de ataque
- [[Aircrack-ng Suite de Herramientas]] — airmon/airodump/aireplay/aircrack
- [[MDK3 Herramienta de Auditoría Inalámbrica]] — beacon flood, disassociation
- [[Wireshark y Tshark]] — captura y filtros 802.11
- [[Macchanger Herramienta de Modificación MAC]] — anonimizar la MAC antes de capturar
