---
aliases:
  - Docker Escape
  - Docker PrivEsc
tags:
  - hacking/escalada
  - hacking/explotacion
  - hacking/laboratorios
  - linux/seguridad
estado: 🟢 Terminado
---
# Abuso de Docker para Escalada de Privilegios y Escape de Contenedores

En esta clase, exploraremos diversas técnicas para abusar de Docker con el objetivo de elevar nuestros privilegios de usuario y escapar del contenedor hacia la máquina host. Examinaremos situaciones específicas y discutiremos las implicaciones de seguridad en cada caso.

> [!info]
> Docker es una plataforma de contenedores que permite empaquetar aplicaciones y sus dependencias en unidades aisladas. Sin embargo, una configuración incorrecta o la exposición de funcionalidades privilegiadas pueden convertirlo en un vector crítico para la escalada de privilegios y el escape a la máquina anfitriona, debido a que los contenedores comparten el kernel del host y, en ciertos escenarios, recursos críticos del sistema.

Las técnicas que se tratarán en esta clase incluyen:

## Uso de Monturas (Bind Mounts) para Acceder a Archivos Privilegiados

Uso de monturas (bind mounts) en el despliegue de contenedores para acceder a archivos privilegiados del sistema host. Analizaremos cómo un atacante puede aprovechar las monturas para manipular los archivos del host y comprometer la seguridad del sistema.

> [!tip]
> Las monturas de volumen (`bind mounts`) permiten que un contenedor acceda directamente a un directorio o archivo del sistema de archivos del host. Si un atacante puede controlar la creación de contenedores y especificar una montura a un directorio sensible como `/etc`, `/root` o incluso el directorio raíz `/`, puede leer, modificar o sobrescribir archivos críticos del sistema host. Por ejemplo, montar `/etc` permite modificar `/etc/passwd` o `/etc/shadow` para crear nuevos usuarios o cambiar contraseñas.
>
> **Ejemplo de comando vulnerable:**
> ```bash
> docker run -v /:/mnt/host_root -it ubuntu bash
> ```
> Una vez dentro del contenedor, el atacante puede acceder al sistema de archivos del host en `/mnt/host_root` y realizar acciones como:
> ```bash
> echo 'root:newpasswordhash:0:0:root:/root:/bin/bash' >> /mnt/host_root/etc/passwd
> ```

## Despliegue de Contenedores con Compartición de Procesos y Permisos Privilegiados

Despliegue de contenedores con la compartición de procesos (`--pid=host`) y permisos privilegiados (`--privileged`). Veremos cómo inyectar un shellcode malicioso en un proceso en ejecución como root, lo que podría permitir al atacante tomar control del sistema.

> [!warning]
> La opción `--pid=host` permite que el contenedor comparta el espacio de nombres de PID del host, lo que significa que los procesos dentro del contenedor pueden ver y manipular los procesos del host. Combinado con `--privileged`, que otorga al contenedor todas las capacidades de Linux y acceso a dispositivos del host, un atacante puede realizar operaciones altamente intrusivas. Por ejemplo, puede usar `ptrace` para inyectar código en procesos del host que se ejecutan con privilegios de `root`.
>
> **Ejemplo de comando vulnerable:**
> ```bash
> docker run --pid=host --privileged -it ubuntu bash
> ```
> Dentro del contenedor, un atacante podría listar procesos del host (`ps aux`) e intentar inyectar código en un proceso `root` utilizando herramientas como `gdb` o `pwn_ptrace`.

## Uso de Portainer para Administrar el Despliegue de Contenedores

Uso de Portainer para administrar el despliegue de contenedores. Discutiremos cómo, mediante el empleo de monturas (bind mounts), un atacante podría ingresar y manipular archivos privilegiados del sistema host y escapar del contenedor.

> [!info]
> Portainer es una interfaz de usuario gráfica para gestionar entornos Docker. Si un atacante obtiene acceso a la interfaz de Portainer (por ejemplo, a través de credenciales comprometidas o una vulnerabilidad web), puede crear y desplegar contenedores con configuraciones arbitrarias, incluyendo `bind mounts` o el modo `--privileged`. Esto le permite realizar las mismas técnicas de escape y escalada de privilegios que si tuviera acceso directo al demonio Docker.
>
> **Escenario de ataque:** Un atacante con acceso a Portainer puede ir a la sección de 