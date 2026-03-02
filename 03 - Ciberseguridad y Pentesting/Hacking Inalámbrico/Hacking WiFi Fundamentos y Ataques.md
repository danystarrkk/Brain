---
aliases:
  - Modo Monitor
  - Ataques WiFi
  - Hacking Inalámbrico
tags:
  - hacking/inalambrico
  - hacking/herramientas
  - linux/comandos
  - redes/conceptos
estado: 🟢 Terminado
---
# Hacking WiFi: Fundamentos y Ataques

## Modo Monitor

El modo monitor nos permite capturar y escuchar todos los paquetes que viajan en el aire. Gracias a este modo, podremos identificar elementos como:
- Puntos de acceso (nombre, MAC, dispositivos, etc.).
- Clientes de los diferentes puntos de acceso, incluyendo su dirección MAC.

Estos dos elementos son cruciales, ya que con ellos comenzaremos a ejecutar diversos ataques de hacking en redes Wi-Fi.

> [!info] Modo Monitor vs. Modo Promiscuo
> El **modo monitor** es una capacidad especial de las tarjetas de red inalámbricas que permite capturar todos los paquetes que se transmiten en un canal específico, sin necesidad de estar asociado a un punto de acceso. A diferencia del **modo promiscuo**, que solo captura paquetes destinados a la interfaz o a la red a la que está conectada, el modo monitor opera a un nivel inferior, permitiendo la inspección de tráfico no dirigido a la estación, incluyendo paquetes de gestión y control 802.11. Es fundamental para herramientas de análisis y ataque Wi-Fi.

## Configurar la Tarjeta de Red en Modo Monitor

Para comenzar, debemos identificar el nombre de la tarjeta de red, lo cual se puede hacer con el comando:

```bash
ifconfig
```

![[Pasted image 20240709115933.png]]

Como se puede observar, tenemos diferentes tipos de interfaces; la que utilizaremos, que es la tarjeta de red Wi-Fi, es `wlan0`.

Para un funcionamiento correcto, es necesario operar como usuario `root`.

Primero, desactivamos la tarjeta con el siguiente comando:

```bash
bash
ifconfig wlan0 down
```

> [!tip] ¿Por qué desactivar la interfaz?
> Desactivar la interfaz (`ifconfig wlan0 down`) antes de cambiar a modo monitor es una buena práctica para evitar conflictos con el sistema operativo y con servicios de red que puedan estar utilizando la interfaz. Esto asegura que la tarjeta esté en un estado "limpio" antes de ser reconfigurada por `airmon-ng`.

Una vez hecho esto, procederemos con los comandos para configurar nuestra tarjeta en modo monitor.

Con nuestra tarjeta de red identificada, utilizaremos [[Aircrack-ng Suite de Herramientas|Aircrack-ng]] y sus herramientas para configurar la tarjeta en modo monitor mediante el siguiente comando:

```bash
airmon-ng start wlan0
```

![[Pasted image 20240709122645.png]]

Como se puede observar, nuestra tarjeta de red ya está en modo monitor. `airmon-ng` suele crear una nueva interfaz virtual con el sufijo `mon` (ej. `wlan0mon`).

> [!info] `airmon-ng` y la nueva interfaz
> Cuando `airmon-ng start wlan0` se ejecuta con éxito, a menudo crea una nueva interfaz virtual (por ejemplo, `wlan0mon` o `mon0`) que es la que realmente opera en modo monitor. Las herramientas de `aircrack-ng` (como `airodump-ng`, `aireplay-ng`) deben ser dirigidas a esta nueva interfaz en modo monitor, no a la original `wlan0`.

Como dato adicional, si ejecutamos `ifconfig`, notaremos que nuestra tarjeta ha desaparecido. Esto es normal, ya que la desactivamos al inicio. Para verificar que está activa, usaremos:

```bash
iwconfig
```

![[Pasted image 20240709122828.png]]

Es importante tener en cuenta que existen dos servicios que pueden causar problemas y es preferible cerrarlos con el siguiente comando:

```bash
pkill dhclient && pkill wpa_supplicant
```

```bash
# también podemos usar el comando
killall dhclient wpa_supplicant 2>/dev/null
```

> [!warning] Servicios que interfieren con el modo monitor
> Los servicios `dhclient` (cliente DHCP) y `wpa_supplicant` (gestor de autenticación WPA/WPA2) intentan gestionar la conexión de red de la interfaz Wi-Fi. Cuando la tarjeta se pone en modo monitor, estos servicios pueden interferir con la captura de paquetes o intentar restablecer la interfaz a un modo gestionado, causando inestabilidad o fallos en las herramientas de `aircrack-ng`. Detenerlos asegura que la interfaz esté completamente bajo el control de las herramientas de auditoría.

Con la tarjeta en modo monitor configurada correctamente, procederemos a activarla con el siguiente comando:

```bash
ifconfig wlan0 up
```

Ahora, si ejecutamos el comando `ifconfig`, podremos ver:
![[Pasted image 20240709123444.png]]

Para detener o deshabilitar el modo monitor, debemos seguir los mismos pasos, con la diferencia de que el comando `airmon-ng` será el siguiente:

```bash
airmon-ng stop wlan0
```

Una vez ejecutado este comando, es crucial reiniciar el servicio de red para que podamos utilizar internet correctamente:

```bash
service networking restart
```

De esta forma, nuestra conexión a internet funcionará correctamente.

## Falsificando la MAC de nuestra tarjeta de Red

Con nuestra tarjeta de red configurada en modo monitor, es altamente recomendable cambiar la dirección MAC para evitar posibles problemas. Esto se puede lograr con la ayuda de [[Macchanger Herramienta de Modificación MAC|macchanger]], y a continuación, veremos el proceso paso a paso.

> [!info] ¿Por qué falsificar la MAC?
> La falsificación de la dirección MAC (MAC spoofing) es una técnica común en auditorías de seguridad inalámbrica. Permite:
> 1.  **Anonimato**: Ocultar la identidad real de la tarjeta de red, dificultando el rastreo.
> 2.  **Evitar listas negras/blancas**: Burlar filtros de MAC implementados en algunos puntos de acceso.
> 3.  **Impersonación**: Simular ser otro dispositivo en la red.
> Es una medida de seguridad y evasión importante en entornos de prueba.

Este proceso es bastante sencillo. Lo primero es verificar la dirección MAC actual de nuestro dispositivo con el comando:

```bash
macchanger -s wlan0
```

![[Pasted image 20240709125950.png]]

Conociendo la MAC original de la tarjeta, procederemos a listar todas las MACs disponibles en `macchanger` para su uso:

```bash
macchanger -l
```

![[Pasted image 20240709130015.png]]

Ahora solo queda elegir una MAC de esta lista; en este caso, usaré una de Sony.
Para cambiar la MAC, primero debemos desactivar nuestra tarjeta de red:

```bash
ifconfig wlan0 down
```

Una vez hecho esto, ejecutaremos el siguiente comando para cambiar la MAC:

```bash
macchanger --mac=5c:a6:e6:08:00:46 wlan0
```

> [!tip] Estructura de la MAC: OUI y NIC
> Una dirección MAC (Media Access Control) de 48 bits se divide en dos partes:
> -   **OUI (Organizationally Unique Identifier)**: Los primeros 24 bits (3 primeros pares hexadecimales) identifican al fabricante del hardware. Por ejemplo, `5c:a6:e6` podría corresponder a un fabricante específico.
> -   **NIC (Network Interface Controller)**: Los últimos 24 bits (3 últimos pares hexadecimales) son un identificador único asignado por el fabricante a esa interfaz de red específica.
> Al falsificar la MAC, a menudo se mantiene el OUI de un fabricante conocido para parecer un dispositivo legítimo, y se modifica el NIC para generar una dirección única.

Es importante entender que los primeros 3 pares corresponden al OUI (Organizationally Unique Identifier) y deben coincidir con la MAC permanente, mientras que los 3 últimos pares (NIC - Network Interface Controller) son los que modificamos, en este caso, con los de Sony.

Con esto, podemos activar la tarjeta de red:

```bash
ifconfig wlan0 up
```

Y si verificamos la MAC, veremos que ha sido cambiada:

![[Pasted image 20240709135123.png]]

## Análisis de Entorno con Airodump

Para este punto, ya debemos tener el modo monitor correctamente configurado y nuestra dirección MAC cambiada.
Con todo lo anterior listo, utilizaremos `Airodump` para visualizar todos los paquetes que se encuentran en el aire con el siguiente comando:

```bash
airodump-ng wlan0
```

![[Pasted image 20240709142924.png]]

Como vemos, ya detecta todos los puntos de acceso.

A continuación, explicaremos la información mostrada en la imagen:

-   Lo primero que tenemos que comprender es la información de la primera línea, la cual nos indica el canal, la hora y la fecha. Es importante saber que al ejecutar `airodump-ng`, este realiza el conocido 'channel hopping', que no es más que el cambio continuo de canal para encontrar diferentes paquetes.
> [!info] Channel Hopping
> El "channel hopping" es la capacidad de `airodump-ng` de escanear automáticamente todos los canales Wi-Fi disponibles (generalmente del 1 al 14 en 2.4 GHz) para descubrir puntos de acceso y clientes. Esto permite una visión completa del entorno inalámbrico. Sin embargo, para ataques específicos, a menudo es necesario fijar `airodump-ng` a un solo canal (`-c <canal>`) para asegurar la captura continua de paquetes de un AP objetivo.
-   El **BSSID**: Es la dirección MAC del punto de acceso. En la imagen, vemos que está repetido porque la parte superior muestra las redes Wi-Fi y la inferior, los clientes conectados a diferentes Wi-Fi.
-   El **PWR**: Representa la potencia de la señal. Un número mayor indica mayor cercanía, y un número menor, mayor distancia. Si todos los valores son -1, significa que el controlador no soporta informes de niveles de señal. Si solo un conjunto tiene -1, indica que las transmisiones del cliente no están al alcance de la tarjeta. Finalmente, si el -1 aparece en la sección de clientes, significa que el controlador no soporta informes de señal para esos clientes.
> [!tip] Interpretación de PWR
> Los valores de PWR (Power) se expresan en dBm (decibelios-milivatios) y son negativos. Un valor de -30 dBm es una señal muy fuerte (cercana), mientras que -90 dBm es una señal muy débil (lejana). Cuanto más cerca de 0, más fuerte es la señal.
-   **Beacons**: Son el número de paquetes de anuncio que envía el AP. Cada punto de acceso envía aproximadamente 10 beacons por segundo a una velocidad de 1 Mbps.
-   **Data**: Corresponde al número de paquetes capturados de transmisión de datos.
-   **s**: Es el número de paquetes de datos medidos en los últimos 10 segundos.
-   **CH**: Nos especifica el canal en el que se encuentra el AP.
-   **MB**: Es la velocidad máxima permitida por el AP.
-   **ENC**: Es el tipo de cifrado detectado (ej. WEP, WPA, WPA2).
-   **CIPHER**: Indica el algoritmo de cifrado utilizado (ej. CCMP para WPA2, TKIP para WPA).
-   **AUTH**: Es el protocolo de autenticación en uso, que puede ser: MGT (redes universitarias, 802.1X), PSKA (claves compartidas WEP), OPN (abierto), o PSK (redes domésticas, WPA/WPA2-PSK).
> [!info] Tipos de Autenticación (AUTH)
> -   **OPN (Open System)**: Redes sin autenticación ni cifrado. Cualquiera puede conectarse.
> -   **WEP (Wired Equivalent Privacy)**: Cifrado obsoleto y vulnerable. Utiliza PSKA (Pre-Shared Key Authentication).
> -   **WPA/WPA2-PSK (Pre-Shared Key)**: Utiliza una clave compartida (contraseña) para autenticar y cifrar el tráfico. Es el más común en redes domésticas.
> -   **WPA/WPA2-EAP/MGT (Enterprise)**: Utiliza un servidor de autenticación (RADIUS) y 802.1X para autenticar a los usuarios individualmente, común en entornos corporativos y universitarios.
-   **ESSID**: Es el nombre de la red (Service Set Identifier).
-   **Probes**: Cuando un dispositivo emite diferentes paquetes de conexión y los capturamos, se listarán los nombres de los AP a los que emitió esos paquetes.
-   **Rate**: Indica la tasa de transmisión seguida de la tasa de retransmisión del dispositivo. Es normal ver una 'e' cuando el AP tiene el QoS (Quality of Service, que ayuda a controlar cómo se distribuye el ancho de banda disponible entre diferentes aplicaciones) habilitado.
-   **Lost**: Hace referencia a los paquetes perdidos en los últimos 10 segundos.
-   **Frames**: Número de paquetes enviados por el cliente al AP.
-   **Notes**: Información adicional.

## Filtrar con Airodump

Primero, debemos fijar una única red. En mi caso, intentaremos con la red llamada 'Pruebas'.

Podemos fijar el objetivo filtrando tanto por BSSID como por ESSID, pero se recomienda usar el ESSID por ser más práctico. Por lo tanto, el comando sería:

```bash
airodump-ng --essid Pruebas -c 11 wlan0
```

Esta es una forma de aplicar el filtro; la otra sería mediante el BSSID:

```bash
airodump-ng --bssid 88:53:D4:92:B9:CC wlan0
```

De cualquiera de estas dos formas podemos aplicar el filtro. En mi caso, usaré la primera:
![[Pasted image 20240709173827.png]]

## Exportación de evidencias

Es importante tener en cuenta que, aunque apliquemos filtros y visualicemos información, no estamos almacenando nada. Por lo tanto, para poder guardar los paquetes encontrados, agregaremos lo siguiente a cualquiera de los dos comandos que usamos para filtrar:

```bash
airodump-ng -w Captura --essid Pruebas -c 11 wlan0
```

![[Pasted image 20240709174202.png]]

Con el comando anterior, almacenamos la información de los paquetes interceptados en un archivo con prefijo `Captura` (ej. `Captura-01.cap`).

## Handshake

El 'handshake' se refiere al proceso inicial en el que dos dispositivos o sistemas establecen una conexión. Durante este intercambio, los dispositivos acuerdan los parámetros de la comunicación, como la velocidad de transferencia de datos, el tipo de cifrado utilizado y otros detalles necesarios para la transmisión de información. Es fundamental para asegurar que ambos extremos de la conexión estén sincronizados y puedan comunicarse de manera efectiva.

Lo que haremos es interceptar este handshake cuando se envíe. Dentro de este paquete, encontraremos la contraseña de nuestra red Wi-Fi, pero cifrada, por lo que aún no podremos obtenerla en texto claro.

> [!info] El Handshake de 4 Vías (WPA/WPA2)
> En redes WPA/WPA2-PSK, el "handshake" es un proceso de cuatro mensajes (EAPOL frames) que ocurre cuando un cliente se conecta a un AP. Su propósito es establecer una clave de sesión temporal (PTK - Pairwise Transient Key) a partir de la clave precompartida (PSK) y otros parámetros aleatorios. Capturar este handshake es crucial para ataques de fuerza bruta o diccionario, ya que contiene un hash de la PSK que puede ser atacado offline.

### Formas de obtener el Handshake

#### Ataque Pasivo

Una de las formas de conseguir el handshake es a través de la paciencia, esperando a que un dispositivo se desconecte y, al reconectarse, logremos capturar el handshake.

#### Ataque de Deautenticación dirigido

Este es un tipo de ataque mediante el cual haremos que un dispositivo específico de esa red se desconecte, provocando que tenga que volver a conectarse y permitiéndonos así obtener el handshake de la red.

Para esto, haremos uso del siguiente comando:

```bash
aireplay-ng -0 10 -e Pruebas -c 4E:5E:E5:84:57:47 wlan0
```

-   `-0` es el modo de ataque, seguido de un valor (en este caso, `10`) para emitir los paquetes de desautenticación.
-   `-e` para especificar el nombre de la red (ESSID).
-   `-c` para especificar la dirección MAC del dispositivo a desconectar.

> [!warning] Ataques de Deautenticación
> Los ataques de desautenticación (Deauthentication Attacks) explotan una vulnerabilidad en el estándar 802.11, que no requiere autenticación para enviar paquetes de desautenticación. Un atacante puede enviar un paquete de desautenticación falsificado a un cliente, haciéndole creer que el AP lo está desconectando, o al AP, haciéndole creer que el cliente se está desconectando. Esto fuerza al cliente a reconectarse, lo que genera un nuevo handshake que puede ser capturado. Este ataque es de Capa 2 (Enlace de Datos).

#### Ataque de Deautenticación Global.

Este tipo de ataque se aplica cuando nos encontramos en una red con múltiples dispositivos conectados y no sabemos cuál de ellos es parte de la misma. En este caso, desconectaremos a todos, lo cual se puede hacer de dos formas:

La primera forma implica que un dispositivo emitirá los paquetes de desautenticación. El comando es el siguiente:

```bash
aireplay-ng -0 10 -e Pruebas -c FF:FF:FF:FF:FF:FF wlan0
```

La segunda forma hace lo mismo, pero sin especificar un cliente objetivo (desautenticación global):

```bash
aireplay-ng -0 10 -e Pruebas wlan0 # Nota: -c FF:FF:FF:FF:FF:FF es implícito o se omite para desautenticar a todos los clientes asociados al AP.
```

#### Ataque de Falsa Autenticación (Conceptual)

Lo veremos de forma conceptual, ya que para WPA/WPA2 no es un ataque efectivo. Dejaremos el comando de manera general para su conocimiento:

```bash
aireplay-ng -1 0 -e Pruebas -a 88:53:D4:92:B9:CC -h 08:00:46:33:45:55 wlan0
```

-   `-1` es el identificador del ataque conocido como 'fakeauth' (autenticación falsa).
-   `-e` para especificar el ESSID (nombre de la red).
-   `-a` para especificar el BSSID (MAC del punto de acceso).
-   `-h` para especificar la dirección MAC que tendrá el dispositivo falso.

> [!info] Ataque de Falsa Autenticación (Fakeauth)
> Este ataque es principalmente útil contra redes WEP o para generar tráfico ARP en redes WEP. En WPA/WPA2, una autenticación falsa por sí sola no es suficiente para obtener el handshake o acceder a la red, ya que el proceso de autenticación es más robusto y requiere el intercambio de claves del handshake de 4 vías.

### Secuestro de Ancho de banda de la Red.

Para comprender el funcionamiento de este tipo de ataque, nos familiarizaremos con los conceptos de RTS y CTS:

#### RTS (Request to Send)

**¿Qué es?** RTS (Request to Send) es un mecanismo de control de acceso al medio utilizado en redes inalámbricas basadas en el estándar IEEE 802.11 para prevenir colisiones de datos.

**Funcionamiento:**

1.  **Necesidad de envío:** Cuando una estación inalámbrica tiene datos para enviar, primero envía un paquete RTS al punto de acceso (AP) o a otra estación de destino.
2.  **Duración de la reserva:** El paquete RTS incluye información sobre cuánto tiempo (duración de la reserva) la estación necesita para transmitir sus datos. Esto permite a otras estaciones inalámbricas saber cuándo el canal estará ocupado y evitarán enviar datos durante ese período para evitar colisiones.
3.  **Respuesta CTS:** Después de recibir el RTS, el punto de acceso o la estación de destino envían un paquete CTS (Clear to Send) de vuelta a la estación que originó el RTS. Este CTS también incluye la duración de la reserva, confirmando que el canal está disponible para la transmisión solicitada.
4.  **Transmisión de datos:** Una vez que la estación que originó el RTS recibe el CTS, puede comenzar a transmitir sus datos. Durante este tiempo, otras estaciones inalámbricas respetan la duración de la reserva y no intentan transmitir datos para evitar colisiones.

#### CTS (Clear to Send)

**¿Qué es?** CTS (Clear to Send) es la respuesta enviada por el punto de acceso (AP) o la estación de destino después de recibir un paquete RTS, confirmando que el canal está disponible para la transmisión solicitada.

**Funcionamiento:**

1.  **Recepción del RTS:** Cuando una estación inalámbrica recibe un paquete RTS de otra estación que desea transmitir datos, verifica la duración de la reserva especificada en el RTS.
2.  **Envío del CTS:** Si el canal está libre durante el período especificado en el RTS, la estación de destino responde con un paquete CTS. Este paquete también incluye la duración de la reserva, lo que permite a la estación que originó el RTS comenzar a transmitir sus datos sin temor a colisiones con otras estaciones.
3.  **Inicio de la transmisión:** Tras recibir el CTS, la estación que originó el RTS puede comenzar a transmitir sus datos de manera segura, sabiendo que el canal está reservado exclusivamente para ella durante el tiempo especificado en la duración de la reserva.

Dentro del **Clear to Send** (CTS) tenemos dos divisiones más, que son:

-   **CTS to Self (CTS/Self)**: Este CTS es enviado por un nodo a sí mismo para reservar el canal y asegurar que puede transmitir sin interrupciones.
-   **CTS to Others (CTS/Others)**: Este CTS es enviado en respuesta a un RTS de otro nodo, indicando a otros nodos en la red que deben abstenerse de transmitir datos durante un tiempo específico para evitar colisiones.

> [!info] RTS/CTS y el Problema del Nodo Oculto
> El mecanismo RTS/CTS fue diseñado para mitigar el "problema del nodo oculto" (hidden node problem) en redes inalámbricas. Este problema ocurre cuando dos estaciones no pueden "escucharse" entre sí, pero ambas pueden "escuchar" al mismo AP. Si ambas intentan transmitir al AP simultáneamente, se producirá una colisión. RTS/CTS resuelve esto al hacer que el AP (o la estación receptora) anuncie la duración de la transmisión, creando un "vector de asignación de red" (NAV - Network Allocation Vector) virtual que todas las estaciones que escuchan el CTS deben respetar, absteniéndose de transmitir durante ese período.

De forma general, este es un proceso que ocurre en la red inalámbrica y que permite administrar el envío y la recepción de paquetes de manera exitosa, evitando colisiones y optimizando la transmisión.

Con esto claro, debemos comprender que cuando se emite el CTS, tiene un esquema simple de ver:

![[Pasted image 20240711131535.png]]

Como vemos, la estructura es sencilla, pero haremos énfasis en el campo 'Type', que se define en microsegundos, desde 0 hasta 32,767.
Teniendo esto en cuenta, el propósito es alterar la duración, estableciendo un valor bastante alto. También debemos incluir la dirección MAC del punto de acceso para generar el ataque y, por último, el FCS ('Frame Check Sequence'), que nos permite verificar la integridad de los datos. De esto último no tenemos que preocuparnos del todo, ya que con herramientas como Wireshark podremos copiar el FCS y enviar nuestro paquete modificado.

Ya con los conocimientos básicos claros, podemos comenzar con el ataque:
1.  Capturaremos los paquetes con la ayuda de Wireshark. Es cuestión de abrir la herramienta y escoger la tarjeta de red para ver cómo comienza a capturar los paquetes:
![[Pasted image 20240712214007.png]]

2.  En este punto, agregaremos el siguiente filtro para intentar encontrar la paquetería correspondiente al CTS.

![[Pasted image 20240712214113.png]]

3.  Con esto, debemos buscar la dirección MAC de nuestro AP víctima y escoger uno de los paquetes:
![[Pasted image 20240712234109.png]]

4.  Ya tengo escogido el paquete y, como vemos, consta de 110 microsegundos. Con esto listo, exportaremos el paquete único con la siguiente configuración:

![[Pasted image 20240712234225.png]]

5.  Una vez almacenado, abriremos el archivo con la ayuda de [[GHex Editor Hexadecimal|GHex]] y lo veremos de la siguiente forma:
![[Pasted image 20240712234423.png]]

Como vemos, ya tenemos esto abierto y contaremos 8 pares de dígitos, donde el 9 y 10 son los microsegundos en hexadecimal. Lo comprobamos con ayuda de Python, como en la imagen:

6.  Calcularemos el valor de 30000 en hexadecimal para poder modificar el paquete, y esto lo haremos con Python:
![[Pasted image 20240712234735.png]]

Con esto listo, modificaremos el paquete para que los microsegundos sean 30000 de la siguiente forma:
![[Pasted image 20240712235339.png]]

7.  Guardamos los datos del paquete y salimos.
8.  Lo siguiente es, preferiblemente, comprobar los datos de la captura abriendo el archivo con Wireshark:
![[Pasted image 20240712235431.png]]

Como vemos, el paquete ya tiene los nuevos valores en microsegundos, por lo que ahora, con ayuda de [[TCPReplay]], replicaremos el paquete en la red:

```bash
tcpreplay --intf1=wlan0 --topspeed --loop=2000 ctsframe.pcap 2>/dev/null
```

Lo que hace el comando es especificar la tarjeta con la que se realizará el ataque, seguido de la instrucción para inyectar todos los paquetes de golpe (no uno por uno) y, por último, emitir 2000 de estos paquetes.
En mi caso, mi tarjeta no lo permite y me arroja un error de soporte, pero con una tarjeta compatible, se puede enviar:
![[Pasted image 20240713000335.png]]

> [!warning] Impacto del Secuestro de Ancho de Banda (CTS Flood)
> Al enviar un paquete CTS con una duración extremadamente alta, se le indica a todos los dispositivos en el rango que el canal estará ocupado durante un período prolongado. Esto efectivamente "secuestra" el ancho de banda, impidiendo que otros dispositivos transmitan y causando una denegación de servicio (DoS) en la red Wi-Fi. La efectividad depende de la tarjeta de red del atacante y la capacidad de inyección de paquetes.

### Ataque Beacon Flood

Este tipo de ataque consiste en inundar todo el espacio aéreo con paquetes Beacon. Para comprenderlo bien, primero expliquemos qué son los paquetes Beacon:

#### Paquetes Beacon en redes Wi-Fi:

1.  **Definición**: Los paquetes Beacon son pequeños mensajes de datos emitidos por un punto de acceso (AP) Wi-Fi aproximadamente cada 100 milisegundos. Contienen información básica sobre la red Wi-Fi, como su nombre (SSID), tipo de seguridad (por ejemplo, WPA2), velocidad de conexión y otros parámetros configurados.

2.  **Funcionamiento**:
    -   **Anuncio de red**: El AP envía periódicamente estos paquetes Beacon en una frecuencia predefinida para informar a los dispositivos cercanos de su presencia y configuración.
    -   **Descubrimiento de red**: Los dispositivos Wi-Fi, como laptops, teléfonos inteligentes y otros dispositivos móviles, escanean activamente las señales de estos paquetes Beacon para identificar las redes disponibles a las que pueden conectarse.
    -   **Conexión y roaming**: Una vez que un dispositivo detecta un paquete Beacon de una red conocida y autorizada, puede establecer una conexión y, en el caso de redes Wi-Fi empresariales o grandes, facilita el roaming sin interrupciones al moverse entre diferentes puntos de acceso dentro de la misma red.

De manera general, estos paquetes Beacon no son más que paquetes enviados por el AP con información sobre la conexión, con el objetivo de ser detectados para posibles conexiones.

Centrándonos en el ataque, la idea es emitir una gran cantidad de estos paquetes con información falsa sobre el punto de acceso que se quiere atacar, para así lograr saturar el espectro de onda del AP. Es decir, emitiremos paquetes Beacon en el mismo canal y con el mismo nombre de la red, con el objetivo de que los dispositivos se confundan al intentar conectarse y la conexión se caiga automáticamente. Esto lo haremos con la ayuda de la herramienta [[MDK3 Herramienta de Auditoría Inalámbrica|MDK3]].

> [!warning] Impacto del Ataque Beacon Flood
> Un ataque Beacon Flood inunda el aire con miles de SSIDs falsos. Esto puede causar:
> 1.  **Confusión del usuario**: Los usuarios ven una lista interminable de redes, dificultando la conexión a la red legítima.
> 2.  **Denegación de servicio (DoS)**: Los dispositivos cliente pueden consumir recursos intentando procesar y mostrar todas estas redes falsas, lo que ralentiza o impide su conexión a cualquier red.
> 3.  **Saturación del espectro**: El exceso de tráfico Beacon puede saturar el canal, reduciendo el rendimiento de la red legítima.

Para este tipo de ataque, crearemos un archivo que contenga el nombre de varias redes. Esto se puede hacer de varias formas, pero en mi caso, lo haré con el siguiente comando:

```bash
for i in $(seq 1 10); do echo "MyNetwork$i" >> redes.txt ; done
```

Con esto, ya tenemos nuestro archivo con varias redes:

![[Pasted image 20240713181336.png]]

Con esto, haremos uso de [[MDK3 Herramienta de Auditoría Inalámbrica|MDK3]] con el siguiente comando:

```bash
mdk3 wlan0 b -f redes.txt -a -s 1000 -c 1
```

Y con esto comienza el ataque. No olvidar que la tarjeta de red debe encontrarse en modo monitor.

![[Pasted image 20240713182656.png]]

Con esto, comienza el ataque y se deben ver múltiples redes disponibles, saturando así el espectro de red.

### Ataque de Disassociation Amok Mode

Este tipo de ataque también lo realizaremos con [[MDK3 Herramienta de Auditoría Inalámbrica|MDK3]], y es importante recordar tener `airodump-ng` escuchando:
![[Pasted image 20240713184446.png]]

Debemos crear una 'blacklist' con los BSSID de los dispositivos a desconectar:

![[Pasted image 20240713184650.png]]

Con esto, nos aseguraremos de tener `airodump-ng` en escucha y ejecutaremos el siguiente comando:

```bash
mdk3 wlan0 d -w blacklist -c 11
```

Y con esto se inicia el ataque:

![[Pasted image 20240713185010.png]]

> [!info] Disassociation vs. Deauthentication
> Aunque a menudo se usan indistintamente en el contexto de ataques DoS Wi-Fi, los frames de **desautenticación** (deauthentication) y **desasociación** (disassociation) tienen propósitos ligeramente diferentes en el estándar 802.11:
> -   **Deauthentication**: Termina completamente la conexión de autenticación entre un cliente y un AP. El cliente debe reiniciar todo el proceso de autenticación y asociación para volver a conectarse.
> -   **Disassociation**: Termina la conexión de asociación, pero la autenticación permanece. El cliente solo necesita reasociarse con el AP.
> Ambos pueden ser falsificados para realizar ataques de denegación de servicio, forzando a los clientes a desconectarse y, potencialmente, a generar nuevos handshakes. El "Amok Mode" de MDK3 es una forma agresiva de enviar estos frames.