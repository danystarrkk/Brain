---
aliases:
  - Shellshock
  - Bash Vulnerability
tags:
  - hacking/explotacion
  - linux/seguridad
estado: 🟢 Terminado
---

# Shellshock: Vulnerabilidad en Bash

Un ataque **Shellshock** es un tipo de ataque informático que aprovecha una vulnerabilidad en el **intérprete de comandos Bash** en sistemas operativos basados en Unix y Linux. Esta vulnerabilidad se descubrió en 2014 y se considera uno de los ataques más grandes y generalizados en la historia de la informática.

> [!info] Identificador CVE
> La vulnerabilidad Shellshock es conocida oficialmente como **CVE-2014-6271** y sus variantes posteriores (como CVE-2014-6277, CVE-2014-6278, CVE-2014-7169, CVE-2014-7186, CVE-2014-7187). Estas CVEs describen fallos en la forma en que Bash procesa ciertas construcciones en las variables de entorno, permitiendo la ejecución de código arbitrario.

Esta vulnerabilidad en Bash permite a los atacantes ejecutar comandos maliciosos en el sistema afectado, lo que les permite tomar el control del sistema y acceder a información confidencial, modificar archivos, instalar programas maliciosos, etc.

## Mecanismo de Explotación

La vulnerabilidad Shellshock se produce en el intérprete de comandos Bash, que es utilizado por muchos sistemas operativos Unix y Linux para ejecutar scripts de shell. El problema radica en la forma en que Bash maneja las variables de entorno. Los atacantes pueden inyectar código malicioso en estas variables de entorno, las cuales Bash ejecuta sin cuestionar su origen o contenido.

> [!tip] Sintaxis de la Vulnerabilidad
> La vulnerabilidad se activa cuando Bash procesa una variable de entorno que comienza con una definición de función, seguida inmediatamente por comandos arbitrarios. La sintaxis maliciosa típica es:
> `VAR='() { :; }; COMMAND'`
> Donde `VAR` es el nombre de la variable de entorno, `() { :; };` es una definición de función Bash vacía, y `COMMAND` es el comando arbitrario que el atacante desea ejecutar. Bash, debido al fallo, ejecuta `COMMAND` después de procesar la definición de la función.

## Vector de Ataque Común: User-Agent

Los atacantes pueden explotar esta vulnerabilidad a través de diferentes vectores de ataque. Uno de ellos es a través del **User-Agent**, que es la información que el navegador web envía al servidor web para identificar el tipo de navegador y sistema operativo que se está utilizando. Los atacantes pueden manipular el User-Agent para incluir comandos maliciosos, que el servidor web ejecutará al recibir la solicitud.

> [!warning] Impacto en Servidores Web (CGI)
> Uno de los vectores de ataque más comunes para Shellshock fue a través de scripts CGI (Common Gateway Interface) en servidores web. Cuando un servidor web ejecuta un script CGI, a menudo pasa las cabeceras HTTP (como User-Agent, Referer, Cookie) como variables de entorno al script. Si el script CGI o el propio servidor web invoca Bash para procesar estas variables, la vulnerabilidad puede ser explotada.
>
> **Ejemplo de cabecera HTTP maliciosa:**
> ```
> User-Agent: () { :; }; /bin/bash -c "echo 'Vulnerable!' > /tmp/shellshock_test.txt"
> ```
> Si un servidor web vulnerable procesa esta cabecera, se crearía un archivo `/tmp/shellshock_test.txt` con el contenido "Vulnerable!".
