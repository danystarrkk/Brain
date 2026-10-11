---
aliases:
  - Nmap
  - Reconocimiento de Redes
  - OSINT
  - Evasión de Firewall
  - Enumeración de Puertos
  - Descubrimiento de Hosts
  - Google Hacking
  - Fuzzing Web
  - Burp Suite
tags:
  - hacking/reconocimiento
  - hacking/herramientas
  - hacking/web
  - redes/comandos
  - redes/protocolos
  - scripting/bash
  - programacion/python
  - herramientas/burpsuite
  - hacking/laboratorios
estado: 🟢 Terminado
---
# Nmap y sus Diferentes Modos de Escaneo

**Nmap** es una herramienta de **escaneo de red** gratuita y de código abierto que se utiliza en pruebas de penetración (pentesting) para explorar y auditar redes y sistemas informáticos.

Con Nmap, los profesionales de seguridad pueden identificar los **hosts** conectados a una red, los **servicios** que se están ejecutando en ellos y las **vulnerabilidades** que podrían ser explotadas por un atacante. La herramienta es capaz de detectar una amplia gama de dispositivos, incluyendo enrutadores, servidores web, impresoras, cámaras IP, sistemas operativos y otros dispositivos conectados a una red.

Asimismo, esta herramienta posee una variedad de funciones y características avanzadas que permiten a los profesionales de seguridad adaptarla a sus necesidades específicas. Estas incluyen técnicas de escaneo agresivas, capacidades de scripting personalizadas y un conjunto de herramientas auxiliares que pueden ser utilizadas para obtener información adicional sobre los hosts objetivo.

> [!info] Nmap es mucho más que un escáner de puertos. Su arquitectura modular, especialmente a través del Nmap Scripting Engine (NSE), permite extender sus capacidades para detección de vulnerabilidades, enumeración avanzada, e incluso explotación básica. Es una herramienta fundamental en la fase de reconocimiento activo.

Los parámetros de uso los veremos en [[Nmap Guía Completa|Nmap]] y vamos a ver dos ejemplos:

1.  Primer escaneo para ver puertos abiertos:
```bash
    nmap -p- --open -sS --min-rate 5000 -v -n -Pn <IP> -oG allPorts
```

> [!tip] Explicación del comando:
    > *   `-p-`: Escanea todos los puertos (1-65535).
    > *   `--open`: Muestra solo los puertos que están abiertos.
    > *   `-sS`: Realiza un escaneo SYN "Stealth" (sigiloso), que es más rápido y menos propenso a ser detectado por firewalls básicos.
    > *   `--min-rate 5000`: Establece una tasa mínima de envío de paquetes de 5000 por segundo, acelerando el escaneo.
    > *   `-v`: Aumenta la verbosidad para ver el progreso en tiempo real.
    > *   `-n`: No realiza resolución DNS, lo que acelera el escaneo.
    > *   `-Pn`: Trata a todos los hosts como si estuvieran en línea, omitiendo la fase de descubrimiento de hosts (útil cuando un firewall bloquea pings).
    > *   `<IP>`: La dirección IP del objetivo.
    > *   `-oG allPorts`: Guarda la salida en formato Grepable en un archivo llamado `allPorts`.

2.  Escaneo para descubrir versiones y servicios en puertos específicos:
```bash
    nmap -p<puertos> -sCV <IP> -oN target
```

> [!tip] Explicación del comando:
    > *   `-p<puertos>`: Especifica los puertos a escanear (ej. `-p80,443,22`).
    > *   `-sC`: Ejecuta los scripts por defecto del Nmap Scripting Engine (NSE), que son útiles para enumeración y detección de vulnerabilidades comunes.
    > *   `-sV`: Intenta determinar la versión del servicio que se ejecuta en los puertos abiertos.
    > *   `<IP>`: La dirección IP del objetivo.
    > *   `-oN target`: Guarda la salida normal en un archivo llamado `target`.

## Técnicas de Evasión de Firewall (MTU, Data Length, Source Port, Decoy, etc.)

Cuando se realizan pruebas de penetración, uno de los mayores desafíos es evadir la detección de los **firewalls**, que están diseñados para proteger las redes y sistemas de posibles amenazas. Para superar este obstáculo, Nmap ofrece una variedad de técnicas de evasión que permiten a los profesionales de seguridad realizar escaneos sigilosos y evitar así la detección de los mismos.

Algunos de los parámetros vistos en esta clase son los siguientes:

*   **MTU (`--mtu`)**: La técnica de evasión de **MTU** o "Maximum Transmission Unit" implica ajustar el tamaño de los paquetes que se envían para evitar la detección por parte del firewall. Nmap permite configurar manualmente el tamaño máximo de los paquetes para garantizar que sean lo suficientemente pequeños para pasar por el firewall sin ser detectados.
> [!info] La fragmentación de MTU divide los paquetes en unidades más pequeñas. Algunos firewalls pueden tener dificultades para reensamblar y analizar estos fragmentos, permitiendo que el tráfico de escaneo pase desapercibido. Sin embargo, esta técnica puede aumentar el tráfico de red y ser detectada por sistemas IDS/IPS más avanzados.
*   **Data Length (`--data-length`)**: Esta técnica se basa en ajustar la **longitud de los datos** enviados para que sean lo suficientemente cortos como para pasar por el firewall sin ser detectados. Nmap permite a los usuarios configurar manualmente la longitud de los datos enviados para que sean lo suficientemente pequeños para evadir la detección del firewall.
> [!info] Añadir datos aleatorios a los paquetes (o reducir su longitud) puede alterar la firma de los paquetes de escaneo, haciendo que parezcan tráfico legítimo o dificultando su análisis por parte de los firewalls que buscan patrones específicos.
*   **Source Port (`--source-port`)**: Esta técnica consiste en configurar manualmente el número de **puerto de origen** de los paquetes enviados para evitar la detección por parte del firewall. Nmap permite a los usuarios especificar manualmente un puerto de origen aleatorio o un puerto específico (ej. 53 para DNS, 80 para HTTP) para evadir la detección del firewall.
> [!info] Muchos firewalls están configurados para permitir el tráfico saliente desde puertos bien conocidos (como 53/DNS, 80/HTTP, 443/HTTPS) para facilitar la navegación web y el uso de servicios legítimos. Al falsificar el puerto de origen, un escaneo puede parecer una respuesta a una consulta legítima.
*   **Decoy (`-D`)**: Esta técnica de evasión en Nmap permite al usuario enviar **paquetes falsos** a la red para confundir a los sistemas de detección de intrusos (IDS) y evitar la detección del firewall. El comando **`-D`** permite al usuario enviar paquetes falsos junto con los paquetes reales de escaneo para ocultar su actividad.
> [!info] La técnica Decoy (`-D`) inyecta direcciones IP falsas (decoys) en los paquetes de escaneo. Esto hace que el objetivo vea múltiples fuentes de escaneo, dificultando la identificación del atacante real entre el ruido generado. Es crucial usar IPs que estén activas para que los decoys parezcan legítimos.
*   **Fragmented (`-f`)**: Esta técnica se basa en **fragmentar los paquetes** enviados para que el firewall no pueda reconocer el tráfico como un escaneo. La opción **`-f`** en Nmap permite fragmentar los paquetes y enviarlos por separado para evitar la detección del firewall.
> [!info] La fragmentación de paquetes divide la cabecera TCP en múltiples fragmentos. Algunos firewalls o IDS pueden tener problemas para reensamblar estos fragmentos correctamente, especialmente si están mal implementados o si el tráfico es elevado, lo que permite que el escaneo pase desapercibido.
*   **Spoof-Mac (`--spoof-mac`)**: Esta técnica de evasión se basa en **cambiar la dirección MAC** del paquete para evitar la detección del firewall. Nmap permite al usuario configurar manualmente la dirección MAC para evitar ser detectado por el firewall.
> [!info] La suplantación de MAC (`--spoof-mac`) es útil en redes locales donde los firewalls o sistemas de seguridad pueden registrar o filtrar por direcciones MAC. Al cambiar la MAC, el atacante puede intentar evadir listas negras o hacerse pasar por un dispositivo de confianza.
*   **Stealth Scan (`-sS`)**: Esta técnica es una de las más utilizadas para realizar escaneos sigilosos y evitar la detección del firewall. El comando **`-sS`** permite a los usuarios realizar un escaneo de tipo **SYN** **sin establecer una conexión completa**, lo que permite evitar la detección del firewall.
> [!info] El escaneo SYN (`-sS`), también conocido como "half-open scan", envía un paquete SYN y espera una respuesta SYN/ACK. Si la recibe, el puerto está abierto, pero Nmap envía un RST en lugar de un ACK final, evitando completar el handshake TCP. Esto lo hace menos ruidoso que un escaneo TCP completo.
*   **min-rate (`--min-rate`)**: Esta técnica permite al usuario **controlar la velocidad de los paquetes** enviados para evitar la detección del firewall. El comando **`--min-rate`** permite al usuario reducir la velocidad de los paquetes enviados para evitar ser detectado por el firewall.
> [!info] Reducir la tasa de envío de paquetes (`--min-rate` o `--max-rate`) puede ayudar a evadir sistemas IDS/IPS que detectan escaneos basándose en el volumen o la velocidad del tráfico. Un escaneo muy lento puede pasar desapercibido al mezclarse con el tráfico de red normal.

Es importante destacar que, además de las técnicas de evasión mencionadas anteriormente, existen muchas otras opciones en Nmap que pueden ser utilizadas para realizar pruebas de penetración efectivas y evadir la detección del firewall. Sin embargo, las técnicas que hemos mencionado son algunas de las más populares y ampliamente utilizadas por los profesionales de seguridad para superar los obstáculos que presentan los firewalls en la realización de pruebas de penetración.

## Uso de Scripts y Categorías en Nmap para Aplicar Reconocimiento

Una de las características más poderosas de **Nmap** es su capacidad para automatizar tareas utilizando **scripts personalizados**. Los scripts de Nmap permiten a los profesionales de seguridad automatizar las tareas de reconocimiento y descubrimiento en la red, además de obtener información valiosa sobre los sistemas y servicios que se están ejecutando en ellos. El parámetro **`--script`** de Nmap permite al usuario seleccionar un conjunto de scripts para ejecutar en un objetivo de escaneo específico.

> [!info] El Nmap Scripting Engine (NSE) es un componente clave de Nmap que permite a los usuarios escribir scripts para automatizar una amplia variedad de tareas de red. Estos scripts están escritos en el lenguaje de programación Lua y pueden ser utilizados para detección de vulnerabilidades, enumeración avanzada, detección de malware, y mucho más.

Existen diferentes categorías de scripts disponibles en Nmap, cada una diseñada para realizar una tarea específica. Algunas de las categorías más comunes incluyen:

*   **default**: Esta es la categoría predeterminada en Nmap, que incluye una gran cantidad de scripts de reconocimiento básicos y útiles para la mayoría de los escaneos.
*   **discovery**: Esta categoría se enfoca en descubrir información sobre la red, como la detección de hosts y dispositivos activos, y la resolución de nombres de dominio.
*   **safe**: Esta categoría incluye scripts que son considerados seguros y que no realizan actividades invasivas que puedan desencadenar una alerta de seguridad en la red.
*   **intrusive**: Esta categoría incluye scripts más invasivos que pueden ser detectados fácilmente por un sistema de detección de intrusos o un firewall, pero que pueden proporcionar información valiosa sobre vulnerabilidades y debilidades en la red.
*   **vuln**: Esta categoría se enfoca específicamente en la detección de vulnerabilidades y debilidades en los sistemas y servicios que se están ejecutando en la red.

En conclusión, el uso de scripts y categorías en Nmap es una forma efectiva de automatizar tareas de reconocimiento y descubrimiento en la red. El parámetro **`--script`** permite al usuario seleccionar un conjunto de scripts personalizados para ejecutar en un objetivo de escaneo específico, mientras que las diferentes categorías disponibles en Nmap se enfocan en realizar tareas específicas para obtener información valiosa sobre la red.

---

## Alternativas para la Enumeración de Puertos Usando Descriptores de Archivo

La enumeración de puertos es una tarea crucial en las pruebas de penetración y seguridad de redes. Tal y como hemos visto, Nmap es una herramienta de línea de comandos ampliamente utilizada para esta tarea, pero existen alternativas para realizar la enumeración de puertos de manera efectiva sin utilizar herramientas externas.

Una alternativa a la enumeración de puertos utilizando herramientas externas es aprovechar el poder de los **descriptores de archivo** en sistemas **Unix**. Los descriptores de archivo son una forma de acceder y manipular archivos y dispositivos en sistemas Unix. En particular, la utilización de **/dev/tcp** permite la conexión a un host y puerto específicos como si se tratara de un archivo en el sistema.

> [!info] `/dev/tcp` y `/dev/udp` son pseudo-archivos especiales en sistemas Linux/Unix que permiten interactuar con sockets TCP/UDP directamente desde la línea de comandos de Bash. Al redirigir la entrada/salida a estos "archivos", Bash puede establecer conexiones de red, enviar datos y recibir respuestas, lo que es útil para pruebas rápidas de conectividad o escaneos básicos sin herramientas externas.

Para realizar la enumeración de puertos utilizando **/dev/tcp** en Bash, es posible crear un script que realice una conexión a cada puerto de interés y compruebe si el puerto está abierto o cerrado en función de si se puede enviar o recibir datos. Una forma de hacer esto es mediante el uso de comandos como "**echo**" o "**cat**", aplicando redireccionamientos al **/dev/tcp**. El código de estado devuelto por el comando se puede utilizar para determinar si el puerto está abierto o cerrado.

```bash
#!/bin/bash

function ctrl_c() {
  echo -e "\n\n[!] Saliendo...\n\n"
  exec 3<&-
  exec 3>&-
  tput cnorm
  exit 1
}

trap ctrl_c SIGINT

declare -a ports=($(seq 1 65535))

function checkPort() {
  tput civis
  bash -c "exec 3<>/dev/tcp/$1/$2" 2>/dev/null

  if [ $? -eq 0 ]; then
    echo "[+] Host $1 - Port $2 - Active"
  fi

  exec 3<&-
  exec 3>&-

}

if [ $1 ]; then

  for port in ${ports[@]}; do
    checkPort $1 $port &
  done

else
  echo -e "\n[!] Uso: $0 <ip-address>\n"
fi

wait
```

---
## Descubrimiento de Equipos en la Red Local (ARP e ICMP) y Tips

El descubrimiento de equipos en la red local es una tarea fundamental en la gestión de redes y en las pruebas de seguridad. Existen diferentes herramientas y técnicas para realizar esta tarea, que van desde el escaneo de puertos hasta el análisis de tráfico de red.

En esta clase, nos enfocaremos en las técnicas de descubrimiento de equipos basadas en los protocolos **ARP** e **ICMP**. Además, se presentarán diferentes herramientas que pueden ser útiles para esta tarea, como:

[[Nmap Guía Completa|Nmap]]: `nmap -sn 192.168.0.0/24`
[[Arp Scan Herramienta de Descubrimiento|arp-scan]]: `arp-scan -I eth0 --localnet`
[[netdiscover]]: `netdiscover -i eth0`
[[Masscan]]: `masscan -p21,22,139,445 -Pn 192.168.0.0/16 --rate=10000`

Entre los modos de escaneo que se explican en la clase, se encuentra el uso del parámetro '**-sn**' de Nmap, que permite realizar un escaneo de hosts sin realizar el escaneo de puertos. También se presentan las herramientas netdiscover, arp-scan y masscan, que utilizan el protocolo ARP para descubrir hosts en la red.

> [!tip] El parámetro `-sn` de Nmap (anteriormente `-sP`) realiza un "ping scan" o "host discovery". No escanea puertos, sino que utiliza una combinación de solicitudes ICMP echo, paquetes TCP SYN a puertos comunes (como 443), y paquetes ARP (en redes locales) para determinar si un host está activo. Es ideal para un reconocimiento rápido de la red.

Cada herramienta tiene sus propias ventajas y limitaciones. Por ejemplo, **netdiscover** es una herramienta simple y fácil de usar, pero puede ser menos precisa que **arp-scan** o **masscan**. Por otro lado, **arp-scan** y **masscan** son herramientas más potentes, capaces de descubrir hosts más rápido y en redes más grandes, pero también son más complejas y pueden requerir más recursos.

En definitiva, el descubrimiento de equipos en la red local es una tarea fundamental para cualquier administrador de redes o profesional de seguridad de la información. Con las técnicas y herramientas adecuadas, es posible realizar esta tarea de manera efectiva y eficiente.

Una opción mediante un script manual puede ser el siguiente:

```bash
#!/bin/bash

# ctrl_c
function ctrl_c() {
  echo -e "\n\n[!] Saliendo.....\n\n"
  tput cnorm
  exit 1
}

tput civis
for i in $(seq 1 254); do
  timeout 1 bash -c "ping -c 1 192.168.1.$i" &>/dev/null && echo -e "Host - 192.168.1.$i - Active" &
done
wait
tput cnorm
```

Muchas ocasiones y en especial caso los dispositivos Windows tienen desactivados los paquetes de tipo ICMP que se tramitan mediante ping y para estos casos podemos usar el siguiente script el cual, en concepto, mediante algunos puertos conocidos, identifica que la máquina se encuentre activa.

> [!info] Los sistemas operativos Windows, por defecto, suelen tener el firewall activado y configurado para bloquear las solicitudes ICMP echo (ping) entrantes. Esto se hace para reducir la superficie de ataque y evitar que los atacantes identifiquen fácilmente hosts activos en la red. Por ello, es común recurrir a escaneos de puertos TCP/UDP para el descubrimiento de hosts en estos entornos.

```bash
#!/bin/bash

# ctrl_c
function ctrl_c() {
  echo -e "\n\n[!] Saliendo.....\n\n"
  tput cnorm
  exit 1
}

tput civis
for i in $(seq 1 254); do
  for port in 21 22 23 25 139 445 8080 80; do
    timeout 1 bash -c "echo '' > /dev/tcp/192.168.1.$i/$port" 2>/dev/null && echo -e "Host - 192.168.1.$i - Active" &
  done
done
wait
tput cnorm
```

---
## Validación del Objetivo (Fijando un Target en HackerOne)

En esta clase exploraremos la plataforma **HackerOne**, una plataforma de **Bug Bounty** que permite a empresas y organizaciones que desean ser auditadas, "conectar" con hackers éticos para encontrar vulnerabilidades de seguridad en sus sistemas y aplicaciones de forma legal.

Antes de iniciar una auditoría en esta plataforma, es fundamental fijar un objetivo claro, además de definir el alcance de la auditoría. Esto se logra a través del concepto de "**Scope**", que establece los límites de la auditoría, así como los sistemas y aplicaciones que pueden ser auditados.

En esta clase, se explicará cómo validar un objetivo en HackerOne y cómo definir el alcance de la auditoría a través del Scope. Además, se discutirán los impedimentos y limitaciones que se pueden encontrar durante la fase de auditoría, evitando así posibles malentendidos durante el proceso de reporte de vulnerabilidades.

> [!info] El "Scope" en un programa de Bug Bounty es crítico. Define explícitamente qué activos (dominios, IPs, aplicaciones, funcionalidades) están permitidos para la prueba y cuáles están fuera de los límites. Ignorar el scope puede llevar a acciones legales o la expulsión de la plataforma. También detalla qué tipos de vulnerabilidades son elegibles para recompensa.

*   **Enlace a la web de HackerOne**: [https://www.hackerone.com/](https://www.hackerone.com/)

## Descubrimiento de Correos Electrónicos

En esta clase exploraremos la importancia de la recolección de información en la fase de **OSINT** durante una auditoría, en particular, la recolección de **correos electrónicos**. Los correos electrónicos pueden ser una valiosa fuente de información para la vulneración de posibles paneles de autenticación y la realización de campañas de **phishing**.

Durante la clase se presentan diferentes herramientas online que pueden ayudar en este proceso. Por ejemplo, se explica cómo usar '**hunter.io**' para buscar correos electrónicos asociados a un dominio en particular. También se muestra cómo utilizar '**intelx.io**' para buscar información relacionada con direcciones de correo electrónico, nombres de usuarios y otros detalles.

Otra herramienta interesante que se presenta en la clase es '**phonebook.cz**', que permite buscar correos electrónicos y otros datos de contacto relacionados con empresas de todo el mundo.

Finalmente, se habla sobre el plugin '**Clearbit Connect**' para Gmail, que permite obtener información de contacto en tiempo real y añadirla directamente a los contactos de Gmail.

> [!tip] La recolección de correos electrónicos es una parte fundamental del OSINT. Estos pueden ser utilizados para ataques de ingeniería social, como phishing o spear-phishing, para obtener credenciales o información sensible. También pueden revelar patrones de nomenclatura de usuarios que son útiles para ataques de fuerza bruta o enumeración de usuarios.

A continuación, se proporcionan los enlaces a las herramientas online vistas en esta clase:

*   Hunter: [https://hunter.io/](https://hunter.io/)
*   Intelligence X: [https://intelx.io/](https://intelx.io/)
*   Phonebook.cz: [https://phonebook.cz/](https://phonebook.cz/)
*   Clearbit Connect: [Chrome Extension](https://chrome.google.com/webstore/detail/clearbit-connect-free-ver/pmnhcgfcafcnkbengdcanjablaabjplo)

En conclusión, la recolección de correos electrónicos es una tarea importante en la fase inicial de OSINT y puede proporcionar información valiosa. Sin embargo, es importante tener en cuenta que la recolección de correos electrónicos por sí sola no permite identificar directamente posibles vulnerabilidades en una red o sistema.

---
## Reconocimiento de Imágenes

En esta clase, exploraremos cómo las tecnologías de reconocimiento de imágenes pueden ser utilizadas para obtener información valiosa sobre las personas y los lugares.

Una de las herramientas en línea que vemos en esta clase es '**PimEyes**'. PimEyes es una plataforma en línea que utiliza tecnología de **reconocimiento facial** para buscar imágenes similares en Internet en función de una imagen que se le proporciona como entrada. Esta herramienta puede ser útil en la detección de información personal de una persona, como sus perfiles en redes sociales, direcciones de correo electrónico, números de teléfono, nombres y apellidos, etc.

El funcionamiento de PimEyes se basa en el análisis de **patrones faciales**, que son comparados con una base de datos de imágenes en línea para encontrar similitudes. La plataforma también permite buscar imágenes de personas que aparecen en una foto en particular, lo que puede ser útil en la investigación de casos de acoso o en la búsqueda de personas desaparecidas.

> [!warning] El reconocimiento facial y la búsqueda inversa de imágenes, como la que ofrece PimEyes, plantean serias preocupaciones éticas y de privacidad. Aunque puede ser una herramienta poderosa para OSINT, su uso debe ser estrictamente limitado a fines legales y éticos, como la investigación de personas desaparecidas o la verificación de identidad con consentimiento. El uso indebido puede violar la privacidad de las personas.

*   **Enlace a la web de PimEyes**: [https://pimeyes.com/en](https://pimeyes.com/en)

---
## Enumeración de Subdominios

La enumeración de **subdominios** es una de las fases cruciales en la seguridad informática para identificar los subdominios asociados a un dominio principal.

Los subdominios son parte de un dominio más grande y a menudo están configurados para apuntar a diferentes recursos de la red, como servidores web, servidores de correo electrónico, sistemas de bases de datos, sistemas de gestión de contenido, entre otros.

Al identificar los subdominios vinculados a un dominio principal, un atacante podría obtener información valiosa para cada uno de estos, lo que le podría llevar a encontrar **vectores de ataque potenciales**. Por ejemplo, si se identifica un subdominio que apunta a un servidor web vulnerable, el atacante podría utilizar esta información para intentar explotar la vulnerabilidad y acceder al servidor en cuestión.

> [!info] Los subdominios a menudo albergan aplicaciones menos seguras, versiones de desarrollo, o servicios internos que no están tan protegidos como el dominio principal. Su descubrimiento es vital porque pueden ofrecer puntos de entrada inesperados a la infraestructura de una organización.

Existen diferentes herramientas y técnicas para la enumeración de subdominios, tanto pasivas como activas. Las **herramientas pasivas** permiten obtener información sobre los subdominios sin enviar ninguna solicitud a los servidores identificados, mientras que las **herramientas activas** envían solicitudes a los servidores identificados para encontrar subdominios bajo el dominio principal.

Algunas de las **herramientas pasivas** más utilizadas para la enumeración de subdominios incluyen la búsqueda en motores de búsqueda como Google, Bing o Yahoo, y la búsqueda en registros DNS públicos como **PassiveTotal** o **Censys**. Estas herramientas permiten identificar subdominios asociados con un dominio, aunque no siempre son exhaustivas. Además, existen herramientas como **CTFR** que utilizan registros de certificados **SSL/TLS** para encontrar subdominios asociados a un dominio.

También se pueden utilizar páginas online como **Phonebook.cz** e **Intelx.io**, o herramientas como **sublist3r**, para buscar información relacionada con los dominios, incluyendo subdominios.

Por otro lado, las **herramientas activas** para la enumeración de subdominios incluyen herramientas de fuzzing como **wfuzz** o **gobuster**. Estas herramientas envían solicitudes a los servidores mediante ataques de fuerza bruta, con el objetivo de encontrar subdominios válidos bajo el dominio principal.

A continuación, os adjuntamos los enlaces a las herramientas vistas en esta clase:

*   **Phonebook** (Herramienta pasiva): [https://phonebook.cz/](https://phonebook.cz/)
*   **Intelx** (Herramienta pasiva): [https://intelx.io/](https://intelx.io/)
*   **CTFR** (Herramienta pasiva): [https://github.com/UnaPibaGeek/ctfr](https://github.com/UnaPibaGeek/ctfr)
*   **Gobuster** (Herramienta activa): [https://github.com/OJ/gobuster](https://github.com/OJ/gobuster) [[Gobuster]]
*   **Wfuzz** (Herramienta activa): [https://github.com/xmendez/wfuzz](https://github.com/xmendez/wfuzz) [[Wfuzz]]
*   **Sublist3r** (Herramienta pasiva): [https://github.com/huntergregal/Sublist3r](https://github.com/huntergregal/Sublist3r)

---
## Credenciales y Brechas de Seguridad

La seguridad de la información es un tema crítico en el mundo digital actual, especialmente cuando se trata de datos sensibles como **contraseñas**, **información financiera** o de **identidad**. Los ataques informáticos son una amenaza constante para cualquier empresa u organización, y una de las principales técnicas utilizadas por los atacantes es la **explotación de las credenciales** y **brechas de seguridad**.

Una de las formas más comunes en que los atacantes aprovechan las brechas de seguridad es mediante el uso de leaks de bases de datos. Estos leaks pueden ser el resultado de errores de configuración, vulnerabilidades en el software o ataques malintencionados. Cuando una base de datos se ve comprometida, los atacantes pueden acceder a una gran cantidad de información sensible, como nombres de usuario, contraseñas y otra información personal.

Una vez que los atacantes tienen acceso a esta información, pueden utilizarla para realizar ataques de fuerza bruta, phishing y otros ataques de ingeniería social para acceder a sistemas y cuentas protegidas. En algunos casos, los atacantes pueden incluso vender esta información en el **mercado negro** para que otros atacantes la utilicen.

Es importante entender que muchas de estas bases de datos filtradas y vendidas en línea **son accesibles públicamente** y en algunos casos, incluso se venden por una pequeña cantidad de dinero. Esto significa que cualquier persona puede acceder a esta información y utilizarla para llevar a cabo ataques malintencionados.

> [!warning] El acceso a credenciales filtradas debe manejarse con extrema precaución y ética. Su uso está justificado únicamente en el contexto de pruebas de penetración autorizadas (para verificar si las credenciales de la organización han sido comprometidas) o en investigaciones forenses. El uso no autorizado de estas credenciales es ilegal y puede tener graves consecuencias.

A continuación, se proporciona el enlace a la utilidad online de ejemplo que se muestra en esta clase:

*   **DeHashed**: [https://www.dehashed.com/](https://www.dehashed.com/)

---
## Identificación de las Tecnologías en una Página Web

Desde el punto de vista de la seguridad, es fundamental conocer las **tecnologías** y **herramientas** que se utilizan en una página web. La identificación de estas tecnologías permite a los expertos en seguridad evaluar los riesgos potenciales de un sitio web, identificar vulnerabilidades y diseñar estrategias efectivas para proteger la información sensible y los datos críticos.

Existen diversas herramientas y utilidades en línea que permiten identificar las tecnologías utilizadas en una página web. Algunas de las herramientas más populares incluyen **Whatweb**, **Wappalyzer** y **builtwith.com**. Estas herramientas escanean la página web y proporcionan información detallada sobre las tecnologías utilizadas, como el lenguaje de programación, el servidor web, los sistemas de gestión de contenido, entre otros.

La herramienta **whatweb** es una utilidad de análisis de vulnerabilidades que escanea la página web y proporciona información detallada sobre las tecnologías utilizadas. Esta herramienta también puede utilizarse para identificar posibles vulnerabilidades y puntos débiles en la página web.

**Wappalyzer**, por otro lado, es una extensión del navegador que detecta y muestra las tecnologías utilizadas en la página web. Esta herramienta es especialmente útil para los expertos en seguridad que desean identificar rápidamente las tecnologías utilizadas en una página web sin tener que realizar un escaneo completo.

**Builtwith.com** es una herramienta en línea que también permite identificar las tecnologías utilizadas en una página web. Esta herramienta proporciona información detallada sobre las tecnologías utilizadas, así como también estadísticas útiles como el tráfico y la popularidad de la página web.

> [!tip] La identificación de tecnologías web es el primer paso en el reconocimiento de aplicaciones web. Conocer el CMS (WordPress, Joomla), el servidor web (Apache, Nginx), el lenguaje de programación (PHP, Python), y las librerías JavaScript (jQuery, React) permite al atacante enfocar sus esfuerzos en vulnerabilidades conocidas para esas tecnologías específicas.

A continuación, os proporcionamos los enlaces correspondientes a las herramientas vistas en esta clase:

*   **Whatweb**: [https://github.com/urbanadventurer/WhatWeb](https://github.com/urbanadventurer/WhatWeb) [[WhatWeb Herramienta de Fingerprinting|WhatWeb, Fingerprinting Web]]
*   **Wappalyzer**: [https://addons.mozilla.org/es/firefox/addon/wappalyzer/](https://addons.mozilla.org/es/firefox/addon/wappalyzer/)
*   **Builtwith**: [https://builtwith.com/](https://builtwith.com/)

---
## Fuzzing y Enumeración de Archivos en un Servidor Web

En esta clase, hacemos uso de las herramientas **Wfuzz** y **Gobuster** para aplicar **Fuzzing**. Esta técnica se utiliza para descubrir rutas y recursos ocultos en un servidor web mediante ataques de fuerza bruta. El objetivo es encontrar recursos ocultos que podrían ser utilizados por atacantes malintencionados para obtener acceso no autorizado al servidor.

> [!info] El fuzzing es una técnica de prueba de software que implica inyectar datos semi-aleatorios o inesperados (fuzz) en una aplicación para descubrir errores, fallos o vulnerabilidades. En el contexto web, se utiliza para descubrir directorios, archivos, parámetros o endpoints ocultos que no están enlazados públicamente pero que existen en el servidor.

**Wfuzz** es una herramienta de descubrimiento de contenido y una herramienta de inyección de datos. Básicamente, se utiliza para automatizar los procesos de prueba de vulnerabilidades en aplicaciones web.

Permite realizar ataques de fuerza bruta en parámetros y directorios de una aplicación web para identificar recursos existentes. Una de las **ventajas** de Wfuzz es que es altamente personalizable y se puede ajustar a diferentes necesidades de pruebas. Algunas de las **desventajas** de Wfuzz incluyen la necesidad de comprender la sintaxis de sus comandos y que puede ser más lenta en comparación con otras herramientas de descubrimiento de contenido.

Por otro lado, **Gobuster** es una herramienta de descubrimiento de contenido que también se utiliza para buscar archivos y directorios ocultos en una aplicación web. Al igual que Wfuzz, Gobuster se basa en ataques de fuerza bruta para encontrar archivos y directorios ocultos. Una de las principales **ventajas** de Gobuster es su velocidad, ya que es conocida por ser una de las herramientas de descubrimiento de contenido más rápidas. También es fácil de usar y su sintaxis es simple. Sin embargo, una **desventaja** de Gobuster es que puede no ser tan personalizable como Wfuzz.

> [!tip] La elección entre Wfuzz y Gobuster a menudo depende de la situación. Gobuster es excelente para escaneos rápidos de directorios y subdominios debido a su velocidad. Wfuzz, con su mayor flexibilidad y capacidad de personalización (ej. para fuzzing de parámetros HTTP, cabeceras, o payloads complejos), es preferible para escenarios más complejos o cuando se necesita un control granular sobre las solicitudes.

En resumen, tanto Wfuzz como Gobuster son herramientas útiles para pruebas de vulnerabilidades en aplicaciones web, pero tienen diferencias en su enfoque y características. La elección de una u otra dependerá de tus necesidades y preferencias personales.

A continuación, te proporcionamos el enlace a estas herramientas:

*   **Wfuzz**: [https://github.com/xmendez/wfuzz](https://github.com/xmendez/wfuzz) [[Wfuzz]]
*   **Gobuster**: [https://github.com/OJ/gobuster](https://github.com/OJ/gobuster) [[Gobuster]]

Adicionalmente, otra de las herramientas que examinamos en esta clase, perfecta para la enumeración de recursos disponibles en una plataforma en línea, es **Burp Suite**. Burp Suite es una plataforma que integra características especializadas para realizar pruebas de penetración en aplicaciones web. Una de sus particularidades es la función de **análisis de páginas en línea**, empleada para identificar y enumerar los recursos accesibles en una página web.

Burp Suite cuenta con dos versiones: una **versión gratuita** (**Burp Suite Community Edition**) y una **versión de pago** (**Burp Suite Professional**).

### Burp Suite Community Edition

Es la **versión gratuita** de esta plataforma, viene incluida por defecto en el sistema operativo. Su función principal es desempeñar el papel de **proxy HTTP** para la aplicación, facilitando la realización de pruebas de penetración.

Un **proxy HTTP** es un filtro de contenido de alto rendimiento, ampliamente usado en el hacking con el fin de interceptar el tráfico de red. Esto permite analizar, modificar, aceptar o rechazar todas las solicitudes y respuestas de la aplicación que se esté auditando.

Algunas de las ventajas que la versión gratuita ofrece son:

*   **Gratuidad**: La versión Community Edition es gratuita, lo que la convierte en una opción accesible para principiantes y profesionales con presupuestos limitados.
*   **Herramientas básicas**: Incluye las herramientas esenciales para realizar pruebas de penetración en aplicaciones web, como el Proxy, el Repeater y el Sequencer.
*   **Intercepción y modificación de tráfico**: Permite interceptar y modificar las solicitudes y respuestas HTTP/HTTPS, facilitando la identificación de vulnerabilidades y la exploración de posibles ataques.
*   **Facilidad de uso**: La interfaz de usuario de la Community Edition es intuitiva y fácil de utilizar, lo que facilita su adopción por parte de usuarios con diversos niveles de experiencia.
*   **Aprendizaje y familiarización**: La versión gratuita permite a los usuarios aprender y familiarizarse con las funcionalidades y técnicas de pruebas de penetración antes de dar el salto a la versión Professional.
*   **Comunidad de usuarios**: La versión Community Edition cuenta con una amplia comunidad de usuarios que comparten sus conocimientos y experiencias en foros y blogs, lo que puede ser de gran ayuda para resolver problemas y aprender nuevas técnicas.

A pesar de que la Community Edition no ofrece todas las funcionalidades y ventajas de la versión Professional, sigue siendo una opción valiosa para aquellos que buscan comenzar en el ámbito de las pruebas de penetración o que necesitan realizar análisis de seguridad básicos sin incurrir en costos adicionales.

### Burp Suite Professional

Burp Suite Professional es la **versión de pago** desarrollada por la empresa **PortSwigger**. Incluye, además del proxy HTTP, algunas herramientas de pentesting web como:

*   **Escáner de seguridad automatizado**: Permite identificar vulnerabilidades en aplicaciones web de manera rápida y eficiente, lo que ahorra tiempo y esfuerzo.
*   **Integración con otras herramientas**: Puede integrarse con otras soluciones de seguridad y entornos de desarrollo para mejorar la eficacia de las pruebas.
*   **Extensibilidad**: A través de su API, Burp Suite Professional permite a los usuarios crear y añadir extensiones personalizadas para adaptarse a necesidades específicas.
*   **Actualizaciones frecuentes**: La versión profesional recibe actualizaciones periódicas que incluyen nuevas funcionalidades y mejoras de rendimiento.
*   **Soporte técnico**: Los usuarios de Burp Suite Professional tienen acceso a un soporte técnico de calidad para resolver dudas y problemas.
*   **Informes personalizables**: La herramienta permite generar informes detallados y personalizados sobre las pruebas de penetración y los resultados obtenidos.
*   **Interfaz de usuario intuitiva**: La interfaz de Burp Suite Professional es fácil de utilizar y permite a los profesionales de seguridad trabajar de manera eficiente.
*   **Herramientas avanzadas**: Incluye funcionalidades avanzadas, como el módulo de intrusión, el rastreador de vulnerabilidades y el generador de payloads, que facilitan la identificación y explotación de vulnerabilidades en aplicaciones web.

En conclusión, tanto la Community Edition como la versión Professional de Burp Suite ofrecen un conjunto de herramientas útiles y eficientes para realizar pruebas de penetración en aplicaciones web. Sin embargo, la versión Professional brinda ventajas adicionales.

---
## Google Dorks / Google Hacking (Los 18 Dorks más Usados)

El '**Google Dork**' es una técnica de búsqueda avanzada que utiliza operadores y palabras clave específicas en el buscador de Google para encontrar información que normalmente no aparece en los resultados de búsqueda regulares.

La técnica de '**Google Dorking**' se utiliza a menudo en el hacking para encontrar información sensible y crítica en línea. Es una forma eficaz de recopilar información valiosa de una organización o individuo que puede ser utilizada para realizar pruebas de penetración y otros fines de seguridad.

Al utilizar Google Dorks, un atacante puede buscar información como nombres de usuarios y contraseñas, archivos confidenciales, información de bases de datos, números de tarjetas de crédito y otra información crítica. También pueden utilizar esta técnica para identificar vulnerabilidades en aplicaciones web, sitios web y otros sistemas en línea.

Es importante tener en cuenta que la técnica de Google Dorking **no es ilegal en sí misma**, pero puede ser utilizada con fines maliciosos. Por lo tanto, es crucial utilizar esta técnica con responsabilidad y ética en el contexto de la seguridad informática y el hacking ético.

> [!warning] El Google Dorking es una herramienta poderosa para la fase de reconocimiento, pero su uso debe ser ético y legal. Buscar y acceder a información sensible sin autorización puede tener graves consecuencias legales. Siempre asegúrate de tener el permiso explícito para realizar este tipo de búsquedas en el contexto de una auditoría de seguridad.

## Tabla de Operadores de Búsqueda Avanzada

### Operadores Básicos

| Operador | Descripción | Ejemplo |
|---|---|---|
| `""` | Búsqueda exacta | `"palabra exacta"` |
| `-` | Excluir término | `wordpress -wp-content` |
| `OR` | Uno u otro término | `wordpress OR joomla` |
| `AND` | Ambos términos | `wordpress AND vulnerability` |
| `*` | Comodín | `admin * login` |
| `()` | Agrupación | `(wordpress OR drupal) login` |

### Operadores Avanzados

| Operador | Descripción | Ejemplo |
|---|---|---|
| `site:` | Limita a un sitio específico | `site:example.com` |
| `filetype:` | Busca por tipo de archivo | `filetype:pdf` |
| `ext:` | Extensión de archivo | `ext:php` |
| `inurl:` | Término en la URL | `inurl:admin` |
| `intitle:` | Término en el título | `intitle:"index of"` |
| `intext:` | Término en el texto | `intext:password` |
| `cache:` | Versión en caché | `cache:example.com` |
| `link:` | Páginas que enlazan a | `link:example.com` |
| `related:` | Sitios relacionados | `related:example.com` |
| `info:` | Información del sitio | `info:example.com` |
| `define:` | Definición | `define:hacking` |
| `stocks:` | Información bursátil | `stocks:GOOG` |

### Operadores de Fecha y Números

| Operador | Descripción | Ejemplo |
|---|---|---|
| `..` | Rango numérico | `price 100..500` |
| `before:` | Antes de fecha | `before:2020-01-01` |
| `after:` | Después de fecha | `after:2023-01-01` |
| `daterange:` | Rango de fechas | `daterange:2458150-2458155` |

---
## Identificación y Verificación Externa de la Versión del Sistema Operativo

El tiempo de vida (**TTL**) hace referencia a la cantidad de tiempo o "**saltos**" que se ha establecido que un paquete debe existir dentro de una red antes de ser descartado por un enrutador. El TTL también se utiliza en otros contextos, como el almacenamiento en caché de CDN y el almacenamiento en caché de DNS.

Cuando se crea un paquete de información y se envía a través de Internet, existe el riesgo de que siga pasando de enrutador a enrutador indefinidamente. Para mitigar esta posibilidad, los paquetes se diseñan con una caducidad denominada **tiempo de vida** o **límite de saltos**. El TTL de los paquetes también puede ser útil para determinar cuánto tiempo ha estado en circulación un paquete determinado, y permite que el remitente pueda recibir información sobre la trayectoria de un paquete a través de Internet.

Cada paquete tiene un lugar en el que se almacena un valor numérico que determina cuánto tiempo debe seguir moviéndose por la red. Cada vez que un enrutador recibe un paquete, resta uno al recuento de TTL y lo pasa al siguiente lugar de la red. Si en algún momento el recuento de TTL llega a cero después de la resta, el enrutador descartará el paquete y enviará un mensaje ICMP al host de origen.

¿Qué tiene que ver esto con la identificación del sistema operativo? Bueno, resulta que diferentes sistemas operativos tienen diferentes valores predeterminados de TTL. Por ejemplo, en sistemas operativos Windows, el valor predeterminado de TTL es 128, mientras que en sistemas operativos Linux es 64.

Por lo tanto, si enviamos un paquete a una máquina y recibimos una respuesta que tiene un valor TTL de **128**, es probable que la máquina esté ejecutando **Windows**. Si recibimos una respuesta con un valor TTL de **64**, es más probable que la máquina esté ejecutando **Linux**.

> [!info] Los valores de TTL predeterminados son una pista útil, pero no una prueba definitiva del sistema operativo. Los administradores de red pueden modificar el TTL de los paquetes salientes por diversas razones (ej. optimización de red, seguridad). Además, los dispositivos intermedios (routers, firewalls) decrementan el TTL, por lo que el valor recibido puede ser menor que el valor inicial del sistema operativo objetivo.

Este método **no es infalible** y puede ser engañado por los administradores de red, pero puede ser útil en ciertas situaciones para identificar el sistema operativo de una máquina.

A continuación, se os comparte la página que mostramos en esta clase para identificar el sistema operativo correspondiente a los diferentes valores de TTL existentes.

*   **Subin’s Blog**: [https://subinsb.com/default-device-ttl-values/](https://subinsb.com/default-device-ttl-values/)

Asimismo, os compartimos el script de Python encargado de identificar el sistema operativo en función del TTL obtenido:

```python
#!/usr/bin/python3
#coding: utf-8

import re, sys, subprocess

# python3 whichSystem.py 10.10.10.188 

if len(sys.argv) != 2:
    print("\n[!] Uso: python3 " + sys.argv[0] + " <direccion-ip>\n")
    sys.exit(1)

def get_ttl(ip_address):

    proc = subprocess.Popen(["/usr/bin/ping -c 1 %s" % ip_address, ""], stdout=subprocess.PIPE, shell=True)
    (out,err) = proc.communicate()

    out = out.split()
    out = out[12].decode('utf-8')

    ttl_value = re.findall(r"\d{1,3}", out)[0]

    return ttl_value

def get_os(ttl):

    ttl = int(ttl)

    if ttl >= 0 and ttl <= 64:
        return "Linux"
    elif ttl >= 65 and ttl <= 128:
        return "Windows"
    else:
        return "Not Found"

if __name__ == '__main__':

    ip_address = sys.argv[1]

    ttl = get_ttl(ip_address)

    os_name = get_os(ttl)
    print("\n%s (ttl -> %s): %s\n" % (ip_address, ttl, os_name))
```
