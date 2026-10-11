---
aliases:
  - Linux Filesystem Hierarchy Standard
  - FHS
tags:
  - linux/filesystem
estado: 🟢 Terminado
---
# Estructura de Directorios de Linux

Este documento detalla la jerarquía estándar de directorios en sistemas operativos GNU/Linux, siguiendo los principios del Filesystem Hierarchy Standard (FHS).

## Directorio Raíz (/)

El directorio raíz, simbolizado por el símbolo (`/`), es el **directorio principal a partir del cual se ramifican todos los demás directorios**. Es el punto de partida de todo el sistema de archivos.

> [!info] El Filesystem Hierarchy Standard (FHS) define la estructura principal de directorios en sistemas operativos tipo Unix. Su objetivo es estandarizar la ubicación de archivos y directorios para que los usuarios y el software puedan predecir dónde encontrar ciertos tipos de archivos.

## Directorio /bin

El directorio `/bin` es un **directorio estático y compartible en el que se almacenan archivos binarios/ejecutables necesarios para el funcionamiento básico del sistema**. Estos archivos binarios pueden ser utilizados por todos los usuarios del sistema operativo.

> [!tip] Los comandos esenciales como `ls`, `cp`, `mv`, `rm`, `cat`, `echo`, entre otros, suelen residir en `/bin`. En sistemas modernos, `/bin` a menudo es un enlace simbólico a `/usr/bin` para simplificar la estructura, aunque lógicamente representan ubicaciones distintas según el FHS tradicional.

## Directorio /boot

Es un directorio estático no compartible que **contiene todos los archivos necesarios para el arranque del ordenador, excepto los archivos de configuración**. Algunos de los archivos indispensables para el arranque del sistema que suele almacenar el directorio `/boot` son el kernel y el gestor de arranque Grub.

> [!info] El kernel de Linux (por ejemplo, `vmlinuz-X.Y.Z`) y las imágenes `initramfs` (que contienen un sistema de archivos raíz temporal para cargar módulos y montar el sistema de archivos raíz real) son componentes clave almacenados aquí. GRUB (Grand Unified Bootloader) utiliza archivos de configuración y módulos ubicados en `/boot/grub`.

## Directorio /dev

El sistema operativo GNU/Linux trata los dispositivos de hardware como archivos especiales. **Estos archivos que representan nuestros dispositivos de hardware se hallan almacenados en el directorio `/dev`**.

Algunos de los archivos básicos que podemos encontrar en este directorio son:

*   **`cdrom`**: que representa un dispositivo de CD-ROM.
*   **`sda`**: que representa un disco duro SATA (el primer disco).
*   **`audio`**: que representa una tarjeta de sonido.
*   **`psaux`**: que representa el puerto PS/2.
*   **`lpx`**: que representa una impresora.
*   **`fd0`**: que representa una disquetera.

> [!info] Los archivos en `/dev` no son archivos "reales" en el sentido de que no ocupan espacio en disco. Son interfaces para el kernel que permiten a los programas interactuar con el hardware. Se dividen en dispositivos de bloque (como discos duros, `sda`) y dispositivos de carácter (como terminales, `tty`, o puertos serie, `ttyS0`). El sistema `udev` gestiona la creación dinámica de estos archivos de dispositivo.

## Directorio /etc

El directorio `/etc` es un **directorio estático que contiene los archivos de configuración del sistema operativo**. Este directorio también contiene archivos de configuración para controlar el funcionamiento de diversos programas.

Algunos de los archivos de configuración de la carpeta `/etc` pueden ser sustituidos o complementados por archivos de configuración ubicados en nuestra carpeta personal `/home`.

> [!tip] `/etc` es crucial para la administración del sistema. Contiene archivos como `/etc/passwd` (información de usuarios), `/etc/shadow` (contraseñas cifradas), `/etc/fstab` (montaje de sistemas de archivos), `/etc/network/interfaces` (configuración de red), y configuraciones para servicios como Apache (`/etc/apache2`) o SSH (`/etc/ssh`). Los archivos de configuración suelen ser de texto plano, facilitando su edición.

## Directorio /home

El directorio `/home` se trata de un **directorio variable y compartible**. Este directorio está **destinado a alojar todos los archivos personales de los distintos usuarios del sistema operativo, a excepción del usuario `root`**. Algunos de los archivos personales almacenados en la carpeta `/home` son fotografías, documentos de ofimática, vídeos, etc.

> [!info] Cada usuario regular tiene su propio subdirectorio dentro de `/home` (ej., `/home/usuario1`). Este directorio contiene configuraciones específicas del usuario (a menudo en archivos y directorios ocultos que comienzan con un punto, como `.bashrc`, `.config/`, `.local/`), documentos, descargas y otros datos personales.

## Directorio /lib

El directorio `/lib` es un **directorio estático y que puede ser compartible**. Este directorio **contiene bibliotecas compartidas que son necesarias para arrancar los ejecutables que se almacenan en los directorios `/bin` y `/sbin`**.

> [!info] Las bibliotecas compartidas (archivos `.so` en Linux, como `libc.so.6`) son fragmentos de código que múltiples programas pueden usar simultáneamente, lo que ahorra espacio en disco y memoria. En sistemas de 64 bits, es común encontrar `/lib64` para bibliotecas de 64 bits. Al igual que `/bin`, `/lib` a menudo es un enlace simbólico a `/usr/lib` en sistemas modernos.

## Directorio /mnt

El directorio `/mnt` **tiene la finalidad de alojar los puntos de montaje de los distintos dispositivos de almacenamiento** como, por ejemplo, discos duros externos, particiones de unidades externas, etc.

> [!tip] `/mnt` se utiliza tradicionalmente para montajes temporales y manuales de sistemas de archivos. Por ejemplo, si necesitas acceder a una partición específica de otro disco duro para una tarea puntual, podrías montarla en `/mnt/mi_particion`.

## Directorio /media

La función del directorio `/media` es similar a la del directorio `/mnt`. Este directorio **contiene los puntos de montaje de los medios de almacenamiento extraíbles** como, por ejemplo, memorias USB, lectores de CD-ROM, unidades de disquete, etc.

> [!info] A diferencia de `/mnt`, `/media` está diseñado para montajes automáticos de dispositivos extraíbles por parte del sistema o del entorno de escritorio (ej., GNOME, KDE). Cuando insertas una memoria USB, el sistema la montará automáticamente en un subdirectorio dentro de `/media` (ej., `/media/usuario/NOMBRE_USB`).

## Directorio /opt

El contenido almacenado en el directorio `/opt` **es estático y compartible**. **La función de este directorio es almacenar programas de terceros o adicionales que no vienen con nuestro sistema operativo** como, por ejemplo, Spotify, Google Earth, Google Chrome, TeamViewer, etc.

> [!tip] `/opt` se utiliza para instalar paquetes de software grandes y autónomos que no siguen la estructura de directorios estándar del FHS. Cada aplicación suele tener su propio subdirectorio dentro de `/opt` (ej., `/opt/google/chrome`). Esto facilita la desinstalación y evita conflictos con los paquetes del sistema.

## Directorio /proc

El directorio `/proc` **se trata de un sistema de archivos virtual**. Este sistema de archivos virtual **nos proporciona información acerca de los distintos procesos y aplicaciones que se están ejecutando en nuestro sistema operativo**.

> [!info] `/proc` es un pseudo-sistema de archivos (o `procfs`) que reside completamente en la memoria. No contiene archivos reales en el disco. Cada directorio numérico dentro de `/proc` corresponde a un ID de proceso (PID) y contiene información sobre ese proceso. También expone parámetros del kernel que pueden ser leídos y modificados (ej., `/proc/sys/kernel/hostname`).

## Directorio /root

El directorio `/root` se trata de un **directorio variable no compartible**. El directorio `/root` **es el directorio `/home` del administrador del sistema** (usuario `root`).

> [!warning] Es crucial que el directorio `/root` esté separado de `/home` para mantener la configuración y los datos del superusuario aislados de los usuarios regulares. Esto es una medida de seguridad importante.

## Directorio /sbin

El directorio `/sbin` se trata de un **directorio estático y compartible**. Su función es similar al directorio `/bin`, pero a diferencia del directorio `/bin`, el directorio `/sbin` **almacena archivos binarios/ejecutables que solo puede ejecutar el usuario `root`** o administrador del sistema.

> [!tip] Comandos de administración del sistema como `fdisk`, `mkfs`, `shutdown`, `reboot`, `ifconfig` (en sistemas más antiguos) o `ip` (en sistemas modernos) suelen residir en `/sbin`. Al igual que `/bin`, en muchos sistemas modernos `/sbin` es un enlace simbólico a `/usr/sbin`.

## Directorio /srv

El directorio `/srv` se usa **para almacenar directorios y datos que usan ciertos servidores que podamos tener instalados en nuestro ordenador**.

> [!info] `/srv` es el lugar para datos específicos de servicios. Por ejemplo, si tienes un servidor web Apache, los archivos de tu sitio web podrían estar en `/srv/www`. Si tienes un servidor FTP, los datos compartidos podrían estar en `/srv/ftp`.

## Directorio /tmp

El directorio `/tmp` es **donde se crean y se almacenan los archivos temporales y las variables para que los programas puedan funcionar de forma adecuada**.

> [!warning] El contenido de `/tmp` es volátil y puede ser borrado en cada reinicio del sistema o periódicamente por un servicio como `systemd-tmpfiles`. Nunca almacenes datos importantes aquí. En muchos sistemas, `/tmp` se monta como un `tmpfs` (sistema de archivos en memoria RAM) para mejorar el rendimiento y garantizar la limpieza.

## Directorio /usr

El directorio `/usr` es un **directorio compartido y estático**. Este directorio es el que **contiene la gran mayoría de programas instalados** en nuestro sistema operativo.

Todo el contenido almacenado en la carpeta `/usr` es accesible para todos los usuarios y **su contenido es solo de lectura**.

> [!info] `/usr` significa "Unix System Resources" y es uno de los directorios más grandes. Contiene subdirectorios importantes como:
> *   `/usr/bin`: Binarios de usuario (la mayoría de los ejecutables de aplicaciones).
> *   `/usr/sbin`: Binarios de administración del sistema no esenciales para el arranque.
> *   `/usr/lib`: Bibliotecas compartidas.
> *   `/usr/local`: Software instalado manualmente por el administrador.
> *   `/usr/share`: Datos compartidos entre programas (documentación, iconos, fuentes).
> *   `/usr/src`: Código fuente del kernel y otros programas.

## Directorio /var

El directorio `/var` **contiene archivos de datos variables y temporales como, por ejemplo, los registros del sistema (logs), los registros de programas que tenemos instalados en el sistema operativo, archivos spool, etc.**

**La principal función del directorio `/var` es la de detectar problemas y solucionarlos**. Se recomienda ubicar el directorio `/var` en una partición propia, y en caso de no ser posible, es recomendable ubicarlo fuera de la partición raíz.

> [!tip] `/var` es crucial para la depuración y el monitoreo. Contiene:
> *   `/var/log`: Archivos de registro del sistema y de aplicaciones (ej., `syslog`, `auth.log`, logs de Apache).
> *   `/var/spool`: Datos para tareas en cola (ej., correos electrónicos salientes, trabajos de impresión).
> *   `/var/tmp`: Archivos temporales que se conservan entre reinicios (a diferencia de `/tmp`).
> *   `/var/www`: Contenido de sitios web (en algunas configuraciones).
> Una partición separada para `/var` evita que un crecimiento excesivo de logs o datos variables llene la partición raíz, lo que podría causar inestabilidad en el sistema.

## Directorio /sys

Directorio que **contiene información similar a la del directorio `/proc`**. Dentro de esta carpeta podemos encontrar **información estructurada y jerárquica acerca del kernel del sistema, de las particiones y sistemas de archivos, de los controladores (drivers)**, etc.

> [!info] `/sys` es un pseudo-sistema de archivos (`sysfs`) que expone la estructura de dispositivos y el estado del kernel. Permite a los programas interactuar con el hardware y el kernel de una manera más estructurada que `/proc`. Por ejemplo, puedes encontrar información sobre dispositivos PCI en `/sys/bus/pci` o controlar el brillo de la pantalla en `/sys/class/backlight`.

## Directorio /lost+found

Directorio que se crea en las particiones de disco con un sistema de archivos `ext` después de ejecutar herramientas para restaurar y recuperar el sistema operativo como, por ejemplo, `fsck`.

Si nuestro sistema no ha presentado problemas, este directorio estará completamente vacío. En el caso de que hayan habido problemas, **este directorio contendrá archivos y directorios que han sido recuperados tras un fallo del sistema operativo**.

> [!warning] Cuando `fsck` (File System Check) encuentra bloques de datos o inodos que no están asociados a ningún archivo o directorio conocido, los coloca en `/lost+found`. Estos elementos suelen estar renombrados con números de inodo. Es un lugar de "cuarentena" para datos recuperados que requieren inspección manual para determinar su origen y utilidad.