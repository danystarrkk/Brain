--------
**WebDAV** (**Web Distributed Authoring and Versioning**) es una extensión del protocolo HTTP que permite a los usuarios **acceder** y **manipular** **archivos** en un servidor web a través de una conexión segura.

Cuando hablamos de enumerar un servidor WebDAV, a lo que nos referimos es al proceso de recopilar información sobre los recursos disponibles en el servidor WebDAV. Los atacantes pueden utilizar herramientas de enumeración de WebDAV para buscar recursos protegidos en el servidor, como archivos de configuración, contraseñas y otros datos confidenciales. Los atacantes pueden utilizar la información recopilada durante la enumeración para planificar ataques más sofisticados contra el servidor.

Asimismo, esta fase inicial de reconocimiento implica el intentar identificar las **extensiones de archivo** permitidas en el servidor. Una vez detectadas las extensiones de archivo permitidas, los atacantes pueden aprovecharse de esto para cargar y ejecutar archivos que contengan código malicioso. Si los archivos maliciosos se cargan y ejecutan con éxito en el servidor web, los atacantes pueden obtener acceso no autorizado al servidor y comprometer la seguridad del sistema.

Una de las herramientas que vemos en esta clase es **Davtest**. La herramienta Davtest es una herramienta de línea de comandos que se utiliza para realizar pruebas de penetración en servidores WebDAV. Davtest puede utilizarse para enumerar recursos protegidos en un servidor WebDAV, así como para probar la configuración de seguridad del servidor. Davtest también puede utilizarse para probar la autenticación y la autorización del servidor, y para detectar vulnerabilidades conocidas.

Otra de las herramientas que vemos en esta clase es **Cadaver**. Cadaver es otra herramienta de línea de comandos que se utiliza para interactuar con servidores WebDAV. Cadaver permite a los usuarios navegar por los recursos del servidor, cargar y descargar archivos, y ejecutar comandos en el servidor. Cadaver también puede utilizarse para realizar pruebas de penetración en servidores WebDAV, como la enumeración de recursos protegidos y la explotación de vulnerabilidades conocidas.

Para prevenir la enumeración y explotación de WebDAV, es importante que los administradores de sistemas implementen medidas de seguridad adecuadas en el servidor. Esto puede incluir la limitación de los recursos disponibles en el servidor y la utilización de autenticación y autorización fuertes. Además, es importante que los usuarios protejan sus contraseñas y eviten el uso de contraseñas débiles o fáciles de adivinar.



## Notas Practica

nos hemos montado con docker un webDAV para practicar y pues bueno al intentar verlo mediante el navegador nos pide credenciales:

![[Pasted image 20260227161233.png]]

En si lo que tenemos que hacer es primero enumerar y podemos hacer algo con [[Whatweb]]

![[Pasted image 20260227161334.png]]

como observamos la herramienta nos detecta que es un webDAV y que además pues no tenemos permisos.

Vamos a usar una herramienta llamada `davtest` que nos permite mediante credenciales validas subir múltiples archivos con diferentes extensiones con el objetivo de identificar cuales nos permite, que archivos son interpretados cuales son solo almacenados y mas.

![[Pasted image 20260227161648.png]]

Como vemos intentamos con credenciales no validas y pues no podemos hacer mucho, en caso de ser correcta seria:

![[Pasted image 20260227161802.png]]

como podemos observar nos muestra los archivos que intenta subir mediante el método `PUT`, luego los intenta ejecutar y al final nos indica todos los archivos que fueron posibles subir y los que fueron posibles de ejecutar.

En el caso de no tener la contraseña podemos hacer fuerza bruta de forma rápida y manual aprovechando la herramienta de `davtest` de la siguiente manera:

```bash
cat /usr/share/seclists/Passwords/Leaked-Databases/rockyou-15.txt | while read password; do response=$(davtest -url http://127.0.0.1 -auth admin:$password 2>&1 | grep -i succeed); if [ $response ]; then echo "[+] La contraseña es: $password"; break; fi; done
```

![[Pasted image 20260227163905.png]]


 ahora para interactuar con la maquina ya teniendo la contraseña puede ser `cadaver` para interactuar de forma mas manual:

![[Pasted image 20260227164038.png]]