---
aliases:
  - SUID
  - Set User ID
  - Privilegios SUID
tags:
  - linux/seguridad
  - hacking/escalada
estado: 🟢 Terminado
---
# Privilegios SUID (Set User ID) en Linux

## Introducción a SUID

Un privilegio **SUID** (**Set User ID**) es un permiso especial que se puede establecer en un archivo binario ejecutable en sistemas Unix/Linux. Este permiso otorga al usuario que ejecuta el archivo los **mismos privilegios** que el **propietario** del archivo, pero solo durante la ejecución de dicho binario.

Por ejemplo, si un archivo binario tiene establecido el permiso SUID y es propiedad del usuario `root`, cualquier usuario que lo ejecute adquirirá temporalmente los mismos privilegios que el usuario `root`. Esto le permitirá realizar acciones que normalmente no podría hacer como un usuario normal, como modificar archivos del sistema o ejecutar comandos restringidos.

> [!info] Funcionamiento de SUID
> Cuando un archivo con el bit SUID se ejecuta, el ID de usuario efectivo (EUID) del proceso se cambia al ID de usuario del propietario del archivo, mientras que el ID de usuario real (RUID) permanece como el del usuario que inició el proceso. Esto permite que el programa realice operaciones privilegiadas sin que el usuario tenga esos privilegios de forma permanente. Un ejemplo clásico es el comando `passwd`, que necesita escribir en `/etc/shadow` (propiedad de `root`) para cambiar contraseñas, pero es ejecutado por usuarios normales.

## Abuso de Privilegios SUID

El abuso de privilegios SUID es una técnica comúnmente utilizada por los atacantes para elevar su nivel de acceso en un sistema comprometido. Si un atacante es capaz de obtener acceso a un archivo binario con permisos SUID que tiene una vulnerabilidad o puede ser manipulado para ejecutar comandos arbitrarios, puede ejecutar comandos con privilegios especiales y realizar acciones maliciosas en el sistema.

> [!warning] Riesgos de Seguridad
> La presencia de binarios SUID mal configurados o vulnerables es una de las vías más comunes para la escalada de privilegios locales en sistemas Linux. Un atacante con acceso de usuario limitado puede explotar estos binarios para obtener privilegios de `root`.

## Prevención del Abuso de Privilegios SUID

Para prevenir el abuso de privilegios SUID, se recomienda limitar el número de archivos con permisos SUID y asegurarse de que solo se otorguen a archivos que requieran este permiso para funcionar correctamente. Además, es importante monitorear regularmente el sistema para detectar cambios inesperados en los permisos de los archivos y para buscar posibles brechas de seguridad.

> [!tip] Mejores Prácticas de Seguridad
> *   **Auditoría Regular**: Realizar auditorías periódicas para identificar todos los archivos SUID en el sistema.
> *   **Principio de Mínimo Privilegio**: Solo otorgar el bit SUID a binarios esenciales y bien revisados.
> *   **Parches y Actualizaciones**: Mantener el sistema operativo y todas las aplicaciones actualizadas para mitigar vulnerabilidades conocidas en binarios SUID legítimos.
> *   **Restricción de PATH**: Asegurarse de que el `PATH` de los usuarios no privilegiados no incluya directorios donde puedan colocar binarios maliciosos que podrían ser ejecutados por un programa SUID.

## Detección y Ejemplos de Explotación

### Cómo encontrar archivos SUID

Para identificar archivos con el bit SUID establecido en un sistema Linux, se puede utilizar el comando `find`.

```bash
find / -perm -4000 -type f 2>/dev/null
```

*   `find /`: Busca desde el directorio raíz.
*   `-perm -4000`: Busca archivos con el bit SUID establecido (el `4` en `4000` representa el bit SUID).
*   `-type f`: Limita la búsqueda a archivos regulares (excluye directorios, enlaces simbólicos, etc.).
*   `2>/dev/null`: Redirige los errores (como "Permiso denegado" en directorios a los que no se tiene acceso) a `/dev/null` para una salida más limpia.

> [!info] Explicación del Permiso `4000`
> En el sistema de permisos octales de Unix, el primer dígito representa permisos especiales:
> *   `4`: SUID (Set User ID)
> *   `2`: SGID (Set Group ID)
> *   `1`: Sticky Bit
> Por lo tanto, `-perm -4000` busca archivos donde el bit SUID está *activado*.

### Ejemplos Comunes de Binarios SUID Explotables

Algunos binarios SUID son conocidos por tener vulnerabilidades o por permitir la ejecución de comandos arbitrarios si se manipulan correctamente. Sitios como [GTFOBins](https://gtfobins.github.io/) documentan cómo explotar binarios Unix/Linux para escalar privilegios.

Ejemplos de binarios que a menudo se encuentran con SUID y pueden ser explotados:

*   **`find`**: Si `find` tiene SUID, se puede usar para ejecutar comandos como `root`.

```bash
    find . -exec /bin/sh -p \; -quit
```

El `-p` en `/bin/sh -p` es crucial, ya que invoca una shell privilegiada que mantiene los permisos efectivos del proceso padre (en este caso, `root`).

*   **`nmap`**: Versiones antiguas de `nmap` con SUID permitían ejecutar comandos arbitrarios.

```bash
    nmap --interactive
    nmap> !sh
```

*   **`vim` / `less` / `more`**: Algunos editores o paginadores de texto pueden permitir la ejecución de comandos de shell si tienen SUID.

```bash
    vim -c ':!/bin/sh'
```

*   **`bash`**: Si `bash` tiene SUID, es una vulnerabilidad crítica.

```bash
    bash -p
```

El flag `-p` previene la degradación de privilegios al iniciar la shell, manteniendo el EUID del propietario del archivo (root).

> [!warning] Verificación de Binarios SUID
> Siempre se debe verificar la legitimidad y la configuración de seguridad de cualquier binario SUID encontrado en un sistema. Un binario SUID que no es esencial o que tiene una vulnerabilidad conocida representa un riesgo significativo.
