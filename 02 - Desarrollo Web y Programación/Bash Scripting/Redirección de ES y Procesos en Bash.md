---
aliases:
  - Redirección de E/S y Procesos en Bash
tags:
  - linux/bash
estado: 🟢 Terminado
---
# Redirección de Entrada/Salida y Gestión de Procesos en Bash

## Stderr (Standard Error)

Cuando hablamos del *stderr* (Standard Error) nos referimos a los mensajes de error que obtenemos al ejecutar comandos. El sistema los reconoce con el descriptor de archivo número 2. Podemos redirigirlos o suprimirlos de la siguiente manera:

```bash
whoam 2>/dev/null
```

> [!info] El descriptor de archivo `2` está reservado para `stderr`. El operador `>` se utiliza para redirigir la salida. En este caso, `2>/dev/null` redirige la salida de error del comando `whoam` (que es un comando incorrecto y generará un error) al dispositivo `/dev/null`.

La ruta `/dev/null` es un dispositivo especial en sistemas tipo Unix que actúa como un "agujero negro": cualquier dato escrito en él se descarta, y cualquier lectura de él devuelve EOF (End Of File) inmediatamente. Es útil para suprimir la salida no deseada. Con esta redirección, los errores no se mostrarán en la consola.

## Stdout (Standard Output)

Cuando hablamos del *stdout* (Standard Output) nos referimos a la salida estándar que un comando correctamente ejecutado muestra por consola. El sistema lo referencia con el descriptor de archivo número 1 (implícito). Para redirigir o suprimir esta salida, tenemos varias formas:

```bash
whoami > /dev/null   # Con esto solo vemos los errores en caso de producirse.
whoami > /dev/null 2>&1  # Con esto convertimos los errores en stdout y tampoco aparecen.
whoami &>/dev/null  # Con esto ya no vemos ni stdout ni el stderr.
```

> [!tip] **Explicación de las redirecciones de Stdout:**
> *   `whoami > /dev/null`: Redirige solo la salida estándar (`stdout`, descriptor 1) a `/dev/null`. Si el comando `whoami` generara un error, este se seguiría mostrando en la consola, ya que `stderr` (descriptor 2) no ha sido redirigido.
> *   `whoami > /dev/null 2>&1`: Esta es una secuencia común. Primero, `>` redirige `stdout` a `/dev/null`. Luego, `2>&1` redirige `stderr` (descriptor 2) al mismo destino que `stdout` (descriptor 1). Esto significa que tanto la salida estándar como los errores se envían a `/dev/null` y no aparecen en la consola. El orden es crucial: `2>&1` debe ir después de `> /dev/null` para que `stderr` se redirija al *nuevo* destino de `stdout`.
> *   `whoami &>/dev/null`: Este es un atajo moderno (disponible en Bash) para `> /dev/null 2>&1`. Redirige tanto `stdout` como `stderr` a `/dev/null` de forma concisa.

## Procesos en Segundo Plano

Para entender esto, es fundamental comprender la relación entre procesos padres e hijos. Cuando ejecutamos un comando desde nuestra consola (shell), la consola actúa como el proceso padre, y el comando ejecutado se convierte en un proceso hijo. Por ejemplo, si ejecutamos Firefox desde la consola, Firefox será un proceso hijo de nuestra shell.

Cuando ejecutamos Firefox, nuestra consola queda "bloqueada" o "inutilizable" mientras el proceso se ejecuta en primer plano. Para evitar esto y permitir que la consola siga siendo interactiva, podemos enviar el proceso a segundo plano utilizando el signo `&` al final del comando:

```bash
firefox &>/dev/null & 
```

En este ejemplo, se ha añadido la redirección `&>/dev/null` para suprimir cualquier salida (estándar o de error) en la consola, y el `&` final envía el proceso a segundo plano. Sin embargo, este proceso sigue siendo un hijo de la shell actual. Si cerramos la consola, el proceso padre (`shell`) terminará, y por defecto, sus procesos hijos (como Firefox) también recibirán una señal de terminación (SIGHUP) y se cerrarán. Para "independizar" el proceso hijo del proceso padre, utilizamos el comando `disown`:

```bash
bash -c "firefox &>/dev/null & disown"
```

Con `disown`, nuestro proceso hijo se independiza de la shell actual, lo que nos permite cerrar la consola sin que el proceso se termine.

> [!info] **Gestión de Procesos en Segundo Plano:**
> *   **`&` (Ampersand):** Envía un comando a segundo plano, permitiendo que la shell siga siendo interactiva. El proceso sigue siendo hijo de la shell y recibirá una señal SIGHUP si la shell se cierra.
> *   **`disown`:** Elimina un trabajo de la tabla de trabajos de la shell, impidiendo que la shell envíe una señal SIGHUP al proceso cuando se cierra. El proceso continuará ejecutándose incluso si la shell padre termina. Es importante ejecutar `disown` en la misma shell que lanzó el proceso, o en una subshell si se quiere independizar completamente desde el inicio, como en el ejemplo `bash -c "firefox &>/dev/null & disown"`.
> *   **`nohup`:** Un comando que se puede usar antes de otro comando para que este ignore la señal SIGHUP. Por ejemplo, `nohup firefox &`. Esto también independiza el proceso de la shell, pero a menudo redirige la salida a un archivo `nohup.out` por defecto si no se especifica otra redirección.

## Descriptores de Archivos (File Descriptors)

Los descriptores de archivos (File Descriptors, FD) son valores numéricos enteros que el kernel asigna a cada archivo o recurso de E/S abierto por un proceso. Los descriptores 0, 1 y 2 están reservados para `stdin` (entrada estándar), `stdout` (salida estándar) y `stderr` (error estándar) respectivamente. Podemos crear descriptores adicionales de la siguiente manera:

```bash
exec 3<> file
```

Esto abre o crea el archivo `file` y le asigna el descriptor de archivo `3`. El operador `<>` indica que el archivo se abre para lectura y escritura.

A través de este descriptor de archivo, podemos redirigir la salida estándar (`stdout`) o la salida de error (`stderr`) a nuestro archivo. Por ejemplo, para redirigir `stdout` al descriptor `3` (que apunta a `file`):

```bash
whoami >&3
```

Como se observa, la salida estándar (`stdout`) del comando `whoami` se redirigió al archivo `file` a través del descriptor `3`.

> [!warning] **Modo de Apertura y Redirección:**
> Cuando se redirige la salida a un archivo con `>` o `>&`, el archivo se trunca (su contenido se borra) si ya existe. Para añadir contenido al final del archivo (modo *append*) sin borrar el existente, se utiliza `>>` o `>>&`. Al usar descriptores de archivo personalizados con `exec`, el comportamiento de escritura dependerá de cómo se maneje el descriptor y las operaciones subsiguientes.

Para cerrar un descriptor de archivo específico, se utiliza la siguiente sintaxis:

```bash
exec 3>&-
```

Es importante destacar que esto solo cierra el descriptor de archivo, liberando el recurso; no elimina el archivo físico.

Esta es la base de los descriptores de archivo, pero con ellos se pueden realizar operaciones más avanzadas. A continuación, se presentan algunos ejemplos:

```bash
exec 5<> data # Abre el archivo 'data' para lectura y escritura, asignándole el descriptor 5.
```

```bash
exec 8>&5 # Crea una copia del descriptor de archivo 5 y la asigna al descriptor 8. Cualquier escritura en el descriptor 8 también se reflejará en el archivo original asociado al descriptor 5.
```

```bash
exec 5<> files # Abre el archivo 'files' para lectura y escritura, asignándole el descriptor 5.
```

```bash
exec 6>&5- # Crea una copia del descriptor de archivo 5 y la asigna al descriptor 6, y luego cierra el descriptor de archivo 5 original.
```

> [!info] **El comando `exec` para descriptores de archivo:**
> El comando `exec` en Bash es muy potente para manipular descriptores de archivo.
> *   `exec N<> file`: Abre `file` en el descriptor `N` para lectura y escritura.
> *   `exec N> file`: Abre `file` en el descriptor `N` para escritura (truncando si existe).
> *   `exec N< file`: Abre `file` en el descriptor `N` para lectura.
> *   `exec N>&M`: Redirige el descriptor `N` al mismo destino que el descriptor `M`.
> *   `exec N<&M`: Redirige el descriptor `N` al mismo origen que el descriptor `M`.
> *   `exec N>&-`: Cierra el descriptor `N`.
> Estas operaciones modifican el entorno de E/S de la shell actual o del script, afectando a todos los comandos subsiguientes.