---
aliases:
  - Ghidra
  - Reverse Engineering
tags:
  - hacking/reversing
  - hacking/herramientas
estado: 🟢 Terminado
---
# Ghidra - Herramienta de Ingeniería Inversa

Ghidra es un framework de ingeniería inversa de software (SRE) desarrollado por la Agencia de Seguridad Nacional (NSA) de Estados Unidos. Es una herramienta potente y gratuita que ayuda a los analistas de ciberseguridad a comprender cómo funcionan los programas compilados, lo que es fundamental para el análisis de malware, la búsqueda de vulnerabilidades y la auditoría de seguridad.

| Categoría    | Comando/Función           | Descripción                                                                                                          | Atajo de Teclado |
| ------------ | ------------------------- | -------------------------------------------------------------------------------------------------------------------- | ---------------- |
| **Proyecto** | `File > New Project`      | Crear un nuevo proyecto de Ghidra.                                                                                   | `Ctrl + N`       |
|              | `File > Import`           | Importar un archivo binario (ejecutable, biblioteca) para su análisis.                                               | `Ctrl + I`       |
|              | `File > Save Project`     | Guardar el estado actual del proyecto de Ghidra.                                                                     | `Ctrl + S`       |
| **Análisis** | `Analysis > Auto Analyze` | Realiza un análisis automático del binario, identificando funciones, referencias, tipos de datos y flujo de control. |                  |
|              |                           |                                                                                                                      |                  |
> El análisis automático de Ghidra es un paso crucial que utiliza una serie de analizadores predefinidos para reconstruir la lógica del programa. Esto incluye la identificación de funciones, la determinación de tipos de datos, la resolución de llamadas a funciones y la construcción del grafo de flujo de control (CFG). Es la base para una decompilación efectiva.

| Categoría | Ruta del Menú | Descripción | Atajo (Shortcut) |
| :--- | :--- | :--- | :--- |
| **Análisis** | Analysis > One Shot | Permite ejecutar analizadores específicos de forma individual. | - |
| **Visualización** | Window > Function Graph | Muestra una representación gráfica del flujo de control de una función. | - |
| **Navegación** | Window > Code Browser | La ventana principal para interactuar con el código desensamblado y decompilado. | Ctrl + Shift + C |
| **Navegación** | Window > Symbol Tree | Muestra una vista jerárquica de los símbolos (funciones, variables globales, etiquetas). | Ctrl + Shift + S |
| **Navegación** | Window > Listing | La vista de desensamblado que muestra el código máquina y su representación en ensamblador. | Ctrl + Shift + L |
| **Edición** | Edit > Undo/Redo | Deshacer o rehacer la última acción realizada. | Ctrl + Z / Ctrl + Y |
| **Edición** | Edit > Rename | Renombrar una variable, función, etiqueta o tipo de dato. | L |

> [!tip] Renombrar Elementos
> Renombrar variables y funciones con nombres descriptivos es una práctica fundamental en la ingeniería inversa. Permite transformar el código ofuscado o genérico en algo comprensible, facilitando la identificación de la lógica del programa y sus componentes clave.

| Categoría | Ruta del Menú | Descripción | Atajo (Shortcut) |
| :--- | :--- | :--- | :--- |
| **Edición** | Edit > Comment | Agregar comentarios al código desensamblado o decompilado. | ; |
| **Vistas** | Window > Decompile | Muestra la representación de alto nivel del código (pseudo-código) generado a partir del ensamblador. | F5 |

> [!info] Vista de Decompilación
> La vista de decompilación es una de las características más potentes de Ghidra. Transforma el código ensamblador de bajo nivel en un pseudo-código similar a C/C++, lo que permite a los analistas comprender la lógica del programa de manera mucho más rápida y eficiente que leyendo directamente el ensamblador.

| Categoría | Ruta del Menú | Descripción | Atajo (Shortcut) |
| :--- | :--- | :--- | :--- |
| **Memoria** | Window > Memory Map | Muestra la distribución de la memoria del binario, incluyendo secciones y permisos. | - |
| **Datos** | Window > Data Type Manager | Permite gestionar y definir tipos de datos personalizados, estructuras y uniones. | - |
| **Scripting** | Window > Script Manager | Administra y ejecuta scripts de Ghidra. | - |
| **Scripting** | Analysis > Run Script | Ejecuta un script seleccionado para automatizar tareas. | - |
| **Scripting** | Python > Interactive | Abre una consola interactiva de Python para ejecutar comandos y scripts. | - |

> [!tip] Automatización con Scripting
> Ghidra soporta scripting en Java y Python. Esto es invaluable para automatizar tareas repetitivas, como la búsqueda de patrones específicos, la modificación masiva de símbolos, la extracción de datos o la integración con otras herramientas. La API de Ghidra expone gran parte de su funcionalidad para ser controlada programáticamente.

| Categoría | Ruta del Menú | Descripción | Atajo (Shortcut) |
| :--- | :--- | :--- | :--- |
| **Búsqueda** | Search > For Strings | Buscar cadenas de texto incrustadas en el binario. | Ctrl + F |
| **Búsqueda** | Search > For Functions | Buscar funciones por nombre o patrón. | Ctrl + Shift + F |
| **Búsqueda** | Search > For Patterns | Buscar patrones de bytes específicos en el binario. | - |
| **Funciones** | Function > Create Function | Definir manualmente una nueva función en el código. | F |
| **Funciones** | Function > Edit Function | Editar propiedades de una función (nombre, tipo de retorno, parámetros). | Ctrl + F |
| **Funciones** | Function > Stack | Abre el editor de pila para analizar y modificar el uso de la pila. | - |

> [!warning] Editor de Pila
> El editor de pila es crucial para corregir la interpretación de Ghidra sobre las variables locales y los parámetros de una función. Una correcta definición de la pila es fundamental para que la decompilación genere un pseudo-código preciso y legible.
