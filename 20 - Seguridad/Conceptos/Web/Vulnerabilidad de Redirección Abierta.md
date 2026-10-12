---
aliases:
  - Open Redirect
  - Redirección Abierta
tags:
  - sec/web
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Server-Side Request Forgery (SSRF)]]"
  - "[[Vulnerabilidades OAuth en PortSwigger Labs]]"
  - "[[Cross-Site Scripting XSS]]"
  - "[[HTTP Host header attacks]]"
  - "[[Vulnerabilidades CORS y Laboratorios]]"
fuente: "https://owasp.org/www-community/attacks/Unvalidated_Redirect"
---

# Vulnerabilidad de Redirección Abierta (Open Redirect)

La **Redirección Abierta** (*Open Redirect*) ocurre cuando una aplicación redirige al usuario a una URL **controlada por el atacante** sin validar el destino. El navegador del usuario confía en el dominio legítimo — que emite un `302 Location: https://evil.com` perfectamente válido — y pierde cualquier señal visual de que sale del sitio. El impacto real no es "cambiar una URL": es **robar la confianza del usuario y de los filtros que confían en ese dominio** (OAuth, allowlists de links, clientes de correo, previews de mensajería).

> [!info] Open Redirect en una frase
> Un redirect acepta un parámetro como `?next=`, `?url=`, `?redirect=`, `?return=`, `?dest=` y lo copia al header `Location`. Si no valida contra una allowlist, cualquier destino externo es posible — y ese es el problema: el atacante construye un link `legitima-empresa.com/login?next=https://evil.com` que supera la inspección humana y muchos filtros automáticos.

## Mecánica: cómo redirige la web

### Motores de redirección

| Mecanismo | Cómo se ve | Control del atacante |
|---|---|---|
| **HTTP 3xx** | `HTTP/1.1 302 Found` + `Location: <url>` | El header `Location` |
| **Meta refresh** | `<meta http-equiv="refresh" content="0;url=https://...">` | El atributo `url` |
| **JS location** | `window.location = "<url>"` o `location.href` | El valor asignado |
| **Header `Refresh`** | `Refresh: 0; url=<url>` (cabezal no estándar, usado por ASP/PHP legacy) | El valor del header |

### Anatomía de una URL — dónde vive el bypass

```
https://usuario:contraseña@subdominio.dominio.com:443/ruta?query=1#fragment
└─┬──┘ └─────┬─────┘ └────────┬────────┘└┬┘ └───┬───┘ └──┬────┘ └───┬────┘
 esquema    userinfo            host      puerto  path    query    fragment
```

Los filtros débiles comparan **strings** ("¿empieza por `https://mi-dominio.com`?"), pero los navegadores **parsean** la URL. Cualquier divergencia entre lo que compara el filtro y lo que parsea el navegador es el bypass. El fragmento (`#`) y el userinfo (`@`) son las zonas favoritas: el filtro ve "mi-dominio.com" y el navegador va a otra parte.

> [!warning] Redirección ≠ solo phishing
> Un Open Redirect bien encadenado tiene impacto alto:
> 1. **Robo de tokens OAuth**: el `redirect_uri` con subdominio abierto permite capturar `?code=`/`?token=` — ver [[Vulnerabilidades OAuth en PortSwigger Labs]].
> 2. **Bypass de SSRF**: un filtro que solo permite "dominios internos" cae ante un redirect legítimo que el backend **sigue** (`follow redirect`) hacia `169.254.169.254` — ver [[Server-Side Request Forgery (SSRF)]].
> 3. **XSS vía scheme**: `?url=javascript:alert(1)` en redirecciones por JS/meta.
> 4. **Encadenamiento con Host header**: generar redirects masivos desde el propio dominio — ver [[HTTP Host header attacks]] y [[Envenenamiento de Caché Web]].
> El impacto "botnet/fuerza bruta" que a veces se atribuye a los Open Redirects (víctimas usadas como reflejo de tráfico) es teórico y raro en la práctica; el impacto real demostrado es phishing + cadenas técnicas.

## Catálogo de bypass de filtros

| Filtro evadido | Payload | Por qué funciona |
|---|---|---|
| "Debe empezar por `/`" | `//evil.com` | `//evil.com` es una *network-path reference*: el navegador hereda el esquema y va a evil.com |
| Idem, con barra invertida | `/\evil.com` o `/\/evil.com` | Muchos navegadores normalizan `\` → `/` |
| Bloqueo de `//` inicial | `https:evil.com`, `http:evil.com` | ElUrlParser acepta el esquema sin `//` (bypass documentado en esta nota) |
| URL-encoding simple | `%2f%2fevil.com`, `/%2f%2fevil.com` | Doble decodificación (proxy+app) lo restaura a `//` |
| Bloqueo de punto | `evil%2ecom` → `%252eevil.com` | **Doble URL-encoding**: si el filtro decodifica una vez y la app/navegador dos, pasa el punto |
| Filtro "solo path interno" | `/\/evil.com`, `//evil%2ecom` | Combinación de normalizaciones |
| Whitelist por dominio | `https://mi-dominio.com.evil.com` | Prefijo falso: el string *contiene* el dominio permitido pero el host real es `evil.com` |
| Whitelist por sufijo | `https://evil.com?mi-dominio.com`, `https://evil.com#@mi-dominio.com` | El filtro busca el dominio en cualquier parte; `?`/`#` cortan la lectura real |
| Parser diferencial userinfo | `https://mi-dominio.com@evil.com` | El host real es `evil.com`; el filtro ve `mi-dominio.com` |
| Puerto confuso | `https://mi-dominio.com:443.evil.com` | Si el puerto no se parsea bien, host real = `evil.com` |
| Subdominio inexistente permitido | `https://notasubdomain-mi-dominio.com` | Filtro por *contiene* en vez de igualdad de host |
| Backslash/unicode | `https:/／evil.com` (U+FF0F fullwidth) | Normalización unicode en navegador, no en filtro |
| NUL byte legacy | `https://mi-dominio.com%00.evil.com` | En parseadores viejos (Java <7u40) truncaba |

> [!tip] Estrategia de prueba
> 1. Cambiar el valor del parámetro a `https://google.com` → si redirige, hay control (confirmación básica).
> 2. Aplicar la tabla de bypass de arriba en orden: `//evil.com`, `/\evil.com`, `https:evil.com`, `%2f%2fevil.com`, `%252e`, `@`, prefijos.
> 3. Verificar **siempre en el navegador final** (seguir la redirección): un `302` no garantiza que el navegador llegue al destino — el filtro pudo reescribir el `Location`.

## Metodología de prueba

1. **Descubrir parámetros de redirect**: en URLs y formularios: `next`, `url`, `redirect`, `return`, `returnTo`, `goto`, `dest`, `destination`, `redir`, `rurl`, `callback`, `continue`, `target`, `view`, `login_url`, `logout_url`. Fuzzing con Burp Param Miner / wordlists.
2. **Confirmar**: `?next=https://google.com` → `302 Location: https://google.com`.
3. **Clasificar**: ¿refleja cualquier host? ¿solo subdominios? ¿solo paths relativos? ¿solo tras login (post-auth redirect)?
4. **Bypass**: recorrer el catálogo hasta conseguir un destino externo arbitrario.
5. **Impacto**:
   - ¿El redirect ocurre tras autenticación? → phishing de credenciales con dominio real en la barra.
   - ¿La app es OAuth/SSO? → probar `redirect_uri` (ver [[Vulnerabilidades OAuth en PortSwigger Labs]]).
   - ¿Hay SSRF? → encadenar el redirect seguido por el backend (ver [[Server-Side Request Forgery (SSRF)]]).
   - ¿Scheme acepta `javascript:`? → XSS (ver [[Cross-Site Scripting XSS]]).
6. **Reportar**: PoC con el dominio real + destino arbitrario + cadena de impacto.

## Defensa (con código)

### 1. Allowlist de destinos — la única defensa completa

```python
# SEGURO (Flask): allowlist de hosts + solo rutas internas por defecto
from urllib.parse import urlparse
ALLOWED = {"mi-dominio.com", "apps.mi-dominio.com"}

def safe_redirect(target: str) -> str:
    # Los destinos por defecto son rutas internas
    if not target:
        return "/dashboard"
    parsed = urlparse(target)
    # Rechazar esquemas inesperados y hosts fuera de allowlist
    if parsed.scheme not in ("http", "https"):
        return "/dashboard"
    if parsed.netloc.lower() not in ALLOWED:
        return "/dashboard"
    return target

@app.route("/go")
def go():
    return redirect(safe_redirect(request.args.get("next")), code=302)
```

```php
// SEGURO (PHP): mapeo por identificador, nunca URL cruda
$routes = [
    'dashboard' => '/dashboard',
    'profile'   => '/account/profile',
];
$key = $_GET['next'] ?? 'dashboard';
header('Location: ' . ($routes[$key] ?? '/dashboard'), true, 302);
```

```javascript
// SEGURO (Node/Express) — librería de allowlist de hosts
import allowlistRedirect from 'express-allowlist-redirect';   // patrón
app.get('/go', (req, res) => {
  const ok = ['mi-dominio.com'];
  const u = new URL(req.query.next ?? '/', 'https://mi-dominio.com');
  if (!ok.includes(u.hostname)) return res.redirect('/dashboard');
  res.redirect(u.toString());
});
```

### 2. Buenas prácticas adicionales

- **Nunca** construir `Location` concatenando strings con input: siempre parsear y comparar **host exacto** (igualdad, no `contains`, no `startsWith`).
- Avisar al usuario: página intermedia "Vas a salir a <dominio externo>".
- No usar el parámetro tras logout/login: mapear a identificadores internos.
- En OAuth, `redirect_uri` debe ser allowlist exacta por cliente — ver [[Vulnerabilidades OAuth en PortSwigger Labs]].

## Notas Prácticas

El botón que estamos observando nos permite movilizarnos a otra web, aplica un redireccionamiento para mandarnos a otro sitio:

![[{781A3A4D-0329-4D81-8683-31B056D085DE}.png]]

![[Pasted image 20260227200408.png]]

Vamos a interceptar esto para ver cómo se está tramitando:

![[{8B5F7609-8398-4983-86C5-889A9712BA74}.png]]

Observamos que nos permite modificar el redireccionamiento. Si cambiamos por `google.es`:

![[Pasted image 20260227200628.png]]

![[Pasted image 20260227200638.png]]

Como vemos, esto funciona. El problema de esto es que un Open Redirect permite construir URLs del dominio legítimo que redirigen a cualquier sitio: phishing que supera la inspección de la barra de direcciones y de filtros de correo/mensajería, y encadenado con otras vulnerabilidades (OAuth, SSRF con follow-redirect, Host header), amplifica su impacto.

> [!info] Cadenas con Open Redirect
> Un Open Redirect rara vez es el impacto final: es el **pegamento** entre otros bugs — bypass de whitelist en SSRF, robo de tokens en OAuth, XSS vía `javascript:`, poisoning de `Location` en caché. Evaluarlo siempre como parte de la cadena, no en aislamiento.

Para evitar este tipo de vulnerabilidades se usan ciertas protecciones. Veamos en una máquina similar cómo burlar una de ellas.

### Bypassing Filtros de Caracteres

Tenemos otra web y vamos directo a la captura por parte de Burp Suite:

![[Pasted image 20260227201130.png]]

Podemos ver que es prácticamente igual, pero al momento de modificarla vamos a ver lo siguiente:

![[Pasted image 20260227201223.png]]

Vemos que no nos permite usar el `.` (punto). Por lo tanto, para burlar esto podríamos implementar URL-encoding de la siguiente manera:

![[Pasted image 20260227201347.png]]

Como vemos, el servidor sabe que eso es un punto y no permite pasar. Pero, ¿y si hacemos URL-encode al `%` que tenemos allí?

> [!tip] URL Encoding Doble
> El doble URL encoding (`%252E` para un punto) es una técnica común para evadir filtros que solo decodifican la URL una vez. Si el servidor decodifica el `%25` a `%` y luego el `%2E` a `.`, el filtro inicial podría ser eludido. Esto es efectivo cuando el filtro se aplica antes de la decodificación completa de la URL.

![[Pasted image 20260227201441.png]]

Como observamos, ya logramos burlar el filtro y pasamos, y en el navegador es funcional.

### Bypassing Filtros de Puntos y Barras

En este caso, tenemos algo un poco más complicado y es que no podemos usar ni puntos ni barras (`.` ni `/`). La forma de burlarla es la misma con URL-encoding:

![[Pasted image 20260227201758.png]]

En ocasiones puede funcionar, pero cuidado, no porque estemos ya en el `302` significa que logramos pasar. Si continuamos el flujo, veremos en realidad que no logramos llegar:

![[Pasted image 20260227201932.png]]

Lo que podemos hacer es que para dominios con `https` no es necesario el uso de las `//` (doble barra), por lo que podemos hacer:

> [!tip] Redirecciones sin `//`
> Algunos navegadores y servidores interpretan URLs que comienzan con `https:` (o `http:`) sin las dobles barras (`//`) como válidas, asumiendo que el resto es el dominio. Por ejemplo, `https:google.com` puede ser interpretado como `https://google.com`. Esta es una técnica de bypass útil cuando las barras están filtradas.

![[Pasted image 20260227202040.png]]

Y en este caso, sí logramos pasar.

## Errores comunes

**Del atacante:**
1. Conformarse con el `302`: no verificar en el navegador que el destino final es el atacante (el backend puede reescribir o validar en otra capa).
2. Probar solo `https://evil.com` y rendirse con filtros: los bypasses de la tabla (`//`, `@`, doble encoding, `https:` sin barras) resuelven el 90% de los filtros reales.
3. Olvidar parámetros en cadenas de flujo: logout, "continuar después del pago", emails de verificación — suelen tener redirects menos vigilados.
4. Reportar "solo phishing" y perder cadenas de mayor severidad (OAuth code theft, SSRF bypass, cache poisoning).

**Del defensor:**
1. Validar con `startsWith("https://mi-dominio.com")`: `https://mi-dominio.com.evil.com` pasa.
2. Validar con `contains($dominio)`: `https://evil.com?x=mi-dominio.com` pasa.
3. "Sanear" tildando/quitan­do caracteres en vez de parsear y comparar host exacto.
4. Aceptar `javascript:`/`data:` porque "solo es un redirect": es XSS directo en muchos escenarios.

## Laboratorios y referencias

- **OWASP — Unvalidated Redirects and Forwards**: https://owasp.org/www-community/attacks/Unvalidated_Redirect
- **PortSwigger — SSRF labs** (bypass con open redirect): https://portswigger.net/web-security/ssrf
- **PayloadsAllTheThings — URL Redirection**: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/URL%20Redirection
- Notas internas: [[Server-Side Request Forgery (SSRF)]], [[Vulnerabilidades OAuth en PortSwigger Labs]], [[HTTP Host header attacks]]

## Cheat sheet

| Paso | Payload |
|---|---|
| Confirmar | `?next=https://google.com` → `302` |
| Bypass #1 | `//evil.com` |
| Bypass #2 | `/\evil.com`, `/\/evil.com` |
| Bypass #3 | `https:evil.com` (sin `//`) |
| Bypass #4 | `%2f%2fevil.com`, `%252eevil.com` (doble encode) |
| Bypass #5 | `https://mi-dominio.com@evil.com` (userinfo) |
| Bypass #6 | `https://mi-dominio.com.evil.com` (prefijo falso) |
| XSS | `?next=javascript:alert(1)` |
| SSRF chain | redirect a `http://169.254.169.254/` con follow del backend |
| Defensa | Parsear URL + allowlist exacta de host; nunca `startsWith`/`contains` |

## Relacionadas

- [[Server-Side Request Forgery (SSRF)]] — el encadenado más peligroso: redirect como bypass de whitelist
- [[Vulnerabilidades OAuth en PortSwigger Labs]] — robo de `code`/`token` vía redirect
- [[Cross-Site Scripting XSS]] — `javascript:` en redirecciones por JS/meta
- [[HTTP Host header attacks]] — generación de redirects desde el propio dominio
- [[Vulnerabilidades CORS y Laboratorios]] — confianza cruzada de orígenes, primo hermano del redirect
- [[Envenenamiento de Caché Web]] — `Location` envenenado en caché
