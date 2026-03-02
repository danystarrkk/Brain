---
aliases:
  - Linux Smart Enumeration
  - LSE
  - Permisos Linux
tags:
  - linux/seguridad
  - hacking/escalada
  - hacking/herramientas
estado: 🟢 Terminado
---
# Abuso de Permisos en Linux y LSE

En sistemas Linux, los archivos y directorios tienen permisos que se utilizan para controlar el acceso a ellos. Los permisos se dividen en tres categorías principales: **propietario**, **grupo** y **otros**. Cada categoría puede tener permisos de **lectura (r)**, **escritura (w)** y **ejecución (x)**. Los permisos de un archivo pueden ser modificados por el propietario o por el superusuario del sistema.

> [!info] Representación de Permisos
> Los permisos se representan comúnmente de dos formas:
> *   **Simbólica**: `rwx` para el propietario, `r-x` para el grupo, `r--` para otros. Por ejemplo, `chmod u=rwx,g=rx,o=r archivo.txt`.
> *   **Octal**: Un número de tres o cuatro dígitos. Cada permiso (r, w, x) tiene un valor numérico: `r=4`, `w=2`, `x=1`. La suma de estos valores para cada categoría forma el dígito octal. Por ejemplo, `rwx` es `4+2+1=7`, `r-x` es `4+0+1=5`, `r--` es `4+0+0=4`. Así, `rwxr-xr--` se representa como `754`. El primer dígito opcional (ej. `4755`) se refiere a permisos especiales como SUID, SGID o Sticky Bit.

El abuso de permisos incorrectamente implementados ocurre cuando los permisos de un archivo crítico son configurados de forma errónea, permitiendo a un usuario no autorizado acceder o modificar el archivo. Esto puede permitir a un atacante leer información confidencial, modificar archivos importantes, ejecutar comandos maliciosos o incluso obtener acceso de superusuario (root) al sistema.

> [!warning] Riesgos de Permisos Incorrectos
> Un archivo con permisos de escritura para "otros" (`o+w`) en una ubicación crítica (como `/etc/passwd` o scripts de inicio) puede ser modificado por cualquier usuario del sistema, lo que podría llevar a la creación de nuevos usuarios, la modificación de contraseñas o la inyección de código malicioso que se ejecute con privilegios elevados.

De esta forma, un atacante experimentado podría aprovecharse de esta falla para elevar sus privilegios en el mejor de los casos. Una de las herramientas encargadas de aplicar este reconocimiento en el sistema es '**lse**'. Linux Smart Enumeration (LSE) es una herramienta de enumeración de seguridad para sistemas operativos basados en Linux, diseñada para ayudar a los administradores de sistemas y auditores de seguridad a identificar y evaluar vulnerabilidades y debilidades en la configuración del sistema.

LSE está diseñado para ser fácil de usar y proporciona una salida clara y legible para facilitar la identificación de problemas de seguridad. La herramienta utiliza comandos de Linux estándar y se ejecuta en la línea de comandos, lo que significa que no se requiere software adicional. Además, enumera una amplia gama de información del sistema, incluyendo usuarios, grupos, servicios, puertos abiertos, tareas programadas (cron jobs), permisos de archivos, variables de entorno y configuraciones del firewall, entre otros.

> [!tip] Uso Básico de LSE
> Para ejecutar LSE, generalmente se descarga el script y se le otorgan permisos de ejecución.
> ```bash
> git clone https://github.com/diego-treitos/linux-smart-enumeration.git
> cd linux-smart-enumeration
> chmod +x lse.sh
> ./lse.sh -l 0 # Ejecuta LSE con nivel de verbosidad 0 (menos ruidoso)
> ```
> LSE ofrece diferentes niveles de verbosidad (`-l`) y opciones para enfocarse en tipos específicos de enumeración, lo que lo hace muy versátil para auditorías de seguridad.

A continuación, se os proporciona el enlace directo al proyecto de GitHub correspondiente a esta herramienta:

- **Linux Smart Enumeration**: [https://github.com/diego-treitos/linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration)