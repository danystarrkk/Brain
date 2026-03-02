---
aliases: MDK3
tags:
  - hacking/herramientas
  - hacking/inalambrico
estado: 🟢 Terminado
---
# MDK3: Herramienta de Auditoría y Pruebas de Penetración Inalámbrica

**MDK3** es una herramienta de prueba de penetración diseñada para explorar y explotar vulnerabilidades en redes inalámbricas. Es ampliamente utilizada por investigadores de seguridad y profesionales de pruebas de penetración para evaluar la seguridad de redes Wi-Fi. Aquí se proporciona información detallada sobre MDK3, incluyendo una lista de sus parámetros principales.

## Información sobre MDK3

MDK3 es una herramienta de hacking ético que se enfoca en la auditoría y pruebas de penetración de redes Wi-Fi. Permite llevar a cabo diversos ataques y pruebas sobre redes inalámbricas, especialmente aquellas protegidas por el protocolo WPA/WPA2.

> [!info] Modos de Operación de MDK3
> MDK3 opera en diferentes modos, cada uno diseñado para un tipo específico de ataque o prueba. Para su funcionamiento, requiere que la tarjeta de red inalámbrica esté en modo monitor (monitor mode) y, en algunos casos, que soporte inyección de paquetes. Esto es crucial para la manipulación de tramas 802.11.

## Características Principales de MDK3

1.  **Ataques de Deautenticación**: Puede deautenticar clientes conectados a un punto de acceso (AP) específico, forzándolos a desconectarse y luego intentar reconectarse.
    > [!tip] Mecanismo de Deautenticación
    > Los ataques de deautenticación explotan el protocolo 802.11, que permite a cualquier estación enviar una trama de deautenticación a otra estación o AP sin necesidad de autenticación previa. MDK3 puede falsificar la dirección MAC del AP o del cliente para enviar estas tramas, causando la desconexión forzada. Esto es útil para capturar handshakes WPA/WPA2 cuando los clientes intentan reconectarse.

2.  **Ataques de Denegación de Servicio (DoS)**: Puede inundar una red Wi-Fi con tráfico malicioso para sobrecargarla y provocar la denegación de servicio.
    > [!warning] Tipos de DoS con MDK3
    > MDK3 puede realizar varios tipos de ataques DoS:
    > *   **Beacon Flood**: Crea miles de APs falsos con SSIDs aleatorios, saturando la lista de redes disponibles y confundiendo a los clientes.
    > *   **Authentication Flood**: Envía un gran número de solicitudes de autenticación falsas a un AP, agotando sus recursos y previniendo que clientes legítimos se conecten.
    > *   **Probe Request Flood**: Inunda el AP con solicitudes de sondeo, lo que puede llevar a una sobrecarga.

3.  **Ataques de Degradación de Rendimiento**: Puede degradar el rendimiento de una red Wi-Fi, haciendo que sea inutilizable o muy lenta.
    > [!info] Impacto en el Rendimiento
    > Estos ataques buscan saturar el medio inalámbrico o los recursos del AP. Al inyectar una gran cantidad de tramas inválidas o al mantener el canal ocupado, MDK3 puede reducir drásticamente el ancho de banda efectivo y aumentar la latencia para todos los usuarios conectados.

4.  **Exploración de Canales y Redes**: Puede escanear y enumerar redes Wi-Fi disponibles en diferentes canales, ayudando en la detección de redes ocultas y en la optimización de la cobertura.
    > [!tip] Detección de Redes Ocultas
    > Aunque una red no transmita su SSID (red oculta), MDK3 puede detectarla al observar las tramas de respuesta de sondeo (probe responses) o las tramas de asociación/re-asociación de clientes que ya conocen la red.

5.  **Generación de Tráfico Personalizado**: Puede generar tráfico personalizado para realizar pruebas específicas de rendimiento o para simular condiciones de carga en la red.
    > [!info] Flexibilidad en la Inyección
    > Esta característica permite a los pentesters crear y enviar tramas 802.11 con contenido específico, lo que es útil para probar la robustez de los APs frente a tramas malformadas o para simular escenarios de tráfico complejos.

## Parámetros y Opciones Principales de MDK3

Aquí tienes una lista de algunos parámetros y opciones comúnmente utilizados en MDK3, junto con una breve explicación:

*   **`b`**: Ataque de deautenticación básico. Envía tramas de deautenticación a un AP o cliente específico.
    *   Ejemplo: `mdk3 <interfaz> d -b <MAC_AP>`

*   **`d`**: Ataque de deautenticación con un número determinado de intentos. Permite especificar cuántas tramas de deautenticación se enviarán.
    *   Ejemplo: `mdk3 <interfaz> d -d <MAC_AP> -c <MAC_CLIENTE> -n 100` (envía 100 tramas)

*   **`a`**: Ataque de deautenticación contra todos los clientes. Dirige tramas de deautenticación a todos los clientes asociados a un AP objetivo.
    *   Ejemplo: `mdk3 <interfaz> d -a <MAC_AP>`

*   **`m`**: Ataque de deautenticación contra todos los APs y clientes. Un ataque de deautenticación masivo que afecta a todas las redes y clientes en el rango.
    *   Ejemplo: `mdk3 <interfaz> d -m`

*   **`e`**: Ataque de deautenticación de clientes con intentos infinitos. Envía continuamente tramas de deautenticación para mantener a los clientes desconectados.
    *   Ejemplo: `mdk3 <interfaz> d -e <MAC_AP>`

*   **`c`**: Ataque de deautenticación selectiva basado en clientes o APs específicos. Se usa en combinación con otros modos para especificar objetivos.
    *   Ejemplo: `mdk3 <interfaz> d -c <MAC_CLIENTE>` (deautentica un cliente específico)

*   **`w`**: Ataque de denegación de servicio (DoS) por inundación de redes (Beacon Flood). Crea APs falsos para saturar el entorno.
    *   Ejemplo: `mdk3 <interfaz> b -w`

*   **`g`**: Genera tráfico en la red objetivo para analizar su rendimiento (Packet Flooding).
    *   Ejemplo: `mdk3 <interfaz> f -g -t <MAC_AP>`

*   **`x`**: Genera tráfico para pruebas de fuzzing o para saturar el canal. (MDK3 se enfoca en la inyección de paquetes; la captura de tráfico resultante debe realizarse con herramientas externas como Wireshark o Airodump-ng).

*   **`i`**: Genera tráfico de autenticación (Authentication Flood).
    *   Ejemplo: `mdk3 <interfaz> a -i -t <MAC_AP>`

*   **`p`**: Genera tráfico de *probe* (Probe Request Flood).
    *   Ejemplo: `mdk3 <interfaz> p -p -t <MAC_AP>`

*   **`r`**: Genera tráfico de *reaver* (Reaver-like traffic generation). Esto es para simular o ayudar en ataques WPS, similar a la herramienta Reaver.
    *   Ejemplo: `mdk3 <interfaz> w -r` (WPS attack mode, a menudo usado con Reaver)
