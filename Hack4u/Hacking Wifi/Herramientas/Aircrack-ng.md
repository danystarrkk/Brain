-----
# Qué es?

**Aircrack-ng** es una suite de herramientas de seguridad informática diseñada para evaluar y mejorar la seguridad de redes inalámbricas, especialmente Wi-Fi. Utilizada principalmente en entornos de pruebas de penetración y auditorías de seguridad, Aircrack-ng permite a los usuarios realizar diversas actividades relacionadas con la seguridad de redes, incluyendo:

- **Captura de paquetes:** Permite la captura de tráfico de red inalámbrica para el análisis posterior.
  
- **Análisis de redes Wi-Fi:** Proporciona información detallada sobre redes Wi-Fi cercanas, incluyendo nombres de red (ESSID), direcciones MAC de puntos de acceso (BSSID), canales utilizados y clientes conectados.

- **Inyección de paquetes:** Facilita la inyección de tráfico de red simulado para pruebas de inseguridad y vulnerabilidades.

- **Crackeo de claves de cifrado:** Especialmente conocido por su capacidad para crackear claves WEP y WPA/WPA2-PSK mediante el uso de diccionarios de contraseñas y análisis de paquetes capturados.

- **Desencriptación de tráfico:** Permite la desencriptación de capturas de tráfico Wi-Fi cifrado (WEP y WPA) para el análisis de datos.

Aircrack-ng es una herramienta poderosa pero debe utilizarse con responsabilidad y en entornos autorizados, ya que el uso indebido puede violar la privacidad y las leyes de seguridad informática. Es ampliamente utilizada por profesionales de seguridad informática, administradores de redes y entusiastas de la tecnología para asegurar y fortalecer las configuraciones de red inalámbrica.
# Uso de Comandos

## Airmon-ng

- **Descripción:** Este comando se utiliza para gestionar interfaces inalámbricas y activar el modo monitor, necesario para la captura de paquetes.

- **Uso básico:**

  ```bash
  airmon-ng start wlan0
  ```

Este comando activará el modo monitor en la interfaz `wlan0`.

- **Parámetros adicionales:**
  - `start <interfaz>`: Inicia el modo monitor en la interfaz especificada.
  - `stop <interfaz>`: Detiene el modo monitor en la interfaz especificada.
  - `check`: Verifica las interfaces y sus estados actuales.

## Airodump-ng

- **Descripción:** Airodump-ng se utiliza para capturar y mostrar información detallada sobre redes Wi-Fi cercanas, incluyendo BSSIDs, canales, y estaciones asociadas.

- **Uso básico:**

  ```bash
  airodump-ng wlan0mon
  ```

Este comando mostrará una lista de redes Wi-Fi detectadas y estaciones asociadas en la interfaz `wlan0mon`.

- **Parámetros adicionales:**
  - `-c <canal>`: Especifica el canal Wi-Fi para escanear.
  - `--bssid <BSSID>`: Filtra por un BSSID específico.
  - `-w <archivo>`: Guarda la captura en un archivo específico.
  - `--output-format <formato>`: Especifica el formato de salida (pcap, csv, etc.).

## Aireplay-ng

- **Descripción:** Aireplay-ng se utiliza para generar y transmitir paquetes de red para pruebas de inyección y destrucción de redes Wi-Fi.

- **Uso básico (ataque de des autenticación):**

  ```bash
  aireplay-ng -0 5 -a 00:11:22:33:44:55 wlan0mon
  ```

Este comando envía 5 paquetes de des autenticación a la dirección MAC `00:11:22:33:44:55` en la interfaz `wlan0mon`.

- **Parámetros adicionales:**
  - `-1`, `-2`, `-3`: Modos específicos de ataque (deauth, ARP request replay, interactive packet replay).
  - `-b <BSSID>`: Especifica el BSSID del objetivo.
  - `-h <MAC>`: Especifica la dirección MAC del atacante.
  - `-e <ESSID>`: Especifica el nombre de la red Wi-Fi (ESSID).

## Aircrack-ng

- **Descripción:** Aircrack-ng se utiliza para crackear claves de redes Wi-Fi cifradas, como WEP y WPA/WPA2-PSK.

- **Uso básico (crackear clave WPA/WPA2-PSK):**

  ```bash
  aircrack-ng -a2 -b 00:11:22:33:44:55 -w diccionario.txt captura-01.cap
  ```

Este comando intenta crackear la clave WPA/WPA2-PSK utilizando un diccionario de contraseñas.

- **Parámetros adicionales:**
  - `-a1`, `-a2`: Modos de ataque (WEP, WPA/WPA2).
  - `-b <BSSID>`: Especifica el BSSID del objetivo.
  - `-w <archivo>`: Utiliza un archivo de diccionario para intentar descifrar la clave.
  - `-n <num-paquetes>`: Número de paquetes de datos capturados para el ataque WEP.

## Airdecap-ng

- **Descripción:** Airdecap-ng se utiliza para desencriptar archivos de captura de tráfico WEP y WPA.

- **Uso básico (desencriptar captura WEP):**

  ```bash
  airdecap-ng -w claveWEP.txt captura-wep.cap
  ```

Este comando desencripta la captura de tráfico WEP utilizando la clave especificada en `claveWEP.txt`.

- **Parámetros adicionales:**
  - `-b <BSSID>`: Especifica el BSSID del punto de acceso.
  - `-e <ESSID>`: Especifica el ESSID (nombre de la red) del punto de acceso.