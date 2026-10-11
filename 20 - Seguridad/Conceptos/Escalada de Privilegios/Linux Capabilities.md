---
aliases: ["Capabilities Linux", "Linux Privileges"]
tags: ["linux/seguridad"]
estado: 🟢 Terminado
---
# Linux Capabilities

En sistemas Linux, las **capabilities** son una funcionalidad de seguridad que permite a los usuarios realizar acciones que normalmente requieren privilegios de superusuario (**root**), sin tener que concederles acceso completo de superusuario. Esto se hace para mejorar la seguridad, ya que un proceso que solo necesita ciertos privilegios puede obtenerlos sin tener que ejecutarse como `root`.

> [!info] Principio de Mínimo Privilegio
> Las capabilities implementan el principio de mínimo privilegio (PoLP), permitiendo una granularidad fina en la asignación de permisos. En lugar de un modelo binario (root/no-root), se pueden otorgar permisos específicos, reduciendo la superficie de ataque en caso de compromiso de un proceso.

Las capabilities se dividen en 3 tipos principales:

-   **Permisos efectivos (effective capabilities)**: Son los permisos que se aplican directamente al proceso que los posee. Estos permisos determinan las acciones que el proceso puede realizar. Por ejemplo, la capability "**CAP_NET_ADMIN**" permite al proceso modificar la configuración de red.
-   **Permisos heredados (inheritable capabilities)**: Son los permisos que se heredan por los procesos hijos que son creados. Estos permisos pueden ser adicionales a los permisos efectivos que ya posee el proceso padre. Por ejemplo, si un proceso padre tiene la capability "**CAP_NET_ADMIN**" y esta capability se configura como heredable, entonces los procesos hijos también tendrán la capability "**CAP_NET_ADMIN**".
-   **Permisos permitidos (permitted capabilities)**: Son los permisos que un proceso tiene permitidos. Esto incluye tanto permisos efectivos como heredados. Un proceso solo puede ejecutar acciones para las que tiene permisos permitidos. Por ejemplo, si un proceso tiene la capability "**CAP_NET_ADMIN**" y la capability "**CAP_SETUID**" configurada como permitida, entonces el proceso puede modificar la configuración de red y cambiar su **UID** (**User ID**).

Ahora bien, algunas capabilities pueden suponer un riesgo desde el punto de vista de la seguridad si se les asignan a determinados binarios. Por ejemplo, la capability **cap_setuid** permite a un proceso establecer el UID (User ID) de un proceso a otro valor diferente al suyo, lo que puede permitir que un usuario malintencionado ejecute código malicioso con privilegios elevados.

> [!warning] Riesgos de `cap_setuid`
> La capability `cap_setuid` es una de las más peligrosas si se asigna incorrectamente. Permite a un proceso cambiar su ID de usuario efectivo, lo que puede ser explotado para escalar privilegios a `root` si el proceso puede ejecutar código arbitrario o si se combina con otras vulnerabilidades. Es crucial auditar cuidadosamente los binarios con esta capability.

## Listar Capabilities con `getcap`

Para **listar** las capabilities de un archivo binario en Linux, puedes usar el comando **getcap**. Este comando muestra las capabilities efectivas, heredables y permitidas del archivo. Por ejemplo, para ver las capabilities del archivo binario **/usr/bin/ping**, puedes ejecutar el siguiente comando en la terminal:

```bash
getcap /usr/bin/ping
```

La salida del comando mostrará las capabilities asignadas al archivo:

```
/usr/bin/ping = cap_net_admin,cap_net_raw+ep
```

En este caso, el binario `ping` tiene dos capabilities asignadas: `cap_net_admin` y `cap_net_raw+ep`. La última capability (`cap_net_raw+ep`) indica que el archivo tiene el bit de ejecución elevado (**ep**) y la capability `cap_net_raw` asignada.

> [!tip] Binarios Comunes con Capabilities
> Además de `ping`, otros binarios comunes que suelen tener capabilities asignadas incluyen `traceroute` (`cap_net_raw`), `dumpcap` (parte de Wireshark, puede tener `cap_net_raw` y `cap_net_admin`), y a veces herramientas de red específicas. Siempre es buena práctica revisar las capabilities de binarios ejecutables en el sistema para identificar posibles vectores de escalada de privilegios.

## Asignar Capabilities con `setcap`

Para **asignar** una capability a un archivo binario, puedes utilizar el comando **setcap**. Este comando establece las capabilities efectivas, heredables y permitidas para el archivo especificado.

Por ejemplo, para otorgar la capability **cap_net_admin** al archivo binario **/usr/bin/my_program**, puedes ejecutar el siguiente comando en la terminal:

```bash
sudo setcap cap_net_admin+ep /usr/bin/my_program
```

En este caso, el comando otorga la capability `cap_net_admin` al archivo `/usr/bin/my_program`, y también establece el bit de ejecución elevado (**ep**). Ahora, el archivo `my_program` tendrá permisos para administrar la configuración de red.

> [!info] Implicaciones de `setcap`
> Asignar capabilities con `setcap` debe hacerse con extrema precaución. Otorgar capabilities a un binario es similar a otorgar permisos SUID/SGID, pero con un control más granular. Un binario comprometido con capabilities elevadas puede ser explotado para realizar acciones no autorizadas, incluso si el usuario que lo ejecuta no tiene privilegios de `root`.

## El Bit de Ejecución Elevado (`ep`)

El **bit de ejecución elevado** (en inglés, *elevated execution bit* o "**ep**") es un atributo especial que se puede establecer en un archivo binario en Linux. Este atributo se utiliza en conjunción con las capabilities para permitir que un archivo se ejecute con permisos especiales, incluso si el usuario que lo ejecuta no tiene privilegios de superusuario.

Cuando un archivo binario tiene el bit de ejecución elevado establecido, se puede ejecutar con las capabilities efectivas asignadas al archivo, en lugar de las capabilities del usuario que lo ejecuta. Esto significa que el archivo puede realizar acciones que normalmente solo están permitidas a los usuarios con privilegios elevados.

> [!info] `ep` vs. SUID/SGID
> El bit `ep` (effective/permitted) en capabilities es una alternativa más segura y granular a los bits SUID (Set User ID) y SGID (Set Group ID) tradicionales. Mientras que SUID/SGID otorgan los permisos completos del propietario/grupo del archivo, las capabilities con `ep` solo otorgan los permisos específicos definidos, reduciendo el riesgo de una escalada de privilegios completa.

Es importante señalar que los permisos permitidos pueden ser limitados aún más mediante el uso de un mecanismo de control de acceso obligatorio (MAC, *Mandatory Access Control*), como **SELinux** o **AppArmor**, que restringen las acciones que los procesos pueden realizar en función de la política de seguridad del sistema.

> [!info] Interacción con MAC (SELinux/AppArmor)
> Los sistemas MAC como SELinux y AppArmor operan a un nivel superior de control de acceso. Incluso si un proceso tiene una capability permitida, un MAC puede denegar la acción si no cumple con la política de seguridad definida. Esto proporciona una capa adicional de defensa, asegurando que las capabilities no puedan ser abusadas para eludir las políticas de seguridad del sistema.