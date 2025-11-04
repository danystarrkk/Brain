-----
## PadBuster - Oracle Padding Attack Tool

|Comando/Opcíón|Descripción|Ejemplo|
|---|---|---|
|**Sintaxis Básica**|`padbuster URL encrypted_sample block_size [opciones]`||
|**Parámetros Requeridos**|||
|`URL`|URL objetivo|`http://sitio.com/vuln.php`|
|`encrypted_sample`|Texto cifrado a analizar|`D8AF6F...`|
|`block_size`|Tamaño de bloque (8, 16)|`16`|
|**Opciones Principales**|||
|`-plaintext`|Texto plano conocido|`-plaintext "user=admin"`|
|`-ciphertext`|Texto cifrado para usar|`-ciphertext "A1B2C3..."`|
|`-encoding`|Codificación (0-3)|`-encoding 1`|
|`-0`|Codificación Base64|`-encoding 0`|
|`-1`|Codificación Hex|`-encoding 1`|
|`-bruteforce`|Fuerza bruta para descifrar|`-bruteforce`|
|`-log`|Generar archivo de log|`-log output.txt`|
|`-noiv`|No usar IV en payload|`-noiv`|
|`-prefix`|Prefijo de texto plano|`-prefix "user="`|
|`-postfix`|Sufijo de texto plano|`-postfix "&role=user"`|
|`-error`|Cadena de error|`-error "Padding error"`|
|**Modos de Operación**|||
|**Descifrado**|Analizar texto cifrado|`padbuster URL ciphertext 16`|
|**Cifrado**|Generar texto cifrado|`padbuster URL ciphertext 16 -plaintext "admin"`|
|**Explotación**|Explotar vulnerabilidad|`padbuster URL cipherte`|