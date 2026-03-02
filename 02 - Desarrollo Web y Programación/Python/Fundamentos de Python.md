---
aliases: 
  - Python
  - Historia de Python
  - Características de Python
  - Python 2 vs Python 3
  - PIP2 vs PIP3
tags:
  - programacion/python
estado: 🟢 Terminado
---
# Historia de Python

Python es un lenguaje de programación de alto nivel, interpretado y con una filosofía que enfatiza una sintaxis que favorece un código legible. Fue creado por Guido van Rossum en el Centro para las Matemáticas y la Informática (CWI) en los Países Bajos, y su lanzamiento inicial fue en 1991. Van Rossum modeló Python con la idea de superar las limitaciones del lenguaje ABC, buscando crear un lenguaje que pudiera hacer todo lo que ABC hacía, pero con una sintaxis clara que fuera fácil de aprender y de usar.

> [!info]
> El lenguaje ABC fue desarrollado en el CWI como un reemplazo para BASIC, diseñado para la enseñanza y la creación de prototipos. Aunque innovador en su momento, carecía de la extensibilidad y la capacidad de manejar excepciones que Guido van Rossum buscaba para Python.

Desde su concepción, Python ha estado enfocado en el principio de legibilidad y simplicidad. Esto no solo hace que sea más fácil de aprender, sino que también permite a los programadores expresar conceptos complejos en menos líneas de código en comparación con otros lenguajes de programación.

Python se ha desarrollado bajo una licencia de código abierto, lo que significa que cualquiera puede contribuir al desarrollo del lenguaje. Esta filosofía colaborativa ha ayudado a Python a crecer y adaptarse rápidamente a las cambiantes necesidades de la industria de la tecnología.

Uno de los documentos más importantes en la comunidad de Python es el ‘**PEP 20**‘, conocido como “**El Zen de Python**“. Este documento contiene 19 aforismos que capturan la esencia de la filosofía de Python. Algunos ejemplos notables incluyen “Lo bello es mejor que lo feo”, “Explícito es mejor que implícito” y “Si la implementación es difícil de explicar, es una mala idea”.

> [!tip]
> Las **PEPs (Python Enhancement Proposals)** son documentos de diseño que proporcionan información a la comunidad de Python, o describen una nueva característica para Python, sus procesos o su entorno. Son el mecanismo principal para proponer nuevas características, recopilar aportaciones de la comunidad y documentar las decisiones de diseño para Python.

La filosofía de Python también enfatiza la importancia de la eficiencia y la practicidad. Python se ha diseñado para ser altamente extensible, lo que permite a los programadores escribir nuevos módulos y extensiones en C o C++ para realizar operaciones críticas rápidamente.

> [!info]
> La implementación de referencia de Python, conocida como **CPython**, está escrita en C. Esto permite que Python se integre fácilmente con bibliotecas y sistemas existentes escritos en C/C++. Sin embargo, CPython utiliza un **Global Interpreter Lock (GIL)**, que permite que solo un hilo nativo ejecute bytecode de Python a la vez, incluso en sistemas con múltiples núcleos. Esto puede limitar el rendimiento en tareas intensivas en CPU que requieren paralelismo real, aunque no afecta a las operaciones de E/S (entrada/salida).

En resumen, Python es más que un lenguaje de programación; es una comunidad y una filosofía de diseño que continúa guiando su desarrollo. Con su énfasis en la claridad, la colaboración y la eficiencia, Python se ha solidificado como uno de los lenguajes de programación más populares y queridos del mundo.

# Características y Ventajas de Python

Python es un lenguaje de programación de alto nivel, interpretado y de propósito general que se ha popularizado por su sintaxis legible y clara. Es un lenguaje versátil que permite a los programadores trabajar rápidamente e integrar sistemas de manera más efectiva.

## Características Principales

*   **Sintaxis simple y fácil de aprender**: Python es famoso por su legibilidad, lo que facilita el aprendizaje para los principiantes y permite a los desarrolladores expresar conceptos complejos en menos líneas de código que serían necesarias en otros lenguajes.
*   **Interpretado**: Python es procesado en tiempo de ejecución por el intérprete. Puedes ejecutar el programa tan pronto como termines de escribir los comandos, sin necesidad de compilar.

    > [!info]
    > Aunque Python es un lenguaje interpretado, el código fuente no se ejecuta directamente. Primero, el intérprete de Python lo compila en un formato intermedio llamado **bytecode** (archivos `.pyc`). Este bytecode es luego ejecutado por la Máquina Virtual de Python (PVM). Esta compilación a bytecode ocurre automáticamente y es transparente para el usuario, lo que da la impresión de que el lenguaje es puramente interpretado.

*   **Tipado dinámico**: Python sigue las variables en tiempo de ejecución, lo que significa que puedes cambiar el tipo de datos de una variable en tus programas.

    > [!tip]
    > El tipado dinámico contrasta con el **tipado estático**, donde el tipo de una variable se declara explícitamente y se verifica en tiempo de compilación. En Python, no necesitas declarar el tipo de una variable; el intérprete lo infiere en tiempo de ejecución. Por ejemplo:
    > ```python
    > x = 10          # x es un entero
    > x = "hola"      # Ahora x es una cadena
    > ```

*   **Multiplataforma**: Python se puede ejecutar en una variedad de sistemas operativos como Windows, Linux y macOS.
*   **Bibliotecas extensas**: Python cuenta con una gran biblioteca estándar que está disponible sin cargo alguno para todos los usuarios.

    > [!info]
    > Más allá de la biblioteca estándar, el ecosistema de Python es vasto, con miles de bibliotecas de terceros disponibles a través de PyPI (Python Package Index). Algunas de las más populares incluyen:
    > *   **NumPy y Pandas**: Para computación numérica y análisis de datos.
    > *   **Matplotlib y Seaborn**: Para visualización de datos.
    > *   **Django y Flask**: Para desarrollo web.
    > *   **TensorFlow y PyTorch**: Para aprendizaje automático y redes neuronales.
    > *   **Requests**: Para realizar solicitudes HTTP.

*   **Soporte para múltiples paradigmas de programación**: Python soporta varios estilos de programación, incluyendo programación orientada a objetos, imperativa y funcional.

    > [!tip]
    > *   **Orientada a Objetos (OOP)**: Permite organizar el código en objetos que combinan datos y funciones. Python es un lenguaje completamente orientado a objetos.
    > *   **Imperativa**: Se enfoca en cómo el programa logra un resultado, con secuencias de comandos que cambian el estado del programa.
    > *   **Funcional**: Trata la computación como la evaluación de funciones matemáticas y evita el cambio de estado y los datos mutables. Python soporta conceptos como funciones de orden superior y lambdas.

## Ventajas de Usar Python

*   **Productividad mejorada**: La simplicidad de Python aumenta la productividad de los desarrolladores ya que les permite enfocarse en resolver el problema en lugar de la complejidad del lenguaje.
*   **Amplia comunidad**: Una comunidad grande y activa significa que es fácil encontrar ayuda, colaboración y contribuciones de terceros.
*   **Aplicabilidad en múltiples dominios**: Python se utiliza en una variedad de aplicaciones, desde desarrollo web hasta inteligencia artificial, ciencia de datos y automatización.
*   **Compatibilidad y colaboración**: Python se integra fácilmente con otros lenguajes y herramientas, y es una excelente opción para equipos de desarrollo colaborativos.

Con estas características y ventajas, Python se ha establecido como un lenguaje clave en el desarrollo de software moderno. Su facilidad de uso y su amplia aplicabilidad lo hacen una elección excelente tanto para programadores principiantes como para expertos.

# Diferencias entre Python 2, Python 3, PIP2 y PIP3

Python 2 y Python 3 son dos versiones del lenguaje de programación Python, cada una con sus propias características y diferencias clave. PIP2 y PIP3 son las herramientas de gestión de paquetes correspondientes a cada versión, utilizadas para instalar y administrar bibliotecas y dependencias.

## Python 2 vs Python 3

*   **Sintaxis de `print`**: En Python 2, `print` es una declaración, mientras que en Python 3, `print()` es una función, lo que requiere el uso de paréntesis.

    > [!example]
    > **Python 2:**
    > ```python
    > print "Hola Mundo"
    > print "El valor es", 42
    > ```
    > **Python 3:**
    > ```python
    > print("Hola Mundo")
    > print("El valor es", 42)
    > ```

*   **División de enteros**: Python 2 realiza una división entera por defecto cuando ambos operandos son enteros, mientras que Python 3 realiza una división real (flotante) por defecto.

    > [!example]
    > **Python 2:**
    > ```python
    > print 7 / 2  # Salida: 3
    > print 7 / 2.0 # Salida: 3.5
    > ```
    > **Python 3:**
    > ```python
    > print(7 / 2)  # Salida: 3.5
    > print(7 // 2) # Salida: 3 (división entera explícita)
    > ```

*   **Unicode**: Python 3 usa Unicode (tipo `str`) como tipo de dato por defecto para representar cadenas, lo que facilita el manejo de texto en diferentes idiomas. Python 2 utiliza ASCII por defecto para `str` y tiene un tipo `unicode` separado.

    > [!info]
    > En Python 3, todas las cadenas son Unicode por defecto, lo que simplifica enormemente el manejo de texto global. Los datos binarios se manejan con el tipo `bytes`. En Python 2, la distinción entre `str` (bytes) y `unicode` era una fuente común de errores.

*   **Librerías**: Muchas librerías populares de Python 2 han sido actualizadas o reescritas para Python 3, con mejoras y nuevas funcionalidades. Algunas librerías de Python 2 no tienen un equivalente directo en Python 3.
*   **Soporte**: Python 2 llegó al final de su vida útil (End-of-Life, EOL) el 1 de enero de 2020, lo que significa que ya no recibe actualizaciones, ni siquiera para correcciones de seguridad. Se recomienda encarecidamente migrar a Python 3 para cualquier desarrollo nuevo o mantenimiento de sistemas existentes.

    > [!warning]
    > Continuar utilizando Python 2 en entornos de producción después de su EOL expone los sistemas a vulnerabilidades de seguridad no parcheadas y a la falta de soporte de la comunidad y de nuevas bibliotecas.

## PIP2 vs PIP3

*   **Gestión de paquetes**: PIP2 y PIP3 son herramientas que permiten instalar paquetes para Python 2 y Python 3, respectivamente. Es importante usar la versión correcta para garantizar la compatibilidad con la versión de Python que estés utilizando.
*   **Comandos de instalación**: El uso de `pip` o `pip3` antes de un comando determina si el paquete se instalará en Python 2 o Python 3. Algunos sistemas operativos pueden requerir especificar `pip2` o `pip3` explícitamente para evitar ambigüedades, especialmente si ambas versiones de Python están instaladas.

    > [!example]
    > Para instalar un paquete para Python 3:
    > ```bash
    > pip3 install nombre_del_paquete
    > ```
    > Para instalar un paquete para Python 2 (si aún lo usas):
    > ```bash
    > pip2 install nombre_del_paquete
    > ```
    > O simplemente `pip install nombre_del_paquete` si `pip` está aliñado a `pip3` en tu sistema.

*   **Ambientes virtuales**: Es una buena práctica usar ambientes virtuales para mantener separadas las dependencias de proyectos específicos y evitar conflictos entre versiones de paquetes para Python 2 y Python 3.

    > [!tip]
    > Los **ambientes virtuales** (`venv` en Python 3 o `virtualenv` para Python 2/3) crean directorios aislados que contienen una instalación de Python y sus propias bibliotecas `site-packages`. Esto permite que diferentes proyectos utilicen diferentes versiones de bibliotecas sin interferir entre sí. Para crear y activar un ambiente virtual en Python 3:
    > ```bash
    > python3 -m venv mi_entorno
    > source mi_entorno/bin/activate
    > # Ahora puedes usar pip install y los paquetes se instalarán en mi_entorno
    > ```

La transición de Python 2 a Python 3 ha sido significativa en la comunidad de desarrolladores de Python, y es fundamental que los programadores comprendan las diferencias y sepan cómo trabajar con ambas versiones del lenguaje y sus herramientas asociadas.