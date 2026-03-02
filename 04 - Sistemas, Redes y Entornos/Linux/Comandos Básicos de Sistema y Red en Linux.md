---
aliases:
  - Linux Info
  - System Info
  - Network Commands
tags:
  - linux/comandos
  - redes/comandos
estado: 🟢 Terminado
---
# Comandos Básicos de Sistema y Red en Linux

## Listar Puertos Abiertos

Estos comandos permiten identificar los puertos de red que están en uso o en escucha en el sistema.

```bash
# Listar puertos abiertos (TCP y UDP)
netstat -nat
```
> [!info] `netstat` (Network Statistics) es una herramienta clásica para monitorear conexiones de red.
> *   `-n`: Muestra direcciones y números de puerto numéricamente (sin resolver nombres de host o servicios).
> *   `-a`: Muestra todos los sockets (listening y non-listening).
> *   `-t`: Muestra solo conexiones TCP.
> *   `-u`: Muestra solo conexiones UDP.
> *   `-p`: Muestra el PID y el nombre del programa asociado al socket (requiere privilegios de root).
> Aunque sigue siendo funcional, `netstat` está considerado obsoleto en muchos sistemas Linux modernos a favor de `ss`.

```bash
# Listar sockets en escucha (TCP y UDP)
ss -nltp
```
> [!info] `ss` (Socket Statistics) es una herramienta más moderna y eficiente que `netstat` para inspeccionar sockets.
> *   `-n`: Muestra direcciones y números de puerto numéricamente.
> *   `-l`: Muestra solo sockets en estado de escucha (listening).
> *   `-t`: Muestra solo sockets TCP.
> *   `-p`: Muestra el proceso que posee el socket.
> `ss` puede ser significativamente más rápido que `netstat`, especialmente en sistemas con muchas conexiones, ya que obtiene la información directamente del kernel.

```bash
# Ver conexiones TCP a través del sistema de archivos proc
cat /proc/net/tcp
```
> [!info] El directorio `/proc` es un sistema de archivos virtual que proporciona una interfaz a las estructuras de datos internas del kernel. `/proc/net/tcp` expone información detallada sobre las conexiones TCP activas, incluyendo el estado del socket, direcciones IP locales y remotas, puertos, y más, en un formato hexadecimal. Es la fuente de datos que utilizan herramientas como `netstat` y `ss`.

```bash
# Listar procesos que usan un puerto específico
lsof -i:<puerto>
```
> [!info] `lsof` (List Open Files) es una utilidad muy potente que lista todos los archivos abiertos por procesos. En el contexto de red, permite identificar qué proceso está utilizando un puerto específico.
> *   `-i`: Selecciona archivos de Internet.
> *   `:<puerto>`: Filtra por el número de puerto. Por ejemplo, `lsof -i:80` mostrará los procesos que usan el puerto 80.
> `lsof` puede mostrar no solo sockets de red, sino también archivos regulares, directorios, pipes, etc., lo que lo convierte en una herramienta de diagnóstico muy versátil.

## Listar Procesos del Sistema

Estos comandos permiten visualizar los procesos que se están ejecutando en el sistema.

```bash
# Listar procesos con formato detallado
ps -faux
```
> [!info] `ps` (Process Status) muestra información sobre los procesos en ejecución.
> *   `-f`: Formato "full" (completo), que incluye UID, PID, PPID, C, STIME, TTY, TIME, CMD.
> *   `-a`: Muestra procesos de todos los usuarios.
> *   `-u`: Muestra procesos por usuario/propietario.
> *   `-x`: Muestra procesos sin un TTY de control.
> La combinación `-faux` es muy común para obtener una vista jerárquica y detallada de todos los procesos en el sistema, incluyendo los que no están asociados a una terminal.

## Listar Comandos Ejecutados en el Sistema

Este comando muestra la línea de comando completa de los procesos en ejecución.

```bash
# Listar la línea de comando completa de los procesos
ps -eo command
```
> [!info] `ps -eo command` es útil para ver exactamente qué comandos se están ejecutando en el sistema en un momento dado.
> *   `-e`: Selecciona todos los procesos.
> *   `-o command`: Especifica que solo se muestre la columna `command`, que contiene la línea de comando completa con sus argumentos.
> Para ver el historial de comandos ejecutados por un usuario en una sesión de shell, se puede consultar el archivo `~/.bash_history` (para Bash) o usar el comando `history`. Sin embargo, esto solo muestra comandos ejecutados en el shell, no por procesos del sistema o scripts.

## Conexiones con Netcat (nc)

`netcat` es una herramienta versátil para leer y escribir datos a través de conexiones de red, a menudo llamada la "navaja suiza" de las redes.

```bash
# Dejar un puerto en escucha (listener)
nc -lnvp <puerto>
```
> [!info] Opciones de `nc` para un listener:
> *   `-l`: Modo de escucha (listener).
> *   `-n`: No resolver nombres de host (numérico).
> *   `-v`: Modo verboso (muestra más información).
> *   `-p <puerto>`: Especifica el puerto local para escuchar.
> Este comando es fundamental para establecer un "shell inverso" o para recibir datos en un puerto específico.

```bash
# Conectarse a un puerto remoto
nc <host> <puerto>
```
> [!info] Este comando permite establecer una conexión TCP a un host y puerto específicos. Una vez conectado, cualquier entrada de teclado se envía al host remoto y cualquier salida del host remoto se muestra en la terminal local. Es útil para:
> *   **Probar la conectividad de puertos**: Verificar si un puerto está abierto en un servidor remoto.
> *   **Banner Grabbing**: Obtener información sobre el servicio que se ejecuta en un puerto (ej. `nc google.com 80` y luego escribir `GET / HTTP/1.0`).
> *   **Transferencia de archivos**: Se puede usar para enviar o recibir archivos de forma sencilla.

## Verificar Comandos en Tiempo Real

El comando `watch` ejecuta repetidamente un comando y muestra su salida en pantalla completa, permitiendo observar cambios en tiempo real.

```bash
# Ejecutar un comando y ver su salida actualizada cada segundo
watch -n 1 <comando>
```
> [!tip] `watch` es extremadamente útil para monitorear cambios dinámicos.
> *   `-n <segundos>`: Especifica el intervalo de actualización en segundos.
> *   `<comando>`: El comando a ejecutar.
> Ejemplos de uso:
> *   `watch -n 1 ls -l`: Ver cambios en un directorio.
> *   `watch -n 1 'df -h'`: Monitorear el uso del disco.
> *   `watch -n 1 'cat /var/log/auth.log | tail -n 10'`: Observar las últimas líneas de un log.

## Ver Usuarios Conectados

Este comando muestra los usuarios que están actualmente logueados en el sistema.

```bash
# Ver usuarios conectados
who
```
> [!info] `who` muestra información sobre los usuarios actualmente logueados en el sistema, incluyendo el nombre de usuario, la terminal (TTY), la fecha y hora de inicio de sesión, y la dirección IP de la que se conectaron (si es remota).
> Otros comandos relacionados:
> *   `w`: Muestra los usuarios logueados y qué están haciendo.
> *   `last`: Muestra un historial de inicios de sesión y cierres de sesión en el sistema.

## Ver Variables de Entorno de mi Sesión

Este comando muestra todas las variables de entorno de la sesión actual.

```bash
# Ver variables de entorno de mi sesión
printenv
```
> [!info] Las variables de entorno son pares clave-valor que afectan el comportamiento de los procesos en el sistema. `printenv` muestra todas las variables de entorno definidas para el proceso actual.
> *   `PATH`: Define los directorios donde el shell busca ejecutables.
> *   `HOME`: El directorio principal del usuario.
> *   `USER`: El nombre de usuario actual.
> *   `SHELL`: El shell predeterminado del usuario.
> Otro comando similar es `env`, que también muestra las variables de entorno. `set` muestra variables de entorno y variables de shell.

## Ver la Versión del Kernel en el Sistema

Este comando muestra la versión del kernel de Linux en ejecución.

```bash
# Ver la versión del kernel en el sistema
uname -r
```
> [!info] `uname` (Unix Name) es una utilidad que imprime información del sistema.
> *   `-r`: Muestra la versión del kernel.
> *   `-a`: Muestra toda la información disponible (nombre del kernel, nombre del host, versión del kernel, tipo de procesador, plataforma de hardware, sistema operativo).
> La versión del kernel es crucial para identificar posibles vulnerabilidades o para compilar módulos específicos del kernel.