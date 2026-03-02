---
aliases: Tcpreplay
tags:
  - redes/comandos
  - linux/comandos
  - hacking/herramientas
estado: 🟢 Terminado
---
# Tcpreplay

**Tcpreplay** es una herramienta de código abierto utilizada para la reproducción de tráfico de red capturado previamente. Permite reenviar paquetes de red guardados en archivos de captura PCAP a través de una interfaz de red. Esto es útil para diversos propósitos como pruebas de rendimiento, pruebas de seguridad, depuración de redes y simulación de tráfico real en entornos controlados.

> [!info] Mecanismo de Operación
> Tcpreplay opera a nivel de enlace de datos (Capa 2 del modelo OSI) y puede inyectar paquetes directamente en la interfaz de red. Esto le permite simular condiciones de red muy específicas, incluyendo ataques de denegación de servicio (DoS) o la validación de reglas de firewall y sistemas de detección de intrusiones (IDS/IPS) con tráfico real o modificado. Es crucial que la interfaz de salida esté en modo promiscuo o que el tráfico sea dirigido correctamente para su captura o análisis.

### Funcionalidades de Tcpreplay

- **Reproducción de tráfico**: Permite reproducir exactamente el tráfico de red capturado desde un archivo PCAP a través de una interfaz de red. Esto es útil para repetir escenarios de tráfico específicos para pruebas o análisis.

- **Modificación de tráfico**: Tcpreplay puede modificar parámetros de los paquetes, como direcciones IP, puertos, tiempos de retardo y otros campos, antes de reenviarlos.

> [!tip] Modificación de Tráfico Avanzada
> La capacidad de modificar campos de paquetes es fundamental para pruebas de seguridad. Por ejemplo, se pueden alterar direcciones IP de origen para simular ataques de spoofing, cambiar puertos para evadir firewalls básicos, o modificar payloads para probar la robustez de aplicaciones ante entradas inesperadas. Tcpreplay utiliza una serie de opciones para realizar estas modificaciones de forma programática.

- **Soporte de múltiples interfaces**: Puede trabajar con múltiples interfaces de red simultáneamente, permitiendo la reproducción de tráfico en varias redes o segmentos de red.

### Comandos más usados de Tcpreplay

```bash
tcpreplay -i eth0 file.pcap   # Reenvía el archivo PCAP especificado a través de la interfaz de red eth0.

tcpreplay -t -i eth0 file.pcap   # Reproduce el tráfico en tiempo invertido (de fin a inicio).

tcpreplay -K -i eth0 file.pcap   # Mantén los tiempos de retardo originales entre los paquetes.

tcpreplay -M 1000 -i eth0 file.pcap   # Limita la velocidad de reproducción a 1000 paquetes por segundo.
```

> [!info] Diferencia entre `-K` y `-F`
> La opción `-K` (Keep) intenta mantener los tiempos de retardo *relativos* entre los paquetes tal como fueron capturados, lo que es útil para simular el comportamiento temporal exacto de una red. Por otro lado, `-F` (Fast) reproduce los paquetes a la máxima velocidad posible que la interfaz y el sistema puedan manejar, ignorando los tiempos de retardo originales, lo cual es ideal para pruebas de estrés o rendimiento donde se busca saturar un enlace o dispositivo.

### Parámetros y opciones de Tcpreplay

- **-i <interfaz>**: Especifica la interfaz de red por donde se enviarán los paquetes.
  
- **-t**: Reproduce el tráfico en tiempo invertido.

- **-K**: Mantiene los tiempos de retardo originales entre los paquetes.

- **-M <velocidad>**: Limita la velocidad de reproducción a un número específico de paquetes por segundo.

- **-D <velocidad>**: Divide el archivo PCAP y reenvía paquetes en paralelo para aumentar la velocidad de reproducción.
> [!tip] Paralelización con `-D`
> La opción `-D` es útil en entornos de alto rendimiento donde un solo hilo de reproducción no es suficiente para alcanzar la velocidad deseada. Al dividir el archivo PCAP y procesarlo en paralelo, Tcpreplay puede saturar enlaces de red de mayor ancho de banda, lo que es crucial para pruebas de estrés en infraestructuras modernas.

- **-C <número>**: Especifica cuántas veces se reproducirá el archivo PCAP.

- **-x <número>**: Reproduce solo un número específico de paquetes desde el archivo PCAP.

- **-T**: Comprueba la suma de verificación de los paquetes antes de enviarlos.
> [!warning] Sumas de Verificación y Dispositivos de Red
> La opción `-T` es importante porque muchos dispositivos de red (routers, firewalls, NICs) pueden descartar paquetes con sumas de verificación incorrectas. Sin embargo, algunas NICs modernas realizan la descarga de sumas de verificación (checksum offloading), lo que significa que la NIC calcula la suma de verificación en hardware. En estos casos, Tcpreplay podría enviar paquetes con sumas de verificación "incorrectas" desde su perspectiva, pero la NIC las corregirá antes de la transmisión física. Es vital entender el comportamiento de la NIC de salida.

- **-A**: Anula la dirección MAC de origen y la reemplaza por la propia.
> [!info] Spoofing de MAC de Origen
> La opción `-A` es una forma rápida de asegurar que los paquetes salientes tengan la dirección MAC de la interfaz de envío. Esto es útil para evitar problemas de enrutamiento o filtrado basados en MAC si el archivo PCAP original contiene direcciones MAC de origen que no son válidas en la red de destino.

- **-q**: Modo silencioso, sin mensajes de estado.

- **-F**: Reproduce los paquetes a la velocidad en que fueron capturados.

- **-l**: Lista las interfaces de red disponibles y luego sale.

- **-m <dirección MAC>**: Cambia la dirección MAC de origen de los paquetes.

- **-n <número>**: Anula el número de secuencia TCP y UDP.

- **-p <número>**: Anula los puertos de origen y destino en el 20% de los paquetes.

- **-P <número>**: Anula los puertos de origen y destino en el 100% de los paquetes.

- **-r <dirección MAC>**: Anula la dirección MAC de destino de los paquetes.

- **-R <número>**: Anula los números de secuencia TCP y UDP en el 100% de los paquetes.

- **-s**: Anula las direcciones IP de origen y destino.
> [!tip] Anulación de IP de Origen y Destino
> Las opciones `-s`, `-S`, `-u`, `-w`, `-W`, `-y`, `-Y` ofrecen un control granular sobre el spoofing de direcciones IP y MAC. Por ejemplo, `-s` es útil para simular un ataque desde una subred diferente o para probar cómo un sistema responde a tráfico con IPs no esperadas. La diferencia entre las opciones que afectan al 20% y al 100% de los paquetes permite simular escenarios más realistas donde no todo el tráfico está spoofed.

- **-S**: Anula las direcciones IP de origen y destino en el 20% de los paquetes.

- **-u**: Anula las direcciones IP de origen y destino en el 100% de los paquetes.

- **-v**: Aumenta el nivel de depuración.

- **-w**: Anula la dirección IP de origen.

- **-W**: Anula las direcciones IP de origen y destino en el 100% de los paquetes.

- **-y**: Anula la dirección MAC de destino.

- **-Y**: Anula las direcciones IP de origen y destino en el 100% de los paquetes.