---------

La vulnerabilidad **Local File Inclusion** (**LFI**) es una vulnerabilidad de seguridad informática que se produce cuando una aplicación web **no valida adecuadamente** las entradas de usuario, permitiendo a un atacante **acceder a archivos locales** en el servidor web.

En muchos casos, los atacantes aprovechan la vulnerabilidad de LFI al abusar de parámetros de entrada en la aplicación web. Los parámetros de entrada son datos que los usuarios ingresan en la aplicación web, como las URL o los campos de formulario. Los atacantes pueden manipular los parámetros de entrada para incluir rutas de archivo local en la solicitud, lo que puede permitirles acceder a archivos en el servidor web. Esta técnica se conoce como “**Path Traversal**” y se utiliza comúnmente en ataques de LFI.

En el ataque de Path Traversal, el atacante utiliza caracteres especiales y caracteres de escape en los parámetros de entrada para navegar a través de los directorios del servidor web y acceder a archivos en ubicaciones sensibles del sistema.

Por ejemplo, el atacante podría incluir “**../**” en el parámetro de entrada para navegar hacia arriba en la estructura del directorio y acceder a archivos en ubicaciones sensibles del sistema.

Para prevenir los ataques LFI, es importante que los desarrolladores de aplicaciones web validen y filtren adecuadamente la entrada del usuario, limitando el acceso a los recursos del sistema y asegurándose de que los archivos sólo se puedan incluir desde ubicaciones permitidas. Además, las empresas deben implementar medidas de seguridad adecuadas, como el cifrado de archivos y la limitación del acceso de usuarios no autorizados a los recursos del sistema.

A continuación, se os proporciona el enlace directo a la herramienta que utilizamos al final de esta clase para abusar de los ‘**Filter Chains**‘ y conseguir así ejecución remota de comandos:

- **PHP Filter Chain Generator**: [https://github.com/synacktiv/php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator)

## Notas de la practica

En este caso como prueba de concepto por ahora vamos a jugar con scripts en php para entender la vulnerabilidad donde tendremos que levantar el servicio `service start apache2` y en la carpeta `/var/www/html`  vamos a crear un index.php y la primera versión sera:

```php
<?php
  $filename = $_GET['filename'];
  include($filename);
?>
```

En esta web mediante el parámetro filename podremos cargar archivos, como por ejemplo un test que tengo en la misma carpeta:

![[Pasted image 20250825120251.png]]

ahora el problema llega cuando al no contar con una correcta sanitización del código y dejar libre el output a los usuario ellos podrían listar archivos no solo de la carpeta sino del sistema también de la siguiente manera:

![[Pasted image 20250825120409.png]]

Esta es la base de un LFI el mas básico de todos.

Bueno vamos a ver una segunda versión del index.php con una pequeña sanitización:

```php
<?php
  $filename = $_GET['filename'];
  include("/var/www/html/" . $filename);
?>
```

![[Pasted image 20250825120524.png]]

Por concepto esto si es vulnerable ya que nosotros podemos retroceder directorios muchos con el objetivo de llegar a la raíz del sistema y luego cargar los archivos.

Bueno vamos con una versión 3 del `index.php` donde vamos a sanitisar el `../`:
```php
<?php
  $filename = $_GET['filename'];
  $filename = str_replace("../", "", $filename); // remplaza los ../ por nada
  include("/var/www/html/" . $filename);
?>
```

bueno si ahora lo intentamos veremos que no funciona:

![[Pasted image 20250825120934.png]]

pero algo a considerar es que por ejemplo si usan `str_replace()` esta no es una función recursiva por lo que solo elimina la primera coincidencia por lo que lo podemos burlar con un `....//`:

![[Pasted image 20250825121054.png]]

como vemos logramos de nuevo listar y burlar esta pequeña sanitización.

En ocasiones los desarrolladores también van a jugar con expresiones regulares para evitar que se puede hacer esto como lo veremos en esta version de `index.php` :
```php
<?php
  $filename = $_GET['filename'];
  $filename = str_replace("../", "", $filename); // remplaza los ../ por nada

  if(preg_match("/\/etc\/passwd/", $filename) === 1){
    echo "\n[!] No es posible visualizar el contenido de este archivo\n";
  }else{
    include("/var/www/html/" . $filename);
  }
?>
```

esto nos da como resultado lo siguiente:

![[Pasted image 20250825121458.png]]

com podemos observar no podemos listar por el match con `passwd` en este caso ya no podremos ver el archivo.

Algo que tenemos que tener en cuenta es que el empleo de `preg_match` no es optimo ya que por concepto en Linux podemos hacer un `/etc/passwd` y es lo mismo que un `/etc//////passwd` o `/etc/././././passwd`  o también con el uso de `?` que serial `/etc/pass?d`lo que esto al no tener el match completo lo daría como valido:

![[Pasted image 20250825122136.png]]

Bueno un desarrollador también pude hacer lo siguiente:

```php
<?php
  $filename = $_GET['filename'];
  $filename = str_replace("../", "", $filename); // remplaza los ../ por nada

  include("/var/www/html" . $filename . ".php"); //fuerza que tenga un .php al final
?>
```


Esto lo que ocasiona es que los archivos que se busquen siempre tengan la extension `.php` al final provocando que que no encuentre el archivo en el sistema y de fallo:

![[Pasted image 20250825122426.png]]

en versiones anteriores de php específicamente para versiones inferiores a la 5.3 es que podemos nosotros al final de la cadena ingresar un `nullbyte (%00)`  el cual provoca que 

![[Pasted image 20250825123154.png]]

como vemos con el `nullbyte` nos permite pasar, el `nullbyte` en el php interactivo se lo pone como `\0` pero es lo mismo.

Ahora algo que se puede hacer es filtrar por los ultimos caracteres para filtrar por los nombres de archivos y bloquearlos de la siguiente manera:
```php
<?php
  $filename = $_GET['filename'];
  $filename = str_replace("../", "", $filename); // remplaza los ../ por nada

  if(substr($filename,-6,6) != "passwd"){
    include($filename);
  }else{
    echo "[!] Archivo no valido";
  }
?>
```

en este caso ya nos excluye de golpe:

![[Pasted image 20250825123624.png]]

ahora en versiones antiguas de php suele funcionar que nosotros para evitar que se encuentre de forma exitosa el nombre del archivo podemos agregar `/.` que hace referencia al directorio actual y esto lograría burlar esa seguridad.

![[Pasted image 20250825123841.png]]

otra de las validaciones es que se puede tratar de filtrar por los últimos caracteres buscando que estos sean la extension de la siguiente manera:
```php
<?php
  $filename = $_GET['filename'];
  $filename = str_replace("../", "", $filename); // remplaza los ../ por nada

  if(substr($filename,-4,4) === ".txt"){
    include($filename);
  }else{
    echo "[!] Archivo no valido";
  }
?>
```

Esto de igual forma podemos pasarlo con el `/.` y eso ya permite pasar:

![[Pasted image 20250825124249.png]]

Esto es algo básico de como se aplica las diferentes formas de pasar algunas verificaciones.

Estos son los conceptos Básicos de un LFI.

Ahora vamos con algo mas practico oséa una maquina vulnerable la cual tiene una web con el siguiente contenido:

![[Pasted image 20250825125719.png]]

si entramos en las opciones y vemos la url nos vamos a dar cuenta que en ella apunta a la pagina, esto quiere decir que de alguna forma esta llamando al archivo que contiene esa pagina:

![[Pasted image 20250825125901.png]]

podemos intentar los métodos anteriores a ver si alguno nos funciona y logramos ver el contenido de archivos internos en las maquinas:

![[Pasted image 20250825130009.png]]

vemos que no nos deja pero algo que podemos intentar es ver si el recurso de `courses.php` existe:

![[Pasted image 20250825130108.png]]

vemos que si existe en concepto y lo que puede esta haciendo la web es convertir ese `courses` en un `courses.php` entonces le concatena a la petición el `.php` 

Ahora lo que sobre entendemos es que la web esta de cada una de estas paginas interpretando el código de los archivos.php lo que quiere decir que cualquier código en php lo va a interpretar.

Nosotros algo que podemos hacer es usar wrappers como el de `php://filter/convert.base64-encode/resource=` y quedaría de la siguiente manera:

![[Pasted image 20250825130937.png]]

como podemos observar tenemos un texto en base64 que es el el contenido de index por o que podemos hacer lo inverso y tendríamos el código fuente de la web:

![[Pasted image 20250825131332.png]]

podemos ver ya el contenido del archivo, donde en realidad hasta podemos ver que de alguna forma incluye otro archivo de conexión que seria `db_connect.php`  y esta en la ruta de admin por lo que podríamos apuntar al archivo para ver si podemos sacar algo:

![[Pasted image 20250825131606.png]]

una vez mas tenemos el contenido del archivo y si lo revisamos veremos:

![[Pasted image 20250825131636.png]]
al parecer tenemos información privilegiada listando los archivos del proyecto.

En este caso nos creamos para practicar un docker nosotros mismo donde usamos la ultima versión de ubuntu y jugando con Port for warding logramos tener una web que nos interpreta el código php por lo que no podemos ver que es lo que este código contiene:

![[Pasted image 20250825132759.png]]
lo que intentamos es ver es eso y en si los comentarios y las declaraciones.

![[Pasted image 20250825132829.png]]
lo que vemos en la web, no aparecen los comentarios ni nada de código en si porque se esta interpretando el código `.php`

Vamos a usar el wrapper que ya conocemos, el de `php://filter/convert.base64-encode/resource=` de la siguiente manera:

![[Pasted image 20250825133032.png]]

como podemos observar ya tenemos el código en base64 y podemos tratar de verlo:

![[Pasted image 20250825133121.png]]

aparte de este wrapper tenemos muchos mas como son los siguientes:
- `php://filter/read=string.rot13/resource=`
![[Pasted image 20250825133347.png]]
para que salga en este caso necesitamos verlo en modo código fuente.
![[Pasted image 20250825133612.png]]

- `php://filter/convert.iconv.utf-8.utf-16/resource=`
![[Pasted image 20250825133903.png]]
de igual forma lo veremos en modo código fuente.

- `php://input` mediante el método POST podemos ejecutar comando,  en caso de ser GET tenemos que cambiarlo a POST y listo:
![[Pasted image 20250825134640.png]]
con esto ya podemos entablar una reverseshell.

- `data://text/plain,base64,<cadena en base64>`
![[Pasted image 20250825134943.png]]
también estamos ejecutando comandos y para aclarar la cadena en base64 es código en php que lo puedes convertir como tu quieras:
![[Pasted image 20250825135042.png]]
podemos programar lo que nosotros queramos, esto por ejemplo para ejecutar comandos a voluntad quedaría de la siguiente manera:
![[Pasted image 20250825135332.png]]
con esto ya podemos ejecutar comando a voluntad, en ocasiones no funciona cuando tiene símbolos de url-encode por lo que con ctrl+u en BurpSuite seleccionando la cadena en base64 podemos url-encode y ya funcionaria:
![[Pasted image 20250825135448.png]]

Mediante la siguiente herramienta:
- **PHP Filter Chain Generator**: [https://github.com/synacktiv/php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator)

Lo que esta herramienta nos permite es de alguna forma introducir código php en la web y que la web lo interprete, esto con el objetivo de lograr una ejecución de comandos, entonces vamos a usar la herramienta.
![[Pasted image 20250825143447.png]]

esto podemos pasarle a la web a ver que logramos:

![[Pasted image 20250825143525.png]]
vemos que logramos ejecutar el comando y vamos a mejorarlo para que mediante un parámetro lo podamos cambiar:

![[Pasted image 20250825143715.png]]

y ahora podemos intentar ejecutar los comandos:

![[Pasted image 20250825143815.png]]
esta toda la cadena y la final le ponemos el `&cmd=id`.