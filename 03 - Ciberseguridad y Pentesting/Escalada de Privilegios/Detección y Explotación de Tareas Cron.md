---
aliases:
  - Cron Jobs
  - Explotación de Cron
  - Pspy
tags:
  - hacking/escalada
  - linux/seguridad
  - hacking/herramientas
estado: 🟢 Terminado
---
# Detección y Explotación de Tareas Cron

Una tarea **cron** es una tarea programada en sistemas Unix/Linux que se ejecuta en un momento determinado o en intervalos regulares de tiempo. Estas tareas se definen en un archivo **crontab** que especifica qué comandos deben ejecutarse y cuándo deben ejecutarse.

> [!info] Ubicaciones Comunes de Crontabs
> Los archivos `crontab` pueden residir en varias ubicaciones:
> - `/etc/crontab`: El crontab del sistema, que permite especificar el usuario bajo el cual se ejecutará cada tarea.
> - `/etc/cron.d/`: Directorio para scripts de cron específicos de paquetes o servicios. Cada archivo aquí es un crontab en sí mismo.
> - `/var/spool/cron/crontabs/`: Directorio que contiene los crontabs individuales de los usuarios. Cada usuario tiene su propio archivo `crontab` aquí, que puede editarse con el comando `crontab -e`.
> - `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/`: Directorios donde se colocan scripts que se ejecutan con la frecuencia indicada por el nombre del directorio.

La detección y explotación de tareas cron es una técnica utilizada por los atacantes para elevar su nivel de acceso en un sistema comprometido. Por ejemplo, si un atacante detecta que un archivo está siendo ejecutado por el usuario "root" a través de una tarea cron que se ejecuta a intervalos regulares de tiempo, y se da cuenta de que los permisos definidos en el archivo están incorrectamente configurados, podría manipular el contenido del mismo para incluir instrucciones maliciosas, las cuales serían ejecutadas de forma privilegiada como el usuario 'root', dado que corresponde al usuario que está ejecutando dicho archivo.

## Herramientas para la Detección de Tareas Cron

Para detectar tareas cron, los atacantes pueden utilizar herramientas como **Pspy**. Pspy es una herramienta de línea de comandos que monitorea las tareas que se ejecutan en segundo plano en un sistema Unix/Linux y muestra las nuevas tareas que se inician.

> [!tip] Funcionamiento de Pspy
> Pspy opera monitoreando el sistema de archivos `/proc` (procfs), que es un sistema de archivos virtual que proporciona información sobre los procesos en ejecución. Específicamente, Pspy observa los cambios en los directorios `/proc/[PID]` y los archivos dentro de ellos, como `cmdline` (línea de comandos) y `status` (estado del proceso), para identificar nuevos procesos y sus argumentos. Esto le permite detectar la ejecución de tareas cron, scripts y otros programas en tiempo real sin necesidad de permisos de root (aunque con root puede ver más detalles).

A continuación, se comparte el enlace al proyecto de Github correspondiente a la herramienta Pspy:

- **Herramienta Pspy**: [https://github.com/DominicBreuker/pspy](https://github.com/DominicBreuker/pspy)

## Recomendaciones para la Seguridad de Tareas Cron

Con el objetivo de reducir las posibilidades de que un atacante logre explotar las tareas cron en un sistema, se recomienda llevar a cabo alguno de los siguientes puntos:

- **Limitar el número de tareas cron**: Es importante limitar el número de tareas cron que se ejecutan en el sistema y asegurarse de que solo se otorgan permisos a tareas que requieren privilegios especiales para funcionar correctamente. Esto disminuye la superficie de ataque y reduce las posibilidades de que un atacante pueda encontrar una tarea cron vulnerable.

- **Verificar los permisos de las tareas cron**: Es crucial revisar los permisos de los archivos y directorios asociados a las tareas cron para asegurarse de que solo se otorgan permisos a usuarios y grupos autorizados. Además, se recomienda evitar otorgar permisos de superusuario a las tareas cron, a menos que sea estrictamente necesario.
> [!warning] Permisos Inseguros
> Un error común es que los scripts ejecutados por cron tengan permisos de escritura para usuarios no privilegiados. Por ejemplo, si un script en `/usr/local/bin/script.sh` es ejecutado por `root` vía cron, y un usuario de bajo privilegio tiene permisos de escritura sobre `script.sh`, ese usuario podría modificar el script para ejecutar comandos arbitrarios como `root`. Se deben usar `chmod` y `chown` para asegurar que solo el propietario (idealmente `root`) y los grupos autorizados tengan permisos de escritura.

- **Supervisar regularmente el sistema**: Es importante monitorear regularmente el sistema para detectar cambios inesperados en las tareas cron y para buscar posibles brechas de seguridad. Además, se recomienda utilizar herramientas de monitoreo de seguridad para detectar actividades sospechosas en el sistema.

- **Configurar los registros de la tarea cron**: Se recomienda habilitar la opción de registro para las tareas cron, para poder identificar cualquier actividad sospechosa en las tareas definidas y para poder llevar un registro de las actividades realizadas por cada una de estas.
> [!info] Registro de Cron
> En la mayoría de los sistemas Linux, el demonio `cron` registra sus actividades a través de `syslog`. Los registros de cron suelen encontrarse en archivos como `/var/log/syslog`, `/var/log/messages`, o un archivo específico como `/var/log/cron` (dependiendo de la configuración de `rsyslog` o `syslog-ng`). Es fundamental revisar estos registros periódicamente para detectar ejecuciones anómalas o errores que puedan indicar un intento de explotación.