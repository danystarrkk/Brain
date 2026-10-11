---
aliases: ["PortSwigger File Path Traversal"]
tags: [hacking/web]
estado: 🟢 Terminado
---

# PortSwigger: Labs de Path Traversal

## Lab: File path traversal, simple case

Este es un caso simple donde el objetivo es listar el archivo `/etc/passwd`.

![[Pasted image 20250923124439.png]]

Los puntos inyectables usuales son cualquiera de los campos donde podemos pasar información. Podemos usar el que se indica en la imagen para intentar la inyección.

En este caso, vemos que no es posible en ese punto. Otro parámetro que podemos usar es el que carga las imágenes:
![[Pasted image 20250923124722.png]]

Vamos a intentar trasladar esto a la URL para ver si nos permite la inyección:
`GET /image?filename=../../../etc/passwd`
![[Pasted image 20250923125644.png]]

> [!info] Explicación de Path Traversal
> El ataque de *Path Traversal* (también conocido como *Directory Traversal*) permite a un atacante acceder a archivos y directorios que están fuera del directorio raíz de la aplicación web. Esto se logra manipulando variables que hacen referencia a rutas de archivos con secuencias como `../` (punto-punto-barra) para moverse hacia arriba en la jerarquía de directorios. En sistemas Linux/Unix, `/etc/passwd` es un objetivo común ya que contiene información sobre los usuarios del sistema.

Aunque la interfaz muestre "no válido", al regresar a la web principal, la inyección ha sido exitosa:
![[Pasted image 20250923125709.png]]

Lab Terminada.

## Lab: File path traversal, traversal sequences blocked with absolute path bypass

Este laboratorio es similar al anterior, pero indica que las secuencias de *path traversal* están bloqueadas. Sin embargo, se pueden usar rutas absolutas. En este caso, lo que hicimos en el laboratorio anterior también es válido:
![[Pasted image 20250923130005.png]]

Vamos a intentarlo para ver qué obtenemos:
`GET /image?filename=/etc/passwd`
![[Pasted image 20250923130423.png]]
Vemos que en este caso, usar la ruta de forma absoluta es suficiente. Si revisamos:

![[Pasted image 20250923130518.png]]

> [!tip] Bypass de Bloqueo de Secuencias de Traversal
> Cuando las secuencias `../` son bloqueadas, una técnica común es intentar rutas absolutas si el sistema de archivos subyacente es conocido (por ejemplo, `/etc/passwd` en Linux o `C:\Windows\System32\drivers\etc\hosts` en Windows). Esto funciona si la aplicación concatena la ruta proporcionada directamente a la raíz del sistema de archivos en lugar de a un directorio base específico de la aplicación.

Lab terminado.

## Lab: File path traversal, traversal sequences stripped non-recursively

En este laboratorio, tenemos una seguridad mejorada donde ya no se permite el uso de `../`, ya que el sistema de alguna forma lo borra o lo quita. Vamos a verificar:
![[Pasted image 20250923131046.png]]

Vemos que nos sale como no válido. En este punto, una forma que a veces logra engañar al sistema es usar `....//....//....//` (o `..%2f..%2f` si se requiere URL encoding). Esto funciona porque si el filtro no es recursivo, solo eliminará una instancia de `../` en la primera pasada de todas las coincidencias, dejando aún otra secuencia `../` intacta. Vamos a ver si esto funciona:
`GET /image?filename=....//....//....//etc/passwd`
![[Pasted image 20250923131226.png]]

> [!warning] Filtros no Recursivos
> Los filtros de seguridad que eliminan secuencias de *path traversal* deben ser implementados de forma recursiva. Si un filtro solo elimina la primera ocurrencia o no itera hasta que no queden más secuencias, un atacante puede evadirlo duplicando las secuencias, como `....//` que se convierte en `../` después de una pasada no recursiva.

Perfecto, esto sí funcionó. Vamos a ver si con esto completamos el lab:

![[Pasted image 20250923131252.png]]

Lab Terminado.

## Lab: File path traversal, traversal sequences stripped with superfluous URL-decode

Al parecer, ahora tenemos más seguridad donde se eliminan las secuencias `...//...//` al usar URL-decode. Vamos a intentar burlar esto:

![[Pasted image 20250923131748.png]]

Vemos que no funciona de forma directa, y nada de lo que conocemos por el momento lo hace. En ocasiones, lo que hacemos es URL-encode de las barras `/` con el objetivo de que el sistema, al hacer un URL-decode, nos lo interprete. Esto puede funcionar haciéndolo solo una vez, donde el URL-encode de `/` es `%2f`. Pero en ocasiones falla y necesitamos también hacer URL-encode al `%`, lo que resultaría en `%25`. Así, la cadena para `/` sería `%252f`.
`GET /image?filename=..%252f..%252f..%252fetc%252fpasswd`
![[Pasted image 20250923132125.png]]

> [!info] Doble URL Encoding para Bypass
> Algunos sistemas de seguridad aplican un URL-decode antes de filtrar las secuencias de *path traversal*. Si un atacante codifica las barras (`/` como `%2f`) y luego codifica el signo de porcentaje (`%` como `%25`), la secuencia `..%252f` se convierte en `../` después de dos pasadas de URL-decode. Esto es efectivo contra filtros que solo realizan un único URL-decode antes de la sanitización.

Perfecto, esto ha funcionado.

![[Pasted image 20250923132143.png]]

Lab Terminado.

## Lab: File path traversal, validation of start of path

En esta ocasión, vamos a ver cómo en la URL se nos obliga a usar la ruta `/var/www/images`. Esto puede parecer una buena idea, pero a la vez no, ya que si no tenemos más que eso, podemos aprovecharnos aún de los `../` para modificarla:
`GET /image?filename=/var/www/images/../../../etc/passwd`
![[Pasted image 20250923132549.png]]

Verifiquemos:

![[Pasted image 20250923132613.png]]

> [!tip] Bypass de Prefijos de Ruta Obligatorios
> Si una aplicación impone un prefijo de ruta obligatorio (ej. `/var/www/images/`), pero no valida la ruta completa después de la concatenación, un atacante puede usar secuencias `../` para "escapar" del directorio impuesto. Por ejemplo, `/var/www/images/../../../etc/passwd` se resuelve a `/etc/passwd` en el sistema de archivos.

Lab Terminada.

## Lab: File path traversal, validation of file extension with null byte bypass

En esta ocasión, se está empleando una forma algo más persistente, y es que concatena siempre la extensión de la imagen al final.

Lo que vamos a hacer en este caso es jugar con el conocido *null byte* (`%00`) que nos permite anular esa concatenación de extensión. Algo a tener en cuenta es que esto funciona en versiones antiguas de PHP y otros lenguajes/sistemas, por lo que en versiones nuevas ya no lo vamos a poder hacer.
`GET /image?filename=../../../etc/passwd%00.png`
![[Pasted image 20250923133121.png]]

Vemos cómo esto ya es posible. Si revisamos:
![[Pasted image 20250923133143.png]]

> [!warning] Vulnerabilidad de Null Byte (`%00`)
> El *null byte* (`%00`) es un carácter especial que, en algunos lenguajes de programación (como C/C++ y versiones antiguas de PHP), marca el final de una cadena. Cuando se utiliza en una ruta de archivo, el sistema operativo o la función de manejo de archivos pueden truncar la cadena en el `%00`, ignorando cualquier carácter posterior. Esto permite a un atacante eludir validaciones de extensión de archivo, por ejemplo, transformando `archivo.txt%00.jpg` en `archivo.txt`. Esta vulnerabilidad ha sido mitigada en versiones modernas de software.

Lab Terminado.