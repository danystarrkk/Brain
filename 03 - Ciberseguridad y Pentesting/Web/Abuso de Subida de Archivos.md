---
aliases: ["File Upload Vulnerabilities", "Web Shell Upload"]
tags:
  - hacking/web
  - hacking/explotacion
  - hacking/laboratorios
  - herramientas/burpsuite
estado: 🟢 Terminado
---
El abuso de subida de archivos es un tipo de ataque que se produce cuando un atacante aprovecha las vulnerabilidades en las aplicaciones web que permiten a los usuarios **cargar archivos** en el servidor. Este tipo de ataque se conoce comúnmente como un ataque de “**subida de archivos maliciosos**“.

En un ataque de subida de archivos maliciosos, un atacante carga un archivo malicioso en una aplicación web, que luego se almacena en el servidor. Si, por ejemplo, el atacante consigue subir un archivo PHP y el servidor web lo almacena, podría conseguir ejecución remota de comandos y tomar el control del servidor.

Los atacantes también pueden utilizar técnicas de “**falsificación de tipos de archivos**” (MIME type spoofing) para engañar a una aplicación web con el objetivo de que acepte un archivo malicioso como si fuera un archivo legítimo.

En esta clase, exploraremos algunas de las técnicas más utilizadas para explotar la fase de subida de archivos en aplicaciones web. Aprenderás cómo los atacantes pueden cargar contenido malicioso, además de eludir diferentes restricciones implementadas para lograrlo.

> [!info]
> Las vulnerabilidades de subida de archivos pueden tener un impacto severo, desde la defacement del sitio web hasta la ejecución remota de código (RCE), lo que permite al atacante tomar el control total del servidor. La clave para la explotación a menudo reside en la validación inadecuada del lado del servidor de la extensión, el tipo MIME, el tamaño o el contenido del archivo.

## PortSwigger

### Lab: Remote code execution via web shell upload

En este laboratorio, tendremos que leer el contenido de un archivo dentro del entorno. Disponemos de credenciales; veamos qué encontramos.

![[{0CDD3600-337A-457D-AE3E-229BD9B6ACCF}.png]]

Como podemos observar, una vez que iniciamos sesión, tenemos un apartado donde podemos subir un `avatar`. Intentaremos subir uno y capturar el punto exacto de la acción para ver cómo se procesa:

![[{E1F73A5B-71B8-43CC-90FB-6790AEA3CBCA}.png]]

Vemos que el archivo se subió. Ahora, veamos cómo se está procesando esto por detrás:
![[{5314E630-F561-4747-AA78-5826C68F6F2E}.png]]

Vemos cómo se procesa la imagen. Intentaremos diferentes métodos que nos permitan la subida de archivos, utilizando la siguiente petición:
![[{86175515-F3BE-40A8-B0EA-A0EBFB765236}.png]]

Perfecto. En este caso, modificamos el `Content-Type` de la petición para que el archivo que estamos subiendo sea de tipo PHP. Si este se interpreta, lograremos la ejecución de código. Vemos que también se menciona que el archivo podría estar en `/avatar/cmd.php` o en `/cmd.php`. Verificaremos si alguna de estas rutas funciona y logramos cargar el archivo.

Ninguna de las rutas anteriores fue correcta, ya que la ruta adecuada era:
![[{6FCBAB57-B772-4FEC-B9D5-B965D9AA900A}.png]]

Esto lo podemos obtener si logramos ver desde dónde se está cargando la supuesta imagen, utilizando la herramienta de inspección del navegador de la siguiente manera:
![[{2022A9C7-2FFE-4C7D-9142-E7301A968044}.png]]

Con esto, intentaremos inyectar un comando mediante el parámetro `cmd` que definimos:
![[{C2B6E0FE-67BB-445E-AA23-BB3CB8429096}.png]]

Con esto, intentaremos ver el contenido del archivo solicitado:

![[{A35E3687-3C63-476D-AC63-1EFC2BDBCD67}.png]]

Completamos el laboratorio:

![[{9B2A9935-AD3B-4608-BA7C-1804C396759D}.png]]

Laboratorio terminado.

> [!tip]
> Para explotar una subida de *web shell*, es crucial identificar la ruta de almacenamiento del archivo. A menudo, esta ruta puede deducirse de la respuesta del servidor, de la URL de la imagen cargada o mediante la inspección del código fuente. El `Content-Type` en la cabecera HTTP es un indicador inicial del tipo de archivo, pero una validación robusta debe ir más allá de esta cabecera, ya que es fácilmente manipulable por el atacante.

### Lab: Web shell upload via Content-Type restriction bypass

En este laboratorio, al parecer, tendremos una restricción al definir el tipo de archivo (MIME type) que debemos eludir. Vamos a intentarlo:

Vamos a iniciar sesión e intentar una subida de archivos inicialmente:
![[{C7033E40-39B9-4CE4-90EC-CF6FBD8645DC}.png]]

Como vemos, nos indica que no podemos subir el archivo de tipo `application/x-php` y que solo se admiten imágenes de tipo JPEG o PNG.

Capturaremos la petición al momento de intentar subir el archivo para ver qué encontramos:

![[{78CA2C85-CB35-4870-87E1-57630CABAD29}.png]]

Vemos que la cabecera `Content-Type` lleva el tipo de archivo. Lo primero que podemos intentar es cambiarla para ver si así nos permite pasar la validación:

![[{2F4F7558-E5C7-404B-88EF-E6122DE47AC1}.png]]

![[{D42E63CC-3F16-443D-A5A7-3809F0B47939}.png]]

Pues ya logramos pasar la validación y el archivo se subió. Ahora intentaremos cargar ese archivo y ver si podemos leer el contenido del archivo solicitado:

![[{BE81ECDC-3D16-40A6-96DC-DB63D9A13AA6}.png]]

![[{EAF2CD8F-2BAE-413F-8A30-41B7D815CE52}.png]]

Podemos terminar el laboratorio:
![[{81DE2997-E689-4EA6-88C8-FABEF995C865}.png]]

Laboratorio terminado.

> [!info]
> La falsificación de tipos MIME (MIME type spoofing) es una técnica común para eludir las validaciones de subida de archivos. Si una aplicación solo valida el `Content-Type` enviado en la cabecera HTTP, un atacante puede simplemente modificar este valor (por ejemplo, de `application/x-php` a `image/jpeg`) para que el servidor acepte un archivo malicioso. La validación robusta debe incluir la verificación de los *magic numbers* del archivo o el procesamiento del contenido real del archivo, no solo la cabecera HTTP.

### Lab: Web shell upload via path traversal

En este caso, al igual que en los dos anteriores, tendremos una máquina vulnerable a la subida de archivos. Sin embargo, en esta ocasión no es posible la ejecución directa de los mismos, por lo que debemos encontrar algún tipo de `path traversal` que nos permita acceder a ellos.

Como siempre, primero intentamos subir el archivo a la máquina y veamos qué sucede:

![[{04751C9B-9E6E-49A3-862E-BA8A887BC171}.png]]

Perfecto, al parecer logramos subir archivos sin problemas. Vamos a ver si podemos abrirlos:

![[{AA3D3797-DDAB-4B26-866B-9301C0B83CC5}.png]]

Perfecto, vemos que se abre, pero no nos está interpretando el código. Esto es un problema.

Vamos a revisar la web para ver qué más podemos encontrar:

![[{D77E8274-6AAB-4D47-8548-7B29E491A13B}.png]]

Algo que se me viene a la mente en este momento es intentar un `Path Traversal` en el nombre del archivo. Si está mal programado, podría guardar el archivo en un directorio superior. Esto significa que podríamos almacenar el archivo dentro de `/files/` en lugar de `/files/avatars`. Veamos qué sucede:

![[{42DF1880-B5BC-4ADB-99D7-FDD35D243C34}.png]]

Puede ser que lográramos el cometido, pues al parecer se insertó el `../` en el nombre, esto con ayuda de `URL-encode` en el `/`. Por lo tanto, vamos a ver si podemos cargar ahora el archivo:

![[{78D8A92D-2954-4360-8FA9-C031A4516556}.png]]

Pues perfecto, esto ya funcionó. Vamos a intentar enviar el código:

![[{6A1CB378-6240-4820-89E7-7A87B16D9A46}.png]]

Laboratorio terminado.

> [!warning]
> Las vulnerabilidades de *Path Traversal* (`../`) permiten a un atacante acceder a directorios y archivos fuera del directorio restringido. En el contexto de la subida de archivos, esto puede significar que un archivo malicioso se guarda en una ubicación donde puede ser ejecutado por el servidor web, en lugar de un directorio seguro para subidas. Es crucial que las aplicaciones validen y saneen rigurosamente los nombres de archivo para evitar secuencias de *path traversal* y codificaciones URL (`%2f`, `%2e%2e%2f`).

### Lab: Web shell upload via extension blacklist bypass

En este laboratorio, al parecer, se está empleando una *blacklist* de extensiones que debemos intentar eludir.

Comenzaremos intentando subir el archivo tal cual para ver qué nos responde:

![[{E5C935D1-7713-4884-AD75-5B96C735CCB6}.png]]

Como vemos, no nos permite la subida de archivos con extensión `.php`.

Lo que haremos en este punto es intentar capturar la petición y modificarla, ya que existen diferentes extensiones que permiten que los archivos PHP se interpreten de igual forma:

![[{FECB0377-46E0-4427-8FE6-F9D84F9E5BE1}.png]]

Vemos una lista de posibles extensiones que podemos usar en PHP. Intentaremos ver si alguna es válida:

![[{D584C3F8-6438-4197-829A-96776635C68C}.png]]

Vemos que `.php3` fue válido y ya subimos el archivo. Intentaremos ver si podemos ejecutar comandos y obtener el secreto:

![[{05B2CC30-0B76-4481-8BCA-569EC43AD201}.png]]

Vemos que sí se subió, pero no se interpreta. Seguiremos intentando con otras extensiones para ver qué logramos.

Después de varios intentos, vemos que no es posible que nos interprete la información. Otro truco que podemos usar es definir una extensión personalizada y hacer que el servidor la interprete como PHP de la siguiente manera:

![[{22ABD5B0-368E-416A-AB89-356DB7718745}.png]]

Vemos que con `addType application/x-httpd-php .stark` estamos definiendo que los archivos con extensión `.stark` sean interpretados como PHP. Esto se debe a que lo estamos definiendo dentro del archivo `.htaccess`, el cual se encarga de ajustar la configuración y permisos dentro de los directorios donde se implementa PHP.

Ahora podemos intentar subir un archivo malicioso con extensión `.stark` y ver si lo interpreta:

![[{32FFACB7-8F93-42D2-BD63-2352881943A5}.png]]

![[{6F2C911F-2492-4603-B51B-07B0571893D6}.png]]

Terminamos el laboratorio:
![[{402611D6-D855-445B-8C84-3CA1B66F9001}.png]]

Laboratorio terminado.

> [!tip]
> Las *blacklists* de extensiones son inherentemente menos seguras que las *whitelists*. Una *blacklist* intenta bloquear extensiones conocidas como peligrosas (ej. `.php`, `.asp`), pero siempre puede haber extensiones alternativas (ej. `.phtml`, `.php3`, `.php4`, `.php5`, `.php7`, `.phps`, `.phar`) o configuraciones de servidor (`.htaccess` en Apache) que permitan la ejecución de código. Una *whitelist* solo permite extensiones explícitamente seguras (ej. `.jpg`, `.png`, `.gif`), lo que es un enfoque mucho más robusto.

### Lab: Web shell upload via obfuscated file extension

En este caso, se está empleando una técnica de detección para las extensiones de archivos que debemos eludir.

En este caso, lo haremos rápidamente. En versiones antiguas de PHP, existía un problema por el cual podíamos, mediante bytes nulos (`%00`), eliminar una extensión después de que el archivo fuera procesado, de la siguiente manera:


![[{82CEAD55-15ED-4D71-8272-AB272623D316}.png]]

![[{6BD262A8-DA3C-4731-A34C-AB750F7C251A}.png]]

Como podemos observar, de esta forma logramos subir el archivo. Ahora es cuestión de ver si podemos ejecutar comandos:

![[{67E5C9FA-298A-48E6-9F72-892AF6289A7F}.png]]

Terminamos el laboratorio:

![[{04FEAFAC-8703-4C2C-A405-5447D308C925}.png]]

Laboratorio terminado.

> [!info]
> La inyección de *null byte* (`%00`) era una técnica común en sistemas antiguos para truncar cadenas de caracteres. Al insertar un *null byte* en el nombre de un archivo (ej. `shell.php%00.jpg`), el sistema operativo o la aplicación podían interpretar que la cadena terminaba en el *null byte*, ignorando la extensión `.jpg` y procesando el archivo como `shell.php`. Esta vulnerabilidad ha sido corregida en la mayoría de los sistemas modernos, pero aún puede encontrarse en aplicaciones heredadas o mal configuradas.

### Lab: Remote code execution via polyglot web shell upload

La idea es volver a leer el archivo `secret` y obtener el código especial para terminar el laboratorio.

Podemos jugar con los *magic numbers* para lograr subir el archivo de la siguiente manera:
![[{ACE649CC-F4F4-45AC-B0D4-CF7616A59CF4}.png]]

![[{8A829388-5694-4E5D-8A06-D09266C5E1F6}.png]]

Se subió correctamente. En este punto, lo que haremos es intentar cargar el archivo:
![[{2D869862-EC6A-4B54-A123-131DCEE742DB}.png]]

Vemos que en este caso nos inserta el *magic number* `GIF8`, con el cual estamos validando los *magic numbers*, y luego inserta el valor del archivo. Vamos a completar el laboratorio:

![[{61EC3200-F74A-422A-A4AA-4EC689B1A95D}.png]]

Laboratorio terminado.

Tenemos otra alternativa: insertar información dentro de una imagen. Tomaremos una imagen cualquiera y, con la ayuda de `exiftool`, veremos sus metadatos primero:
![[{1FB9E64C-21E7-4AA2-9F2F-19AD0A73BB26}.png]]

Vemos los metadatos de la imagen. Intentaremos agregar un comentario a sus metadatos, donde el mismo comentario será malicioso, y lo haremos de la siguiente manera:
![[{1A5595CD-52BE-4ADC-8F90-D27A6E610DD8}.png]]

Si vemos nuevamente la imagen, observaremos que se agregó lo que definimos:
![[{70B0062C-89D5-4FF7-9DCF-406F059CF794}.png]]

Ahora podemos intentar subir esta imagen para ver qué sucede:
![[{1DDBB914-3552-4AA2-B9EC-CF2BB849E24D}.png]]

Como vemos, logramos subir el archivo. En este punto, si logramos acceder al archivo, veremos lo siguiente:

![[{EE392D1D-0D39-4F00-AE4B-67FEB79C3252}.png]]

Esta, como vemos, es una forma algo desordenada porque nos lista toda la data del archivo que era imagen en un comienzo, pero podemos ejecutar comandos de igual forma:

![[{86793291-F642-4063-854E-F84022917374}.png]]

Podemos observar que después de `comment` se está concatenando el contenido del comando. Es cuestión de identificar la información y funcionaría. Esta es una alternativa.

> [!info]
> Los *magic numbers* son secuencias de bytes al inicio de un archivo que identifican su formato. Las aplicaciones pueden validar estos números para asegurar que el archivo es del tipo esperado. Un archivo *políglota* es un archivo que es válido bajo múltiples formatos (ej. una imagen GIF que también contiene código PHP ejecutable). Esto se logra incrustando el código malicioso después del *magic number* y otros metadatos válidos de la imagen. Herramientas como `exiftool` pueden ser utilizadas para manipular metadatos de imágenes y facilitar la creación de estos archivos *políglotas*.

### Lab: Web shell upload via race condition

En esta ocasión, tendremos que aprovecharnos de una condición de carrera. En este caso, la aplicación tiene una validación de archivos robusta, pero logramos eludirla mediante una condición de carrera debido a cómo se está procesando el archivo.

Comenzaremos intentando subir nuestro archivo PHP malicioso:

![[{9554D4FD-08EC-4208-A3A3-AB007089CC33}.png]]

Podemos intentar múltiples métodos, de los cuales ninguno funcionará.

Podemos replicar esto en Burp Suite Repeater para ver qué sucede:

![[{B260EFFC-3EC5-4F58-9C1F-E9A10C14B628}.png]]

Vemos inicialmente un pequeño retardo al responder. Puede que el servidor esté lento, pero también podría ser un indicio de un proceso de validación. Tenemos que probar todo lo que nos resulte curioso o extraño; para eso estamos los *Pentesters*.

Lo que estamos suponiendo es que ese tiempo de demora se debe a un proceso de validación. Es posible que, al momento de la subida, el servidor primero almacene el archivo temporalmente y luego determine si es válido o no. Si no es válido, lo elimina después de ese retardo.

Con esta suposición, inferimos que el archivo existe por un momento en el servidor. Podemos intentar aprovecharnos de esto: si somos lo suficientemente rápidos, podemos intentar cargar el archivo antes de que se elimine y lograr una ejecución de comandos momentánea, por decirlo de alguna forma.

Primero, intentaremos hacerlo de forma manual, enviando rápidamente las dos peticiones para ver qué logramos:

![[{EF23B5BA-E456-410D-9057-60F2629878E9}.png]]

Perfecto, esto funciona. Vamos a intentar automatizarlo creando un grupo de peticiones y enviando ambas consultas en paralelo:

![[{B102BF4B-526A-41EC-BA85-56C22AEE83C8}.png]]

Con esto, funcionará la gran mayoría de las veces, por lo que vamos a intentar ver el secreto:

![[{E9C24CAB-3FFF-4A83-A1BB-92951EF90985}.png]]

Perfecto, vamos a completar el laboratorio:

![[{CA21606A-C4E3-4A70-86B2-BBF79A788185}.png]]

Laboratorio terminado.

> [!warning]
> Las *condiciones de carrera* (Race Conditions) en la subida de archivos ocurren cuando hay una ventana de tiempo entre la validación de un archivo y su procesamiento o eliminación. Un atacante puede explotar esta ventana subiendo un archivo malicioso, y antes de que la validación lo elimine, realizar una segunda petición para ejecutarlo. Herramientas como Burp Suite Intruder o Turbo Intruder son ideales para automatizar y acelerar estas peticiones, aumentando las posibilidades de éxito. Las mitigaciones incluyen el uso de operaciones atómicas, el bloqueo de archivos o el procesamiento de archivos en un entorno aislado antes de moverlos a una ubicación accesible públicamente.