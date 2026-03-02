---
aliases:
  - Elevación de Privilegios Binarios
  - Exim CVE-2016-1531
  - Buffer Overflow SUID
tags:
  - hacking/escalada
  - hacking/explotacion
  - hacking/laboratorios
estado: 🟢 Terminado
---
# Elevación de Privilegios mediante Explotación de Binarios

En esta clase, analizaremos cómo elevar nuestros privilegios de usuario mediante la explotación de dos binarios diferentes como ejemplos ilustrativos en entornos de laboratorio controlados.

> [!info] Contexto de la Clase
> Esta sesión está diseñada para demostrar la importancia de una configuración y protección adecuadas en los sistemas, mostrando cómo binarios legítimos y personalizados pueden ser explotados para obtener privilegios elevados.

## Explotación de Exim (CVE-2016-1531)

El primer ejemplo se enfoca en explotar el binario legítimo **exim-4.84-7**, que presenta una vulnerabilidad identificada como **CVE-2016-1531**. Esta vulnerabilidad permite a un atacante ejecutar comandos privilegiados mediante el abuso de ciertas variables de entorno. Estudiaremos cómo aprovechar esta vulnerabilidad para escalar privilegios y acceder a funciones restringidas.

> [!info] CVE-2016-1531 en Exim
> La vulnerabilidad CVE-2016-1531 en Exim 4.84-7 (y versiones anteriores) es un fallo de escalada de privilegios local que permite a un usuario no privilegiado ejecutar código arbitrario como `root`. Esto se debe a un manejo inadecuado de las variables de entorno `PATH` y `LD_PRELOAD` cuando Exim se ejecuta con privilegios elevados (por ejemplo, como SUID `root`). Un atacante puede manipular estas variables para cargar bibliotecas maliciosas o ejecutar binarios arbitrarios.

### Máquina de Laboratorio para Exim

A continuación, se comparte el enlace de descarga de la máquina **Pluck** de Vulnhub, la cual estaremos utilizando para representar el primer caso de explotación:

- **Máquina Pluck de Vulnhub**: [https://www.vulnhub.com/entry/pluck-1,178/](https://www.vulnhub.com/entry/pluck-1,178/)

## Explotación de Buffer Overflow (ret2libc) en Binario Personalizado

El segundo ejemplo aborda un **Buffer Overflow** en un binario personalizado en una máquina **Linux** de **32 bits** con **protecciones activas** y **ASLR** (Address Space Layout Randomization) habilitado. En este caso, nos centraremos en explotar un **ret2libc** en un binario que posee permisos SUID y cuyo propietario es `root`. A través del buffer overflow, demostraremos cómo inyectar comandos privilegiados y, en consecuencia, elevar los privilegios de usuario.

> [!tip] Ret2libc y ASLR
> **Ret2libc** (Return-to-libc) es una técnica de explotación de buffer overflow que, en lugar de inyectar shellcode directamente, redirige el flujo de ejecución del programa a funciones existentes en la biblioteca estándar de C (libc), como `system()` o `execve()`. Esto es particularmente útil cuando la pila no es ejecutable (protección NX/DEP). Cuando ASLR está habilitado, las direcciones de libc se aleatorizan, lo que requiere técnicas adicionales como la fuga de direcciones (information leak) o el uso de gadgets ROP para determinar las direcciones de las funciones de libc en tiempo de ejecución.

### Entorno de Laboratorio para Buffer Overflow

Por otro lado, se comparte el enlace directo de descarga para Ubuntu 16.04.7 LTS (Xenial Xerus), un sistema operativo adecuado para replicar este escenario:

- **Ubuntu 16.04.7**: [https://releases.ubuntu.com/16.04/](https://releases.ubuntu.com/16.04/)

Por último, se comparte el enlace de descarga al binario el cual estaremos explotando para este segundo caso. Este binario es necesario que lo depositéis en alguna ruta del sistema y que le otorguéis permisos de ejecución:

- **Binario CUSTOM**: [https://hack4u.io/wp-content/uploads/2023/04/custom](https://hack4u.io/wp-content/uploads/2023/04/custom)

> [!warning] Instrucciones para el Binario CUSTOM
> Una vez descargado el binario `custom`, es crucial **quitarle la extensión `.txt`** si el navegador la añade automáticamente. Posteriormente, asegúrate de otorgarle permisos de ejecución (`chmod +x custom`) y, para el escenario de escalada de privilegios, configúralo con el bit SUID y como propiedad de `root` (`sudo chown root:root custom && sudo chmod u+s custom`).