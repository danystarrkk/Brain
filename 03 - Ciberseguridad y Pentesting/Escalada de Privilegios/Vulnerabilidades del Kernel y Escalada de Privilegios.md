---
aliases: ["Vulnerabilidades del Kernel", "Escalada de Privilegios Linux"]
tags:
  - linux/seguridad
  - hacking/escalada
estado: 🟢 Terminado
---

# Vulnerabilidades del Kernel y Escalada de Privilegios

## ¿Qué es el Kernel?

El **kernel** es la parte central del sistema operativo Linux, que se encarga de administrar los recursos del sistema, como la memoria, los procesos, los archivos y los dispositivos. Debido a su papel crítico en el sistema, cualquier vulnerabilidad en el kernel puede tener graves consecuencias para la seguridad del sistema.

> [!info] Componentes Clave del Kernel
> El kernel de Linux se compone de varios subsistemas principales, incluyendo:
> *   **Administrador de Procesos (Scheduler)**: Gestiona la ejecución de los procesos y la asignación de tiempo de CPU.
> *   **Administrador de Memoria**: Controla la asignación y desasignación de memoria a los procesos, incluyendo la memoria virtual.
> *   **Sistema de Archivos Virtual (VFS)**: Proporciona una interfaz unificada para diferentes sistemas de archivos.
> *   **Controladores de Dispositivo**: Permiten la interacción con el hardware.
> *   **Subsistema de Red**: Gestiona las comunicaciones de red.
> Una vulnerabilidad en cualquiera de estos subsistemas puede ser crítica.

## Vulnerabilidades del Kernel

En versiones antiguas del kernel de Linux, se han descubierto vulnerabilidades que pueden ser explotadas para permitir a los atacantes obtener acceso de superusuario (**root**) en el sistema.

> [!warning] Tipos Comunes de Vulnerabilidades del Kernel
> Las vulnerabilidades del kernel no se limitan solo a versiones antiguas. Aunque las versiones modernas incorporan muchas mitigaciones, aún pueden surgir nuevos fallos. Algunos tipos comunes incluyen:
> *   **Desbordamientos de Búfer (Buffer Overflows)**: Escribir más datos de los que un búfer puede contener, sobrescribiendo memoria adyacente.
> *   **Uso Después de Liberar (Use-After-Free - UAF)**: Acceder a memoria que ha sido liberada y potencialmente reasignada para otro propósito.
> *   **Condiciones de Carrera (Race Conditions)**: Fallos que ocurren cuando la salida de un programa depende de la secuencia o el tiempo de otros eventos incontrolables.
> *   **Errores de Puntero Nulo (Null Pointer Dereferences)**: Intentar desreferenciar un puntero que apunta a `NULL`, lo que puede causar una denegación de servicio o, en ciertos contextos, ser explotable.
> *   **Fugas de Información (Information Leaks)**: Exponer direcciones de memoria del kernel u otra información sensible que puede ayudar a eludir mitigaciones como ASLR.

## Escalada de Privilegios

La elevación de privilegios se refiere a la técnica utilizada por los atacantes para obtener permisos elevados en el sistema, como superusuario (**root**), cuando solo tienen permisos limitados. Por ejemplo, un usuario con permisos limitados en el sistema podría utilizar una vulnerabilidad en el kernel para obtener acceso de superusuario y, posteriormente, comprometer el sistema.

> [!info] Importancia de la Escalada de Privilegios
> La escalada de privilegios es un paso crítico en la mayoría de los ataques avanzados. Un atacante que obtiene una shell inicial con bajos privilegios buscará inmediatamente una forma de escalar a `root` o `SYSTEM` (en Windows) para:
> *   Instalar backdoors persistentes.
> *   Acceder a datos sensibles de otros usuarios o del sistema.
> *   Modificar la configuración del sistema o los logs para ocultar su presencia.
> *   Pivotar a otras máquinas en la red con credenciales o hashes extraídos.

## Métodos de Explotación

Las vulnerabilidades del kernel pueden ser explotadas de varias maneras. Por ejemplo, un atacante podría aprovechar una vulnerabilidad en un controlador de dispositivo para obtener acceso al kernel y realizar operaciones maliciosas. Otra forma común en que se explotan las vulnerabilidades del kernel es mediante el uso de técnicas de desbordamiento de búfer, que permiten a los atacantes escribir código malicioso en áreas de memoria reservadas para el kernel.

> [!tip] Desbordamientos de Búfer en el Kernel
> Un desbordamiento de búfer en el contexto del kernel es particularmente peligroso. Si un atacante logra sobrescribir una dirección de retorno o un puntero de función dentro del espacio de memoria del kernel, podría redirigir la ejecución a su propio código malicioso. Este código, ejecutándose en modo kernel, tendría control total sobre el sistema. Las técnicas comunes incluyen:
> *   **Ret2usr (Return-to-User)**: Redirigir la ejecución a código shellcode inyectado en el espacio de usuario.
> *   **ROP (Return-Oriented Programming)**: Encadenar pequeños fragmentos de código existentes en el kernel (gadgets) para lograr la ejecución deseada, eludiendo la protección de no-ejecución (NX/DEP).
> *   **Modificación de Estructuras de Datos del Kernel**: Alterar estructuras como `cred` o `task_struct` para cambiar los UID/GID de un proceso a 0 (root).

## Mitigación

Para mitigar el riesgo de vulnerabilidades del kernel, es importante mantener actualizado el sistema operativo y aplicar parches de seguridad tan pronto como estén disponibles.

> [!info] Estrategias de Mitigación Avanzadas
> Además de mantener el sistema actualizado, existen varias mitigaciones a nivel de kernel y sistema operativo diseñadas para dificultar la explotación:
> *   **ASLR (Address Space Layout Randomization)**: Aleatoriza las direcciones de memoria de las bibliotecas, el stack y el heap para dificultar la predicción de direcciones.
> *   **KASLR (Kernel Address Space Layout Randomization)**: Una variante de ASLR aplicada al kernel, aleatorizando la dirección base del kernel en cada arranque.
> *   **DEP/NX (Data Execution Prevention/No-Execute)**: Marca las páginas de memoria como no ejecutables, impidiendo la ejecución de código desde áreas de datos (como el stack o el heap).
> *   **SMEP (Supervisor Mode Execution Prevention)**: Impide que el kernel ejecute código en páginas de memoria marcadas como de usuario.
> *   **SMAP (Supervisor Mode Access Prevention)**: Impide que el kernel acceda a datos en páginas de memoria marcadas como de usuario.
> *   **Grsecurity/PaX**: Conjunto de parches de seguridad para el kernel de Linux que implementan una variedad de protecciones avanzadas.
> *   **SELinux/AppArmor**: Módulos de seguridad que implementan control de acceso obligatorio (MAC) para restringir lo que los procesos pueden hacer, incluso si se comprometen.
> A pesar de estas mitigaciones, los atacantes buscan constantemente nuevas formas de eludirlas, lo que subraya la importancia de la investigación continua y la aplicación de parches.