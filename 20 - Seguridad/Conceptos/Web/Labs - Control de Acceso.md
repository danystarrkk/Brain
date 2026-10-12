---
tags:
  - ciberseguridad
  - web
  - labs
tipo: concepto
estado: terminado
---

# Labs - Control de Acceso (PortSwigger)

Índice de walkthroughs de los laboratorios de **control de acceso** de PortSwigger Academy, con la metodología que los une. **Control de acceso** = quién puede ver o hacer qué. Cuando falla, un usuario autenticado (o anónimo) accede a **funciones u objetos** que le corresponde a otro rol u otro usuario.

> [!info] Taxonomía del control de acceso
> | Clase | Defecto | Ejemplo |
> |---|---|---|
> | **Vertical** (escalada) | rol bajo accede a funciones de rol alto | usuario normal llega a `/admin` |
> | **Horizontal** (pares) | usuario A accede a objetos de usuario B | `?id=123` → `?id=124` (ver [[Insecure Direct Object References IDOR]]) |
> | **Función ausente** | endpoint simplemente no verifica | `/admin/deleteUser` sin check |
> | **Metadatos confiables** | confiar en headers/parámetros client-side | `X-Original-URL`, `Referer`, `role=true` |
>
> Regla de oro: **el servidor debe autorizar cada request**, no el enrutador/JS del cliente. Cualquier dato que el cliente envía (query, body, cookie, header, método) es atacable.

## Metodología de prueba (aplicada a todos los labs)

1. **Mapear**: recorrer la app como usuario normal y anotar cada endpoint + parámetro relacionado con identidad (`id`, `user`, `role`, `account`) o acción sensible (`delete`, `admin`, `update`).
2. **Enumerar**: fuzzear rutas ocultas (`/admin`, `/admin-panel`, `/console`), parámetros (`roleid`, `isAdmin`), métodos (`GET/POST/PUT/DELETE`) y headers.
3. **Probar horizontal**: cambiar IDs/identificadores (usuario → usuario, objeto → objeto). IDs secuenciales y no secuenciales (buscar leaks).
4. **Probar vertical**: reproducir requests de admin como usuario normal (Repeater), incluidos pasos intermedios de flujos.
5. **Probar metadatos**: `X-Original-URL`, `X-Rewrite-URL`, `X-Forwarded-For`, `Referer`, método HTTP, `Host`.
6. **Verificar**: el cambio persiste y tiene efecto real (no solo un 200).

### Catálogo de payloads y técnicas

| Técnica | Payload / acción |
|---|---|
| Rol por parámetro | `role=admin`, `isAdmin=true`, `roleid=1`, `roleid=2` |
| IDOR de usuario | `?id=administrator`, `?id=carlos`, `?user=carlos` |
| IDOR de objeto | iterar `?postId=1..50`, UUIDs filtrados en la app |
| Rutas admin | `/admin`, `/admin-panel`, `/admin/delete`, `/console`, `/manager/html` |
| Headers de reescritura | `X-Original-URL: /admin`, `X-Rewrite-URL: /admin` |
| Cabecera de origen | `Referer: https://target/admin` |
| Método alternativo | `POST→GET`, `GET→POST`, `PUT`, `DELETE`, `X-HTTP-Method-Override` |
| Parámetro oculto | body con campos extra (`roleid`), cookie manipulada |

## Labs

### Lab: Unprotected admin functionality

El objetivo del lab es eliminar al usuario `carlos`.

La verdad es que no tenemos mayor información, por lo que vamos a usar [[Nmap Guía Completa|Nmap]] para enumerar la web usando el script de `http-enum`.
```bash
nmap --script http-enum -p443 0a8b0015038a9503842a04ee00f400bd.web-security-academy.net
```

![[Pasted image 20250923195359.png]]

No vemos que el `nmap` funcione en este caso, pero esto sería algo que podemos hacer en un principio.

> [!info]
> El script `http-enum` de Nmap es útil para descubrir directorios y archivos comunes en servidores web. Utiliza un diccionario de rutas conocidas y puede ayudar a identificar paneles de administración, archivos de configuración o directorios sensibles que no están enlazados directamente desde la página principal.

En este punto, podemos buscar rutas que nosotros conocemos, como `robots.txt`:
![[Pasted image 20250923195447.png]]

Vemos que nos dice que no está permitido que se indexe el panel de administración, que es `/administrator-panel`, pero nos lo está revelando directamente, por lo que podemos intentar ingresar:
![[Pasted image 20250923195623.png]]

> [!tip]
> El archivo `robots.txt` es un recurso valioso para el reconocimiento. Aunque su propósito es guiar a los rastreadores de motores de búsqueda, a menudo expone rutas sensibles o paneles de administración que los desarrolladores no quieren que sean indexados, pero que pueden ser accesibles directamente.

Perfecto, es cuestión de eliminar al usuario `carlos` y ya estaría:

![[Pasted image 20250923195658.png]]

Lab Terminado.


### Lab: Unprotected admin functionality with unpredictable URL

En este laboratorio, tenemos una funcionalidad de administración no protegida donde la URL es impredecible (no se puede forzar por fuerza bruta).

Bueno, podemos intentar ver el código fuente de la web a ver qué encontramos:
![[Pasted image 20250923195912.png]]

Vemos un código en JavaScript el cual ya nos revela la URL del panel de administración, que es `admin-xj19s8`:

![[Pasted image 20250923200006.png]]

> [!info]
> Es común que las aplicaciones web oculten rutas sensibles en el código fuente de JavaScript. Analizar los archivos `.js` puede revelar endpoints de API, rutas de administración o parámetros ocultos que no son visibles en el HTML renderizado directamente.

Pues esto ha funcionado, vamos a terminar el lab:
![[Pasted image 20250923200038.png]]

Lab Terminado.


### Lab: User role controlled by request parameter

Al parecer, nos habla de que el usuario tiene el rol asignado mediante un parámetro en la solicitud. Se tiene el usuario `wiener:peter` y el panel `/admin`.

Bueno, como nos habla de un parámetro en la solicitud, lo que podemos hacer es intentar capturar la petición y analizarla:
![[Pasted image 20250923200336.png]]

Vemos que, en efecto, tenemos un parámetro en la solicitud. Pasémoslo a `true` y veamos qué sucede al enviar la solicitud:

![[Pasted image 20250923200506.png]]

> [!tip]
> La manipulación de parámetros es una técnica común en el control de acceso. Si un parámetro como `admin=false` o `isAdmin=0` controla los privilegios del usuario, cambiar su valor a `true` o `1` puede otorgar acceso no autorizado. Siempre se debe validar la autorización en el lado del servidor.

Pasamos, ya con esto podemos terminar el lab:

![[Pasted image 20250923200546.png]]

![[Pasted image 20250923200554.png]]

Lab Terminado.


### Lab: User role can be modified in user profile

Al parecer, en nuestro panel de usuario podemos modificar el rol. Además, en la máquina solo se puede usar el panel de administración si eres un usuario con `roleid:2`.

Analizando la petición del campo de email, vemos lo siguiente:
![[Pasted image 20250923201453.png]]

Observamos que pasamos información que le enviamos como `email`, pero nos responde con más información de la que debería, y vemos el campo `roleid`. Podemos intentar modificar esto de la siguiente manera para que el campo `roleid` cambie:

![[Pasted image 20250923201620.png]]

![[Pasted image 20250923201650.png]]

> [!info]
> El `roleid` es un identificador numérico o alfanumérico que representa el nivel de privilegios de un usuario. Si este valor puede ser modificado por el usuario o se filtra en respuestas de la API, un atacante podría elevar sus privilegios simplemente cambiando el `roleid` a un valor que corresponda a un rol de administrador. Es un caso de [[Ataque de Asignación Masiva]]: el servidor acepta campos que el formulario no expone.

> [!warning]
> La fuga de información sensible en las respuestas de la API, como el `roleid` o cualquier otro dato de autorización, es una vulnerabilidad crítica. Los desarrolladores deben asegurarse de que las respuestas solo contengan la información estrictamente necesaria para el usuario autenticado.

Vemos que sí nos cambió y ya podemos acceder al panel de administración. Vamos a completar el lab:

![[Pasted image 20250923201716.png]]

Lab Terminado.

### Lab: User ID controlled by request parameter

En este lab, al parecer controlamos el ID de usuario por un parámetro en la petición, lo que podría permitir un salto a otro usuario:

![[Pasted image 20250923201934.png]]

Vemos que en la web tenemos el parámetro `id` el cual apunta a `wiener`. Podemos intentar cambiarlo a `administrator` a ver qué sucede:
![[Pasted image 20250923202019.png]]

> [!tip]
> Las referencias directas a objetos inseguras (IDOR) ocurren cuando una aplicación expone una referencia directa a un objeto de implementación interno, como un archivo, directorio o clave de base de datos, sin una verificación de autorización adecuada. Manipular un parámetro como `id=123` a `id=administrator` es un ejemplo clásico de IDOR. Ver [[Insecure Direct Object References IDOR]].

Logramos cambiar el usuario con esto, vamos a completar el lab:

![[Pasted image 20250923202145.png]]

Lab Terminado.


### Lab: User ID controlled by request parameter, with unpredictable user IDs

En este caso, la web utiliza IDs no predecibles, por lo que debemos encontrar la forma de encontrar el del usuario `carlos`.

Bueno, en este caso, nosotros tenemos que saber que cuando se publican cosas, en ocasiones logramos ver información de estos mismos filtrada. Vamos a ver si encontramos algo de `carlos`:

![[Pasted image 20250923203143.png]]

> [!info]
> Cuando los IDs de usuario no son secuenciales o predecibles, los atacantes a menudo buscan fugas de información en otras partes de la aplicación. Esto puede incluir comentarios de usuarios, publicaciones, metadatos de archivos o incluso en el código fuente de la página, donde los IDs únicos pueden ser expuestos inadvertidamente.

Parece que conseguimos el identificador de `carlos`.

Con esto podemos intentar saltar al usuario `carlos`:
![[Pasted image 20250923203252.png]]

Ya con esto podemos terminar el laboratorio:

![[Pasted image 20250923203318.png]]

Lab Terminado.


### Lab: User ID controlled by request parameter with data leakage in redirect

Aquí nos habla de que tenemos información filtrada en los datos de la respuesta.

![[Pasted image 20250923203931.png]]

Vamos a ver la respuesta y el redirect de esta a ver qué sucede. Por contexto, estamos cambiando el usuario de `wiener` a `carlos` mediante el parámetro de la URL y capturamos la respuesta:
![[Pasted image 20250923204132.png]]

Vemos un `302 Found` de un redirect. El problema es que vemos mucha información para ser un redirect. Vamos a ver qué es lo que nos da:
![[Pasted image 20250923204248.png]]

> [!warning]
> La fuga de información sensible en las respuestas de redirección HTTP (códigos 3xx) es un riesgo de seguridad. Aunque el navegador siga la cabecera `Location`, el cuerpo de la respuesta del redireccionamiento puede contener datos confidenciales, como API keys, tokens de sesión o IDs de usuario, que no deberían ser expuestos.

Vemos que ya tenemos la API-Key de `carlos`, por lo que vamos a usarla para terminar el lab:

![[Pasted image 20250923204404.png]]

Lab Terminado.


### Lab: User profile data leakage

En esta web, si cambiamos el usuario de `wiener` a `carlos`, vamos a ver lo siguiente:
![[Pasted image 20250923205014.png]]

Vemos que nos muestra la contraseña y, aunque la veamos así, modificando el HTML podríamos verla:
![[Pasted image 20250923205321.png]]

> [!tip]
> A menudo, las contraseñas u otra información sensible se ocultan visualmente en el HTML (por ejemplo, con `type="password"` o `display: none;`) pero siguen presentes en el código fuente de la página. Inspeccionar el DOM con las herramientas de desarrollo del navegador puede revelar estos datos ocultos.

Obtuvimos la contraseña del usuario `administrator` y con esto podemos ingresar como ese usuario:

![[Pasted image 20250923205442.png]]

Perfecto, terminemos el laboratorio.

![[Pasted image 20250923205505.png]]

Lab Terminado.


### Lab: Insecure direct object references

Bueno, vemos algo como parte del chat lo cual procesa diferentes opciones. Vamos a ver una en especial que llama nuestra atención:

![[Pasted image 20250923211336.png]]

Vamos a ver qué procesa:

![[Pasted image 20250923211539.png]]

Vemos información que se procesa en esta petición. Vamos a intentar ver qué pasa si pasamos valores de la siguiente manera:


![[Pasted image 20250923211735.png]]

> [!info]
> Las IDOR no solo se aplican a IDs de usuario, sino también a cualquier objeto referenciado directamente, como mensajes de chat, documentos, o registros de transacciones. Si la aplicación permite iterar a través de estos IDs (secuenciales o no) sin verificar la autorización del usuario para acceder a cada objeto, se produce una IDOR.

Nos movilizamos entre valores y vemos otra información donde encontramos una contraseña que es del usuario `carlos`:

![[Pasted image 20250923212004.png]]

Lab Terminado.

### Lab: URL-based access control can be circumvented

Tenemos restricciones en el front-end que no permiten el acceso a `/admin`. Por lo que leemos, tenemos una cabecera `X-Original-URL` la cual se emplea para evitar el acceso externo a la web.

Si nos habla de una cabecera, lo más probable es que esta se pueda modificar. Vamos a intentarlo:


![[Pasted image 20250923212838.png]]

Vemos que tenemos el acceso denegado en un comienzo. Podemos intentar agregar la cabecera, ya que no está, a ver si nos reescribe su valor:
![[Pasted image 20250923212958.png]]

> [!info]
> Cabeceras como `X-Original-URL`, `X-Rewrite-URL`, `X-Forwarded-For`, o `X-Forwarded-Host` son utilizadas comúnmente por proxies inversos o balanceadores de carga para reescribir o pasar información sobre la solicitud original. Si la aplicación backend confía ciegamente en estas cabeceras para el control de acceso, un atacante puede manipularlas para eludir las restricciones de URL.

Logramos pasar, vamos a completar el lab:
![[Pasted image 20250923213152.png]]

Al momento de eliminar, modificamos ligeramente la solicitud, ya que en la cabecera `X-Original-URL` no se pueden ingresar parámetros directamente, pero la técnica funcionó:
![[Pasted image 20250923213258.png]]

Lab Terminado.


### Lab: Method-based access control can be circumvented

Este laboratorio es sencillo. Lo que sucede es que tenemos un panel de administración `admin-roles` al cual no podemos acceder, pero que se le pasan dos valores `username` y `action`. Esto se procesa mediante `POST`. Podemos intentar cambiar esto a `GET` y que no necesitemos ser un usuario con grandes privilegios:
![[Pasted image 20250923214016.png]]

> [!tip]
> La manipulación del método HTTP (HTTP Method Tampering) es una técnica donde se cambia el método de una solicitud (por ejemplo, de `POST` a `GET` o viceversa) para eludir los controles de acceso. Algunas aplicaciones solo aplican restricciones a un método específico, dejando otros métodos desprotegidos para la misma URL.

Logramos darle permisos de administrador a `carlos`:
![[Pasted image 20250923214407.png]]

Lab Terminado.


### Lab: Multi-step process with no access control on one step

En este laboratorio, tenemos un proceso de múltiples pasos sin control de acceso en uno de los pasos. El objetivo es, de nuevo, obtener privilegios de administrador.

De igual forma que en la anterior, nosotros podemos lograr asignar privilegios de administrador de la siguiente manera:
![[Pasted image 20250923215120.png]]

> [!info]
> En procesos de múltiples pasos (por ejemplo, registro de usuario, cambio de contraseña, configuración de roles), es crucial que cada paso intermedio tenga sus propias verificaciones de autorización. Si un paso intermedio carece de control de acceso, un atacante podría saltarse los pasos iniciales y acceder directamente a una etapa avanzada con privilegios no autorizados. Relacionado: [[Laboratorios de Vulnerabilidades Lógicas y de Autenticación]] (flawed state machine, workflow bypass).

Conociendo la ruta y algunos parámetros, podemos ver cómo logramos asignar privilegios a `wiener`:
![[Pasted image 20250923215200.png]]

Lab Terminado.


### Lab: Referer-based access control

En este laboratorio, vamos a volver a ver algo del `Referer`. El objetivo sigue siendo el mismo: conseguir permisos de administrador para el usuario `wiener`.

De igual forma, configurando la cabecera `Referer`, podemos realizar solicitudes como otro usuario:
![[Pasted image 20250923215953.png]]

![[Pasted image 20250923220011.png]]

> [!tip]
> La cabecera `Referer` (o `Referrer` en su forma correcta) indica la URL de la página web desde la cual se hizo la solicitud actual. Algunas aplicaciones utilizan esta cabecera para implementar controles de acceso básicos, esperando que las solicitudes provengan de una página específica (por ejemplo, `/admin`). Manipular esta cabecera puede engañar a la aplicación para que otorgue acceso.

## Defensas

**1. Autorizar en el servidor, por recurso, en cada request** — deny by default:

```js
// Node.js/Express — middleware de autorización con matriz de roles
const ALLOW = {
  user:  ['GET /api/me', 'GET /api/orders/*'],
  admin: ['GET /api/me', 'GET /api/orders/*', 'DELETE /api/users/*', 'GET /admin/*'],
};

function authorize(req, res, next) {
  const key = `${req.method} ${req.route?.path ?? req.path}`;
  const grants = ALLOW[req.session.role] ?? [];
  const ok = grants.some(g => new RegExp('^' + g.replace(/\*/g, '.*') + '$').test(key));
  if (!ok) return res.status(403).end();
  next();
}
```

```python
# Django — control a nivel de objeto (horizontal)
def order_detail(request, order_id):
    order = get_object_or_404(Order, id=order_id, customer=request.user)  # scope por ownership
    return render(request, 'order.html', {'order': order})
```

**2. Nunca confiar en datos client-side** para autoridad: parámetros de rol, `isAdmin`, headers (`X-Original-URL`, `X-Forwarded-For`), `Referer`, método HTTP. Los roles viven en sesión/servidor; los IDs de objeto se validan contra el dueño.

**3. Denegar por defecto**: rutas admin fuera del alcance del enrutador público; 403 genérico; no "ocultar" la UI como control.

**4. Minimizar fugas**: respuestas con solo los campos necesarios (no `roleid`, no passwords, no API keys en 3xx); IDs no secuenciales ayudan pero **no sustituyen** la autorización.

**5. Auditoría continua**: tests automatizados que recorren cada endpoint como cada rol (matriz rol×recurso) — el control de acceso regresa con cada feature nueva.

## Errores comunes

**Atacante:**
1. Quedarse en la UI: probar solo lo visible; los endpoints intermedios/administrativos existen aunque no haya botón.
2. No probar métodos alternativos en el mismo path (`GET` del recurso que solo "protege" el `POST`).
3. Cambiar un parámetro y no **verificar el efecto real** (200 ≠ permiso concedido).
4. Olvidar que el IDOR puede estar en el **body** o en headers internos, no solo en query.

**Defensor:**
1. Filtrar `/admin` con reglas de proxy y olvidar la autorización en la app (cabeceras de reescritura la saltan).
2. Validar el rol al login y guardarlo como confiable en el cliente (cookie editable / JWT sin verificar).
3. Checks solo en el paso 1 de un flujo multipaso.
4. Respuestas con exceso de datos (`roleid`, hashes, keys) que facilitan el siguiente salto.

## Cheatsheet

| Objetivo | Prueba rápida |
|---|---|
| Ruta oculta | `robots.txt`, `/admin`, fuzz de directorios, JS comments |
| Rol client-side | `?role=admin`, `isAdmin=true`, `roleid=2` |
| Horizontal | `?id=<otro usuario>`; buscar IDs en respuestas |
| Vertical | replay de requests admin como usuario normal |
| Proxy/rewrite | `X-Original-URL: /admin`, `X-Rewrite-URL: /admin` |
| Método | cambiar `POST↔GET`, probar `PUT/DELETE`, override headers |
| Referer | `Referer: https://target/admin/...` |
| Multipaso | ejecutar pasos 2..N sin el paso 1 |

## Referencias

- PortSwigger Web Security Academy — Access control: https://portswigger.net/web-security/access-control
- OWASP API Security — BOLA/BFLA (API1:2023): https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/
- OWASP — Broken Access Control: https://owasp.org/Top10/A01_2021-Broken_Access_Control/

## Relacionadas

- [[Insecure Direct Object References IDOR]] — profundidad en la clase horizontal con labs propios
- [[Laboratorios de Vulnerabilidades Lógicas y de Autenticación]] — lógica de negocio, estados y oráculos (los otros labs de authz)
- [[JSON Web Tokens Explotación y Bypass]] — cuando el rol viaja en el token y la validación falla
- [[GraphQL Introspección y Mutaciones en Ataques IDOR]] — IDOR en el ecosistema GraphQL
- [[Gestión de Autenticación]] — el piso previo: identidad antes que autorización
- [[Ataque de Asignación Masiva]] — campos ocultos que cambian autoridad
