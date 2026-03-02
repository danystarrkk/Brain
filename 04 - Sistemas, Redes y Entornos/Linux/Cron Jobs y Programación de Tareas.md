---
aliases:
  - Cron Jobs
  - Tareas Programadas
  - Crontab
tags:
  - linux/administracion
  - linux/comandos
estado: 🟢 Terminado
---
# Cron Jobs y Programación de Tareas

Las tareas cron son un tipo de tareas que se ejecutan en intervalos de tiempo predefinidos en sistemas operativos tipo Unix/Linux. Saber interpretar este tipo de tareas puede parecer complicado, pero no es imposible. Aprenderemos a hacerlo, tomando en cuenta que su lógica es similar a la de las  y los .

## Estructura de una Tarea Cron

La estructura para comprender las tareas cron se basa en 5 campos (asteriscos), cada uno representando una unidad de tiempo:

![[Pasted image 20240608221843.png]]

Como vemos, los 5 asteriscos (`* * * * *`) representan que la tarea se ejecuta cada minuto de cada hora, todos los días del mes, todos los meses y todos los días de la semana.

> [!info] Granularidad de Cron
> Es importante destacar que la granularidad estándar de `cron` es por minuto. No es posible programar tareas directamente para que se ejecuten cada segundo utilizando la sintaxis estándar de `crontab`. Para ejecuciones sub-minuto, se suelen usar bucles (`sleep`) dentro de un script que se ejecuta cada minuto, o herramientas más avanzadas como `systemd timers`.

Con esto claro, debemos comprender que no solo se utilizan 5 asteriscos, y que cada posición representa un valor específico:

![[Pasted image 20240608221902.png]]

## Campos de Tiempo en Cron

Dependiendo del valor que se especifique en cada campo, se obtendrán diferentes patrones de ejecución.

### Ejemplo 1: Ejecución cada 5 minutos

Consideremos la siguiente configuración:

![[Pasted image 20240608221925.png]]

Como vemos, esta configuración (`*/5 * * * *`) indica que la tarea se ejecutará cada 5 minutos de cada hora. Es decir, se ejecutará a la 1:05, 1:10, 1:15, ..., 2:05, 2:10, etc., hasta terminar las 24 horas del día.

> [!tip] Operador de Paso (`/`)
> El operador de paso (`/`) se utiliza para especificar un intervalo de ejecución. La sintaxis es `*/N`, donde `N` es el paso. Por ejemplo:
> *   `*/5` en el campo de minutos significa "cada 5 minutos".
> *   `0-59/5` es equivalente a `*/5`.
> *   `*/2` en el campo de horas significa "cada 2 horas".
> Este operador es muy útil para programar tareas periódicas sin tener que listar cada valor individualmente.

### Ejemplo 2: Interpretación del Operador de Paso

![[Pasted image 20240608221952.png]]

Cada vez que veamos un asterisco seguido de una barra (`*/N`), significa 'cada N unidades'. Por lo tanto, `*/5` en el campo de minutos significa que la tarea se ejecutará cada 5 minutos.

> [!warning] Consideraciones de Seguridad
> Al configurar tareas cron, es crucial seguir las mejores prácticas de seguridad:
> *   **Ruta Completa**: Siempre especifica la ruta completa a los ejecutables y scripts para evitar problemas con el `PATH` y posibles ataques de secuestro de ruta.
> *   **Permisos**: Asegúrate de que los scripts ejecutados por cron tengan los permisos adecuados y que solo el usuario `root` o el propietario del script puedan modificarlos.
> *   **Salida de Comandos**: Redirige la salida de los comandos a un archivo de log (`>> /var/log/cron_task.log 2>&1`) para monitorear su ejecución y depurar errores, en lugar de enviarla por correo electrónico al usuario, lo cual puede ser ruidoso o fallar.
> *   **Privilegios Mínimos**: Ejecuta las tareas cron con el usuario con menos privilegios posible. Evita ejecutar todo como `root` a menos que sea estrictamente necesario.
> *   **Variables de Entorno**: Ten en cuenta que el entorno de ejecución de cron es mínimo. Define explícitamente cualquier variable de entorno necesaria dentro del crontab o en el script.