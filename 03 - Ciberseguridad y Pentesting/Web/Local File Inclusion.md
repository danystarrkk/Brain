---
aliases:
  - LFI
  - Path Traversal
tags:
  - hacking/web
  - hacking/explotacion
estado: 🟢 Terminado
---
# Vulnerabilidad Local File Inclusion (LFI)

La vulnerabilidad **Local File Inclusion** (**LFI**) es una falla de seguridad informática que se produce cuando una aplicación web **no valida adecuadamente** las entradas de usuario, permitiendo a un atacante **acceder a archivos locales** en el servidor web.

> [!info] ¿Qué es LFI?
> LFI permite a un atacante incluir archivos arbitrarios del sistema de archivos local en la respuesta de una aplicación web. Esto puede llevar a la divulgación de información sensible, ejecución remota de código (RCE) o incluso la toma de control del servidor.

En muchos casos, los atacantes aprovechan la vulnerabilidad de LFI al abusar de parámetros de entrada en la aplicación web. Los parámetros de entrada son datos que los usuarios ingresan en la aplicación web, como las URL o los campos de formulario. Los atacantes pueden manipular los parámetros de entrada para incluir rutas de archivo local en la solicitud, lo que puede permitirles acceder a archivos en el servidor web. Esta técnica se conoce como "**Path Traversal**" y se utiliza comúnmente en ataques de LFI.

> [!tip] Path Traversal
> El Path Traversal, también conocido como Directory Traversal, es una técnica que permite a un atacante acceder a archivos y directorios almacenados fuera del directorio raíz web. Se logra manipulando rutas de archivo con secuencias como `../` (punto-punto-barra) para navegar por la estructura de directorios del sistema.

En el ataque de Path Traversal, el atacante utiliza caracteres especiales y caracteres de escape en los parámetros de entrada para navegar a través de los directorios del servidor web y acceder a archivos en ubicaciones sensibles del sistema.

Por ejemplo, el atacante podría incluir "**../**" en el parámetro de entrada para navegar hacia arriba en la estructura del directorio y acceder a archivos en ubicaciones sensibles del sistema.

Para prevenir los ataques LFI, es importante que los desarrolladores de aplicaciones web validen y filtren adecuadamente la entrada del usuario, limitando el acceso a los recursos del sistema y asegurándose de que los archivos sólo se puedan incluir desde ubicaciones permitidas. Además, las empresas deben implementar medidas de seguridad adecuadas, como el cifrado de archivos y la limitación del acceso de usuarios no autorizados a los recursos del sistema.

> [!warning] Prevención de LFI
> Las medidas preventivas clave incluyen:
> *   **Validación estricta de entrada**: Utilizar listas blancas para los nombres de archivo permitidos.
> *   **Filtrado de caracteres especiales**: Eliminar o escapar secuencias como `../`, `..\`, `/`, `\` de las entradas de usuario.
> *   **Uso de `basename()`**: En PHP, `basename()` puede extraer el nombre de archivo de una ruta, eliminando componentes de directorio.
> *   **Restricción de permisos**: Asegurar que el usuario del servidor web tenga los mínimos privilegios necesarios.
> *   **Configuración de `open_basedir`**: En PHP, esta directiva restringe los archivos que pueden ser accedidos por scripts a un directorio específico.

A continuación, se os proporciona el enlace directo a la herramienta que utilizamos al final de esta clase para abusar de los ‘**Filter Chains**‘ y conseguir así ejecución remota de comandos:

- **PHP Filter Chain Generator**: [https://github.com/synacktiv/php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator)

## Notas de la Práctica

En este caso, como prueba de concepto, por ahora vamos a jugar con scripts en PHP para entender la vulnerabilidad. Tendremos que levantar el servicio `apache2` con `sudo service apache2 start` y en la carpeta `/var/www/html` vamos a crear un `index.php`. La primera versión será:

```php
<?php
  $filename = $_GET['filename'];
  include($filename);
?>
```

> [!info] LFI Básico
> Esta es la forma más directa de LFI. La función `include()` de PHP, cuando se le pasa una entrada de usuario no validada, intentará incluir cualquier archivo especificado por el parámetro `filename`. Si el archivo es un script PHP, se ejecutará; si es un archivo de texto, su contenido se mostrará.

En esta web, mediante el parámetro `filename`, podremos cargar archivos, como por ejemplo un `test.txt` que tengo en la misma carpeta:

![[Pasted image 20250825120251.png]]

Ahora, el problema llega cuando, al no contar con una correcta sanitización del código y dejar libre el *output* a los usuarios, ellos podrían listar archivos no solo de la carpeta, sino del sistema también, de la siguiente manera:

![[Pasted image 20250825120409.png]]

Esta es la base de un LFI, el más básico de todos.

Bueno, vamos a ver una segunda versión del `index.php` con una pequeña sanitización:

```php
<?php
  $filename = $_GET['filename'];
  include("/var/www/html/" . $filename);
?>
```

![[Pasted image 20250825120524.png]]

Por concepto, esto sí es vulnerable, ya que nosotros podemos retroceder directorios con `../` muchas veces con el objetivo de llegar a la raíz del sistema y luego cargar los archivos.

> [!tip] Bypass de Ruta Base
> Aunque se concatene una ruta base (`/var/www/html/`), un atacante puede usar secuencias `../` repetidas para "escapar" de ese directorio. Por ejemplo, `filename=../../../../etc/passwd` podría acceder a `/etc/passwd` si hay suficientes `../` para salir de `/var/www/html/`.

Bueno, vamos con una versión 3 del `index.php` donde vamos a sanitizar el `../`:

```php
<?php
  $filename = $_GET['filename'];
  $filename = str_replace("../", "", $filename); // remplaza los ../ por nada
  include("/var/www/html/" . $filename);
?>
```

Bueno, si ahora lo intentamos, veremos que no funciona:

![[Pasted image 20250825120934.png]]

Pero algo a considerar es que, por ejemplo, si usan `str_replace()`, esta no es una función recursiva, por lo que solo elimina la primera coincidencia. Por lo tanto, lo podemos burlar con un `....//`:

![[Pasted image 20250825121054.png]]

Como vemos, logramos de nuevo listar y burlar esta pequeña sanitización.

> [!warning] Limitaciones de `str_replace()`
> La función `str_replace()` en PHP no es recursiva por defecto. Si se busca reemplazar `../` por una cadena vacía, y la entrada es `....//`, `str_replace()` solo procesará la primera ocurrencia de `../`, dejando `../` restante. Esto permite el bypass con `....//` (que se convierte en `../` después del primer reemplazo) o `....//....//` y así sucesivamente.

En ocasiones, los desarrolladores también van a jugar con expresiones regulares para evitar que se pueda hacer esto, como lo veremos en esta versión de `index.php`:

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

Esto nos da como resultado lo siguiente:

![[Pasted image 20250825121458.png]]

Como podemos observar, no podemos listar por el *match* con `passwd` en este caso; ya no podremos ver el archivo.

Algo que tenemos que tener en cuenta es que el empleo de `preg_match` no es óptimo, ya que por concepto en Linux podemos hacer un `/etc/passwd` y es lo mismo que un `/etc//////passwd` o `/etc/././././passwd` o también con el uso de `?` que sería `/etc/pass?d`, lo que esto, al no tener el *match* completo, lo daría como válido:

![[Pasted image 20250825122136.png]]

> [!tip] Bypass de `preg_match` con rutas flexibles
> Los sistemas operativos Linux son flexibles con las rutas de archivo. Un atacante puede usar:
> *   **Barras múltiples**: `/etc//////passwd` es equivalente a `/etc/passwd`.
> *   **Directorios actuales**: `/etc/././passwd` es equivalente a `/etc/passwd`.
> *   **Wildcards**: `/etc/pass?d` puede coincidir con `/etc/passwd` si la expresión regular no es lo suficientemente estricta.
> Estas técnicas pueden eludir filtros basados en expresiones regulares que buscan cadenas exactas.

Bueno, un desarrollador también puede hacer lo siguiente:

```php
<?php
  $filename = $_GET['filename'];
  $filename = str_replace("../", "", $filename); // remplaza los ../ por nada

  include("/var/www/html" . $filename . ".php"); //fuerza que tenga un .php al final
?>
```

Esto lo que ocasiona es que los archivos que se busquen siempre tengan la extensión `.php` al final, provocando que no encuentre el archivo en el sistema y dé fallo:

![[Pasted image 20250825122426.png]]

En versiones anteriores de PHP, específicamente para versiones inferiores a la 5.3, es que podemos nosotros al final de la cadena ingresar un `nullbyte (%00)`, el cual provoca que:

![[Pasted image 20250825123154.png]]

Como vemos, con el `nullbyte` nos permite pasar. El `nullbyte` en el PHP interactivo se lo pone como `\0`, pero es lo mismo.

> [!warning] Null Byte (`%00`) Truncation
> En versiones antiguas de PHP (especialmente < 5.3), el carácter *null byte* (`%00` en URL-encoding) podía ser usado para truncar una cadena. Si una aplicación añadía una extensión (`.php`) al final de la entrada del usuario, un atacante podía añadir `%00` después del nombre de archivo deseado (ej. `/etc/passwd%00.php`). El *null byte* haría que la función `include()` o similar ignorara todo lo que venía después, permitiendo la inclusión del archivo sin la extensión forzada.

Ahora, algo que se puede hacer es filtrar por los últimos caracteres para filtrar por los nombres de archivos y bloquearlos de la siguiente manera:

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

En este caso, ya nos excluye de golpe:

![[Pasted image 20250825123624.png]]

Ahora, en versiones antiguas de PHP, suele funcionar que nosotros, para evitar que se encuentre de forma exitosa el nombre del archivo, podemos agregar `/.` que hace referencia al directorio actual, y esto lograría burlar esa seguridad.

![[Pasted image 20250825123841.png]]

> [!tip] Bypass de `substr()` con `/.`
> Si un filtro utiliza `substr()` para verificar los últimos caracteres de un nombre de archivo (por ejemplo, para bloquear "passwd"), se puede intentar añadir `/.` al final del nombre de archivo. Por ejemplo, `/etc/passwd/.` En algunos sistemas, `/.` se interpreta como el directorio actual, pero puede alterar la cadena lo suficiente como para eludir la comprobación de `substr()` sin cambiar el archivo al que se apunta.

Otra de las validaciones es que se puede tratar de filtrar por los últimos caracteres buscando que estos sean la extensión de la siguiente manera:

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

Esto, de igual forma, podemos pasarlo con el `/.` y eso ya permite pasar:

![[Pasted image 20250825124249.png]]

Esto es algo básico de cómo se aplican las diferentes formas de pasar algunas verificaciones.

Estos son los conceptos básicos de un LFI.

## Explotación de LFI con PHP Wrappers

Ahora vamos con algo más práctico, o sea, una máquina vulnerable la cual tiene una web con el siguiente contenido:

![[Pasted image 20250825125719.png]]

Si entramos en las opciones y vemos la URL, nos vamos a dar cuenta de que en ella apunta a la página. Esto quiere decir que de alguna forma está llamando al archivo que contiene esa página:

![[Pasted image 20250825125901.png]]

Podemos intentar los métodos anteriores a ver si alguno nos funciona y logramos ver el contenido de archivos internos en las máquinas:

![[Pasted image 20250825130009.png]]

Vemos que no nos deja, pero algo que podemos intentar es ver si el recurso de `courses.php` existe:

![[Pasted image 20250825130108.png]]

Vemos que sí existe en concepto y lo que puede estar haciendo la web es convertir ese `courses` en un `courses.php`, entonces le concatena a la petición el `.php`.

Ahora, lo que sobreentendemos es que la web está, de cada una de estas páginas, interpretando el código de los archivos `.php`, lo que quiere decir que cualquier código en PHP lo va a interpretar.

Nosotros algo que podemos hacer es usar *wrappers* como el de `php://filter/convert.base64-encode/resource=` y quedaría de la siguiente manera:

![[Pasted image 20250825130937.png]]

Como podemos observar, tenemos un texto en Base64 que es el contenido de `index.php`, por lo que podemos hacer lo inverso y tendríamos el código fuente de la web:

![[Pasted image 20250825131332.png]]

Podemos ver ya el contenido del archivo, donde en realidad hasta podemos ver que de alguna forma incluye otro archivo de conexión que sería `db_connect.php` y está en la ruta de `admin`, por lo que podríamos apuntar al archivo para ver si podemos sacar algo:

![[Pasted image 20250825131606.png]]

Una vez más, tenemos el contenido del archivo y si lo revisamos veremos:

![[Pasted image 20250825131636.png]]

Al parecer, tenemos información privilegiada listando los archivos del proyecto.

> [!info] `php://filter` Wrapper
> El *wrapper* `php://filter` permite aplicar filtros de procesamiento de datos a un flujo antes de que sea leído o escrito. Es extremadamente útil en ataques LFI para:
> *   **Divulgación de código fuente**: `php://filter/convert.base64-encode/resource=archivo.php` codifica el contenido del archivo en Base64, permitiendo al atacante decodificarlo y ver el código fuente, incluso si el servidor lo interpreta como PHP.
> *   **Manipulación de datos**: Otros filtros como `string.rot13` o `convert.iconv.*` pueden ser usados para diferentes propósitos, aunque `base64-encode` es el más común para la divulgación.

En este caso, nos creamos para practicar un Docker nosotros mismos, donde usamos la última versión de Ubuntu y, jugando con *Port Forwarding*, logramos tener una web que nos interpreta el código PHP, por lo que no podemos ver qué es lo que este código contiene:

![[Pasted image 20250825132759.png]]

Lo que intentamos es ver eso y en sí los comentarios y las declaraciones.

![[Pasted image 20250825132829.png]]

Lo que vemos en la web, no aparecen los comentarios ni nada de código en sí porque se está interpretando el código `.php`.

Vamos a usar el *wrapper* que ya conocemos, el de `php://filter/convert.base64-encode/resource=` de la siguiente manera:

![[Pasted image 20250825133032.png]]

Como podemos observar, ya tenemos el código en Base64 y podemos tratar de verlo:

![[Pasted image 20250825133121.png]]

Aparte de este *wrapper*, tenemos muchos más, como son los siguientes:

- `php://filter/read=string.rot13/resource=`
![[Pasted image 20250825133347.png]]
Para que salga en este caso, necesitamos verlo en modo código fuente.
![[Pasted image 20250825133612.png]]

- `php://filter/convert.iconv.utf-8.utf-16/resource=`
![[Pasted image 20250825133903.png]]
De igual forma, lo veremos en modo código fuente.

- `php://input` mediante el método POST podemos ejecutar comandos. En caso de ser GET, tenemos que cambiarlo a POST y listo:
![[Pasted image 20250825134640.png]]
Con esto ya podemos entablar una *reverse shell*.

> [!warning] `php://input` Wrapper
> El *wrapper* `php://input` permite leer datos *raw* del cuerpo de la solicitud HTTP. Si una aplicación es vulnerable a LFI y permite la inclusión de `php://input`, un atacante puede enviar código PHP malicioso en el cuerpo de una solicitud POST. Cuando el servidor incluye `php://input`, el código PHP enviado en el cuerpo de la solicitud se ejecutará, lo que puede llevar a la ejecución remota de comandos (RCE).

- `data://text/plain,base64,<cadena en base64>`
![[Pasted image 20250825134943.png]]
También estamos ejecutando comandos y para aclarar, la cadena en Base64 es código en PHP que lo puedes convertir como tú quieras:
![[Pasted image 20250825135042.png]]
Podemos programar lo que nosotros queramos. Esto, por ejemplo, para ejecutar comandos a voluntad, quedaría de la siguiente manera:
![[Pasted image 20250825135332.png]]
Con esto ya podemos ejecutar comandos a voluntad. En ocasiones, no funciona cuando tiene símbolos de *URL-encode*, por lo que con `Ctrl+U` en Burp Suite, seleccionando la cadena en Base64, podemos *URL-encode* y ya funcionaría:
![[Pasted image 20250825135448.png]]

> [!info] `data://` Wrapper
> El *wrapper* `data://` permite incrustar pequeños archivos directamente en la URL. En el contexto de LFI, `data://text/plain,base64,<código_base64>` permite inyectar y ejecutar código PHP codificado en Base64 directamente a través de la URL. Esto es útil cuando no se puede escribir en el sistema de archivos o cuando otras técnicas de inyección están bloqueadas. Es crucial *URL-encode* la cadena Base64 para evitar problemas con caracteres especiales en la URL.

### Ejecución Remota de Comandos con PHP Filter Chain Generator

Mediante la siguiente herramienta:
- **PHP Filter Chain Generator**: [https://github.com/synacktiv/php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator)

Lo que esta herramienta nos permite es de alguna forma introducir código PHP en la web y que la web lo interprete, esto con el objetivo de lograr una ejecución de comandos. Entonces, vamos a usar la herramienta.
![[Pasted image 20250825143447.png]]

Esto podemos pasarle a la web a ver qué logramos:

![[Pasted image 20250825143525.png]]
Vemos que logramos ejecutar el comando y vamos a mejorarlo para que mediante un parámetro lo podamos cambiar:

![[Pasted image 20250825143715.png]]

Y ahora podemos intentar ejecutar los comandos:

![[Pasted image 20250825143815.png]]
Esta es toda la cadena y al final le ponemos el `&cmd=id`.

> [!warning] PHP Filter Chains para RCE
> Las *PHP Filter Chains* son una técnica avanzada para lograr la ejecución remota de código (RCE) a través de LFI, incluso cuando los *wrappers* directos como `php://input` o `data://` están bloqueados. Consisten en encadenar múltiples filtros de PHP (`convert.base64-decode`, `string.strip_tags`, `convert.iconv.*`, etc.) de tal manera que, al final de la cadena, se reconstruye y ejecuta código PHP arbitrario. Herramientas como `php_filter_chain_generator` automatizan la creación de estas cadenas complejas, permitiendo a los atacantes inyectar y ejecutar comandos a través de parámetros de URL.