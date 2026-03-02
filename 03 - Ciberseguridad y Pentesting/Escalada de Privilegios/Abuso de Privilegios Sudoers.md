---
aliases: ["sudoers", "sudo", "privilegios sudo"]
tags:
  - linux/seguridad
  - hacking/escalada
estado: 🟢 Terminado
---
# Abuso de Privilegios Sudoers

El archivo **/etc/sudoers** es un archivo de configuración fundamental en sistemas Linux que se utiliza para controlar el acceso de los usuarios a las diferentes acciones que pueden realizar con privilegios elevados en el sistema. Este archivo contiene una lista de usuarios y grupos de usuarios que tienen permisos para realizar tareas de administración.

> [!info] Formato y Edición de `sudoers`
> El archivo `/etc/sudoers` no debe ser editado directamente con un editor de texto como `vi` o `nano`. En su lugar, se debe utilizar el comando `visudo`. Este comando abre el archivo en un editor (generalmente `vi` por defecto) y realiza una verificación de sintaxis antes de guardar los cambios, previniendo errores que podrían bloquear el acceso `sudo` al sistema.
>
> La sintaxis básica de una entrada en `sudoers` es:
> `usuario HOST=(RUNAS) COMANDO`
>
> Donde:
> *   `usuario`: El nombre de usuario o `%grupo`.
> *   `HOST`: El host desde donde se permite la ejecución (normalmente `ALL`).
> *   `RUNAS`: El usuario o grupo bajo el cual se ejecuta el comando (normalmente `ALL` o `root`).
> *   `COMANDO`: El comando o alias de comando que se puede ejecutar.

El comando "**sudo**" (del inglés "superuser do") permite a los usuarios ejecutar comandos como superusuario (root) o como otro usuario con privilegios especiales, según lo especificado en el archivo `sudoers`.

> [!info] `sudo` vs `su`
> Mientras que `su` (switch user) requiere la contraseña del usuario al que se desea cambiar (o la de root si no se especifica usuario), `sudo` requiere la contraseña del *propio usuario* que ejecuta el comando (a menos que se configure `NOPASSWD`). `sudo` ofrece un control más granular sobre qué comandos puede ejecutar un usuario específico y bajo qué condiciones, sin necesidad de compartir la contraseña de `root`.

Abusar de los privilegios a nivel de `sudoers` es una técnica comúnmente utilizada por los atacantes para elevar su nivel de acceso en un sistema comprometido. Si un atacante es capaz de obtener acceso a una cuenta con permisos de `sudo` en el archivo `sudoers`, puede ejecutar comandos con privilegios especiales y realizar acciones maliciosas en el sistema, llegando incluso a obtener una *shell* de `root`.

> [!warning] Configuraciones Inseguras Comunes
> Una configuración peligrosa en `sudoers` es permitir a un usuario ejecutar comandos como `ALL` o `NOPASSWD: ALL`. Esto significa que el usuario puede ejecutar cualquier comando como `root` sin necesidad de introducir su contraseña. Otras configuraciones de riesgo incluyen permitir la ejecución de binarios que pueden ser utilizados para escalar privilegios, como `find`, `nmap`, `vi`, `less`, `awk`, `perl`, `python`, `bash`, etc., con privilegios de `root`. Sitios como [GTFOBins](https://gtfobins.github.io/) documentan cómo abusar de estos binarios para obtener una *shell* de `root`.

El comando "**sudo -l**" es utilizado para listar los permisos de `sudo` de un usuario en particular. Al ejecutar este comando, se muestra una lista de los comandos que el usuario tiene permiso para ejecutar y bajo qué condiciones.

```bash
sudo -l
```

> [!tip] Interpretando `sudo -l` y `NOPASSWD`
> Al ejecutar `sudo -l`, busca líneas que contengan `(ALL : ALL) ALL` o `(ALL : ALL) NOPASSWD: ALL`. La presencia de `NOPASSWD` es crítica, ya que indica que el usuario puede ejecutar esos comandos sin necesidad de introducir su contraseña. Si un usuario tiene `NOPASSWD` para un comando que puede ser abusado (como un editor de texto, un intérprete de comandos o un binario con capacidades de ejecución de comandos), esto representa una vulnerabilidad directa para la escalada de privilegios.

Para prevenir el abuso de privilegios a nivel de `sudoers`, se recomienda mantener los permisos adecuados en el archivo `sudoers` y limitar el número de usuarios con permisos de `sudo`.

> [!tip] Principio de Mínimo Privilegio
> Aplica siempre el principio de mínimo privilegio: los usuarios solo deben tener los permisos `sudo` estrictamente necesarios para realizar sus tareas. Evita configuraciones amplias como `NOPASSWD: ALL`. En su lugar, especifica comandos exactos y rutas absolutas a los binarios permitidos. Por ejemplo, en lugar de `ALL`, usa `/usr/bin/apt update`.

Además, es importante monitorear regularmente el archivo `sudoers` y buscar cambios inesperados o sospechosos en su contenido.

> [!info] Auditoría y Monitoreo
> Herramientas de monitoreo de integridad de archivos (FIM) como `AIDE` o `Tripwire` pueden ser configuradas para alertar sobre cualquier modificación en `/etc/sudoers`. También es crucial revisar los logs de `sudo` (generalmente en `/var/log/auth.log` o `/var/log/secure`) para detectar intentos de uso de `sudo` no autorizados o inusuales.