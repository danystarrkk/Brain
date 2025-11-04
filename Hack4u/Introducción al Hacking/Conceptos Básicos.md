-------

## Direcciones IP (IPV4 e IPV6)

Las direcciones IP son identificadores numéricos únicos que se utilizan para identificar dispositivos en una red, como ordenadores, routers, servidores y otros dispositivos conectados a Internet.

Existen dos versiones de direcciones IP: **IPv4** e **IPv6**. La versión **IPv4** utiliza un formato de dirección de **32 bits** y se utiliza actualmente en la mayoría de las redes. La versión **IPv6** utiliza un formato de dirección de **128 bits** y se está implementando gradualmente en todo el mundo para hacer frente a la escasez de direcciones IPv4.

Las direcciones IPv4 se representan como cuatro números separados por puntos, como **192.168.0.1**, mientras que las direcciones IPv6 se representan en notación hexadecimal y se separan por dos puntos, como **2001:0db8:85a3:0000:0000:8a2e:0370:7334**.

Mediante el comando `ifconfig` podemos verificar nuestra IP de la siguiente manera:

![[Pasted image 20250821092208.png]]

En este punto vamos a convertir nuestra IP en ceros y unos, de la siguiente manera:

```bash
echo "$(echo "obase=2; 192" | bc).168.0.180"
```

![[Pasted image 20250821092444.png]]

Podemos observar que el valor de 192 en binario es 11000000 y esto lo vamos a hacer con todos:

```bash
echo "$(echo "obase=2; 192" | bc).$(echo "obase=2; 168" | bc).$(echo "obase=2; 0" | bc).$(echo "obase=2; 180" | bc)"
```

![[Pasted image 20250821092931.png]]

Podemos observar que tenemos 4 octetos de 8 bits con un total de 32 bits por lo que respecto a direcciones IPV4 vamos a tener un total de `2^32` que seria: `4294967296` de IPV4 disponibles pero en el mundo ya somos mas humanos que el numero de direcciones IPV4 por lo que se implemento IPV6 la cual nos da un total de `2^128` por lo que el problema se resuelve.

-----
## Direcciones MAC ( OUI y NIC )

La dirección MAC es un número hexadecimal de **12 dígitos** (número binario de **6 bytes**), que está representado principalmente por **notación hexadecimal** de dos puntos.

Mediante el mismo comando de `ifconfig` podremos ver la MAC:

![[Pasted image 20250821093954.png]]

Los primeros **6 dígitos** (digamos **00:0c:29**) del MAC Address, identifican al fabricante, llamado **OUI** (**Identificador Único Organizacional**). El Comité de la Autoridad de Registro de **IEEE** asigna estos prefijos MAC a sus proveedores registrados.

Los seis dígitos más a la derecha representan el **controlador de interfaz de red**, que es asignado por el fabricante.

Es decir, los primeros **3 bytes** (**24 bits**) representan el fabricante de la tarjeta, y los últimos **3 bytes** (**24 bits**) identifican la tarjeta particular de ese fabricante. Cada grupo de **3 bytes** se puede representar con **6 dígitos hexadecimales**, formando un número hexadecimal de **12 dígitos** que representa la MAC completa.

Para una búsqueda de fabricante utilizando direcciones MAC, se requieren al menos los primeros **3 bytes** (**6 caracteres**) de la dirección MAC. Una de las herramientas que vemos en esta clase para lograr dicho fin es [[Macchanger]]  una herramienta de GNU/Linux para la visualización y manipulación de direcciones MAC.

Muchas implementaciones como lo son [[Arp-Scan]] hacen uso de de la MAC para identificar diferentes dispositivos en red:

```bash
sudo arp-scan -I eth0 --local-net
```

![[Pasted image 20250821095020.png]]

como podemos observar tenemos los nombres de los dispositivos.

----
## Protocolos comunes (UDP, TCP) y el famoso Three-Way Handshake

Los protocolos **TCP** (Transmission Control Protocol) y **UDP** (User Datagram Protocol) son dos de los protocolos de red más comunes utilizados en la transferencia de datos a través de redes de ordenadores.

El protocolo **TCP**, es un protocolo **orientado a la conexión** que proporciona una entrega de datos confiable, mientras que el protocolo **UDP**, es un protocolo **no orientado a conexión** el cual no garantiza la entrega de datos y tampoco verifica los errores.

Una parte crucial del protocolo **TCP** es el **Three-Way Handshake**, un procedimiento utilizado para establecer una conexión entre dos dispositivos. Este procedimiento consta de tres pasos: **SYN**, **SYN-ACK** y **ACK**, en los que se sincronizan los números de secuencia y de reconocimiento de los paquetes entre los dispositivos. El Three-Way Handshake es fundamental para establecer una conexión confiable y segura a través de TCP.

De forma grafica vamos a ver como se obtiene el Three-Way-Handshake en la conexiones monitorizando con ayuda de [[Wireshark]] y veremos lo siguiente:

![[Pasted image 20250821100608.png]]

Puertos TCP comunes:

- 21: **FTP** (File Transfer Protocol) – permite la transferencia de archivos entre sistemas.
- 22: **SSH** (Secure Shell) – un protocolo de red seguro que permite a los usuarios conectarse y administrar sistemas de forma remota.
- 23: **Telnet** – un protocolo utilizado para la conexión remota a dispositivos de red.
- 80: **HTTP** (Hypertext Transfer Protocol) – el protocolo que se utiliza para la transferencia de datos en la World Wide Web.
- 443: **HTTPS** (Hypertext Transfer Protocol Secure) – la versión segura de HTTP, que utiliza encriptación SSL/TLS para proteger las comunicaciones web.

Puertos UDP comunes:

- 53: **DNS** (Domain Name System) – un sistema que traduce nombres de dominio en direcciones IP.
- 67/68: **DHCP** (Dynamic Host Configuration Protocol) – un protocolo utilizado para asignar direcciones IP y otros parámetros de configuración a los dispositivos en una red.
- 69: **TFTP** (Trivial File Transfer Protocol) – un protocolo simple utilizado para transferir archivos entre dispositivos en una red.
- 123: **NTP** (Network Time Protocol) – un protocolo utilizado para sincronizar los relojes de los dispositivos en una red.
- 161: **SNMP** (Simple Network Management Protocol) – un protocolo utilizado para administrar y supervisar dispositivos en una red.

Cabe destacar que estos son solo **algunos de los más comunes**. Existen muchos más puertos los cuales operan tanto por TCP como por UDP.

----
## El modelo OSI – ¿En qué consiste y cómo se estructura la actividad de red en capas?

En redes de ordenadores, el modelo **OSI** (Open Systems Interconnection) es una estructura de **siete capas** que se utiliza para describir el proceso de comunicación entre dispositivos. Cada capa proporciona servicios y funciones específicas, que permiten a los dispositivos comunicarse a través de la red.

A continuación, se describen brevemente las siete capas del modelo OSI:

1. **Capa física**: Es la capa más baja del modelo OSI, que se encarga de la transmisión de datos a través del medio físico de la red, como cables de cobre o fibra óptica.

2. **Capa de enlace de datos**: Esta capa se encarga de la transferencia confiable de datos entre dispositivos en la misma red. También proporciona funciones para la detección y corrección de errores en los datos transmitidos.

3. **Capa de red**: La capa de red se ocupa del enrutamiento de paquetes de datos a través de múltiples redes. Esta capa utiliza direcciones lógicas, como direcciones IP, para identificar dispositivos y rutas de red.

4. **Capa de transporte**: La capa de transporte se encarga de la entrega confiable de datos entre dispositivos finales, proporcionando servicios como el control de flujo y la corrección de errores.

5. **Capa de sesión**: Esta capa se encarga de establecer y mantener las sesiones de comunicación entre dispositivos. También proporciona servicios de gestión de sesiones, como la autenticación y la autorización.

6. **Capa de presentación**: La capa de presentación se encarga de la representación de datos, proporcionando funciones como la codificación y decodificación de datos, la compresión y el cifrado.

7. **Capa de aplicación**: La capa de aplicación proporciona servicios para aplicaciones de usuario finales, como correo electrónico, navegadores web y transferencia de archivos.

Comprender la estructura en capas del modelo OSI es esencial para cualquier analista de seguridad, ya que permite tener una visión completa del funcionamiento de la red y de las posibles vulnerabilidades que puedan existir en cada una de las capas.

Esto nos permite identificar de manera efectiva los puntos débiles de la red y aplicar medidas de seguridad adecuadas para protegerla de posibles ataques.

----
## Subnetting – ¿Qué es y cómo se interpreta una máscara de red?

**Subnetting** es una técnica utilizada para dividir una red IP en subredes más pequeñas y manejables. Esto se logra mediante el uso de **máscaras de red**, que permiten definir qué bits de la dirección IP corresponden a la **red** y cuáles a los **hosts**.

Para interpretar una máscara de red, se deben identificar los bits que están en “**1**“. Estos bits representan la porción de la dirección IP que corresponde a la **red**. Por ejemplo, una máscara de red de **255.255.255.0** indica que los primeros **tres octetos** de la dirección IP corresponden a la **red**, mientras que el **último octeto** se utiliza para identificar los **hosts**

Ahora bien, cuando hablamos de **CIDR** (acrónimo de **Classless Inter-Domain Routing**), a lo que nos referimos es a un método de asignación de direcciones IP más eficiente y flexible que el uso de clases de redes IP fijas. Con **CIDR**, una dirección IP se representa mediante una dirección IP base y una máscara de red, que se escriben juntas separadas por una barra (**/**).

Por ejemplo, la dirección IP **192.168.1.1** con una máscara de red de **255.255.255.0** se escribiría como **192.168.1.1/24**.

La máscara de red se representa en notación de prefijo, que indica el número de bits que están en “**1**” en la máscara. En este caso, la máscara de red **255.255.255.0** tiene **24** bits en “**1**” (los primeros tres octetos), por lo que su notación de prefijo es **/24**.

Para calcular la máscara de red a partir de una notación de prefijo, se deben escribir los bits “**1**” en los primeros bits de una dirección IP de 32 bits y los bits “**0**” en los bits restantes. Por ejemplo, la máscara de red **/24** se calcularía como **11111111.11111111.11111111.00000000** en binario, lo que equivale a **255.255.255.0** en decimal.

Vamos a crear un Excel para entender esto de mejor manera:

![[Subnetting.xlsx]]

En cuanto a clases de direcciones IP, existen tres tipos de máscaras de red: **Clase A**, **Clase B** y **Clase C**.

- Las redes de **clase A** usan una máscara de subred predeterminada de **255.0.0.0** y tienen de **0** a **127** como su **primer octeto**. La dirección **10.52.36.11**, por ejemplo, es una dirección de **clase A**. Su primer octeto es **10**, que está entre **1** y **126**, ambos incluidos.
- Las redes de **clase B** usan una máscara de subred predeterminada de **255.255.0.0** y tienen de **128** a **191** como su **primer octeto**. La dirección **172.16.52.63**, por ejemplo, es una dirección de **clase B**. Su primer octeto es **172**, que está entre **128** y **191**, ambos inclusive.
- Las redes de **clase C** usan una máscara de subred predeterminada de **255.255.255.0** y tienen de **192** a **223** como su **primer octeto**. La dirección **192.168.123.132**, por ejemplo, es una dirección de **clase C**. Su primer octeto es **192**, que está entre **192** y **223**, ambos incluidos.

Es importante tener en cuenta que, además de estos tres tipos de máscaras de red, **también existen máscaras de red personalizadas** que se pueden utilizar para crear subredes de diferentes tamaños dentro de una red.

Tal y como mencionamos en la descripción de la clase anterior sobre los CIDRs (**Classless Inter-Domain Routing**), se trata de un método de asignación de direcciones IP que permite dividir una dirección IP en una parte que identifica **la red** y otra parte que identifica **el host**. Esto se logra mediante el uso de una **máscara de red**, que se representa en notación **CIDR** como una **dirección IP base** seguida de un número que indica la **cantidad de bits** que corresponden a la red.

Con **CIDR**, se pueden asignar direcciones IP de forma **más precisa**, lo que reduce la cantidad de direcciones IP desperdiciadas y facilita la administración de la red.

El número que sigue a la dirección IP base en la notación CIDR se llama **prefijo** o **longitud de prefijo**, y representa el **número de bits** en la máscara de red que están en “**1**“.

Por ejemplo, una dirección IP con un prefijo de **/24** indica que los primeros **24 bits** de la dirección IP corresponden a la **red**, mientras que los **8 bits** restantes se utilizan para identificar los **hosts**.

Para calcular la cantidad de hosts disponibles en una red CIDR, se deben contar el número de bits “**0**” en la máscara de red y elevar **2 a la potencia** de ese número. Esto se debe a que cada bit “**0**” en la máscara de red representa un bit que se puede utilizar para identificar un host.

Por ejemplo, una máscara de red de **255.255.255.0** (**/24**) tiene **8 bits** en “**0**“, lo que significa que hay **2^8 = 256** direcciones IP disponibles para los hosts en esa red.

A continuación, se representan algunos ejemplos prácticos de CIDR:

- Una dirección IP con un prefijo de **/28** (**255.255.255.240**) permite hasta **16 direcciones IP** para los hosts (**2^4**), ya que los primeros **28 bits** corresponden a la red.
- Una dirección IP con un prefijo de **/26** (**255.255.255.192**) permite hasta **64 direcciones IP** para los hosts (**2^6**), ya que los primeros **26 bits** corresponden a la red.
- Una dirección IP con un prefijo de **/22** (**255.255.252.0**) permite hasta **1024 direcciones IP** para los hosts (**2^10**), ya que los primeros **22 bits** corresponden a la red.

A continuación, se detalla paso a paso cómo hemos ido calculando cada apartado:

1. **Cálculo de la máscara de red:**

La dirección IP que se nos dio es **192.168.1.0/26**, lo que significa que los primeros **26 bits** de la dirección IP corresponden a la **red** y los últimos **6 bits** corresponden a los **hosts**.

Para calcular la **máscara de red**, necesitamos colocar los primeros **26 bits** en **1** y los últimos **6 bits** en **0**. En binario, esto se ve así:

- **11111111.11111111.11111111.11000000**

Cada octeto de la máscara de red se compone de **8 bits**. El valor de cada octeto se determina convirtiendo los **8 bits** a **decimal**. En este caso, los primeros **24 bits** son todos **1s**, lo que significa que el valor decimal de cada uno de estos octetos es **255**. El último octeto tiene los últimos **6 bits** en **0**, lo que significa que su valor decimal es **192**.

Por lo tanto, la máscara de red para esta dirección IP es **255.255.255.192**.

2. **Cálculo del total de hosts a repartir:**

En este caso, se pueden utilizar los **6 bits** que quedan disponibles para representar la parte de **host**. En una máscara de red de **26 bits**, los **6 bits** restantes representan **2^6 – 2 = 62** hosts disponibles para asignar.

El número máximo de hosts disponibles se calcula como **2^(n) – 2**, donde **n** es la cantidad de bits utilizados para representar la parte de **host** en la máscara de red.

3. **Cálculo del Network ID:**

Para calcular el **Network ID**, lo que debemos hacer es aplicar la máscara de red a la dirección IP de la red. En este caso, la máscara de red es **255.255.255.192**, lo que significa que los primeros **26 bits** de la dirección IP pertenecen a la parte de **red**.

Para calcular el **Network ID**, convertimos tanto la dirección IP como la máscara de red en **binario** y luego hacemos una operación “**AND**” lógica entre los dos. La operación “**AND**” compara los bits correspondientes en ambas direcciones y devuelve un resultado en el que todos los bits coincidentes son iguales a “**1**” y todos los bits no coincidentes son iguales a “**0**“.

En este caso, la dirección **IP** es **192.168.1.0** en decimal y se convierte en binario como **11000000.10101000.00000001.00000000**. La máscara de red es **255.255.255.192** en decimal y se convierte en binario como **11111111.11111111.11111111.11000000**.

Luego, aplicamos la operación “**AND**” entre estos dos valores binarios bit a bit. Los bits correspondientes en ambos valores se comparan de la siguiente manera:

![[Pasted image 20250821113837.png]]

El resultado final es el **Network ID**, que es **192.168.1.0**. Este es el identificador único de la subred a la que pertenecen los hosts.

4. **Cálculo de la Broadcast Address:**

La **Broadcast Address** es la dirección de red que se utiliza para enviar paquetes a **todos los hosts de la subred**. Para calcularla, necesitamos saber el **Network ID** y la **cantidad de hosts** disponibles en la subred.

En el ejemplo que estamos trabajando, ya hemos calculado el **Network ID** como **192.168.1.0**. La cantidad de hosts disponibles se calcula como **2^(n) – 2**, donde **n** es la cantidad de bits utilizados para representar la parte de host en la máscara de red. En este caso, **n** es igual a **6**, ya que hay **6** bits disponibles para la parte de **host**.

Para calcular la **Broadcast Address**, debemos asignar todos los bits de la parte del **host** de la dirección IP a “**1**“. En este caso, la dirección IP es **192.168.1.0** y se convierte en binario como **11000000.10101000.00000001.00000000**.

Para encontrar la dirección **Broadcast**, llenamos con unos la parte correspondiente a los bits de host, es decir, los últimos **6 bits**:

**11000000.10101000.00000001.00111111** (dirección IP con bits de host asignados a “**1**“)

Luego, convertimos este valor binario de regreso a decimal y obtenemos la dirección de Broadcast: **192.168.1.63**. Esta es la dirección a la que se enviarán los paquetes para llegar a todos los hosts de la subred.