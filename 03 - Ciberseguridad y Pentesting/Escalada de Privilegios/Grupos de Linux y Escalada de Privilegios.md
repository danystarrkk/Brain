---
aliases:
  - Linux Groups
  - Docker PrivEsc
  - LXD PrivEsc
tags:
  - hacking/escalada
  - linux/seguridad
estado: 🟢 Terminado
---
# Grupos de Linux y Escalada de Privilegios

En el contexto de Linux, los **grupos** se utilizan para **organizar** a los usuarios y **asignar permisos** para acceder a los recursos del sistema. Los usuarios pueden pertenecer a uno o varios grupos, y los grupos pueden tener diferentes niveles de permisos para acceder a los recursos del sistema.

> [!info] Gestión de Grupos en Linux
> Los comandos principales para la gestión de grupos incluyen:
> *   `groupadd [nombre_grupo]`: Crea un nuevo grupo.
> *   `groupdel [nombre_grupo]`: Elimina un grupo.
> *   `usermod -aG [nombre_grupo] [nombre_usuario]`: Añade un usuario a un grupo secundario.
> *   `groups [nombre_usuario]`: Muestra los grupos a los que pertenece un usuario.
> *   `id [nombre_usuario]`: Muestra el UID, GID y grupos a los que pertenece un usuario.
> *   `/etc/group`: Archivo que almacena la información de los grupos del sistema.

## Grupos Especiales: LXD y Docker

Existen grupos especiales en Linux, como 'lxd' o 'docker', que se utilizan para permitir a los usuarios ejecutar contenedores de manera segura y eficiente. Sin embargo, si un usuario malintencionado tiene acceso a uno de estos grupos, podría aprovecharlo para obtener privilegios elevados en el sistema.

> [!tip] ¿Qué son LXD y Docker?
> *   **LXD**: Es un gestor de contenedores de sistema (system containers) que ofrece una experiencia similar a la de una máquina virtual, pero con la ligereza de los contenedores. Permite a los usuarios ejecutar sistemas operativos completos dentro de contenedores.
> *   **Docker**: Es una plataforma de contenedores de aplicaciones (application containers) que permite empaquetar una aplicación y todas sus dependencias en un contenedor aislado.

## Escalada de Privilegios con el Grupo Docker

Por ejemplo, si un usuario tiene acceso al grupo 'docker', podría utilizar la herramienta Docker para desplegar nuevos contenedores en el sistema. Durante el proceso de despliegue, el usuario podría aprovecharse de las **monturas** (**mounts**) para hacer que ciertos recursos inaccesibles en la máquina host estén disponibles en el contenedor. Al ganar acceso al contenedor como usuario 'root', el usuario malintencionado podría inferir o manipular el contenido de estos recursos desde el contenedor.

> [!warning] Riesgo de Escalada de Privilegios con Docker
> Un usuario en el grupo `docker` tiene efectivamente privilegios de `root` sobre el sistema host. Esto se debe a que puede ejecutar contenedores con la opción `--privileged` o montar el sistema de archivos raíz del host (`/`) dentro del contenedor.
>
> **Ejemplo de Escalada de Privilegios con Docker:**
> ```bash
> # 1. Montar el sistema de archivos raíz del host en un contenedor
> docker run -v /:/mnt --rm -it alpine chroot /mnt sh
>
> # Una vez dentro del contenedor, se tiene acceso de root al sistema de archivos del host.
> # Por ejemplo, se puede leer /etc/shadow o añadir un usuario con privilegios.
> # Ejemplo: Añadir una clave SSH a un usuario existente o crear un nuevo usuario.
> # chroot /mnt sh -c "echo 'newuser:password' | chpasswd"
> # chroot /mnt sh -c "usermod -aG sudo newuser"
> ```
> Este método permite al atacante manipular archivos críticos del sistema host, como `/etc/shadow`, `/etc/passwd`, o incluso inyectar claves SSH en el directorio `~/.ssh` de un usuario privilegiado.

## Mitigación

Para mitigar el riesgo de abuso de grupos de usuario especiales, es importante limitar cuidadosamente el acceso a estos grupos y asegurarse de que solo se asignan a usuarios confiables que realmente necesitan acceder a ellos.

> [!info] Otros Grupos de Riesgo
> Además de `docker` y `lxd`, otros grupos como `kvm` (Kernel-based Virtual Machine) o `disk` también pueden presentar riesgos de seguridad si se asignan a usuarios no confiables, ya que permiten acceso directo a recursos de hardware o dispositivos de bloque. La gestión de permisos y la auditoría de la pertenencia a grupos son cruciales para mantener la seguridad del sistema.