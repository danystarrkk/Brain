---
aliases:
  - Explotación de Vulnerabilidades
  - Shells en Ciberseguridad
  - Payloads Staged y Non-Staged
  - Burp Suite
  - Enumeración de Sistemas Linux
tags:
  - hacking/explotacion
  - herramientas/burpsuite
  - linux/seguridad
  - scripting/bash
  - redes/conceptos
estado: 🟢 Terminado
---
# Introducción a la Explotación de Vulnerabilidades

Una vez completada la fase de reconocimiento inicial, podemos proceder con la fase de explotación. Sin embargo, es crucial comprender algunos conceptos fundamentales antes de iniciar la explotación de vulnerabilidades.

A lo largo de las siguientes secciones, exploraremos diferentes tipos de shells (como las reverse shells, bind shells y forward shells), las cuales nos permitirán establecer conexiones de red y tomar control de un sistema comprometido. Hablaremos sobre los diferentes tipos de payloads (staged y non-staged) y cómo se utilizan para ejecutar código malicioso en el sistema objetivo.

Además, se discutirá la diferencia entre las explotaciones manuales y automatizadas, presentando herramientas que pueden ser utilizadas para automatizar el proceso de explotación de vulnerabilidades.

Por último, se introducirá la herramienta **Burp Suite**, una suite de herramientas para realizar pruebas de penetración y análisis de vulnerabilidades en aplicaciones web.

> [!info] Burp Suite: La navaja suiza del pentesting web
> Burp Suite es una plataforma integrada para realizar pruebas de seguridad en aplicaciones web. Ofrece herramientas para mapear, analizar, encontrar y explotar vulnerabilidades. Es esencial para interceptar, modificar y reenviar tráfico HTTP/S, lo que permite un control granular sobre las interacciones con la aplicación objetivo.

## Reverse Shell, Bind Shell y Forward Shells

En esta sección, veremos las diferencias entre Reverse Shell, Bind Shell y Forward Shell:

- **Reverse Shell**: Es una técnica que permite a un atacante establecer una conexión desde la máquina comprometida (víctima) hacia la máquina del atacante (servidor de escucha). Esto se logra ejecutando un programa malicioso o una instrucción específica en la máquina remota que inicia la conexión de vuelta, permitiendo al atacante tomar el control.

> [!tip] Ventaja de la Reverse Shell
> Las Reverse Shells son particularmente útiles en escenarios donde la máquina objetivo se encuentra detrás de un firewall o NAT que bloquea las conexiones entrantes directas. Al iniciar la conexión desde la víctima, se aprovechan las reglas de salida que suelen ser más permisivas.

- **Bind Shell**: Esta técnica es el opuesto de la Reverse Shell, ya que en lugar de que la máquina comprometida se conecte a la máquina del atacante, es el atacante quien se conecta a la máquina comprometida. El atacante escucha en un puerto determinado y la máquina comprometida acepta la conexión entrante en ese puerto. El atacante luego tiene acceso por consola a la máquina comprometida, lo que le permite tomar el control de la misma.

> [!warning] Consideraciones de la Bind Shell
> Las Bind Shells requieren que la máquina objetivo tenga un puerto abierto y accesible desde la red del atacante. Esto a menudo implica que el firewall de la víctima debe permitir conexiones entrantes a ese puerto, lo que las hace menos efectivas contra sistemas bien protegidos por firewalls perimetrales.

- **Forward Shell**: Esta técnica se utiliza cuando no se pueden establecer conexiones Reverse o Bind debido a reglas de firewall implementadas en la red. Se logra mediante el uso de **mkfifo**, que crea un archivo **FIFO** (**named pipe**), que se utiliza como una especie de “**consola simulada**” interactiva a través de la cual el atacante puede operar en la máquina remota. En lugar de establecer una conexión directa, el atacante redirige el tráfico a través del archivo **FIFO**, lo que permite la comunicación bidireccional con la máquina remota.

> [!info] Mecanismo de la Forward Shell con `mkfifo`
> `mkfifo` crea un "named pipe" o FIFO (First-In, First-Out) en el sistema de archivos. A diferencia de los pipes anónimos, los named pipes persisten en el sistema de archivos y pueden ser abiertos por múltiples procesos. En una Forward Shell, un proceso escribe en el FIFO y otro lee de él, simulando una comunicación interactiva bidireccional a través de un archivo, eludiendo restricciones de red directas.

Es importante entender las diferencias entre estas técnicas para poder determinar cuál es la mejor opción en función del escenario de ataque y las limitaciones de la red.

## Tipos de Payloads (Staged y Non-Staged)

- **Payload Staged**: Es un tipo de payload que se **divide en dos o más etapas**. La primera etapa es una pequeña porción de código (conocida como *stager*) que se envía al objetivo. Su propósito principal es establecer una conexión de red fiable y segura entre el atacante y la máquina objetivo. Una vez que se establece la conexión, el atacante envía la segunda etapa del payload, que es la carga útil real del ataque. Este enfoque permite a los atacantes sortear medidas de seguridad adicionales, ya que la carga útil real no se envía hasta que se establece una conexión segura.

> [!tip] Eficiencia y Evasión con Payloads Staged
> Los payloads Staged son más eficientes en términos de tamaño inicial, lo que reduce la probabilidad de detección por sistemas de seguridad basados en firmas. Además, permiten al atacante adaptar la carga útil final (segunda etapa) en función de la información recopilada durante la primera etapa, ofreciendo mayor flexibilidad y evasión.

- **Payload Non-Staged**: Es un tipo de payload que se envía como **una sola entidad** y **no se divide en múltiples etapas**. La carga útil completa se envía al objetivo en un solo paquete y se ejecuta inmediatamente después de ser recibida. Este enfoque es más simple que el Payload Staged, pero también es más fácil de detectar por los sistemas de seguridad, ya que se envía todo el código malicioso de una sola vez.

> [!warning] Riesgos de Detección con Payloads Non-Staged
> Aunque más simples de implementar, los payloads Non-Staged envían toda la carga útil maliciosa de una sola vez. Esto los hace más susceptibles a ser detectados por sistemas de detección de intrusiones (IDS/IPS) o antivirus que analizan el tamaño y el contenido completo del payload.

Es importante tener en cuenta que el tipo de payload utilizado en un ataque dependerá del objetivo y de las medidas de seguridad implementadas. En general, los payloads Staged son más difíciles de detectar y son preferidos por los atacantes, mientras que los payloads Non-Staged son más fáciles de implementar pero también son más fáciles de detectar.

## Tipos de Explotación (Manuales y Automatizadas)

- **Explotación Manual**: Es un tipo de explotación que se realiza de **manera manual** y requiere que el atacante tenga un conocimiento profundo del sistema y sus vulnerabilidades. En este enfoque, el atacante emplea herramientas y técnicas específicas para identificar, analizar y explotar vulnerabilidades en un sistema objetivo, a menudo requiriendo un entendimiento profundo de la lógica de la aplicación o del sistema operativo. Este enfoque es más lento y requiere más esfuerzo y habilidad por parte del atacante, pero también es más preciso y permite un mayor control sobre el proceso de explotación.

> [!info] Precisión y Discreción de la Explotación Manual
> La explotación manual permite una mayor precisión y un menor "ruido" en la red, lo que reduce las posibilidades de detección. Es ideal para vulnerabilidades complejas o cuando se busca un acceso sigiloso y persistente, ya que el atacante puede adaptar cada paso del proceso.

- **Explotación Automatizada**: Es un tipo de explotación que se realiza **automáticamente** mediante el uso de **herramientas automatizadas**, como scripts o programas diseñados específicamente para identificar y explotar vulnerabilidades en un sistema objetivo. Este enfoque es más rápido y menos laborioso que el enfoque manual, pero también puede ser menos preciso y puede generar más ruido en la red objetivo, lo que aumenta el riesgo de detección.

> [!tip] Velocidad y Escalabilidad de la Explotación Automatizada
> La explotación automatizada es invaluable para escanear y explotar rápidamente un gran número de sistemas o para identificar vulnerabilidades conocidas en entornos amplios. Herramientas como Metasploit Framework o Nessus son ejemplos clave que facilitan este tipo de explotación.

Es importante tener en cuenta que el tipo de explotación utilizado en un ataque dependerá de los objetivos del atacante, sus habilidades y del nivel de seguridad implementado en el sistema objetivo. En general, los ataques de explotación manual son más precisos y discretos, pero también requieren más tiempo y habilidades. Por otro lado, los ataques de explotación automatizada son más rápidos y menos laboriosos, pero también pueden ser más ruidosos y menos precisos.

## Enumeración del Sistema

Algunas de las herramientas que vemos en esta sección son:

- **LSE** (**Linux Smart Enumeration**): Es una herramienta de enumeración para sistemas Linux que permite a los atacantes obtener información detallada sobre la configuración del sistema, los servicios en ejecución, los permisos de archivo y posibles vectores de escalada de privilegios. LSE utiliza una variedad de comandos de Linux para recopilar información y presentarla en un formato fácil de entender. Al utilizar LSE, los atacantes pueden detectar posibles vulnerabilidades y encontrar información valiosa para futuros ataques.

> [!info] Capacidades de LSE
> LSE automatiza la recolección de información crítica para la escalada de privilegios en Linux. Examina configuraciones de red, archivos de configuración sensibles, permisos de SUID/SGID, tareas cron, servicios en ejecución y más, presentando los resultados de manera organizada para facilitar el análisis del atacante.

- **Pspy**: Es una herramienta de enumeración de procesos que permite a los atacantes observar los procesos y comandos que se ejecutan en el sistema objetivo a intervalos regulares de tiempo. Pspy es una herramienta útil para la detección de malware y backdoors, así como para la identificación de procesos maliciosos que se ejecutan en segundo plano sin la interacción del usuario.

> [!tip] Uso de Pspy para la Detección de Actividad
> Pspy es extremadamente útil para observar procesos que se ejecutan periódicamente (como tareas cron) o procesos que se inician y terminan rápidamente, los cuales podrían pasar desapercibidos con herramientas de monitoreo estáticas. Permite identificar patrones de actividad y posibles backdoors o scripts ocultos.

Asimismo, desarrollaremos un script en Bash ideal para detectar tareas y comandos que se ejecutan en el sistema a intervalos regulares de tiempo, abusando para ello del comando ‘**ps -eo user,command**‘ que nos chivará todo lo que necesitamos.

> [!info] `ps -eo user,command`: Un comando fundamental
> El comando `ps -eo user,command` es una herramienta poderosa para listar todos los procesos en ejecución (`-e`) y mostrar el usuario propietario (`user`) junto con el comando completo que inició el proceso (`command`). Esto es crucial para identificar procesos sospechosos, servicios no autorizados o scripts que se ejecutan con privilegios elevados.

A continuación, se proporciona el enlace a estas herramientas:

- **Herramienta LSE**: [https://github.com/diego-treitos/linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration)
- **Herramienta PSPY**: [https://github.com/DominicBreuker/pspy](https://github.com/DominicBreuker/pspy)