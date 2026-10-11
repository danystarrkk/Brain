---
aliases: ["PortSwigger Labs", "Divulgación de Información"]
tags:
  - hacking/web
  - herramientas/burpsuite
  - hacking/laboratorios
  - hacking/explotacion
  - hacking/reconocimiento
estado: 🟢 Terminado
---
# PortSwigger

## Lab: Information disclosure in error messages

El propósito es provocar un error que nos permita filtrar información.

![[{89BB39D5-7691-4A6F-86B0-D5F8AB2CF1FC}.png]]

Vamos a ver que podemos controlar los ID de los productos. Podemos intentar cambiar los valores numéricos por texto para ver si ocurren errores que divulguen información:

![[{781AB5B1-489A-4D73-99D7-3A7D14154138}.png]]

Vemos que sí, se ha provocado un error que revela la versión del software. Con esto, vamos a pasarle la versión y veamos si se resuelve el laboratorio:

![[{DB119A49-F169-430A-B4CF-540383A5DC58}.png]]

> [!info] Divulgación de Información por Errores
> Los mensajes de error detallados pueden revelar información sensible sobre la infraestructura del servidor, la base de datos, el lenguaje de programación o las rutas de archivos. Esto incluye versiones de software, nombres de tablas/columnas de bases de datos, rutas absolutas del sistema de archivos y fragmentos de código fuente. Es crucial configurar los entornos de producción para mostrar mensajes de error genéricos y registrar los detalles internamente.

Lab Terminado.


## Lab: Information disclosure on debug page

En este laboratorio vamos a tener una página de depuración que divulga información. El propósito es encontrar la `Secret_KEY`.

![[{BA410643-FA5D-4C0B-BBDB-E8032F104595}.png]]

En el código fuente de la aplicación web vemos una ruta donde parece estar la página de `phpinfo()`. Vamos a verificar:

![[{8C423E80-F934-47C3-A751-7A9219B5460D}.png]]

> [!warning] Peligros de phpinfo()
> La función `phpinfo()` de PHP genera una página que muestra una gran cantidad de información sobre la configuración actual de PHP, las variables de entorno, los módulos cargados, las variables predefinidas y el estado del servidor. En un entorno de producción, esta página puede ser una mina de oro para un atacante, revelando rutas absolutas, credenciales de bases de datos, claves de API, versiones de software y otros datos sensibles que pueden ser explotados para ataques posteriores.

Listo, con esto podemos enviar la respuesta:

![[{4ED99445-2343-4C99-AC33-1585CB5916A1}.png]]

Lab Terminado.


## Lab: Source code disclosure via backup files

Al parecer, vamos a encontrar archivos de respaldo (backup) mediante los cuales se divulga información.

Bueno, para esto vamos a interceptar las peticiones con Burp Suite y vamos a ver mediante el `Site Map` qué podemos encontrar:

![[{FAC31A64-AD02-4913-B732-69BEB9782672}.png]]

No encontramos nada por ahora, y lo que podemos hacer es buscar por rutas conocidas como `robots.txt`:

![[{E3E1A7B3-4F4F-4AAB-86B5-703950141C5D}.png]]

> [!info] robots.txt
> El archivo `robots.txt` es un estándar de exclusión de robots que indica a los rastreadores de motores de búsqueda qué partes de un sitio web no deben indexar. Aunque no es un mecanismo de seguridad, a menudo revela rutas de directorios que los desarrolladores no quieren que sean públicas, lo que puede ser útil para un atacante en la fase de reconocimiento.

Al parecer, identificamos la ruta `/backup`:

![[{A4392F45-3527-499C-8C61-F5973AD0AF19}.png]]

> [!tip] Búsqueda de Archivos de Respaldo
> Los desarrolladores a menudo crean copias de seguridad de archivos o directorios antes de realizar cambios importantes. Estos archivos pueden contener código fuente, credenciales o configuraciones sensibles. Las extensiones comunes incluyen `.bak`, `.old`, `.zip`, `.tar.gz`, `.rar`, `.7z`, o incluso archivos con un `~` al final del nombre. Herramientas como DirBuster, Dirsearch o Gobuster son útiles para enumerar directorios y archivos ocultos o no enlazados.

Perfecto, encontramos el archivo y podemos ahora sí proporcionar la contraseña que nos pide:

![[{5CD3F0DA-CE47-4CA2-B36D-1E14DA1C0217}.png]]

Lab Terminado.


## Lab: Authentication bypass via information disclosure

El objetivo es obtener el control de administrador comenzando como `wiener` y luego eliminar el usuario `carlos`.

Vamos a ver cómo se está procesando el inicio de sesión primero:

![[{AD7ED10E-3DDF-46A5-9480-09B20C17D97E}.png]]

No vamos a encontrar nada interesante, pero podemos intentar ver qué sucede o cómo se maneja el acceso a la interfaz administrativa.

![[{5AE0F7D6-3D7B-401A-BCAF-C082196EC173}.png]]

Bueno, lo que podemos hacer es utilizar ciertas cabeceras para intentar burlar la protección, haciendo que el servidor crea que las peticiones provienen de una fuente interna o de confianza. Sin embargo, ninguna de las cabeceras conocidas funciona, y la fuerza bruta sobre cabeceras desconocidas sería ineficiente. Pero podemos intentar cambiando por diferentes métodos como POST, HEAD y, en este caso, uno nuevo que es `TRACE`, el cual es un método HTTP utilizado principalmente para depuración y diagnóstico, que devuelve la solicitud recibida por el servidor, incluyendo todas las cabeceras. Vamos a intentarlo:

![[{3CCA8A7C-5EFF-47CD-BE91-58D525570D66}.png]]

> [!info] Método HTTP TRACE
> El método HTTP `TRACE` es un método de diagnóstico que permite a un cliente ver exactamente lo que un servidor recibe en su solicitud. El servidor responde con una copia exacta de la solicitud que recibió, incluyendo todas las cabeceras. Esto puede ser útil para depurar problemas de proxy o de enrutamiento. Sin embargo, también puede ser explotado en ataques de Cross-Site Tracing (XST) si no está deshabilitado, y como se ve en este laboratorio, puede revelar cabeceras personalizadas que el servidor procesa internamente para la autenticación o autorización.

Vemos que sí ha funcionado y vemos muchas más cabeceras, donde destaca la cabecera `X-Custom_IP-Authorization`. Lo que parece ser que si enviamos esto, quizás intentando con una IP local, nos permita el acceso:

![[{2A1B5854-D2DC-428C-B6E7-8DEC840E50CD}.png]]

> [!tip] Cabeceras de Bypass Comunes
> Para intentar bypasses de autenticación o autorización, es común probar cabeceras HTTP que simulan una fuente de confianza o alteran la percepción del servidor sobre el origen de la solicitud. Algunas de estas incluyen: `X-Forwarded-For`, `X-Real-IP`, `X-Originating-IP`, `X-Remote-IP`, `X-Client-IP`, `X-Host`, `X-Custom-IP-Authorization` (como en este caso), entre otras. La clave es identificar cómo el servidor procesa estas cabeceras y si alguna de ellas puede ser manipulada para eludir los controles de seguridad.

Con esta cabecera ya hemos logrado el bypass. En este punto, lo que vamos a hacer es completar el lab eliminando el usuario `Carlos`.

![[{D028E237-30C7-442F-A3D2-3E8E3261085B}.png]]

Lab Terminado.


## Lab: Information disclosure in version control history

Al parecer, tenemos que intentar obtener la contraseña del usuario `administrator` mediante el historial de control de versiones (Git).

En este punto, vamos a intentar una enumeración de rutas con herramientas como Nmap (para puertos abiertos) y Gobuster (para directorios):

![[{16EE65DB-0000-4085-A1AF-3EA945379B46}.png]]

> [!info] Nmap y Gobuster para Reconocimiento
> - **Nmap**: Es una herramienta fundamental para el escaneo de puertos y la detección de servicios. Permite identificar qué puertos están abiertos en un objetivo y qué servicios se están ejecutando en ellos, lo cual es el primer paso para entender la superficie de ataque.
> - **Gobuster**: Es una herramienta de fuerza bruta para directorios y nombres de archivos en servidores web. Utiliza listas de palabras (wordlists) para probar miles de rutas posibles y descubrir recursos ocultos o no enlazados, como directorios de administración, archivos de configuración o, como en este caso, directorios de control de versiones como `.git`.

Nmap no revela rutas web directamente, por lo que procederemos con Gobuster para la enumeración de directorios y archivos:

![[{EF3B21C6-7B19-42F1-A65D-0AC08B38D8C7}.png]]

Vemos que encuentra un directorio `/.git`. Recordemos que cuando se construye un proyecto, es común usar un software de control de versiones como Git. Este crea el directorio `.git` en la raíz del proyecto para almacenar todo el historial de versiones, configuraciones y objetos internos. Lo que vamos a hacer es primero descargar el contenido del directorio `.git`:

![[{27C7BABC-D2C6-460B-AF78-A5B6DCA1608C}.png]]

Esto se hace con `wget -r <URL_DEL_DIRECTORIO_GIT>`.

> [!info] Divulgación de Información en Repositorios Git
> El directorio `.git` contiene todo el historial de un repositorio Git, incluyendo commits, ramas, tags y objetos. Si este directorio es accesible públicamente en un servidor web, un atacante puede descargarlo y reconstruir el repositorio completo. Esto permite revisar el historial de cambios, buscar credenciales, claves API, información sensible que pudo haber sido eliminada en commits posteriores pero que aún reside en el historial, o incluso revertir a versiones anteriores del código.

Ahora vamos a entrar y vamos a realizar un reconocimiento de los commits:

![[{105B7E9B-4E5E-492C-9834-82273AE369B8}.png]]

> [!tip] Comandos Git para Auditoría
> Una vez descargado el directorio `.git`, se pueden usar comandos estándar de Git para auditar el historial:
> - `git log`: Muestra el historial de commits, incluyendo mensajes, autores y fechas.
> - `git log -p`: Muestra el historial de commits con los cambios (diff) introducidos en cada uno.
> - `git show <commit_hash>`: Muestra los detalles de un commit específico, incluyendo los archivos modificados y su contenido.
> - `git checkout <commit_hash> <file_path>`: Permite recuperar una versión específica de un archivo de un commit anterior.

Vemos que tenemos un commit que nos dice que se eliminó la contraseña de la configuración. Esto quiere decir que en el commit anterior estaba presente. Vamos a revisar el commit anterior para ver qué encontramos:

![[{83B56613-C13F-41A8-A5B3-C68E9AA37FF8}.png]]

Ya hemos obtenido la contraseña del administrador. Vamos ahora sí a ver si completamos el lab:

![[{71FA9D5F-7A15-4BDC-8B64-613FCA83E67A}.png]]

Lab Terminado.