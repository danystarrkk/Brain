---
tags:
  - ciberseguridad
  - web
  - session-overloading
tipo: concepto
estado: terminado
---

# Session Puzzling, Session Fixation y Session Variable Overloading

Tres vulnerabilidades distintas que comparten superficie: **la gestión de sesiones**. Session Fixation ataca el *identificador*; Session Puzzling ataca la *máquina de estados* que la aplicación monta en la sesión; Session Variable Overloading ataca las *variables* almacenadas en ella. Entender los tres exige antes entender cómo funcionan las sesiones.

## Mecánica: cómo funcionan las sesiones HTTP

HTTP es **sin estado**: cada petición es independiente. Las sesiones añaden estado artificialmente:

1. El usuario se autentica → el servidor genera un **session ID** (string opaco, impredecible).
2. El ID viaja al cliente (casi siempre como cookie `Set-Cookie: PHPSESSID=...; HttpOnly; Secure; SameSite=Lax`).
3. En cada petición, el cliente devuelve la cookie; el servidor la usa como **llave** para cargar su porción de estado (en memoria, fichero, Redis, base de datos).
4. El estado (rol, carrito, `user_id`, flags de flujo) vive **en el servidor**; el cliente solo tiene la llave.

| Componente | Responsabilidad | Fallo asociado |
|---|---|---|
| **Generación del ID** | Entropía suficiente, impredecibilidad | Adivinanza/robo de ID |
| **Transporte (cookie)** | `Secure`, `HttpOnly`, `SameSite` | Robo vía XSS, fijación vía subdominio |
| **Ciclo de vida** | Regenerar al login, expirar al logout, timeout | Session Fixation |
| **Contenido del estado** | Validar todo lo que se guarda y se lee | Puzzling, Overloading |
| **Mapeo parámetro→estado** | No reflejar inputs en claves de sesión | Puzzling, Overloading |

> [!info] La distinción clave
> **Fixation** = el atacante *conoce* el ID con el que la víctima se autentica. **Puzzling** = el atacante *confunde el estado* de la sesión para que la app salte validaciones (sin necesitar conocer el ID). **Overloading** = el atacante *sobrescribe/inyecta* variables de sesión con valores no previstos por el flujo.

## Session Fixation

El atacante **fija** un ID de sesión válido en el navegador de la víctima *antes* de que esta se autentique. Si la aplicación **no regenera el ID al hacer login**, la sesión post-autenticación sigue siendo la que el atacante ya conoce → accede a la cuenta de la víctima.

### Flujos de fijación

| Vector | Mecanismo |
|---|---|
| **URL parameter** | `http://app.com/?PHPSESSID=attacker_id` — la app acepta el ID desde la URL (frameworks antiguos: ASP, Java `jsessionid`, PHP con `session.use_trans_sid=1`) |
| **Cookie cross-subdominio** | El atacante fija `Domain=.empresa.com` desde un subdominio que controle (ver CORS/lab de subdominios) → la cookie llega a toda la app |
| **Meta/cookie injection** | Si existe [[Vulnerabilidad de Redirección Abierta]] + [[Vulnerabilidades de HTTP Request Smuggling|CRLF injection]], se puede inyectar un `Set-Cookie` |
| **XSS** | Si no hay `HttpOnly`, el JS fija el ID con `document.cookie` — ver [[Cross-Site Scripting XSS]] |
| **Hidden form field** | Formulario que acepta un campo oculto con el ID |

### Secuencia del ataque

1. Atacante obtiene un ID válido: visita `app.com` y recibe `Set-Cookie: SESSION=abc123` (o fuerza la creación con el parámetro de URL).
2. Atacante envía a la víctima: `http://app.com/?SESSION=abc123` (o la siembra por cookie).
3. Víctima inicia sesión normalmente. **La app no cambia el ID**.
4. El atacante usa `SESSION=abc123` → está autenticado como la víctima.

> [!warning] La defensa no es "ocultar el ID", es **rotarlo**
> Incluso si la app ignora IDs externos, cualquier robo previo del ID (proxy, logs, Referer) se vuelve persistente sin regeneración. La única mitigación completa es `session_regenerate_id(True)` en el primer escalón de privilegio.

### Defensa con código

```php
// PHP — regenerar ID al autenticar y al elevar privilegios
session_start();
if (valida_credenciales($user, $pass)) {
    session_regenerate_id(true);   // true = destruye el ID anterior en el store
    $_SESSION['user_id'] = $user->id;
    $_SESSION['auth_time'] = time();
}
```

```java
// Java (Servlet) —Jakarta EE 6+ / Spring Security lo hace por defecto en login
request.changeSessionId();           // JDK 8+ / Servlet 3.1+
// antes de existir changeSessionId: invalidar y recrear
HttpSession old = request.getSession(false);
if (old != null) old.invalidate();
HttpSession neuva = request.getSession(true);
```

```javascript
// Node.js + express-session
app.post('/login', async (req, res) => {
  const user = await checkCredentials(req.body);
  if (user) {
    req.session.regenerate((err) => {      // ID nuevo, store antiguo borrado
      if (err) throw err;
      req.session.userId = user.id;
      res.redirect('/dashboard');
    });
  }
});
```

Atributos de cookie obligatorios:

```php
session_set_cookie_params([
    'lifetime' => 0,
    'path'     => '/',
    'secure'   => true,      // solo HTTPS
    'httponly' => true,      // sin acceso desde JS (mitiga XSS→robo)
    'samesite' => 'Lax',     // mitiga CSRF en navegadores modernos
]);
```

## Session Puzzling

**Session Puzzling** (también *Session Sequence Ambiguity*) es una vulnerabilidad de **máquina de estados**: muchas aplicaciones usan la sesión no solo para datos de usuario, sino para recordar **en qué paso de un flujo** está (¿ya verificó email? ¿ya eligió plan? ¿ya pasó el check de identidad?). Si el atacante puede **inyectar o manipular el nombre/valor de esas claves de estado**, puede hacer que la aplicación crea que un paso intermedio ya se completó, saltándose controles de seguridad.

### Mecanismo clásico

```php
// Flujo de reset de contraseña (vulnerable)
// Paso 1: el email llega por parámetro y se guarda tal cual en la sesión
$_SESSION['email'] = $_GET['email'];          // ← acepta CUALQUIER email

// Paso 2 (en otro endpoint): la app confía en la sesión
if (isset($_SESSION['email'])) {
    showPasswordResetForm($_SESSION['email']);  // el atacante resetea la cuenta ajena
}
```

El atacante solo debe visitar `paso1?email=victim@empresa.com` y luego `paso2`: la aplicación cree que la víctima pasó por el flujo legítimo. La misma familia de bug aparece en:

- Flujos 2FA: `$_SESSION['2fa_ok'] = $_GET['verify']` → bypass de segundo factor.
- Onboarding/elevación: `$_SESSION['role']` poblada desde un parámetro.
- Checks de elegibilidad (KYC, región, edad) guardados desde inputs.

> [!info] Origen del término
> El concepto lo popularizó SANS (Puzzling = *session state confusion*): explotar la secuencia de pasos almacenados en la sesión, a menudo combinando **parameter pollution** de nombres que la app asocia a claves de sesión — ver también [[Ataque de Asignación Masiva]] cuando el framework mapea `?status=...` → `session['status']`.

### Metodología de detección

1. Mapear todos los parámetros aceptados por cada endpoint del flujo (login, verify, reset, checkout, admin).
2. Probar si parámetros de nombre igual a claves de sesión (`?admin=1`, `?step=3`, `?authenticated=true`, `POST email=`) quedan almacenados en la sesión.
3. Continuar el flujo con una sesión "adelantada" artificialmente y observar si los pasos se omiten.
4. Revisar respuestas del servidor: si un `Set-Cookie` o el store de sesión (a veces visible vía ID) refleja inputs del usuario, hay fuga+manipulación.

### Defensa con código

```php
// SEGURO: los estados de flujo solo los pone el servidor, con valores internos
session_start();

// Paso 1: validación real, no aceptar el email como input
$_SESSION['reset_user_id'] = lookupUserByToken($_GET['token']); // token firmado, no email crudo
$_SESSION['reset_step'] = 2;                                    // valor interno, no user-input

// Paso 2: verificar en servidor, no asumir
if (($_SESSION['reset_step'] ?? 0) < 2) {
    http_response_code(403); exit('Flujo inválido');
}
```

```python
# SEGURO (Flask): allowlist de claves de sesión y valores internos
ALLOWED_SESSION_KEYS = {"user_id", "cart", "csrf_token", "2fa_verified"}

def guarda_en_sesion(key, value):
    assert key in ALLOWED_SESSION_KEYS, f"clave de sesión no permitida: {key}"
    session[key] = value
```

## Session Variable Overloading

El atacante envía **masivamente** datos hacia las variables de sesión — ya sea por volumen (DoS) o por **sobrescritura de claves legítimas** (corrupción de estado / elevación):

| Tipo | Vector | Impacto |
|---|---|---|
| **Volumétrico** | Miles de parámetros/cookies que la app copia a `$_SESSION` | Consumo de memoria/ disco del store (sesiones en ficheros/Redis sin límite) → DoS |
| **Sobrescritura** | `?role=admin`, `?user_id=1`, `?price=0` si la app hace `$session[$k]=$v` con claves controladas | Elevación de privilegios, fraude de precio, bypass lógico |
| **Type confusion** | `?auth=true` donde la app espera un objeto → inyección de tipos | Bypass de checks (`strcmp` null, comparaciones débiles — ver [[Ataque de Type Juggling]]) |
| **Serialización** | La sesión se persiste con `serialize()` y el atacante controla parte del grafo | Objeto deserializado → RCE — ver [[Ataques de Deserialización]] |

Caso histórico amplificador: PHP con `register_globals=On` (≤5.3) mapeaba *todo* input directamente al namespace global/sesión; la misma mala práctica sobrevive hoy en frameworks que hacen "mass assignment" a objetos de sesión.

### Defensa con código

```php
// VULNERABLE: copia sin filtro cualquier parámetro a la sesión
foreach ($_GET as $k => $v) { $_SESSION[$k] = $v; }

// SEGURO: schema explícito + límites
$schema = ['nombre' => 'string:64', 'lang' => 'enum:es,en,pt'];
foreach ($schema as $k => $spec) {
    if (!isset($_GET[$k])) continue;
    $_SESSION[$k] = validate($_GET[$k], $spec);   // tipo, longitud y valor acotados
}
```

```ini
; php.ini — límites duros del store de sesión
session.max_input_vars = 1000      ; tope de variables por request
session.gc_maxlifetime = 1800      ; recolección de sesiones viejas
; y en producción: session.save_handler = redis con maxmemory y TTL
```

## Prevención general (checklist)

| Medida | Fixation | Puzzling | Overloading |
|---|---|---|---|
| `session_regenerate_id(true)` en login y elevación | ✔ | | |
| Cookies `Secure` + `HttpOnly` + `SameSite` | ✔ | | ✔ (reduce robo) |
| IDs con ≥64 bits de entropía (CSPRNG) | ✔ | | |
| Timeout de inactividad + invalidación en logout | ✔ | ✔ | |
| Allowlist de claves de sesión; nunca user-input como clave ni valor de estado | | ✔ | ✔ |
| Validar tipo/longitud de todo dato guardado en sesión | | ✔ | ✔ |
| Límites de tamaño y TTL en el store de sesión | | | ✔ |
| No exponer el ID en URL (`use_trans_sid=0`) | ✔ | | |

> [!tip] Entropía del ID
> Un ID débil no es "fijación" pero produce el mismo resultado: `1,2,3...`, `md5(username)`, timestamps o IDs de 8 caracteres son adivinables por fuerza bruta — una debilidad de generación que combina con todo lo anterior. Usar CSPRNG: `bin2hex(random_bytes(32))`.

## Notas Prácticas

En este laboratorio, la cookie de sesión resultó ser un [[JSON Web Tokens Explotación y Bypass|JWT]], lo que abre otra superficie (algoritmo, claims, `none`).

![[Pasted image 20260227173616.png]]

Vamos a acceder con `admin:admin`:

![[Pasted image 20260227173746.png]]

Como observamos, tenemos una cookie de sesión la cual parece ser un JSON Web Token. Veamos qué es lo que se puede hacer:

![[Pasted image 20260227173957.png]]

Observamos que tenemos un panel para cambiar la contraseña, donde vamos a capturar la petición primero:

![[Pasted image 20260227174157.png]]

Vemos cómo se tramita y, al cargar, nos dice:

![[Pasted image 20260227174213.png]]

Nos tiene que llegar un correo para cambiar la contraseña, pero si nosotros en el Burp Suite capturamos la respuesta:

![[Pasted image 20260227174305.png]]

Estamos observando una cookie de sesión nueva emitida **en la respuesta del endpoint de cambio de contraseña** — el flujo emite un token/sesión que no debería existir todavía en el flujo de reset. Podemos intentar movernos a algún apartado específico como `dashboard` y, si lo arrastra, podríamos acceder:

![[Pasted image 20260227174452.png]]

> [!info] Lectura del escenario
> Este patrón es **Session Puzzling** práctico: la aplicación emite/acepta estado de sesión en un punto intermedio del flujo (cambio de contraseña pendiente de email) y luego **arrastra ese estado** a endpoints privilegiados (`dashboard`), saltándose la verificación que el flujo pretendía imponer. El hallazgo se repasa en [[Gestión de Autenticación]].

## Errores comunes

**Del atacante:**
1. Probar fixation solo por URL y olvidar el vector cross-subdominio (`Domain=.padre.com`) — el más frecuente en apps con muchos subdominios.
2. Confundir puzzling con ID guessing: puzzling no requiere conocer el ID, requiere **manipular el estado**; el approach correcto es mapear el flujo, no fuerzar IDs.
3. No probar nombres de claves de sesión como parámetros (`?2fa_verified=1`, `?step=4`, `?cart.total=0`).
4. Descartar JWTs como "solo cookies": rotar/algoritmo/claims son fixation-sobre-claims — ver [[JSON Web Tokens Explotación y Bypass]].

**Del defensor:**
1. Regenerar el ID solo en login y **no** al elevar privilegios (cambiar rol, añadir 2FA, pago) → fixation parcial residual.
2. Asumir que "si la clave no está en el formulario no puede llegar": frameworks con mass-assignment mapean cualquier parámetro — ver [[Ataque de Asignación Masiva]].
3. Almacenar tokens de reset/2FA en la sesión **sin TTL y sin un solo uso**.
4. Cerrar sesión sin invalidar server-side: el ID robado sigue vivo hasta el GC.

## Laboratorios y referencias

- **PortSwigger Academy — Authentication**: labs de bypass de 2FA, password reset, fixation encadenada: https://portswigger.net/web-security/authentication
- **OWASP Session Management Cheat Sheet**: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- **SANS — Session Puzzling**: https://www.sans.org/blog/ (búsqueda: session puzzling)
- **OWASP Mass Assignment**: https://owasp.org/www-project-web-security-testing-guide/
- Notas internas: [[Gestión de Autenticación]], [[JSON Web Tokens Explotación y Bypass]], [[Laboratorios de Vulnerabilidades Lógicas y de Autenticación]]

## Cheat sheet

| Concepto | Ataca | Señal | Defensa clave |
|---|---|---|---|
| **Session Fixation** | El ID | ID igual antes/después del login | `session_regenerate_id(true)` |
| **Session Puzzling** | El flujo/estado | Endpoints que confían en flags de sesión poblables | El servidor pone los estados; allowlist de claves |
| **Session Variable Overloading** | Las variables | Inputs masivos → sesión | Validar tipo/longitud; límites de store |
| Cookies seguras | Transporte | — | `Secure` `HttpOnly` `SameSite` |
| Entropía | Generación | IDs cortos/predictibles | `random_bytes(32)` |

## Relacionadas

- [[Gestión de Autenticación]] — el ciclo completo de auth donde viven estas fallas
- [[JSON Web Tokens Explotación y Bypass]] — cuando el "ID de sesión" es un token firmado débil
- [[Cross-Site Request Forgery (CSRF)]] — abusa de la sesión existente sin fijarla
- [[Ataque de Asignación Masiva]] — el mecanismo que suele poblar claves de sesión ajenas
- [[Cross-Site Scripting XSS]] — robo/fijación de ID cuando falta `HttpOnly`
- [[Laboratorios de Vulnerabilidades Lógicas y de Autenticación]] — práctica combinada
