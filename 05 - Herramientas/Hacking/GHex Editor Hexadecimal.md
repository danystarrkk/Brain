---
aliases:
  - GHex
  - Editor Hexadecimal
tags:
  - hacking/herramientas
estado: 🟢 Terminado
---
**GHex** es una herramienta de edición hexadecimal diseñada para sistemas Linux y otras distribuciones basadas en Unix. Permite a los usuarios visualizar y editar archivos en formato hexadecimal, facilitando la manipulación de datos binarios de manera precisa.

> [!info] Importancia de los Editores Hexadecimales
> Los editores hexadecimales como GHex son fundamentales en ciberseguridad para el análisis de bajo nivel. Permiten a los profesionales examinar y modificar directamente el contenido binario de archivos ejecutables, firmware, volcados de memoria y paquetes de red. Esto es crucial para tareas como la ingeniería inversa de malware, el desarrollo de exploits (modificando opcodes o direcciones), la recuperación de datos, la corrección de archivos corruptos o la alteración de firmas de archivos (magic bytes).

## Uso Básico de GHex

### Inicio de GHex
Inicia GHex desde la línea de comandos con el comando `ghex` o busca su icono en el menú de aplicaciones gráficas de tu entorno de escritorio.

> [!tip] Instalación y Ejecución
> GHex suele estar disponible en los repositorios de la mayoría de las distribuciones Linux. Puedes instalarlo con `sudo apt install ghex` (Debian/Ubuntu) o `sudo dnf install ghex` (Fedora). Al ejecutar `ghex <nombre_archivo>`, se abrirá directamente el archivo especificado.

### Interfaz de Usuario
GHex tiene una interfaz similar a un editor de texto estándar:
- **Panel Hexadecimal:** Muestra los datos del archivo en formato hexadecimal. Cada byte se representa con dos dígitos hexadecimales (00-FF).
- **Panel ASCII:** Muestra la interpretación ASCII de los datos correspondientes. Los caracteres no imprimibles suelen representarse con un punto (`.`).
- **Barra de Herramientas:** Ofrece accesos directos a funciones como abrir, guardar, copiar, pegar y buscar.

> [!info] Correlación Hex-ASCII
> La visualización simultánea en hexadecimal y ASCII es vital. El panel hexadecimal permite ver el valor exacto de cada byte, mientras que el panel ASCII ayuda a identificar cadenas de texto, nombres de funciones, URLs o cualquier dato legible incrustado en el binario. Esta dualidad es clave para entender la estructura y el propósito de los datos.

### Abrir Archivos
Selecciona **Archivo > Abrir** desde la barra de menú o usa `Ctrl+O` para cargar un archivo binario en GHex para su edición.

### Editar Datos
Haz clic en un byte en el panel hexadecimal para modificarlo. GHex permite cambios precisos en datos binarios, útiles para ajustes en configuraciones o software.

> [!warning] Edición de Binarios
> La edición directa de datos binarios requiere precaución extrema. Un cambio incorrecto puede corromper el archivo, hacerlo inoperable o introducir vulnerabilidades. Es recomendable trabajar siempre con copias de seguridad de los archivos originales. En el contexto de la ciberseguridad ofensiva, la edición de binarios puede incluir:
> - **Parcheo de ejecutables:** Modificar instrucciones de salto (`JMP`, `JE`, `JNE`) para alterar el flujo de ejecución, insertar NOP sleds (`0x90`) para alinear código o deshabilitar comprobaciones de licencia.
> - **Modificación de "Magic Bytes":** Alterar los primeros bytes de un archivo para cambiar su tipo (ej. de JPG a PNG, aunque esto no lo hará funcional sin una reestructuración completa).
> - **Inyección de Shellcode:** Insertar código malicioso en secciones ejecutables de un binario.

### Funciones de Edición
Incluye búsqueda y reemplazo de datos, inserción y eliminación de bytes, y la capacidad de copiar y pegar bloques de datos.

### Guardar Cambios
Guarda los cambios con **Archivo > Guardar** o `Ctrl+S`. Asegúrate de tener permisos suficientes para escribir en el archivo.

### Funcionalidades Avanzadas
Admite visualización y edición en múltiples formatos (octal, binario, decimal), además del hexadecimal estándar.

> [!info] Bases Numéricas y Endianness
> La capacidad de visualizar datos en octal, binario y decimal es útil para interpretar diferentes tipos de datos:
> - **Binario:** Esencial para entender flags, permisos o bits específicos.
> - **Octal:** Comúnmente usado en permisos de archivos Unix (ej. `chmod 755`).
> - **Decimal:** Para valores numéricos estándar.
> Además, al interpretar valores multi-byte (como enteros de 16, 32 o 64 bits), es crucial considerar la **endianness** (orden de bytes). GHex muestra los bytes en el orden en que aparecen en el archivo, pero la interpretación de un valor numérico puede variar si el sistema es *little-endian* (byte menos significativo primero) o *big-endian* (byte más significativo primero).