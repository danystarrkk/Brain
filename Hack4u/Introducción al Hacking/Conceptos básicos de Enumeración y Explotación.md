----
## Introducción a la explotación de vulnerabilidades

Una vez aplicada la fase de reconocimiento inicial, ya podríamos proceder con la **fase de explotación**, pero es importante comprender algunos conceptos antes de comenzar con la explotación de vulnerabilidades.

A lo largo de las siguientes clases, exploraremos diferentes tipos de shells (como las reverse shells, bind shells y forward shells), las cuales nos permitirán establecer conexiones de red y tomar control de un sistema comprometido. Hablaremos sobre los diferentes tipos de payloads (staged y non-staged) y cómo se utilizan para ejecutar código malicioso en el sistema objetivo.

Además, se discutirá la diferencia entre las explotaciones manuales y automatizadas, presentando herramientas que pueden ser utilizadas para automatizar el proceso de explotación de vulnerabilidades.

Por último, se introducirá la herramienta **BurpSuite**, una suite de herramientas para realizar pruebas de penetración y análisis de vulnerabilidades en aplicaciones web.

---
## Reverse Shell, Bind Shell y Forward Shells

En esta clase, veremos las diferencias entre Reverse Shell, Bind Shell y Forward Shell:

- **Reverse Shell**: Es una técnica que permite a un atacante conectarse a una máquina remota desde una máquina de su propiedad. Es decir, se establece una conexión desde la máquina comprometida hacia la máquina del atacante. Esto se logra ejecutando un programa malicioso o una instrucción específica en la máquina remota que establece la conexión de vuelta hacia la máquina del atacante, permitiéndole tomar el control de la máquina remota.
- **Bind Shell**: Esta técnica es el opuesto de la Reverse Shell, ya que en lugar de que la máquina comprometida se conecte a la máquina del atacante, es el atacante quien se conecta a la máquina comprometida. El atacante escucha en un puerto determinado y la máquina comprometida acepta la conexión entrante en ese puerto. El atacante luego tiene acceso por consola a la máquina comprometida, lo que le permite tomar el control de la misma.
- **Forward Shell**: Esta técnica se utiliza cuando no se pueden establecer conexiones Reverse o Bind debido a reglas de Firewall implementadas en la red. Se logra mediante el uso de **mkfifo**, que crea un archivo **FIFO** (**named pipe**), que se utiliza como una especie de “**consola simulada**” interactiva a través de la cual el atacante puede operar en la máquina remota. En lugar de establecer una conexión directa, el atacante redirige el tráfico a través del archivo **FIFO**, lo que permite la comunicación bidireccional con la máquina remota.

Es importante entender las diferencias entre estas técnicas para poder determinar cuál es la mejor opción en función del escenario de ataque y las limitaciones de la red.

----
## Tipos de payloads ( Staged y Non-Staged )

- **Payload Staged**: Es un tipo de payload que se **divide en dos** **o más etapas**. La primera etapa es una pequeña parte del código que se envía al objetivo, cuyo propósito es establecer una conexión segura entre el atacante y la máquina objetivo. Una vez que se establece la conexión, el atacante envía la segunda etapa del payload, que es la carga útil real del ataque. Este enfoque permite a los atacantes sortear medidas de seguridad adicionales, ya que la carga útil real no se envía hasta que se establece una conexión segura.
- **Payload Non-Staged**: Es un tipo de payload que se envía como **una sola entidad** y **no se divide en múltiples etapas**. La carga útil completa se envía al objetivo en un solo paquete y se ejecuta inmediatamente después de ser recibida. Este enfoque es más simple que el Payload Staged, pero también es más fácil de detectar por los sistemas de seguridad, ya que se envía todo el código malicioso de una sola vez.

Es importante tener en cuenta que el tipo de payload utilizado en un ataque dependerá del objetivo y de las medidas de seguridad implementadas. En general, los payloads Staged son más difíciles de detectar y son preferidos por los atacantes, mientras que los payloads Non-Staged son más fáciles de implementar pero también son más fáciles de detectar.

---
## Tipos de explotación ( Manuales y Automatizadas )

- **Explotación Manual**: Es un tipo de explotación que se realiza de **manera manual** y requiere que el atacante tenga un conocimiento profundo del sistema y sus vulnerabilidades. En este enfoque, el atacante utiliza herramientas y técnicas específicas para identificar y explotar vulnerabilidades en un sistema objetivo. Este enfoque es más lento y requiere más esfuerzo y habilidad por parte del atacante, pero también es más preciso y permite un mayor control sobre el proceso de explotación.
- **Explotación Automatizada**: Es un tipo de explotación que se realiza **automáticamente** mediante el uso de **herramientas automatizadas**, como scripts o programas diseñados específicamente para identificar y explotar vulnerabilidades en un sistema objetivo. Este enfoque es más rápido y menos laborioso que el enfoque manual, pero también puede ser menos preciso y puede generar más ruido en la red objetivo, lo que aumenta el riesgo de detección.

Es importante tener en cuenta que el tipo de explotación utilizado en un ataque dependerá de los objetivos del atacante, sus habilidades y del nivel de seguridad implementado en el sistema objetivo. En general, los ataques de explotación manual son más precisos y discretos, pero también requieren más tiempo y habilidades. Por otro lado, los ataques de explotación automatizada son más rápidos y menos laboriosos, pero también pueden ser más ruidosos y menos precisos.

-----
## Enumeración del sistema

Algunas de las herramientas que vemos en esta clase son:

- **LSE** (**Linux Smart Enumeration**): Es una herramienta de enumeración para sistemas Linux que permite a los atacantes obtener información detallada sobre la configuración del sistema, los servicios en ejecución y los permisos de archivo. LSE utiliza una variedad de comandos de Linux para recopilar información y presentarla en un formato fácil de entender. Al utilizar LSE, los atacantes pueden detectar posibles vulnerabilidades y encontrar información valiosa para futuros ataques.
- **Pspy**: Es una herramienta de enumeración de procesos que permite a los atacantes observar los procesos y comandos que se ejecutan en el sistema objetivo a intervalos regulares de tiempo. Pspy es una herramienta útil para la detección de malware y backdoors, así como para la identificación de procesos maliciosos que se ejecutan en segundo plano sin la interacción del usuario.

Asimismo, desarrollaremos un script en Bash ideal para detectar tareas y comandos que se ejecutan en el sistema a intervalos regulares de tiempo, abusando para ello del comando ‘**ps -eo user,command**‘ que nos chivará todo lo que necesitamos.

A continuación, se proporciona el enlace a estas herramientas:

- **Herramienta LSE**: [https://github.com/diego-treitos/linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration)
- **Herramienta PSPY**: [https://github.com/DominicBreuker/pspy](https://github.com/DominicBreuker/pspy)