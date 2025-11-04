-----------

El abuso de subidas de archivos es un tipo de ataque que se produce cuando un atacante aprovecha las vulnerabilidades en las aplicaciones web que permiten a los usuarios **cargar archivos** en el servidor. Este tipo de ataque se conoce comúnmente como un ataque de “**subida de archivos maliciosos**“.

En un ataque de subida de archivos maliciosos, un atacante carga un archivo malicioso en una aplicación web, que luego se almacena en el servidor. Si por ejemplo el atacante consigue subir un archivo PHP y el servidor web lo almacena, podría conseguir ejecución remota de comandos y tomar el control del servidor.

Los atacantes también pueden utilizar técnicas de “**falsificación de tipos de archivos**” para engañar a una aplicación web con el objetivo de que acepte un archivo malicioso como si fuera un archivo legítimo.

En esta clase, exploraremos algunas de las técnicas más utilizadas para explotar la fase de subida de archivos en aplicaciones web. Aprenderás cómo los atacantes pueden cargar contenido malicioso, además de eludir diferentes restricciones implementadas para lograrlo.

## PortSwigger

### Lab: Remote code execution via web shell upload

En este laboratorio vamos atener que leer el contenido de un archivo que tenemos dentro del laboratorio, tenemos credenciales y pues veamos que encontramos.

![[{0CDD3600-337A-457D-AE3E-229BD9B6ACCF}.png]]

Como podemos observar una vez que iniciamos sesión tenemos un apartado donde podemos subir un `avatar`, vamos a intentar subir uno y a capturar el punto exacto de la acción para ver como esto es que se tramita:

![[{E1F73A5B-71B8-43CC-90FB-6790AEA3CBCA}.png]]

vemos que se subió el archivo, ahora veamos como esta que se esta tramitando esto por detras:
![[{5314E630-F561-4747-AA78-5826C68F6F2E}.png]]

vemos como es que se tramita la imagen y vamos a intentar diferentes métodos que nos permitan la subida de archivos donde vamos a usar la siguiente petición:
![[{86175515-F3BE-40A8-B0EA-A0EBFB765236}.png]]

perfecto en este caso modificamos el contenido de la web para que el archivo que estamos subiendo es de tipo PHP y además si este se interpreta lograremos la ejecución de código, vemos que también nos habla de que esta o en `/avatar/cmd.php` o en `/cmd.php` vemos si alguno funciona y logramos cargar el archivo.

Ninguno fue correcto ya que la ruta correcta era:
![[{6FCBAB57-B772-4FEC-B9D5-B965D9AA900A}.png]]

esto lo podemos obtener si es que logramos ver de donde esta cargando la supuesta imagen en inspeccionar de la siguiente manera:
![[{2022A9C7-2FFE-4C7D-9142-E7301A968044}.png]]

con esto vamos a intentar ya mediante el parámetro `cmd` que definimos inyectar un comando:
![[{C2B6E0FE-67BB-445E-AA23-BB3CB8429096}.png]]

con esto vamos ya a intentar ver el archivo que se nos solicito:

![[{A35E3687-3C63-476D-AC63-1EFC2BDBCD67}.png]]

Completemos el laboratorio:

![[{9B2A9935-AD3B-4608-BA7C-1804C396759D}.png]]

Laboratorio terminado.


### Lab: Web shell upload via Content-Type restriction bypass

En este laboratorio al parecer vamos a tener una restricción al definir el Tipo de archivo y tenemos que sobrepasarla, vamos a intentarlo:


Vamos a iniciar sesión e intentar una subida de archivos en primera estancia:
![[{C7033E40-39B9-4CE4-90EC-CF6FBD8645DC}.png]]

como vemos nos dice que no podemos subir el archivo de typo `application/x-php` y que solo se admiten imágenes de tipo jpeg o png.

Bueno vamos a capturar la petición al momento de intentar subir el archivo a ver que nos encontramos:

![[{78CA2C85-CB35-4870-87E1-57630CABAD29}.png]]

vemos que la cabecera lleva el tipo de archivo, lo primero que podemos intentar hacer es cambiarla a ver si así nos permite pasar:

![[{2F4F7558-E5C7-404B-88EF-E6122DE47AC1}.png]]

![[{D42E63CC-3F16-443D-A5A7-3809F0B47939}.png]]

Pues ya logramos pasar y el archivo se subió, vamos a intentar ahora cargar ese archivo y ver si podemos leer el archivo que nos piden:

![[{BE81ECDC-3D16-40A6-96DC-DB63D9A13AA6}.png]]

![[{EAF2CD8F-2BAE-413F-8A30-41B7D815CE52}.png]]

Podemos terminar el laboratorio:
![[{81DE2997-E689-4EA6-88C8-FABEF995C865}.png]]

Lab Terminado.


### Lab: Web shell upload via path traversal

En este caso al igual que en los otros dos anteriores vamos atener una maquina vulnerable a la subida de archivos pero en esta ocasión no es posible la ejecución de la vista de los mismos por lo que debemos encontrar algún tipo de `path traversal` que nos permita abrirlo.

Como siempre primero intentamos subir el archivo a la maquina y veamos que es lo que sucede:

![[{04751C9B-9E6E-49A3-862E-BA8A887BC171}.png]]

Perfecto al parecer logramos subir archivos sin problemas, vamos a ver si podemos abrirlo:

![[{AA3D3797-DDAB-4B26-866B-9301C0B83CC5}.png]]

perfecto vemos que se abre pero no nos esta interpretando el código, esto es malo.

Vamos revisar la web a ver que mas podemos encontrar:

![[{D77E8274-6AAB-4D47-8548-7B29E491A13B}.png]]

Bueno algo que se me viene a la mente en este momento es intentar el `Path traversal` en el nombre, si esta mal programado puede que esto guarde el archivo en la ruta anterior, eso quiere decir que podríamos almacenar el archivo dentro de `/files/` en lugar de `/files/avatars`, vamos a ver que sucede:

![[{42DF1880-B5BC-4ADB-99D7-FDD35D243C34}.png]]

puede ser que lográramos el cometido, pues al parecer se inserto el `../` en el nombre, esto con ayuda de `url-encode` en el `/` por lo que vamos a ver si podemos cargar ahora el archivo:

![[{78D8A92D-2954-4360-8FA9-C031A4516556}.png]]

Pues perfecto, esto ya funciono, vamos a intentar enviar el código:

![[{6A1CB378-6240-4820-89E7-7A87B16D9A46}.png]]

Lab Terminada.


### Lab: Web shell upload via extension blacklist bypass

En este laboratorio al parecer se esta empleando una blacklist de extensiones la cual tenemos que intentar pasar.

Vamos a comenzar intentando subir el archivo tal cual para ver que nos responde:

![[{E5C935D1-7713-4884-AD75-5B96C735CCB6}.png]]

Como vemos no nos permite la subida de archivos con extension `.php`.

Lo que vamos a hacer en este punto es intentar capturar la petición y cambiar ya que veremos que podemos implementar diferentes extensiones que de igual forma permite que los archivos PHP se interpreten:

![[{FECB0377-46E0-4427-8FE6-F9D84F9E5BE1}.png]]

vemos una lista de posibles extensiones que podemos usar en PHP, vamos a intentar a ver si alguna es valida:

![[{D584C3F8-6438-4197-829A-96776635C68C}.png]]

vemos que `.php3` fue valido y ya subimos el archivo, vamos a intentar ver si podemos ejecutar comandos y ver el secreto:

![[{05B2CC30-0B76-4481-8BCA-569EC43AD201}.png]]

vemos que si se subió pero no se interpreta, vamos a seguir intentando con otros a ver que logramos.

Luego de varios vamos aver que no es posible el que nos interprete la información y otro truco que podemos usar es definir una extension y que esta la interprete como PHP de la siguiente manera:

![[{22ABD5B0-368E-416A-AB89-356DB7718745}.png]]

vemos que con `addType application/x-httpd-php .stark` estamos definiendo que a los archivos con extension `.stark` los interprete como PHP, esto es porque lo estamos definiendo dentro del archivo `.htacces` el cual se encarga de ajustar permisos dentro de directorios en los cuales se implementan PHP.

Ahora podemos intentar subir un archivo malicioso con extension `.stark` y vemos si lo interpreta:

![[{32FFACB7-8F93-42D2-BD63-2352881943A5}.png]]

![[{6F2C911F-2492-4603-B51B-07B0571893D6}.png]]

Terminemos el lab:
![[{402611D6-D855-445B-8C84-3CA1B66F9001}.png]]

Lab Terminado.


### Lab: Web shell upload via obfuscated file extension

Bueno en este caso se esta empleando una técnica de detección para las extensiones de archivos la cual tenemos que pasar.

En este caso lo vamos a hacer muy deprisa y es que en versiones antiguas de PHP se tenia un problema y es que podíamos mediante bytes nulos eliminar una extensión luego de ser  procesada de la siguiente manera:


![[{82CEAD55-15ED-4D71-8272-AB272623D316}.png]]

![[{6BD262A8-DA3C-4731-A34C-AB750F7C251A}.png]]

Como podemos observar de esta forma logramos subir el archivo, ahora es cuestión de ver si podemos ejecutar comandos:

![[{67E5C9FA-298A-48E6-9F72-892AF6289A7F}.png]]

Terminemos el laboratorio:

![[{04FEAFAC-8703-4C2C-A405-5447D308C925}.png]]

Lab Terminado.


### Lab: Remote code execution via polyglot web shell upload

La idea es volver a leer el archivo secret y tenemos que obtener el código especial para terminarlo.

Podemos jugar con los magics numbers para lograr subir el archivo de la siguiente manera:
![[{ACE649CC-F4F4-45AC-B0D4-CF7616A59CF4}.png]]

![[{8A829388-5694-4E5D-8A06-D09266C5E1F6}.png]]

se subió correctamente, en este punto lo que vamos a hacer es intentar cargar el archivo:
![[{2D869862-EC6A-4B54-A123-131DCEE742DB}.png]]

vemos que en este caso nos inserta el `GIF8` con el cual estamos validando los magics numbers y luego no inserta el valor del archivo, vamos completar el lab:

![[{61EC3200-F74A-422A-A4AA-4EC689B1A95D}.png]]

Lab Terminado.

Bueno tenemos nosotros otra alternativa y es insertar información dentro de una imagen, vamos a tener una imagen cualquiera y con ayuda de `exiftool` vamos a ver sus metadatos primero:
![[{1FB9E64C-21E7-4AA2-9F2F-19AD0A73BB26}.png]]

vemos los metadatos de la imagen, vamos a intentar agregar un comentario a sus metadatos donde el mismo comentario va a ser malicioso y lo hacemos de la siguiente manera:
![[{1A5595CD-52BE-4ADC-8F90-D27A6E610DD8}.png]]

si vemos nuevamente la imagen vamos a ver que se agrego lo que definimos:
![[{70B0062C-89D5-4FF7-9DCF-406F059CF794}.png]]

ahora podemos intentar subir esta imagen a ver que sucede:
![[{1DDBB914-3552-4AA2-B9EC-CF2BB849E24D}.png]]

Como vemos logramos subir el archivo, en este punto si logramos ingresar en el archivo vamos a ver lo siguiente:

![[{EE392D1D-0D39-4F00-AE4B-67FEB79C3252}.png]]

esta como vemos es una forma algo desordenada porque nos lista toda la data del archivo que era imagen en un comienzo pero podemos ejecutar comandos de igual forma:

![[{86793291-F642-4063-854E-F84022917374}.png]]

Podemos observar que después de `comment` se esta concatenando el contenido de del comando y bueno es cuestión de identificar la información y funcionaria, esta es una alternativa.


### Lab: Web shell upload via race condition

Vemos que esta ocasión vamos atener que aprovecharnos de una condición de carrera, donde en este caso tenemos una validación de archivos robusta pero logramos pasar de la misma mediante una condición de carrera debido a como se esta procesando.

Bueno nosotros vamos de comienzo a intentar subir nuestro PHP malicioso:

![[{9554D4FD-08EC-4208-A3A3-AB007089CC33}.png]]

bueno podemos intentar multiples métodos de los cuales no va a funcionar ninguno.

Podemos replicar esto en el repeater a ver que sucede:

![[{B260EFFC-3EC5-4F58-9C1F-E9A10C14B628}.png]]

Vemos de comienzo un pequeño retardo al responder, puede que el servidor este lento pero también puede que no, tenemos que probar todo lo que nos resulte curiosos o extraño para eso estamos los Pentesters.

Lo que estamos suponiendo es que ese tiempo que se demora esta realizando un procesado el cual puede que al momento de subir puede primero almacenarlo y luego determinar si es valido o no y eliminarlo donde si no es valido tarda ese tiempo y lo borra.

Con esta suposición decimos que el archivo existe por un momento en el servidor del cual podemos tratar de aprovecharnos ya que si somos lo suficientemente rápidos podemos tratar de cargar el archivo antes de que se elimine y lograr una ejecución de comandos momentánea por decirlo de alguna forma.

vamos primero a intentarlo de forma manual mandando al tiro las dos peticiones a ver que logramos:

![[{EF23B5BA-E456-410D-9057-60F2629878E9}.png]]

perfecto esto funciona, vamos a intentar automatizarlo creando un grupo y enviando las dos consultas en paralelo:

![[{B102BF4B-526A-41EC-BA85-56C22AEE83C8}.png]]

ya con esto funcionara la gran mayoría de veces por lo que vamos a intentar ver el secreto:

![[{E9C24CAB-3FFF-4A83-A1BB-92951EF90985}.png]]

perfecto vamos a completar el lab:

![[{CA21606A-C4E3-4A70-86B2-BBF79A788185}.png]]

Lab Terminado.



