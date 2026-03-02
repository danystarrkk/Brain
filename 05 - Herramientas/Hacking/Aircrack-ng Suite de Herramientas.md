---
aliases:
  - Aircrack-ng
  - Auditoría Wi-Fi
tags:
  - hacking/inalambrico
estado: 🟢 Terminado
---
# Aircrack-ng: Suite de Herramientas para Auditoría Inalámbrica

**Aircrack-ng** es una suite de herramientas de seguridad informática diseñada para evaluar y mejorar la seguridad de redes inalámbricas, especialmente Wi-Fi. Utilizada principalmente en entornos de pruebas de penetración y auditorías de seguridad, Aircrack-ng permite a los usuarios realizar diversas actividades relacionadas con la seguridad de redes, incluyendo:

- **Captura de paquetes:** Permite la captura de tráfico de red inalámbrica para el análisis posterior.
  > [!info] Captura de Paquetes
  > La captura de paquetes se realiza poniendo la interfaz inalámbrica en modo monitor. Esto permite a la tarjeta de red escuchar todo el tráfico en el canal seleccionado, independientemente de si los paquetes están dirigidos a ella o no. Los formatos de captura comunes incluyen `.cap` (libpcap), que puede ser analizado con herramientas como Wireshark.

- **Análisis de redes Wi-Fi:** Proporciona información detallada sobre redes Wi-Fi cercanas, incluyendo nombres de red (ESSID), direcciones MAC de puntos de acceso (BSSID), canales utilizados y clientes conectados.
  > [!tip] Identificación de Redes
  > El BSSID (Basic Service Set Identifier) es la dirección MAC del punto de acceso, mientras que el ESSID (Extended Service Set Identifier) es el nombre de la red Wi-Fi visible para los usuarios. Conocer ambos es crucial para ataques dirigidos.

- **Inyección de paquetes:** Facilita la inyección de tráfico de red simulado para pruebas de inseguridad y vulnerabilidades.
  > [!warning] Inyección de Paquetes
  > La inyección de paquetes requiere una tarjeta de red inalámbrica compatible y *drivers* específicos que soporten esta funcionalidad. Es fundamental para ataques como la desautenticación o la reinyección de paquetes ARP para acelerar la captura de IVs en WEP.

- **Crackeo de claves de cifrado:** Especialmente conocido por su capacidad para crackear claves WEP y WPA/WPA2-PSK mediante el uso de diccionarios de contraseñas y análisis de paquetes capturados.
  > [!info] Métodos de Crackeo
  > Para WEP, Aircrack-ng explota debilidades en el algoritmo RC4 y la reutilización de Vectores de Inicialización (IVs). Para WPA/WPA2-PSK, se basa en ataques de diccionario contra el *handshake* de 4 vías capturado, o ataques de fuerza bruta si el diccionario no es suficiente.

- **Desencriptación de tráfico:** Permite la desencriptación de capturas de tráfico Wi-Fi cifrado (WEP y WPA) para el análisis de datos.
  > [!tip] Análisis Post-Explotación
  > Desencriptar el tráfico capturado es esencial para el análisis forense y para entender qué datos se transmitieron una vez que la clave ha sido obtenida. Esto puede revelar credenciales, información sensible o patrones de comunicación.

Aircrack-ng es una herramienta poderosa, pero debe utilizarse con responsabilidad y en entornos autorizados, ya que el uso indebido puede violar la privacidad y las leyes de seguridad informática. Es ampliamente utilizada por profesionales de seguridad informática, administradores de redes y entusiastas de la tecnología para asegurar y fortalecer las configuraciones de red inalámbrica.

## Uso de Comandos

### Airmon-ng

- **Descripción:** Este comando se utiliza para gestionar interfaces inalámbricas y activar el modo monitor, necesario para la captura de paquetes.

- **Uso básico:**

  ```bash
  airmon-ng start wlan0
  ```

Este comando activará el modo monitor en la interfaz `wlan0`.
  > [!info] Modo Monitor
  > El modo monitor (también conocido como modo RFMON o modo de monitoreo) permite a una interfaz inalámbrica capturar todos los paquetes que "ve" en un canal específico, sin asociarse a un punto de acceso. Es crucial para el reconocimiento pasivo y la inyección de paquetes. Al iniciar `airmon-ng start wlan0`, a menudo se crea una nueva interfaz virtual como `wlan0mon` o `mon0`.

- **Parámetros adicionales:**
  - `start <interfaz>`: Inicia el modo monitor en la interfaz especificada.
  - `stop <interfaz>`: Detiene el modo monitor en la interfaz especificada.
  - `check`: Verifica las interfaces y sus estados actuales, identificando procesos que podrían interferir con el modo monitor (ej. NetworkManager).

### Airodump-ng

- **Descripción:** Airodump-ng se utiliza para capturar y mostrar información detallada sobre redes Wi-Fi cercanas, incluyendo BSSIDs, canales y estaciones asociadas.

- **Uso básico:**

  ```bash
  airodump-ng wlan0mon
  ```

Este comando mostrará una lista de redes Wi-Fi detectadas y estaciones asociadas en la interfaz `wlan0mon`.
  > [!tip] Información Recopilada
  > Airodump-ng muestra en tiempo real el BSSID, ESSID, canal, potencia de la señal (PWR), número de paquetes de datos (#Data), y el número de clientes conectados (STATION). Esta información es vital para seleccionar un objetivo y planificar un ataque.

- **Parámetros adicionales:**
  - `-c <canal>`: Especifica el canal Wi-Fi para escanear. Es recomendable fijar un canal (`-c <canal>`) para concentrar la captura en una red específica y mejorar la eficiencia.
  - `--bssid <BSSID>`: Filtra por un BSSID específico, mostrando solo la información de ese punto de acceso.
  - `-w <archivo>`: Guarda la captura en un archivo específico (ej. `captura.cap`). Esto es esencial para el crackeo posterior de claves WPA/WPA2.
  - `--output-format <formato>`: Especifica el formato de salida (pcap, csv, netxml, kismet, etc.). El formato `pcap` es el más común para análisis con Wireshark.

### Aireplay-ng

- **Descripción:** Aireplay-ng se utiliza para generar y transmitir paquetes de red para pruebas de inyección y destrucción de redes Wi-Fi.

- **Uso básico (ataque de desautenticación):**

  ```bash
  aireplay-ng -0 5 -a 00:11:22:33:44:55 wlan0mon
  ```

Este comando envía 5 paquetes de desautenticación a la dirección MAC `00:11:22:33:44:55` (BSSID del AP) en la interfaz `wlan0mon`.
  > [!warning] Ataque de Desautenticación
  > El ataque de desautenticación (`-0`) envía paquetes de desautenticación a clientes conectados a un AP, forzándolos a desconectarse y, potencialmente, a reconectarse. Durante la reconexión, se puede capturar el *handshake* de 4 vías WPA/WPA2, necesario para el crackeo de la clave.

- **Parámetros adicionales:**
  - `-1`, `-2`, `-3`: Modos específicos de ataque.
    - `-1`: Ataque de autenticación falsa (para WEP).
    - `-2`: Ataque de reinyección de ARP (para WEP, para generar IVs).
    - `-3`: Ataque de reinyección estándar (para WEP).
  - `-b <BSSID>`: Especifica el BSSID del objetivo (dirección MAC del AP).
  - `-h <MAC>`: Especifica la dirección MAC del atacante (su propia MAC, a menudo falsificada).
  - `-e <ESSID>`: Especifica el nombre de la red Wi-Fi (ESSID) para ataques dirigidos.
  - `-c <MAC_cliente>`: Especifica la dirección MAC de un cliente específico para desautenticar solo a ese cliente.

### Aircrack-ng

- **Descripción:** Aircrack-ng se utiliza para crackear claves de redes Wi-Fi cifradas, como WEP y WPA/WPA2-PSK.

- **Uso básico (crackear clave WPA/WPA2-PSK):**

  ```bash
  aircrack-ng -a2 -b 00:11:22:33:44:55 -w diccionario.txt captura-01.cap
  ```

Este comando intenta crackear la clave WPA/WPA2-PSK utilizando un diccionario de contraseñas.
  > [!info] Proceso de Crackeo
  > Para WPA/WPA2, Aircrack-ng necesita un archivo `.cap` que contenga el *handshake* de 4 vías. Luego, compara los *hashes* del *handshake* con los *hashes* generados a partir de las palabras del diccionario. Si encuentra una coincidencia, la clave es revelada. La eficiencia depende de la calidad del diccionario y la complejidad de la contraseña.

- **Parámetros adicionales:**
  - `-a1`, `-a2`: Modos de ataque.
    - `-a1`: Ataque WEP.
    - `-a2`: Ataque WPA/WPA2-PSK.
  - `-b <BSSID>`: Especifica el BSSID del objetivo para filtrar la captura.
  - `-w <archivo>`: Utiliza un archivo de diccionario para intentar descifrar la clave. Este es el método más común para WPA/WPA2.
  - `-n <num-paquetes>`: Número de paquetes de datos capturados para el ataque WEP. Cuantos más IVs se capturen, mayor la probabilidad de éxito en WEP.
  - `-K`: Utiliza el ataque PTW (Pyshkin, Tews, Weinmann) para WEP, que es más rápido y eficiente.

### Airdecap-ng

- **Descripción:** Airdecap-ng se utiliza para desencriptar archivos de captura de tráfico WEP y WPA.

- **Uso básico (desencriptar captura WEP):**

  ```bash
  airdecap-ng -w claveWEP.txt captura-wep.cap
  ```

Este comando desencripta la captura de tráfico WEP utilizando la clave especificada en `claveWEP.txt`.
  > [!tip] Desencriptación de Tráfico
  > Una vez que se ha obtenido la clave (WEP o WPA/WPA2), Airdecap-ng permite desencriptar un archivo `.cap` previamente capturado. Esto genera un nuevo archivo `.cap` con el tráfico en texto claro, que puede ser analizado con Wireshark para extraer información útil.

- **Parámetros adicionales:**
  - `-b <BSSID>`: Especifica el BSSID del punto de acceso para filtrar la desencriptación.
  - `-e <ESSID>`: Especifica el ESSID (nombre de la red) del punto de acceso.
  - `-p <clave_wpa>`: Especifica la clave WPA/WPA2 directamente en la línea de comandos.
  - `-w <archivo_clave>`: Especifica un archivo que contiene la clave WEP o WPA/WPA2.