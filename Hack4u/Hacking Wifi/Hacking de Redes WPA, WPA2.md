-------

## Modo Monitor

El modo monitor nos permite capturar y escuchar todos los paquetes que viajan en el aire gracias a este modo vamos a poder identificar cosas como:
- Puntos de acceso, su nombre, Mac, dispositivos, etc.
- Clientes De los diferentes puntos de acceso con su información como la mac.

Estas dos cosas son las mas importantes porque con estos vamos a comenzar a tratar de hacer los diferentes ataques para el Hacking de la red WIFI.

## Configurar la Tarjeta de Red en Modo Monitor

Bueno lo primero que tenemos que hacer es identificar el nombre de la tarjeta de red y esto lo podemos hacer con el comando:

```bash
ifconfig
```

![[Pasted image 20240709115933.png]]

Como se puede observar tenemos los diferentes tipos de interfaces, el que vamos a usar que es la tarjeta de red wifi es **wlan0**.

Para que todo funcione de manera correcta tenemos que estar como usuarios root.

Lo primero es dar de baja a la tarjeta y lo hacemos con el siguiente comando:

```bash
ifconfig wlan0 down
```

ya con esto vamos a continuar con los comandos para poner nuestra tarjeta en modo monitor.


Ya con nuestra tarjeta de red identificada Vamos a hacer uso [[Aircrack-ng]] y sus diferentes herramientas para poner nuestra tarjeta de red en modo monitor con el siguiente comando:

```bash
airmon-ng start wlan0
```

![[Pasted image 20240709122645.png]]

Como podemos observar ya tenemos nuestra tarjeta de red en modo monitor.

Como dato si hacemos un ifconfig vamos a ver que nuestra tarjeta desapareció y esto es normal ya que la dimos de baja al comienzo por lo que para saber que se encuentra activa vamos a hacer:

```bash
iwconfig
```

![[Pasted image 20240709122828.png]]

Una cosa que tenemos que tener en cuenta es que tenemos dos servicios los cuales nos pueden dar problemas y son preferibles cerrarlos con el siguiente comando:

```bash
pkill dhclient && pkill wpa_supplicant

# tambien podemos usar el comando

killall dhcliente wpa_supplicant 2>/dev/null
```

Ya con esto tenemos la tarjeta en modo monitor configurada correctamente por lo que vamos a levantar la tarjeta con el siguiente comando:

```bash
ifconfig wlan0 up
```

Ya con esto podemos ver si ejecutamos el comando `ifconfig`:
![[Pasted image 20240709123444.png]]

Para detener o quitar el modo monitor tenemos que seguir los mismo pasos con la diferencia de que al usar el comando de `arimon-ng` será el siguiente:

```bash
airmon-ng stop wlan0
```

ya con este comando hecho algo muy importante a hacer para que nosotros podamos usar la red de manera correcta es ejecutar el siguiente comando:

 ```bash
 service networking restart
```

de esta forma nuestro internet funcionara de manera correcta.

## Falsificando la MAC de nuestra tarjeta de Red

Ya con nuestra tarjeta de red en modo monitor configurada lo mas recomendable es cambiar la MAC para evitar diferentes problemas, esto lo podemos hacer con ayuda de [[Macchanger]] y aquí vamos a ver paso a paso el como hacerlo.

Bueno esto es bastante sencillo y lo primero es ver la dirección mac de nuestro dispositivo con el comando:

```bash
macchanger -s wlan0
```

![[Pasted image 20240709125950.png]]

Ya conociendo el la MAC original de la tarjeta vamos a listar todas las MACS que tiene macchanger para poder usar:

```bash
macchanger -l
```

![[Pasted image 20240709130015.png]]

Ya solo nos queda escoger una mac de esta lista, en este caso voy a usar la de Sony.
Para poder cambiar la mac primero tenemos que dar de baja nuestra tarjeta de red:

```bash
ifconfig wlan0 down
```

ya con esto vamos a ejecutar el siguiente comando para cambiar la mac:


```bash
macchanger --mac=5c:a6:e6:08:00:46 wlan0
```

Tenemos que comprender que los primeros 3 pares son del OUI por lo que tiene que ir iguales que la mac permanente y los 3 últimos (NIC) pares son los que cambiamos con lo de sony

ya con esto podemos levantar la tarjeta de red:

```bash
ifconfig wlan0 up
```

y si revisamos la mac veremos que ya se a cambiado:

![[Pasted image 20240709135123.png]]

## Análisis de Entorno con Airodump

	Para este punto ya tenemos que tener correctamente configurado el modo monitor y cambiada nuestra MAC.
Con todo lo anterior listo vamos a hacer uso de `Airodump` para poder ver todos los paquetes que se encuentran en el aire con el siguiente comando:

```bash
airodump-ng wlan0
```

![[Pasted image 20240709142924.png]]

Como vemos ya nos detecta todos los puntos accesos.

Bueno vamos a comprender la Información de la imagen:

- Lo primero que tenemos que comprender es la información de la primera línea la cual nos indica el canal, tiempo y fecha. Tenemos que saber que al ejecutar airodump-ng este hacer el conocido channel hopping que no es mas que el cambio continuo de canal para encontrar diferentes paquetes.
- El **BSSID**: Esta es la dirección mac del punto de acceso, vemos en la imagen que tenemos repetido esto es porque lo de arriba son las Redes Wifi y las de Abajo son Clientes conectados a diferentes WIFIS.
- El **PWR**: Esto no es mas que la señal por lo que mientras mas cerca nos encontramos con un mayor numero y mientras mas lejos estemos nos encontraremos con un menor numero, si todos tiene un valor de -1 quiere decir que el controlador no admite informes de niveles de señal, pero si vemos que tenemos solo un conjunto con -1 quiere decir que las transmisiones el cliente no están al alcance de la tarjeta, por ultimo si el -1 esta en la parte de clientes quiere decir que el controlador no admite informes de señal.
- **Beacons**: Estos los números de paquetes de anuncios que envía el AP, cada punto de acceso envía alrededor de 10 beacons por segundo a la velocidad de un 1M.
- **Data**: Corresponde al número de paquetes capturados de transmisión de datos.
- **#/s**: No es mas que los números de paquetes de datos medidos en los últimos 10 segundos. 
- **CH**: Nos especifica el canal en el que se encuentra.
- **MB**: Esta es la velocidad máxima permitida por el AP.
- **ENC**: Este es el cifrado que detecta.
- **CIPHER**: Nos Indica el cifrado empleado.
- **AUTH**: Es el protocolo de autenticación que se esta empleando que podemos tener: MGT(Redes Universitarias), PSKA(Claves compartidas web), OPN(Abierto), PSK(Redes domesticas).
- **ESSID**: Es el nombre de la red.
- **Probes**: Esto es cuando un dispositivo emite diferentes paquetes de conexión y los capturamos nos listara los nombres de los AP a los que emitió los paquetes.
- **Rate**: Nos indica la taza de transmisión seguido de la taza de retransmisión del dispositivo, es normal ver la e cuando el AP tiene el QoS (Quality of Service -> Básicamente, el QoS ayuda a controlar cómo se distribuye el ancho de banda disponible entre diferentes aplicaciones) habilitado.
- **Lost**: Hace referencia a los paquetes perdidos de los últimos 10 segundos.
- **Frames**: Numero de paquetes enviados por el cliente al AP.
- **Notes**: Es información adicional.

## Filtrar con Airodump

Lo primero que tenemos que hacer es fijar una única red en mi caso vamos a intentar con la que se llama Pruebas.

Podemos Fijar el objetivo filtrando tanto por la BSSID como por el ESSID pero por recomendación usamos el ESSID ya que es mas practico, por lo tanto el comando seria:

```bash
airodump-ng --essid Pruebas -c 11 wlan0
```

Esta es una forma la cual podemos hacer el filtro y la otra seria mediante el bssid:

```bash
airodump-ng --bssid 88:53:D4:92:B9:CC wlan0
```

de cualquiera de estas dos formas podemos aplicar el filtro, en mi caso voy a usar el primero:
![[Pasted image 20240709173827.png]]
## Exportación de evidencias

Algo importante a tener en cuenta es que aunque apliquemos los filtros y demos cosas no estamos almacenando nada por lo tanto para nosotros poder almacenar esos paquetes que encontramos vamos a agregar lo siguiente a cualquiera de los dos comandos que vimos y usamos al filtrar:

```bash
airodump-ng -w Captura --essid Pruebas -c 11 wlan0
```

![[Pasted image 20240709174202.png]]

Ya con el comando anterior almacenamos la información de los paquetes que interceptamos.

## Handshake

El "handshake" se refiere al proceso inicial en el que dos dispositivos o sistemas establecen una conexión. Durante este intercambio, los dispositivos acuerdan los parámetros de la comunicación, como la velocidad de transferencia de datos, el tipo de codificación utilizada y otros detalles necesarios para la transmisión de información. Es fundamental para asegurar que ambos extremos de la conexión estén sincronizados y puedan comunicarse de manera efectiva.

Lo que vamos a hacer es que cuando se envíe este handshake  nosotros lo vamos a interceptar, dentro de este paquete vamos a encontrar la contraseña de nuestro wifi pero encriptada por lo que no que aun no podremos obtener la contraseña en texto claro.

## Formas de obtener el Handshake

### Ataque Pasivo
Una de las formas de conseguir el handshake es a través de la paciencia esperando a que un dispositivo se desconecte y al reconectarse logremos capturar el handshake.

### Ataque de Deautenticación dirigido
Este es un tipo de ataque a través del cual vamos a hacer que un dispositivo específico de esa red se desconecte provocando así que se tenga que volver a conectar y permitiéndonos obtener el handshake de la red.

Para esto vamos a hacer uso del siguiente comando:

```bash
aireplay-ng -0 10 -e Pruebas -c 4E:5E:E5:84:57:47 wlan0
```

- -0 es el modo de ataque seguido de un valor, en este caso el 10 para emitir los paquetes de deautenticación.
- -e para especificar el nombre de la red.
- -c para especificar el dispositivo a desconectar.

### Ataque de Deautenticación Global.

Este tipo de ataque es cuando nos encontramos en una red la cual tiene múltiples dispositivos desconectados y no sabemos cual de ellos es el parte de la misma por lo que vamos a desconectar a todos y podemos hacerlo de dos formas:

La primera inserta un dispositivo el cual va a emitir los paquetes de deautenticación y el comando es el siguiente: 

```bash
aireplay-ng -0 10 -e Pruebas -c FF:FF:FF:FF:FF:FF wlan0
```

La segunda forma hace lo mismo pero sin especificar nada:

```bash
aireplay-ng -0 10 -e Pruebas -c FF:FF:FF:FF:FF:FF wlan0
```

### Ataque de Falsa Autenticación (Conceptual)

Lo vamos a ver de forma conceptual ya que para WPA/WPA2  no es algo que podamos usar, por lo que vamos a dejar el comando de manera general para que lo conozcan:

```bash
aireplay-ng -1 0 -e Pruebas -a 88:53:D4:92:B9:CC -h 08:00:46:33:45:55 wlan0
```

- -1 es el identificador del aquel ataque que se lo conoce como: fakeauth
- -e para especificar el ESSID
- -a para especificar el BSSID
- -h para especificar una mac que va a tener el falso dispositivo.

### Secuestro de Ancho de banda de la Red.

Para comprender el funcionamiento de este tipo de ataque vamos a familiarizarnos con lo que es RTS y CTS:
#### RTS (Request to Send)

**¿Qué es?** RTS (Request to Send) es un mecanismo de control de acceso al medio utilizado en redes inalámbricas basadas en el estándar IEEE 802.11 para prevenir colisiones de datos.

**Funcionamiento:**

1. **Necesidad de envío:** Cuando una estación inalámbrica tiene datos para enviar, primero envía un paquete RTS al punto de acceso (AP) o a otra estación de destino.
   
2. **Duración de la reserva:** El paquete RTS incluye información sobre cuánto tiempo (duración de la reserva) la estación necesita para transmitir sus datos. Esto permite a otras estaciones inalámbricas saber cuándo el canal estará ocupado y evitarán enviar datos durante ese período para evitar colisiones.

3. **Respuesta CTS:** Después de recibir el RTS, el punto de acceso o la estación de destino envían un paquete CTS (Clear to Send) de vuelta a la estación que originó el RTS. Este CTS también incluye la duración de la reserva, confirmando que el canal está disponible para la transmisión solicitada.

4. **Transmisión de datos:** Una vez que la estación que originó el RTS recibe el CTS, puede comenzar a transmitir sus datos. Durante este tiempo, otras estaciones inalámbricas respetan la duración de la reserva y no intentan transmitir datos para evitar colisiones.

#### CTS (Clear to Send)

**¿Qué es?** CTS (Clear to Send) es la respuesta enviada por el punto de acceso (AP) o la estación de destino después de recibir un paquete RTS, confirmando que el canal está disponible para la transmisión solicitada.

**Funcionamiento:**

1.  **Recepción del RTS:** Cuando una estación inalámbrica recibe un paquete RTS de otra estación que desea transmitir datos, verifica la duración de la reserva especificada en el RTS.

2. **Envío del CTS:** Si el canal está libre durante el período especificado en el RTS, la estación de destino responde con un paquete CTS. Este paquete también incluye la duración de la reserva, lo que permite a la estación que originó el RTS comenzar a transmitir sus datos sin temor a colisiones con otras estaciones

3. **Inicio de la transmisión:** Tras recibir el CTS, la estación que originó el RTS puede comenzar a transmitir sus datos de manera segura, sabiendo que el canal está reservado exclusivamente para ella durante el tiempo especificado en la duración de la reserva.

Dentro de el **Clear to Send** tenemos dos divisiones mas que son:

- **CTS to Self (CTS/Self)**: Este CTS es enviado por un nodo a sí mismo para reservar el canal y asegurar que puede transmitir sin interrupciones.
- **CTS to Others (CTS/Others)**: Este CTS es enviado en respuesta a un RTS de otro nodo, indicando a otros nodos en la red que deben abstenerse de transmitir datos durante un tiempo específico para evitar colisiones.

De forma general este es un proceso que se da en la red inalámbrica el cual permite administrar el envió y paquetes y la recepción de los mismos de una manera exitosa evitando el choque de paquetes y optimizando la transmición.

Con esto claro tenemos que comprender que cuando se mite el CTS tiene un esquema simple de ver:

![[Pasted image 20240711131535.png]]

Como vemos la estructura es sencilla pero en lo que vamos a tener énfasis en el tipo el cual es definido en micro segundos desde el 0 hasta el 32,767.
Teniendo esto en cuenta el propósito es alterar la duración ubicando un valor bastante alto, también debemos poner la dirección address del punto de acceso para generar el ataque y por ultimo el FCS ("Frame Check Sequence") el cual nos permite verificar la integridad de los datos, de esto no tenemos que preocuparnos del todo ya que con herramientas como [[Brain/Hack4u/Hacking Wifi/Herramientas/Wireshark]]  vamos a poder copiar el FCS y enviar nuestro paquete modificado.

Ya con los conocimientos básicos claros podemos comenzar con el ataque:
1. Vamos a capturar los paquetes con la ayuda de [[Brain/Hack4u/Hacking Wifi/Herramientas/Wireshark]], es cuestión de abrir y escoger la tarjeta de red para ver como ya comienza a capturar los paquetes:
![[Pasted image 20240712214007.png]]

2. En este punto vamos a agregar el siguiente filtro para intentar encontrar la paquetería correspondiente al CTS.

![[Pasted image 20240712214113.png]]

3. Ya con esto tenemos que buscar por la mac de nuestro AP victima y escoger uno de los paquetes:
![[Pasted image 20240712234109.png]]

4. ya tengo escogido el paquete y como vemos consta de 110 microsegundos con esto listo vamos a exportar paquete único con la siguiente configuración:

![[Pasted image 20240712234225.png]]

5. Ya almacenado vamos a abrir el archivo con ayuda de [[Ghex]] y lo veremos de la siguiente forma:
![[Pasted image 20240712234423.png]]

Como vemos ya tenemos abierto esto y vamos a contar 8 pares de dígitos donde el 9 y 10 son los microsegundo en hexadecimal, lo comprobamos con ayuda de Python como en la imagen:

6. Vamos a calcular el valor de 30000 en hexadecimal para poder modificar el paquete y esto lo hacemos con python:
![[Pasted image 20240712234735.png]]

ya con esto listo vamos a modificar el paquete para que los microsegundos sean 30000 de la siguiente forma:
![[Pasted image 20240712235339.png]]

7. Guardamos los datos del paquete y salimos
8. Lo siguiente es de preferencia comprobar los datos de la captura abriendo esto con Wireshark:
![[Pasted image 20240712235431.png]]

como vemos el paquete ya tiene los nuevos valores en microsegundos por lo que ahora con ayuda de [[TCPReplay]] vamos a replicar el paquete en la red:

```bash
tcpreplay --intf1=wlan0 --topspeed --lop=2000 ctsframe.pcap 2>/dev/null
```

lo que hace el comando es especificar la tarjeta con la que se va a hacer el ataque seguido decimos inyecta todos los paquetes de golpe no 1x1 y por ultimo emite 2000 de estos paquetes
En mi caso mi tarjeta no me lo permite pero con una tarjeta que si se puede enviar a mi me da un error de soporte:
![[Pasted image 20240713000335.png]]

### Ataque Beacon Flood

Este tipo de ataque se trata de inundar todo el espacio aéreo de paquetes Beacon, pero para comprender bien esto primero expliquemos los paquetes Beacon:
#### Paquetes Beacon en redes Wi-Fi:

1. **Definición**: Los paquetes beacon son pequeños mensajes de datos emitidos por un punto de acceso (AP) Wi-Fi aproximadamente cada 100 milisegundos. Contienen información básica sobre la red Wi-Fi, como su nombre (SSID), tipo de seguridad (por ejemplo, WPA2), velocidad de conexión y otros parámetros configurados.
 
 2. **Funcionamiento**:
	- **Anuncio de red**: El AP envía periódicamente estos paquetes beacon en una frecuencia predefinida para informar a los dispositivos cercanos de su presencia y configuración.
	- **Descubrimiento de red**: Los dispositivos Wi-Fi, como laptops, teléfonos inteligentes y otros dispositivos móviles, escanean activamente las señales de estos paquetes beacon para identificar las redes disponibles a las que pueden conectarse.
	- **Conexión y roaming**: Una vez que un dispositivo detecta un paquete beacon de una red conocida y autorizada, puede establecer una conexión y, en el caso de redes Wi-Fi empresariales o grandes, facilita el roaming sin interrupciones al moverse entre diferentes puntos de acceso dentro de la misma red.

De manera general esto paquetes Beacon no son mas que paquetes enviados por el AP con información sobre la conexión con el objetivo de ser detectados para posibles conexiones.

Centrándonos en el ataque la idea es emitir un montón de estos paquetes con información falsa sobre el punto de acceso al cual se quiere atacar para así lograr dañar el espectro de onda del AP, lo que queremos decir que es vamos a emitir los paquetes beacon en el mismo canal y con el mismo nombre de la red con el objetivo de que los dispositivos se vuelvan locos al intentarse conectar y automáticamente se caiga la conexión, esto lo vamos a hacer con ayuda de la herramienta [[MDK3]].


Para este tipo de ataque vamos a creamos un archivo que contenga el nombre de varias redes, esto lo podemos hacer de varias formas pero en mi caso lo voy a hacer con el siguiente comando:
```bash
for i in $(seq 1 10); do echo "MyNetwork$i" >> redes.txt ; done
```

Con esto ya tenemos nuestro archivo con varias redes:

![[Pasted image 20240713181336.png]]

ya con esto vamos a hacer uso de [[MDK3]] con el siguiente comando:

```bash
mdk3 wlan0 b -f redes.txt -a -s 1000 -c 1
```

y con esto comienza el ataque, no olvidarse que la tarjeta de red se debe encontrar en modo monitor.

![[Pasted image 20240713182656.png]]

ya con esto comienza el ataque y se deben ver múltiples redes disponibles saturando así el espectro de red.

### Ataque de Disassociation Amok Mode

Este tipo de ataque también lo vamos a hacer con uso de [[MDK3]] y recordar tener airodump-ng escuchando:
![[Pasted image 20240713184446.png]]

tenemos que crear una lista una blacklist con los bssid de los dispositivos a desconectar:

![[Pasted image 20240713184650.png]]

con esto vamos a encargarnos de tener airodump en escucha y ejecutar el siguiente comando:

```bash
mdk3 wlan0 d -w blacklist -c 11
```

y con esto se inicia el ataque:

![[Pasted image 20240713185010.png]]


