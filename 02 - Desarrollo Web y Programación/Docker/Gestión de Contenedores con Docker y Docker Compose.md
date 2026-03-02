---
aliases:
  - Docker
  - Contenedores
  - Dockerfile
  - Docker Compose
tags:
  - hacking/herramientas
  - hacking/laboratorios
  - linux/comandos
  - programacion/python
estado: 🟢 Terminado
---
# Gestión de Contenedores con Docker y Docker Compose

## Introducción a Docker

**Docker** es una plataforma de **contenedores** de software que permite crear, distribuir y ejecutar aplicaciones en entornos aislados. Esto significa que se pueden empaquetar las aplicaciones con todas sus dependencias y configuraciones en un contenedor que se puede mover fácilmente de una máquina a otra, independientemente de la configuración del sistema operativo o del hardware.

> [!info] Contenedores vs. Máquinas Virtuales
> A diferencia de las máquinas virtuales (VMs) que virtualizan el hardware completo y ejecutan un sistema operativo invitado, los contenedores Docker comparten el kernel del sistema operativo del host. Esto los hace mucho más ligeros, rápidos de iniciar y eficientes en el uso de recursos, lo que es ideal para entornos de desarrollo, pruebas y despliegue rápido.

Algunas de las ventajas que se presentan a la hora de practicar hacking usando Docker son:

-   **Aislamiento**: Los contenedores de Docker están aislados entre sí, lo que significa que si una aplicación dentro de un contenedor es comprometida, el resto del sistema no se verá afectado.
-   **Portabilidad**: Los contenedores de Docker se pueden mover fácilmente de un sistema a otro, lo que los hace ideales para desplegar entornos vulnerables para prácticas de hacking.
-   **Reproducibilidad**: Los contenedores de Docker se pueden configurar de forma precisa y reproducible, lo que es importante en el hacking para poder recrear escenarios de ataque.

En las próximas clases, se mostrará cómo instalar Docker, cómo crear y administrar contenedores, y cómo usar Docker para desplegar entornos vulnerables para practicar hacking.

## Instalación de Docker

Para instalar **Docker** en Linux, se puede utilizar el comando `apt install docker.io`, que instalará el paquete Docker desde el repositorio de paquetes del sistema operativo. Es importante mencionar que, dependiendo de la distribución de Linux que se esté utilizando, el comando puede variar. Por ejemplo, en algunas distribuciones como CentOS o RHEL se utiliza `yum install docker` en lugar de `apt install docker.io`.

Una vez que Docker ha sido instalado, es necesario iniciar el **demonio** de Docker para que los contenedores puedan ser creados y administrados. Para iniciar el demonio de Docker, se puede utilizar el comando `service docker start`. Este comando iniciará el servicio del demonio de Docker, que es responsable de gestionar los contenedores y asegurarse de que funcionen correctamente.

> [!tip] Inicio del servicio Docker con `systemctl`
> En sistemas Linux modernos que utilizan `systemd` (como la mayoría de las distribuciones actuales), es más común y recomendado usar `systemctl` para gestionar servicios. Para iniciar Docker, se usaría `sudo systemctl start docker`. Para habilitarlo para que se inicie automáticamente en cada arranque, se usaría `sudo systemctl enable docker`.

## Definiendo la estructura básica de Dockerfile

Un archivo **Dockerfile** se compone de varias secciones, cada una de las cuales comienza con una **palabra clave** en **mayúsculas**, seguida de uno o más argumentos.

Algunas de las secciones más comunes en un archivo Dockerfile son:

-   **FROM**: Se utiliza para especificar la imagen base desde la cual se construirá la nueva imagen. **Usar la versión específica del sistema operativo en lugar de 'latest'.**
> [!warning] Evitar `latest` en `FROM`
> Aunque `latest` puede ser conveniente para pruebas rápidas, su uso en entornos de producción o para la creación de imágenes reproducibles es desaconsejado. La etiqueta `latest` puede apuntar a diferentes versiones de la imagen base con el tiempo, lo que puede llevar a inconsistencias y problemas de compatibilidad en futuras compilaciones. Siempre es mejor especificar una versión concreta, como `ubuntu:22.04` o `debian:stable`.
-   **RUN**: Se utiliza para ejecutar comandos en el interior del contenedor, como la instalación de paquetes o la configuración del entorno.
-   **COPY**: Se utiliza para copiar archivos desde el sistema host al interior del contenedor.
-   **CMD**: Se utiliza para especificar el comando que se ejecutará cuando se arranque el contenedor.

Además de estas secciones, también se pueden incluir otras instrucciones para configurar el entorno, instalar paquetes adicionales, exponer puertos de red y más.

Un ejemplo de la creación del Dockerfile es:

```dockerfile
FROM ubuntu:22.10

LABEL org.opencontainer.authors="Daniel Estrella"
```

## Creación y construcción de imágenes

Para crear una imagen de Docker, es necesario tener un archivo **Dockerfile** que defina la configuración de la imagen. Una vez que se tiene el Dockerfile, se puede utilizar el comando `docker build` para construir la imagen. Este comando buscará el archivo 'Dockerfile' en el directorio actual y utilizará las instrucciones definidas en el mismo para construir la imagen.

Algunas de las instrucciones que vemos en esta clase son:

-   `docker build`: Es el comando que se utiliza para construir una imagen de Docker a partir de un Dockerfile.

    La sintaxis básica es la siguiente:

    `➜ docker build [opciones] ruta_al_Dockerfile`

    El parámetro `-t` se utiliza para etiquetar la imagen con un nombre y una etiqueta. Por ejemplo, si se desea etiquetar la imagen con el nombre `mi_imagen` y la etiqueta `v1`, se puede usar la siguiente sintaxis:

    `➜ docker build -t mi_imagen:v1 ruta_al_Dockerfile`

    El punto (`.`) al final de la ruta al Dockerfile se utiliza para indicar al comando que busque el Dockerfile en el directorio actual. Si el Dockerfile no se encuentra en el directorio actual, se puede especificar la ruta completa al Dockerfile en su lugar. Por ejemplo, si el Dockerfile se encuentra en `/home/usuario/proyecto/`, se puede usar la siguiente sintaxis:

    `➜ docker build -t mi_imagen:v1 /home/usuario/proyecto/`

-   `docker pull`: Es el comando que se utiliza para descargar una imagen de Docker desde un registro de imágenes.

    La sintaxis básica es la siguiente:

    `➜ docker pull nombre_de_la_imagen:etiqueta`

    Por ejemplo, si se desea descargar la imagen `ubuntu` con la etiqueta `latest`, se puede usar la siguiente sintaxis:

    `➜ docker pull ubuntu:latest`

-   `docker images`: Es el comando que se utiliza para listar las imágenes de Docker que están disponibles en el sistema.

    La sintaxis básica es la siguiente:

    `➜ docker images [opciones]`

Durante la construcción de la imagen, Docker descargará y almacenará en caché las capas de la imagen que se han construido previamente, lo que hace que las compilaciones posteriores sean más rápidas.

> [!info] Capas de Docker y Caching
> Cada instrucción en un Dockerfile (como `FROM`, `RUN`, `COPY`) crea una nueva "capa" en la imagen. Docker almacena en caché estas capas. Si una instrucción y sus dependencias no han cambiado desde la última compilación, Docker reutilizará la capa en caché, acelerando significativamente el proceso de construcción. Esto es crucial para la eficiencia y la reproducibilidad.

## Carga de instrucciones en Docker y desplegando nuestro primer contenedor

El comando `docker run` se utiliza para crear y arrancar un contenedor a partir de una imagen. Algunas de las opciones más comunes para el comando `docker run` son:

-   `-d` o `--detach`: Se utiliza para arrancar el contenedor en segundo plano, en lugar de en primer plano.
-   `-i` o `--interactive`: Se utiliza para permitir la entrada interactiva al contenedor.
-   `-t` o `--tty`: Se utiliza para asignar un seudoterminal al contenedor.
-   `--name`: Se utiliza para asignar un nombre al contenedor.

Para arrancar un contenedor a partir de una imagen, se utiliza el siguiente comando:

`➜ docker run [opciones] nombre_de_la_imagen`

Por ejemplo, si se desea arrancar un contenedor a partir de la imagen `mi_imagen`, en segundo plano y con un seudoterminal asignado, se puede utilizar la siguiente sintaxis:

`➜ docker run -dit mi_imagen`

> [!tip] Explicación de `-dit`
> *   `-d` (detach): Ejecuta el contenedor en segundo plano, liberando la terminal.
> *   `-i` (interactive): Mantiene STDIN abierto incluso si no está adjunto, permitiendo interactuar con el contenedor más tarde.
> *   `-t` (tty): Asigna un pseudo-TTY, lo que permite una interfaz de línea de comandos interactiva dentro del contenedor. Esencial para comandos como `bash`.

Una vez que el contenedor está en ejecución, se puede utilizar el comando `docker ps` para listar los contenedores que están en ejecución en el sistema. Algunas de las opciones más comunes son:

-   `-a` o `--all`: Se utiliza para listar todos los contenedores, incluyendo los contenedores detenidos.
-   `-q` o `--quiet`: Se utiliza para mostrar sólo los identificadores numéricos de los contenedores.

Por ejemplo, si se desea listar todos los contenedores que están en ejecución en el sistema, se puede utilizar la siguiente sintaxis:

`➜ docker ps -a`

Para ejecutar comandos en un contenedor que ya está en ejecución, se utiliza el comando `docker exec` con diferentes opciones. Algunas de las opciones más comunes son:

-   `-i` o `--interactive`: Se utiliza para permitir la entrada interactiva al contenedor.
-   `-t` o `--tty`: Se utiliza para asignar un seudoterminal al contenedor.

Por ejemplo, si se desea ejecutar el comando `bash` en el contenedor con el identificador `123456789`, se puede utilizar la siguiente sintaxis:

`➜ docker exec -it 123456789 bash`

## Comandos comunes para la gestión de contenedores

A continuación, se detallan algunos de los comandos vistos en esta clase:

-   `docker rm $(docker ps -a -q) --force`: Este comando se utiliza para eliminar todos los contenedores en el sistema, incluyendo los contenedores detenidos. La opción `-q` se utiliza para mostrar sólo los identificadores numéricos de los contenedores, y la opción `--force` se utiliza para forzar la eliminación de los contenedores que están en ejecución. Es importante tener en cuenta que la eliminación de todos los contenedores en el sistema puede ser peligrosa, ya que puede borrar accidentalmente contenedores importantes o datos importantes. Por lo tanto, se recomienda tener precaución al utilizar este comando.
> [!warning] Precaución con `docker rm --force`
> El uso de `--force` puede resultar en la pérdida de datos no guardados o la interrupción de servicios críticos si se aplica a contenedores en producción. Siempre verifica los contenedores que vas a eliminar antes de ejecutar este comando, especialmente en entornos que no sean de laboratorio.
-   `docker rm id_contenedor`: Este comando se utiliza para eliminar un contenedor específico a partir de su identificador. Es importante tener en cuenta que la eliminación de un contenedor eliminará también cualquier cambio que se haya realizado dentro del contenedor, como la instalación de paquetes o la modificación de archivos.
-   `docker rmi $(docker images -q)`: Este comando se utiliza para eliminar todas las imágenes de Docker en el sistema. La opción `-q` se utiliza para mostrar sólo los identificadores numéricos de las imágenes. Es importante tener en cuenta que la eliminación de todas las imágenes de Docker en el sistema puede ser peligrosa, ya que puede borrar accidentalmente imágenes importantes o datos importantes. Por lo tanto, se recomienda tener precaución al utilizar este comando.
> [!warning] Precaución con `docker rmi`
> Eliminar imágenes puede liberar espacio en disco, pero también significa que Docker tendrá que volver a descargar o construir esas imágenes si se necesitan de nuevo. Asegúrate de que no necesitas las imágenes antes de eliminarlas.
-   `docker rmi id_imagen`: Este comando se utiliza para eliminar una imagen específica a partir de su identificador. Es importante tener en cuenta que la eliminación de una imagen eliminará también cualquier contenedor que se haya creado a partir de esa imagen. Si se desea eliminar una imagen que tiene contenedores en ejecución, se deben detener primero los contenedores y luego eliminar la imagen.

## Port Forwarding en Docker y uso de monturas

El port forwarding, también conocido como reenvío de puertos, nos permite redirigir el tráfico de red desde un puerto específico en el host a un puerto específico en el contenedor. Esto nos permitirá acceder a los servicios que se ejecutan dentro del contenedor desde el exterior.

Para utilizar el port forwarding, se utiliza la opción `-p` o `--publish` en el comando `docker run`. Esta opción se utiliza para especificar la redirección de puertos y se puede utilizar de varias maneras. Por ejemplo, si se desea redirigir el puerto 80 del host al puerto 8080 del contenedor, se puede utilizar la siguiente sintaxis:

`➜ docker run -p 80:8080 mi_imagen`

> [!info] Sintaxis de Port Forwarding
> La sintaxis general es `host_port:container_port`. Esto significa que cualquier conexión al `host_port` en la máquina anfitriona será redirigida al `container_port` dentro del contenedor. Si el `host_port` se omite (ej. `-p 8080`), Docker asignará un puerto aleatorio y disponible en el host.

Esto redirigirá cualquier tráfico entrante en el puerto 80 del host al puerto 8080 del contenedor. Si se desea especificar un protocolo diferente al protocolo TCP predeterminado, se puede utilizar la opción `-p` con un formato diferente. Por ejemplo, si se desea redirigir el puerto 53 del host al puerto 53 del contenedor utilizando el protocolo UDP, se puede utilizar la siguiente sintaxis:

`➜ docker run -p 53:53/udp mi_imagen`

Las monturas, por otro lado, nos permiten compartir un directorio o archivo entre el sistema host y el contenedor. Esto nos permitirá persistir la información entre ejecuciones de contenedores y compartir datos entre diferentes contenedores.

Para utilizar las monturas, se utiliza la opción `-v` o `--volume` en el comando `docker run`. Esta opción se utiliza para especificar la montura y se puede utilizar de varias maneras. Por ejemplo, si se desea montar el directorio `/home/usuario/datos` del host en el directorio `/datos` del contenedor, se puede utilizar la siguiente sintaxis:

`➜ docker run -v /home/usuario/datos:/datos mi_imagen`

> [!info] Sintaxis de Monturas (Volúmenes)
> La sintaxis general es `host_path:container_path`. Esto crea un "bind mount", donde el contenido del `host_path` se sincroniza con el `container_path`. Los cambios realizados en cualquiera de los dos lugares se reflejarán en el otro. Esto es fundamental para la persistencia de datos y para inyectar configuraciones o scripts desde el host.

Esto montará el directorio `/home/usuario/datos` del host en el directorio `/datos` del contenedor. Si se desea especificar una opción adicional, como la de montar el directorio en modo de solo lectura, se puede utilizar la opción `-v` con un formato diferente. Por ejemplo, si se desea montar el directorio en modo de solo lectura, se puede utilizar la siguiente sintaxis:

`➜ docker run -v /home/usuario/datos:/datos:ro mi_imagen`

En la siguiente clase, veremos cómo desplegar máquinas vulnerables usando Docker-Compose.

### Introducción a Docker Compose

**Docker Compose** es una herramienta de orquestación de contenedores que permite definir y ejecutar aplicaciones multi-contenedor de manera fácil y eficiente. Con Docker Compose, podemos describir los diferentes servicios que componen nuestra aplicación en un archivo YAML y, a continuación, utilizar un solo comando para ejecutar y gestionar todos estos servicios de manera coordinada.

En otras palabras, Docker Compose nos permite definir y configurar múltiples contenedores en un solo archivo YAML, lo que simplifica la gestión y la coordinación de múltiples contenedores en una sola aplicación. Esto es especialmente útil para aplicaciones complejas que requieren la interacción de varios servicios diferentes, ya que Docker Compose permite definir y configurar fácilmente la conexión y la comunicación entre estos servicios.

> [!tip] Beneficios de Docker Compose
> Docker Compose simplifica la gestión de entornos complejos al:
> *   **Definir servicios**: Todos los componentes de la aplicación (base de datos, frontend, backend) se definen en un solo archivo `docker-compose.yml`.
> *   **Redes automáticas**: Crea una red interna para que los contenedores se comuniquen entre sí por nombre de servicio.
> *   **Orquestación**: Permite iniciar, detener y reconstruir todos los servicios con un solo comando.
> *   **Reproducibilidad**: Asegura que el entorno de desarrollo sea idéntico al de producción o al de otros desarrolladores.

## Despliegue de máquinas vulnerables con Docker-Compose

El despliegue es bastante sencillo teniendo en cuenta que las máquinas están enteramente configuradas para funcionar, por lo que vamos a hacer uso del comando que se especifique en el repositorio de la máquina que vayamos a descargar. La web que se usará para todas las máquinas vulnerables será:

-   **Vulhub**: [https://github.com/vulhub/vulhub](https://github.com/vulhub/vulhub)

Y el comando que comúnmente se usa para levantar las máquinas es:

```bash
docker-compose up -d
```

> [!info] `docker-compose up -d`
> *   `up`: Construye, (re)crea, inicia y adjunta a los contenedores para un servicio.
> *   `-d` (detach): Ejecuta los contenedores en segundo plano. Si no se usa, los logs de todos los servicios se mostrarán en la terminal actual.

Por último, el repositorio es muy grande y es muy pesado como para clonarlo todo, por lo que vamos a hacer uso de una herramienta llamada `downgit`, la cual podemos instalar mediante `pip` con un `pip install downgit`. Para clonar una máquina en específico, entraremos a su carpeta específica y copiaremos la URL para luego hacer en consola un `downgit get <URL>` y listo.

> [!tip] `downgit` para repositorios grandes
> `downgit` es una herramienta muy útil para descargar subdirectorios específicos de un repositorio de GitHub sin tener que clonar todo el repositorio. Esto ahorra tiempo y espacio en disco, especialmente cuando solo se necesita un laboratorio o exploit específico de un repositorio extenso como Vulhub.
