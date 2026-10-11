---
aliases:
  - PATH Hijacking
  - Secuestro de PATH
tags:
  - linux/seguridad
  - hacking/escalada
estado: 🟢 Terminado
---
# PATH Hijacking

**PATH Hijacking** es una técnica utilizada por los atacantes para **secuestrar comandos** de un sistema Unix/Linux mediante la manipulación de la variable de entorno **PATH**. La variable PATH es una variable de entorno que define las rutas de búsqueda para los archivos ejecutables en el sistema.

> [!info] La variable de entorno `PATH`
> La variable `PATH` es una lista de directorios que el sistema operativo busca para encontrar un comando cuando este se ejecuta sin una ruta absoluta. Los directorios se listan en un orden específico, separados por dos puntos (`:`). Cuando un usuario escribe un comando, el shell busca en cada directorio de izquierda a derecha hasta que encuentra un ejecutable con ese nombre. Por ejemplo, si `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`, el sistema buscará primero en `/usr/local/sbin`, luego en `/usr/local/bin`, y así sucesivamente.

En algunos binarios compilados, algunos de los comandos definidos internamente pueden ser indicados con una ruta relativa en lugar de una ruta absoluta. Esto significa que el binario busca los archivos ejecutables en las rutas especificadas en la variable PATH, en lugar de utilizar la ruta absoluta del archivo ejecutable.

Si un atacante es capaz de alterar la variable PATH y crear un nuevo archivo con el mismo nombre de uno de los comandos definidos internamente en el binario, puede lograr que el binario ejecute la versión maliciosa del comando en lugar de la versión legítima.

Por ejemplo, si un binario compilado utiliza el comando “**ls**” sin su ruta absoluta en su código y el atacante crea un archivo malicioso llamado “**ls**” en una de las rutas especificadas en la variable PATH, el binario ejecutará el archivo malicioso en lugar del comando legítimo “**ls**” cuando sea llamado.

> [!warning] Escenarios de Explotación Comunes
> El PATH Hijacking es particularmente efectivo en escenarios donde:
> *   Un usuario tiene permisos para modificar su propia variable `PATH` o la de un proceso específico.
> *   Un programa con privilegios elevados (como un binario SUID) ejecuta comandos con rutas relativas. Si un atacante puede inyectar una ruta maliciosa al inicio del `PATH` de ese proceso, puede lograr la escalada de privilegios.
> *   Scripts o programas que no sanitizan adecuadamente el entorno antes de ejecutar comandos externos.

Para prevenir el PATH Hijacking, se recomienda utilizar **rutas absolutas** en lugar de rutas relativas en los comandos definidos internamente en los binarios compilados. Además, es importante asegurarse de que las rutas en la variable PATH sean controladas y limitadas a las rutas necesarias para el sistema. También se recomienda utilizar la opción de permisos de ejecución para los archivos ejecutables solo para los usuarios y grupos autorizados.

> [!tip] Mejores Prácticas de Seguridad
> *   **Uso de rutas absolutas**: Siempre que sea posible, especifique la ruta completa a los ejecutables (ej. `/bin/ls` en lugar de `ls`).
> *   **Sanitización del entorno**: Los programas que ejecutan comandos externos deben limpiar la variable `PATH` o establecerla a un valor seguro y conocido antes de la ejecución.
> *   **Principio de menor privilegio**: Limitar los permisos de los usuarios y los programas para modificar variables de entorno críticas.
> *   **Verificación de integridad**: Implementar mecanismos para verificar la integridad de los ejecutables críticos del sistema.
> *   **Restricción de directorios en PATH**: Evitar añadir directorios de usuario o directorios con permisos de escritura para usuarios no privilegiados al `PATH` global o de usuarios con privilegios.