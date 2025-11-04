------

**Tcpreplay** es una herramienta de código abierto utilizada para la reproducción de tráfico de red capturado previamente. Permite reenviar paquetes de red guardados en archivos de captura pcap a través de una interfaz de red. Esto es útil para diversos propósitos como pruebas de rendimiento, pruebas de seguridad, depuración de redes, y simulación de tráfico real en entornos controlados.

### Funcionalidades de Tcpreplay

- **Reproducción de tráfico**: Permite reproducir exactamente el tráfico de red capturado desde un archivo pcap a través de una interfaz de red. Esto es útil para repetir escenarios de tráfico específicos para pruebas o análisis.

- **Modificación de tráfico**: Tcpreplay puede modificar parámetros de los paquetes, como direcciones IP, puertos, tiempos de retardo, y otros campos, antes de reenviarlos.

- **Soporte de múltiples interfaces**: Puede trabajar con múltiples interfaces de red simultáneamente, permitiendo la reproducción de tráfico en varias redes o segmentos de red.

### Comandos más usados de Tcpreplay

```bash
tcpreplay -i eth0 file.pcap   # Reenvía el archivo pcap especificado a través de la interfaz de red eth0.

tcpreplay -t -i eth0 file.pcap   # Reproduce el tráfico en tiempo invertido (de fin a inicio).

tcpreplay -K -i eth0 file.pcap   # Mantén los tiempos de retardo originales entre los paquetes.

tcpreplay -M 1000 -i eth0 file.pcap   # Limita la velocidad de reproducción a 1000 paquetes por segundo.
```

### Parámetros y opciones de Tcpreplay

- **-i \<interfaz\>**: Especifica la interfaz de red por donde se enviarán los paquetes.
  
- **-t**: Reproduce el tráfico en tiempo invertido.

- **-K**: Mantiene los tiempos de retardo originales entre los paquetes.

- **-M \<velocidad\>**: Limita la velocidad de reproducción a un número específico de paquetes por segundo.

- **-D \<velocidad\>**: Divide el archivo pcap y reenvía paquetes en paralelo para aumentar la velocidad de reproducción.

- **-C \<número\>**: Especifica cuántas veces se reproducirá el archivo pcap.

- **-x \<número\>**: Reproduce solo un número específico de paquetes desde el archivo pcap.

- **-T**: Comprueba la suma de verificación de los paquetes antes de enviarlos.

- **-A**: Anula la dirección MAC de origen y la reemplaza por la propia.

- **-q**: Modo silencioso, sin mensajes de estado.

- **-F**: Reproduce los paquetes a la velocidad en que fueron capturados.

- **-l**: Lista las interfaces de red disponibles y luego sale.

- **-m \<dirección MAC\>**: Cambia la dirección MAC de origen de los paquetes.

- **-n \<número\>**: Anula el número de secuencia TCP y UDP.

- **-p \<número\>**: Anula los puertos de origen y destino en el 20% de los paquetes.

- **-P \<número\>**: Anula los puertos de origen y destino en el 100% de los paquetes.

- **-r \<dirección MAC\>**: Anula la dirección MAC de destino de los paquetes.

- **-R \<número\>**: Anula los números de secuencia TCP y UDP en el 100% de los paquetes.

- **-s**: Anula las direcciones IP de origen y destino.

- **-S**: Anula las direcciones IP de origen y destino en el 20% de los paquetes.

- **-u**: Anula las direcciones IP de origen y destino en el 100% de los paquetes.

- **-v**: Aumenta el nivel de depuración.

- **-w**: Anula la dirección IP de origen.

- **-W**: Anula las direcciones IP de origen y destino en el 100% de los paquetes.

- **-y**: Anula la dirección MAC de destino.

- **-Y**: Anula las direcciones IP de origen y destino en el 100% de los paquetes.

