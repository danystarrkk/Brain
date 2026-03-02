---
aliases:
  - RFI
  - Remote File Inclusion
tags:
  - hacking/web
  - hacking/explotacion
  - hacking/laboratorios
  - hacking/herramientas
estado: 🟢 Terminado
---

La vulnerabilidad **Remote File Inclusion** (**RFI**) es una falla de seguridad en la que un atacante puede **incluir archivos remotos** en una aplicación web vulnerable. Esto puede permitir al atacante ejecutar código malicioso en el servidor web y comprometer el sistema.

> [!info]
> La RFI ocurre cuando una aplicación web no valida adecuadamente las entradas del usuario que se utilizan para incluir archivos. Si un atacante puede controlar la ruta del archivo a incluir, puede forzar a la aplicación a cargar y ejecutar código desde un servidor remoto bajo su control.

En un ataque de RFI, el atacante utiliza una entrada del usuario, como una URL o un campo de formulario, para incluir un archivo remoto en la solicitud. Si la aplicación web no valida adecuadamente estas entradas, procesará la solicitud y devolverá el contenido del archivo remoto al atacante.

> [!tip]
> Las funciones de inclusión de archivos en lenguajes como PHP (`include()`, `require()`, `include_once()`, `require_once()`) son las principales causas de RFI si no se implementan con validación estricta. Es crucial filtrar y sanear todas las entradas de usuario que puedan afectar rutas de archivos.

Un atacante puede utilizar esta vulnerabilidad para incluir archivos remotos maliciosos que contienen código malicioso, como virus o troyanos, o para ejecutar comandos en el servidor vulnerable. En algunos casos, el atacante puede dirigir la solicitud hacia un recurso PHP alojado en un servidor de su propiedad, lo que le brinda un mayor grado de control en el ataque.

> [!warning]
> La RFI es una vulnerabilidad de alta severidad que a menudo conduce a la ejecución remota de código (RCE), permitiendo a un atacante tomar el control total del servidor web. Esto puede escalar a la exfiltración de datos, defacement del sitio o incluso el acceso a la red interna.

A continuación, se proporciona el enlace al proyecto de Github correspondiente al laboratorio que estaremos desplegando para practicar esta vulnerabilidad:

- **DVWP**: [https://github.com/vavkamil/dvwp](https://github.com/vavkamil/dvwp)

Asimismo, se os comparte el enlace directo para la descarga del plugin ‘**Gwolle Guestbook**‘ de WordPress:

- **Gwolle Guestbook**: [https://es.wordpress.org/plugins/gwolle-gb/](https://es.wordpress.org/plugins/gwolle-gb/)

## Notas de la Práctica

En este caso, utilizamos una máquina que modificamos ligeramente con el objetivo de hacerla vulnerable a este tipo de ataque. Lo primero que hicimos fue realizar fuzzing para determinar posibles plugins de WordPress en la aplicación web:

```bash
wfuzz -c --hc=404 -t 200 -w wp-plugins.fuzz.txt http://localhost:31337/FUZZ
```

> [!info]
> `wfuzz` es una herramienta potente para el fuzzing de aplicaciones web. El comando `wfuzz -c --hc=404 -t 200 -w wp-plugins.fuzz.txt http://localhost:31337/FUZZ` busca directorios o archivos de plugins de WordPress, ignorando las respuestas con código HTTP 404 (Not Found) y utilizando 200 hilos para una mayor velocidad. El diccionario `wp-plugins.fuzz.txt` contiene nombres comunes de plugins.

![[Pasted image 20250825164045.png]]

Como podemos observar, identificamos un plugin certero: `gwolle-gb`. Para este plugin, utilizaremos `searchsploit` para verificar si existe alguna vulnerabilidad conocida:

![[Pasted image 20250825164203.png]]

Como podemos observar, sí existe una vulnerabilidad. A continuación, analizaremos qué hace exactamente para explotarla:

![[Pasted image 20250825164448.png]]

La vulnerabilidad, al parecer, permite enviar peticiones a un servidor externo a través de la URL proporcionada.

![[Pasted image 20250825164437.png]]

Ahora, procederemos a cargar un archivo `wp-load.php` con el siguiente contenido:

```php
<?php
	system('whoami');
?>
```

> [!tip]
> Este payload PHP utiliza la función `system()` para ejecutar un comando del sistema operativo (`whoami`) y mostrar su salida. En un escenario real, se podría reemplazar `whoami` con comandos más avanzados para establecer una shell inversa, leer archivos sensibles o manipular el sistema de archivos.

![[Pasted image 20250825164648.png]]

Observamos que ya es posible ejecutar comandos.

> [!info]
> En resumen, la vulnerabilidad RFI permite que la aplicación web interprete y ejecute archivos externos que contienen código malicioso. En este laboratorio, logramos que la aplicación cargara un archivo PHP desde un servidor controlado por el atacante, lo que resultó en la ejecución remota de comandos y la confirmación de la identidad del usuario del servidor web (`whoami`).
