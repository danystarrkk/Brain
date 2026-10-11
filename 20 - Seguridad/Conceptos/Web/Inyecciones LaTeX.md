---
aliases: ["LaTeX Injection"]
tags:
  - hacking/web
  - hacking/explotacion
estado: 🟢 Terminado
---
# Inyecciones LaTeX

Las **inyecciones LaTeX** son un tipo de ataque que se aprovecha de las vulnerabilidades en las aplicaciones web que permiten a los usuarios ingresar **texto formateado** en LaTeX. LaTeX es un sistema de composición de textos que se utiliza comúnmente en la escritura académica y científica.

> [!info] Mecanismo de Ataque
> Los ataques de inyección LaTeX ocurren cuando un atacante ingresa código LaTeX malicioso en un campo de entrada de texto que luego es procesado por una aplicación web. Este código LaTeX puede ser diseñado para explotar vulnerabilidades en la aplicación, permitiendo la **ejecución de código malicioso** en el servidor subyacente.

## Ejemplos de Explotación

Un ejemplo de una inyección LaTeX podría ser un ataque que aprovecha la capacidad de LaTeX para incluir gráficos y archivos externos en una aplicación web. Un atacante podría enviar un código LaTeX que incluya un enlace a un archivo malicioso, como un virus o un troyano, que podría infectar el servidor o los sistemas de la red.

> [!tip] Comandos LaTeX Maliciosos Comunes
> Algunos comandos LaTeX que pueden ser abusados en un ataque de inyección incluyen:
> *   `\input{/etc/passwd}`: Para leer archivos locales del sistema de archivos del servidor.
> *   `\includegraphics{/path/to/malicious.png}`: Para intentar cargar imágenes desde rutas arbitrarias, lo que podría ser utilizado para Server-Side Request Forgery (SSRF) o para intentar cargar archivos no-imagen.
> *   `\write18{command}`: Si la opción `\write18` (también conocida como `\shellescape`) está habilitada, permite la ejecución de comandos del sistema operativo directamente desde LaTeX. Por ejemplo, `\write18{ls -la /}`.
> *   `\url{http://malicious.com/payload}`: Aunque no ejecuta directamente, puede ser usado para exfiltración de datos o para cargar recursos externos maliciosos, dependiendo de cómo se procese la URL.

## Prevención y Mitigación

Para evitar las inyecciones LaTeX, las aplicaciones web deben validar y limpiar adecuadamente los datos que se reciben antes de procesarlos en LaTeX. Esto incluye la eliminación de caracteres especiales y la limitación de los comandos que pueden ser ejecutados en LaTeX.

> [!warning] Estrategias de Saneamiento
> Es crucial implementar una lista blanca (whitelist) de comandos y caracteres LaTeX permitidos, en lugar de una lista negra (blacklist). La lista negra es inherentemente insegura, ya que los atacantes pueden encontrar formas de evadirla. Se deben deshabilitar funciones peligrosas como `\write18` si no son estrictamente necesarias para la funcionalidad de la aplicación.

También es importante que las aplicaciones web se ejecuten con privilegios mínimos en la red y que se monitoreen regularmente las actividades de la aplicación para detectar posibles inyecciones. Además, se debe fomentar la educación sobre la seguridad en el uso de LaTeX y cómo evitar la introducción de código malicioso.

> [!info] Defensa en Profundidad
> Además del saneamiento de entradas, otras medidas de seguridad importantes incluyen:
> *   **Principio de Mínimo Privilegio**: Ejecutar el proceso de compilación de LaTeX con los permisos de usuario más restrictivos posibles para limitar el daño potencial de una explotación.
> *   **Sandboxing**: Aislar el entorno de compilación de LaTeX en un contenedor o máquina virtual para limitar el impacto de una posible explotación al resto del sistema.
> *   **Monitoreo y Logging**: Registrar y analizar los intentos de compilación de LaTeX y los errores generados para detectar patrones de ataque o actividades sospechosas.
> *   **Actualizaciones**: Mantener el motor LaTeX y las librerías de la aplicación actualizadas para parchear vulnerabilidades conocidas y asegurar que se utilizan las versiones más seguras.

## Laboratorio Práctico

A continuación, se os proporciona el enlace correspondiente al proyecto de Github que nos descargamos para desplegar un laboratorio vulnerable donde poder practicar esta vulnerabilidad: