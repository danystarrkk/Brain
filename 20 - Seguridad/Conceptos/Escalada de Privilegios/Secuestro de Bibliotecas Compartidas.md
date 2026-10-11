---
aliases:
  - Shared Library Hijacking
  - DLL Hijacking (Linux)
  - LD_PRELOAD Attack
tags:
  - hacking/explotacion
  - linux/seguridad
  - hacking/laboratorios
estado: 🟢 Terminado
---
# Secuestro de Bibliotecas Compartidas (Shared Library Hijacking)

Las **bibliotecas compartidas** son archivos que contienen funciones y recursos utilizados por múltiples programas. Cuando un programa requiere una función de una biblioteca compartida, el sistema operativo busca la biblioteca y la enlaza dinámicamente durante la ejecución del programa. Sin embargo, si el sistema no encuentra la biblioteca en las rutas predeterminadas, puede buscarla en otros directorios.

> [!info] Mecanismo de Enlace Dinámico
> El enlace dinámico permite que los programas compartan código de bibliotecas, reduciendo el tamaño de los ejecutables y facilitando las actualizaciones. El cargador dinámico (`ld.so` en Linux) es responsable de localizar y cargar las bibliotecas necesarias en tiempo de ejecución.

## Mecanismo de Ataque: Secuestro de Bibliotecas

Un atacante puede aprovechar esta situación creando una **biblioteca compartida maliciosa** con el mismo nombre que una biblioteca legítima y colocándola en un directorio donde el sistema la buscará antes que la versión legítima. Cuando el programa intenta cargar la biblioteca, el sistema cargará la versión maliciosa en lugar de la legítima, permitiendo al atacante ejecutar código arbitrario con los privilegios del programa víctima.

> [!warning] Orden de Búsqueda de Bibliotecas
> En sistemas Linux, el orden de búsqueda de bibliotecas compartidas generalmente sigue esta secuencia:
> 1.  Rutas especificadas en la variable de entorno `LD_LIBRARY_PATH`.
> 2.  Caché de bibliotecas (`/etc/ld.so.cache`), generado por `ldconfig`.
> 3.  Directorios predeterminados (`/lib`, `/usr/lib`, etc.).
> Un atacante puede manipular `LD_LIBRARY_PATH` o colocar una biblioteca maliciosa en un directorio con mayor prioridad en la ruta de búsqueda.

> [!tip] Identificación de Dependencias con `ldd`
> Para identificar las bibliotecas compartidas de las que depende un ejecutable, se puede usar el comando `ldd`. Por ejemplo:
> ```bash
> ldd /usr/bin/programa_ejemplo
> ```
> Esto mostrará la lista de bibliotecas requeridas y sus rutas. Si una biblioteca aparece como "not found", es un posible vector para el secuestro.

## Herramientas y Laboratorios

En esta clase, analizaremos cómo se lleva a cabo el secuestro de bibliotecas de objetos compartidos enlazados dinámicamente y cómo identificar situaciones en las que esta técnica puede ser aplicada.

A continuación, se comparte una de las herramientas utilizadas para analizar la ejecución de un programa escrito en C/C++:

*   **Herramienta Uftrace**: Una herramienta para rastrear y analizar la ejecución de funciones en programas. Es útil para entender el flujo de llamadas a funciones y las dependencias de bibliotecas.
    *   [https://github.com/namhyung/uftrace](https://github.com/namhyung/uftrace)

Asimismo, se comparte el enlace directo a la plataforma **AttackDefense**, donde se resolverá un reto que involucra esta misma temática:

*   **AttackDefense**: Plataforma de laboratorios de ciberseguridad ofensiva y defensiva.
    *   [https://attackdefense.com](https://attackdefense.com/)