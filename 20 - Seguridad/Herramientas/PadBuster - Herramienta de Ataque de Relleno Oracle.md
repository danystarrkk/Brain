---
aliases:
  - PadBuster
tags:
  - hacking/web
  - hacking/herramientas
  - hacking/explotacion
estado: 🟢 Terminado
---
# PadBuster - Herramienta de Ataque de Relleno Oracle

PadBuster es una herramienta diseñada para explotar vulnerabilidades de tipo "Padding Oracle Attack". Este tipo de ataque permite a un atacante descifrar o cifrar datos arbitrarios cuando una aplicación web revela si el relleno (padding) de un bloque de texto cifrado es válido o no.

> [!info] ¿Qué es un Padding Oracle Attack?
> Un ataque de relleno (Padding Oracle Attack) explota una debilidad en la implementación de cifrado en modo de bloque (como CBC) que revela si el relleno de un bloque de texto cifrado es correcto o incorrecto. Al enviar repetidamente bloques de texto cifrado modificados y observar las respuestas del servidor (por ejemplo, mensajes de error HTTP 500 o 200), un atacante puede deducir byte a byte el texto plano original. Este ataque es particularmente efectivo contra esquemas de relleno como PKCS#7.

| Comando/Opción | Descripción | Ejemplo |
|---|---|---|
| **Sintaxis Básica** | `padbuster URL encrypted_sample block_size [opciones]` | |
| **Parámetros Requeridos** | | |
| `URL` | URL objetivo donde se procesa el texto cifrado | `http://sitio.com/vulnerable.php` |
| `encrypted_sample` | Texto cifrado de ejemplo a analizar | `D8AF6F...` |
| `block_size` | Tamaño del bloque de cifrado en bytes (comúnmente 8 o 16 para AES/DES) | `16` |
| **Opciones Principales** | | |
| `-plaintext` | Texto plano conocido para cifrar o comparar | `-plaintext "user=admin"` |
| `-ciphertext` | Texto cifrado específico para usar en la operación | `-ciphertext "A1B2C3..."` |
| `-encoding` | Tipo de codificación del texto cifrado (0: Base64, 1: Hex, 2: URL, 3: ASCII) | `-encoding 1` |
| `-0` | Equivalente a `-encoding 0` (Codificación Base64) | `-encoding 0` |
| `-1` | Equivalente a `-encoding 1` (Codificación Hexadecimal) | `-encoding 1` |
| `-bruteforce` | Realizar fuerza bruta para descifrar el texto | `-bruteforce` |
| `-log` | Generar un archivo de registro con los detalles del ataque | `-log output.txt` |
| `-noiv` | No incluir el Vector de Inicialización (IV) en el payload enviado | `-noiv` |
| `-prefix` | Prefijo de texto plano a añadir al payload | `-prefix "user="` |
| `-postfix` | Sufijo de texto plano a añadir al payload | `-postfix "&role=user"` |
| `-error` | Cadena de texto que indica un error de relleno en la respuesta del servidor | `-error "Padding error"` |

> [!tip] Entendiendo el `block_size` y el `IV`
> El `block_size` es crucial porque los cifrados de bloque operan sobre bloques de datos de tamaño fijo. AES, por ejemplo, usa un tamaño de bloque de 16 bytes. El Vector de Inicialización (IV) es un bloque de datos aleatorio que se combina con el primer bloque de texto plano antes del cifrado en modos como CBC. Su propósito es asegurar que bloques de texto plano idénticos produzcan bloques de texto cifrado diferentes, aumentando la seguridad. La opción `-noiv` se usa cuando el IV no es parte del texto cifrado que se manipula directamente, sino que se maneja de otra forma por la aplicación (ej. se envía en una cabecera HTTP o se genera en el servidor).

> [!warning] La importancia de la `cadena de error`
> Para que PadBuster funcione eficazmente, es fundamental identificar una respuesta del servidor que indique un error de relleno. Esta "cadena de error" puede ser un mensaje explícito en el cuerpo de la respuesta, un código de estado HTTP diferente (ej. 500 en lugar de 200), o incluso una diferencia en el tiempo de respuesta. Sin una forma fiable de distinguir entre un relleno válido e inválido, el ataque no puede progresar.

### Modos de Operación

| Modo | Descripción | Ejemplo |
|---|---|---|
| **Descifrado** | Analizar y descifrar un texto cifrado existente | `padbuster http://sitio.com/vulnerable.php D8AF6F... 16 -encoding 1 -error "Invalid padding"` |
| **Cifrado** | Generar un nuevo texto cifrado a partir de un texto plano | `padbuster http://sitio.com/vulnerable.php D8AF6F... 16 -plaintext "admin=true" -encoding 1 -error "Invalid padding"` |
| **Explotación** | Explotar la vulnerabilidad para manipular datos | `padbuster http://sitio.com/vulnerable.php D8AF6F... 16 -plaintext "user=admin&role=admin" -encoding 1 -error "Invalid padding"` |