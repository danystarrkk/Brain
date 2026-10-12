---
aliases: ["API", "API REST", "APIs web", "Interfaz de Programación"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-02-01
actualizado: 2026-10-11
relacionado:
  - "[[Formato Json]]"
  - "[[Java Script]]"
  - "[[Acceder a Html]]"
  - "[[Eventos]]"
  - "[[Node JS]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTTP/Overview"
---

# Qué es una API

Una **API** (*Application Programming Interface*, "interfaz de programación de aplicaciones") es un **contrato formal que define cómo un programa puede pedirle cosas a otro**, sin necesidad de saber cómo funciona por dentro. El término no es exclusivo de la web: el [[Document Object Model (DOM)|DOM]] del navegador es una API (`document.querySelector(...)` es una llamada a una API); `fetch`, `localStorage`, `prompt` y hasta `Math.round` son APIs.

El caso que nos interesa es la **API web**: un contrato que permite que un cliente (un frontend, una app móvil, un script) se comunique con un servidor a través de HTTP. Su propósito es la **integración sin reconstrucción**: si querés mapas en tu web, desarrollar un motor de mapas propio es costoso y absurdo — usás la API de Google Maps (o la de Leaflet/OpenStreetMap) y consumís un servicio ya construido.

> [!info] La idea central en una frase
> Una API **oculta la implementación** y **expone una superficie estable**. Vos pedís "la lista de usuarios" sin saber si atrás hay PostgreSQL, un cache en Redis o treinta microservicios. Ese desacoplamiento es lo que hace posible el software moderno.

## Vocabulario mínimo

| Término | Qué es | Ejemplo |
|---|---|---|
| **API** | El contrato completo: reglas, formatos, operaciones disponibles | "La API de GitHub" |
| **Endpoint** | Una URL concreta del contrato sobre la que se puede hacer una petición | `https://api.github.com/repos/owner/repo` |
| **Recurso** | La entidad que se manipula (siempre sustantivos, en plural) | usuarios, pedidos, productos |
| **Cliente** | Quien hace la petición (frontend web, app móvil, otro servidor) | tu aplicación con `fetch` |
| **Servidor** | Quien procesa la petición y responde | la API desplegada en la nube |
| **SDK** | *Software Development Kit*: librería + herramientas + documentación que un proveedor publica para facilitar el uso de su API desde un lenguaje concreto | `@google/maps` para JavaScript |
| **Token / API key** | Credencial que identifica al cliente y autoriza el uso | `Authorization: Bearer eyJhbGci...` |
| **Contrato** | La documentación de qué existe, qué recibe y qué devuelve cada endpoint | OpenAPI/Swagger, documentación del proveedor |

El modelo de funcionamiento es **cliente-servidor**: el cliente construye una petición HTTP respetando las reglas del contrato, el servidor la procesa y devuelve una respuesta **estructurada** (casi siempre en [[Formato Json|JSON]]) que el cliente interpreta y renderiza.

## API vs librería vs framework

Tres palabras que se confunden todo el tiempo porque las tres "reutilizan código ajeno", pero responden a problemas distintos:

| | **API** | **Librería** | **Framework** |
|---|---|---|---|
| **Qué es** | Un contrato de comunicación | Un conjunto de funciones que **llamás vos** | Un esqueleto de aplicación **que llama al código de vos** |
| **Control** | Vos pedís, el otro responde | Vos tenés el flujo (`main`) | El framework tiene el flujo (***inversión de control***) |
| **Ejemplo** | REST API de Twitter, DOM, `fetch` | GSAP (ver [[Librería GSAP]]), axios | React (ver [[ReactJS - Fundamentos]]), Angular |
| **Límite** | Puede estar en otro proceso/máquina/red | Corre en tu proceso | Estructura tu proyecto entero |

> [!tip] El test rápido de la inversión de control
> Si **vos** llamás al código → librería. Si **el código** te llama a vos en momentos que él decide (ciclo de vida de componentes, hooks de middleware) → framework. Una API es una capa más: puede estar detrás de una librería (axios es una librería *cliente* de la API `fetch`/HTTP).

## REST: la arquitectura dominante en la web

**REST** (*Representational State Transfer*) no es un protocolo ni un formato: es un **conjunto de restricciones de diseño** propuestas por Roy Fielding (2000) para sistemas distribuidos. Una API "REST" (o "RESTful") se organiza así:

1. **Todo es un recurso**, identificado por una URL única: `/usuarios`, `/usuarios/42`.
2. **Los recursos se manipulan con los verbos HTTP** estándar (GET, POST, PUT, PATCH, DELETE) — ver la tabla más abajo.
3. **El cliente envía y recibe representaciones** del recurso, normalmente JSON (pero podría ser XML o HTML). JSON es el formato de facto: legible, soportado nativamente en JS (ver [[Formato Json]]).
4. **Statelessness**: cada petición lleva **toda** la información necesaria. El servidor no recuerda "quién es" el cliente entre una petición y la siguiente; la identidad viaja en cada request (token, cookie de sesión). Esto es lo que permite escalar: cualquier instancia del servidor puede atender cualquier petición.
5. **Cacheabilidad**: las respuestas declaran si pueden guardarse (`Cache-Control`, `ETag`) y el cliente/servidor reutilizan lo cacheado sin volver a pedir.
6. **Interfaz uniforme**: URLs como sustantivos + verbos estándar + códigos de estado con significado. Nadie inventa `/getUserData?action=delete`.

Ejemplo de diseño de recurso:

```
GET    /api/v1/usuarios          → lista (soporta paginación y filtros)
GET    /api/v1/usuarios/42       → un usuario
POST   /api/v1/usuarios          → crea (devuelve 201 + Location)
PUT    /api/v1/usuarios/42       → reemplaza completo
PATCH  /api/v1/usuarios/42       → modificación parcial
DELETE /api/v1/usuarios/42       → elimina (devuelve 204)
```

> [!warning] "REST" mal usado
> Que un endpoint sea una URL que devuelve JSON no lo hace REST. Los pecados típicos: usar GET para mutar (`/borrarUsuario?id=3`), verbos en la URL en vez de HTTP methods, y responder siempre 200 sin importar qué pasó. Ver "Errores comunes" abajo.

## Métodos HTTP: la tabla definitiva

| Método | Significado | ¿Cambia el servidor? | ¿Idempotente? | ¿Seguro? | Éxito típico | Cuerpo en la petición | Cuerpo en la respuesta |
|---|---|---|---|---|---|---|---|
| `GET` | Leer recurso | No | Sí | Sí | `200 OK` | No permitido | Sí (el recurso) |
| `HEAD` | Igual que GET pero **sin cuerpo** (solo cabeceras) | No | Sí | Sí | `200 OK` | No | No |
| `POST` | Crear recurso / ejecutar acción | Sí | **No** | No | `201 Created` | Sí (JSON) | Sí (el recurso creado) |
| `PUT` | Reemplazar recurso **completo** | Sí | Sí | No | `200`/`204` | Sí (JSON completo) | Opcional |
| `PATCH` | Modificación **parcial** | Sí | Depende | No | `200 OK` | Sí (JSON parcial) | Sí |
| `DELETE` | Eliminar recurso | Sí | Sí | No | `204 No Content` | No | Opcional |
| `OPTIONS` | Qué métodos/headers acepta (preflight CORS) | No | Sí | Sí | `204` | No | Cabeceras |

- **Idempotente**: repetir la llamada N veces deja el servidor en el mismo estado que hacerla una vez. `PUT` y `DELETE` deben serlo (un reintento de red no puede duplicar nada). `POST` no lo es — por eso las APIs de pago usan **claves de idempotencia** para que un reintento no cobre dos veces.
- **Seguro**: no tiene efectos secundarios. Solo `GET`/`HEAD` lo son; por eso **nunca** se usa GET para acciones (los crawlers de Google, los prefetch del navegador y el historial "ejecutan" GETs ajenos).

### Semántica PUT vs POST vs PATCH

| Escenario | Método | Ejemplo |
|---|---|---|
| "Creá este usuario nuevo, asignale ID" | `POST /usuarios` | registro, alta de pedido |
| "Reemplazá el perfil completo" | `PUT /usuarios/42` | formulario que envía todos los campos |
| "Cambiame solo el email" | `PATCH /usuarios/42` | toggle, edición parcial |
| "Quitá el recurso" | `DELETE /usuarios/42` | baja de cuenta |

## Códigos de estado: qué devolvió el servidor

El código de estado es parte del contrato: le dice al cliente qué clase de resultado obtuvo, **sin abrir el cuerpo**.

| Rango | Familia | Significado |
|---|---|---|
| `1xx` | Informativo | Procesando (rara vez se ve en JS) |
| `2xx` | **Éxito** | La petición funcionó |
| `3xx` | Redirección / caché | El recurso está en otro lado o ya lo tenés |
| `4xx` | **Error del cliente** | La petición está mal hecha o no autorizada — culpa de quien llama |
| `5xx` | **Error del servidor** | El servidor falló — no se arregla cambiando la petición |

Códigos que vas a usar constantemente:

| Código | Nombre | Cuándo | Nota para el frontend |
|---|---|---|---|
| `200` | OK | Éxito con cuerpo | El caso feliz |
| `201` | Created | Recurso creado vía POST | Suele traer el recurso nuevo en el cuerpo y su URL en `Location` |
| `204` | No Content | Éxito sin devolver nada | `DELETE` exitoso; `res.json()` fallaría (no hay cuerpo) |
| `301`/`308` | Moved / Permanent Redirect | La URL cambió | El navegador sigue la pista solo |
| `304` | Not Modified | Caché válida (`ETag`) | Ahorra transferencia; fetch lo maneja solo |
| `400` | Bad Request | JSON malformado, campo inválido | Mostrar el detalle del cuerpo al usuario/dev |
| `401` | Unauthorized | **No autenticado** (sin token o token vencido) | Redirigir a login; refrescar token |
| `403` | Forbidden | Autenticado pero **sin permiso** | No reintentar con las mismas credenciales |
| `404` | Not Found | El recurso no existe | Cuidado: puede ser también "no te dejamos ver" |
| `405` | Method Not Allowed | Usaste un método que ese endpoint no acepta | Error de integración |
| `409` | Conflict | Estado actual impide la operación | Ej.: email ya registrado |
| `422` | Unprocessable Entity | Sintaxis ok, reglas de negocio inválidas | Errores de validación campo a campo |
| `429` | Too Many Requests | **Rate limit** superado | Ver "Rate limiting" abajo |
| `500` | Internal Error | El servidor explotó | Reintento con backoff + avisar |
| `502`/`503` | Bad Gateway / Service Unavailable | Servicio caído o en mantenimiento | Reintento con backoff |
| `504` | Gateway Timeout | El servidor upstream no respondió a tiempo | Reintento con backoff |

> [!warning] `200` con error adentro
> Algunas APIs responden `200 OK` con `{"error": "..."}` en el cuerpo. Es un anti-patrón: `fetch` solo ve el `200` y el cliente cree que todo salió bien. Si heredás una API así, el chequeo de éxito tiene que mirar el cuerpo, no solo el status.

## Cabeceras HTTP que importan

| Cabecera | Dirección | Función |
|---|---|---|
| `Content-Type` | request/response | Qué formato trae el cuerpo: `application/json`, `multipart/form-data` |
| `Accept` | request | Qué formatos acepta el cliente: `application/json` |
| `Authorization` | request | Credencial: `Bearer <token>` o `Basic base64` |
| `User-Agent` | request | Quién hace la petición (navegador, app) |
| `Cache-Control` | response | Política de caché: `max-age=3600`, `no-store` |
| `ETag` / `If-None-Match` | response/request | Identificador de versión del cuerpo; permite `304 Not Modified` |
| `Location` | response | URL del recurso creado (con `201`) o de la redirección (`3xx`) |
| `Set-Cookie` | response | Guarda cookies de sesión en el navegador |
| `Retry-After` | response | Segundos a esperar tras un `429`/`503` |

## El contrato JSON

El cuerpo casi siempre es JSON: texto con una estructura de objetos/arreglos que JS interpreta nativamente (ver [[Formato Json]] para el formato en detalle).

```json
{
  "id": 42,
  "nombre": "Ada",
  "email": "ada@ejemplo.com",
  "roles": ["admin", "editor"],
  "activo": true,
  "direccion": null
}
```

Convenciones de contrato que conviene conocer:

- **Nombres de campos**: muchas APIs usan `snake_case` (`created_at`), JS prefiere `camelCase` (`createdAt`). Elegí uno y documentalo; si consumís APIs ajenas, convertí en la capa de cliente.
- **Envelope (sobre)**: respuestas de lista envueltas en metadatos de paginación: `{ "data": [...], "page": 2, "totalPages": 9 }`.
- **`null` vs ausente**: `null` significa "explícitamente sin valor"; la ausencia de la clave puede significar "no aplica". Las APIs RESTful bien diseñadas son consistentes con esto.
- **Fechas**: se envían en ISO 8601 (`"2026-10-11T14:30:00Z"`) — en JS, `new Date("2026-10-11T14:30:00Z")` las parsea sin ambigüedad (los formatos locales no).

## Autenticación: quién sos

| Esquema | Cómo funciona | Ventaja | Desventaja | Uso típico |
|---|---|---|---|---|
| **API key** | Un string fijo en la URL/header identifica al *proyecto* | Simple | No identifica al *usuario*; si se filtra, hay que rotarla | Acceso servidor-a-servidor (mapas, clima) |
| **Basic Auth** | `Authorization: Basic base64(user:pass)` en cada petición | Trivial | Credenciales en claro en cada request; solo con HTTPS | Tools internas |
| **Cookie de sesión** | El servidor guarda la sesión y emite una cookie; el navegador la adjunta solo | El navegador maneja todo; protección CSRF integrable | Acoplado al navegador; statefulness del lado servidor | Web clásica (MPA, ver [[Uso de Java Script]]) |
| **JWT (Bearer token)** | El servidor firma un token autocontenido (`header.payload.firma`); el cliente lo adjunta en cada petición | **Stateless**: cualquier instancia del servidor valida el token sin consultar la base | No se puede revocar antes de expirar; pesado si guardás mucho estado; cuidado con el almacenamiento en el cliente | SPAs, APIs de móvil (ver [[Uso de Java Script]]) |
| **OAuth 2.0 / OpenID Connect** | Delegación: "entrá con Google" emite tokens vía un proveedor | No manejás contraseñas propias | Complejo de implementar bien | Login social, APIs de terceros |

> [!warning] Dónde guardan el frontend los tokens
> `localStorage` es accesible por cualquier JS de la página: un XSS (ver [[Recomendaciones]]) roba el token al instante. Cookies `httpOnly` no son accesibles por JS (mitigan el robo por XSS, pero exigen protección CSRF). Es un trade-off real, no un detalle — se decide en cada proyecto.

Un **JWT** es un string de tres partes separadas por puntos: `header.payload.firma`, donde el payload es base64 de JSON con datos del usuario (`sub`, `exp`...). No está cifrado: cualquiera lo lee; la **firma** garantiza que nadie lo alteró. Que sea *self-contained* es su poder (validás sin ir a la base) y su riesgo (no hay "cerrar sesión" real hasta que expire).

> [!info] API keys y seguridad
> Una API key es una **llave de uso**, no una contraseña: identifica al cliente para facturación y rate limit, pero **no autentica usuarios**. En el navegador solo van keys marcadas como "restringidas" (por dominio/referrer); cualquier key expuesta en un repo o bundle público debe considerarse filtrada.

## `fetch`: cómo las consume un frontend

`fetch` es la API nativa del navegador para hacer peticiones HTTP (nace como promesa, así que funciona con `async/await`):

```js
async function getUsuario(id) {
  try {
    const res = await fetch(`https://api.ejemplo.com/v1/usuarios/${id}`, {
      method: 'GET',                       // por defecto
      headers: {
        'Accept': 'application/json',
        'Authorization': `Bearer ${token}`,
      },
      // body: JSON.stringify(datos),     // solo en POST/PUT/PATCH
      // signal: controller.signal,       // para AbortController
    });

    // ¡Cuidado! fetch NO lanza error con 4xx/5xx:
    // solo rechaza la promesa si falla la RED.
    if (!res.ok) {
      const detalle = await res.text();
      throw new Error(`HTTP ${res.status}: ${detalle}`);
    }

    if (res.status === 204) return null;  // No Content: no hay .json()
    return await res.json();              // ya parseado
  } catch (err) {
    console.error('Falló la petición:', err.message);
    throw err;
  }
}
```

Enviar datos:

```js
await fetch('https://api.ejemplo.com/v1/usuarios', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ nombre: 'Ada', email: 'ada@ejemplo.com' }),
});
```

Abortar una petición que ya no importa (ej.: el usuario siguió escribiendo en el buscador):

```js
let controller;
async function buscar(termino) {
  controller?.abort();                      // cancela la anterior
  controller = new AbortController();
  const res = await fetch(`/api/buscar?q=${encodeURIComponent(termino)}`, {
    signal: controller.signal,
  });
  return res.json();
}
```

> [!tip] Ir más allá
> `fetch` cubre el 90% de los casos. Para patrones más altos — reintentos, caché por petición, deduplicación, queries con estado — existen librerías como TanStack Query o SWR, y en [[ReactJS - Fundamentos|React]] el patrón *loader* de React Router. Este nota es el mapa; el detalle de selección del DOM está en [[Acceder a Html]] y el de JSON en [[Formato Json]].

## CORS: por qué tu API "no funciona" en el navegador

El navegador aplica la **same-origin policy**: el JS de `https://tu-app.com` no puede leer respuestas de `https://api-otra.com` salvo que **esa API lo permita**. CORS (*Cross-Origin Resource Sharing*) es el protocolo de permisos:

1. El cliente hace la petición (si es "compleja", el navegador primero manda un preflight `OPTIONS`).
2. El servidor responde con `Access-Control-Allow-Origin: https://tu-app.com` (o `*`).
3. Si el origen está permitido, el navegador **entrega** la respuesta al JS; si no, el fetch falla con un error CORS.

**Lo crucial**: el CORS **no es un bug del frontend ni se arregla del lado del cliente**. Es una restricción del navegador para proteger al usuario; un script de Node o `curl` no la tiene. Soluciones legítimas:

| Solución | Cuándo |
|---|---|
| Pedir al dueño de la API que permita tu origen | Lo correcto; la API debe enviar `Access-Control-Allow-Origin` |
| Proxy en tu dev server (Vite: `server.proxy`, ver [[ReactJS - Vite]]) | Desarrollo local |
| Proxy/serverless en tu propio backend | Producción: tu servidor llama a la API (sin CORS) y la repasa |
| "Deshabilitar CORS" en el navegador | **Nunca** en producción: solo anula la protección en TU navegador, el de tus usuarios sigue bloqueando |

Preflight: cualquier petición con `Authorization`, `Content-Type: application/json` o métodos no-simple dispara primero un `OPTIONS` que pregunta "¿me dejás hacer POST con esas cabeceras?". Si tu servidor solo implementa el GET, el preflight falla y el navegador ni siquiera intenta el POST — síntoma clásico: "funciona con `curl`, no funciona en la web".

## Rate limiting: el derecho a decir "no"

Las APIs limitan cuántas peticiones aceptan por ventana de tiempo (por API key, por IP o por usuario) para proteerse del abuso y de los bucles disparados por errores de cliente.

- **`429 Too Many Requests`**: se superó el límite. La respuesta suele traer `Retry-After: 30` (segundos) o headers `X-RateLimit-Remaining` / `X-RateLimit-Reset`.
- **Cómo consumir bien**: respetá `Retry-After`, y si no viene, aplicá **backoff exponencial** con jitter:

```js
async function conReintentos(fn, intentos = 3) {
  for (let i = 0; i <= intentos; i++) {
    try {
      return await fn();
    } catch (err) {
      const esReintentable =
        err.status === 429 || err.status === 503 || err.status >= 500;
      if (!esReintentable || i === intentos) throw err;
      const espera = 2 ** i * 500 + Math.random() * 300; // 0.5s, 1s, 2s + jitter
      await new Promise((r) => setTimeout(r, espera));
    }
  }
}
```

> [!warning] Reintentos ciegos matan APIs
> Reintentar un `POST` sin clave de idempotencia puede **duplicar** el recurso (dos pedidos, dos cobros). Reintentá de forma agresiva solo lecturas (`GET`) y con criterio las mutaciones.

## Patrones de manejo de errores en el cliente

| Patrón | Idea |
|---|---|
| **Chequear `res.ok`** | Todo éxito vive en 2xx; lanzá un error con `res.status` para que el `catch` lo maneje |
| **Separar tipos de fallo** | `TypeError` de red (sin conexión), `AbortError` (cancelado) y error HTTP (4xx/5xx) se muestran distinto al usuario |
| **Timeout** | `AbortSignal.timeout(8000)` (o un `setTimeout` + `abort`) para no dejar la UI colgada |
| **Reintento con backoff** | Solo para 429/5xx y GETs (arriba) |
| **Degradar con gracia** | Datos stale > mensaje de error; nunca una pantalla en blanco |
| **Error del servidor ≠ error del usuario** | 4xx: mensaje accionable ("no tenés permiso"); 5xx: "algo falló de nuestro lado, probá más tarde" |

Distinción fundamental para depurar: **un 4xx lo arreglás en el código que llama; un 5xx lo arregla el equipo del servidor.**

## Cómo un frontend consume una API (de punta a punta)

1. **Descubrir el contrato**: leé la documentación — endpoints, auth, límites.
2. **Capa de acceso**: centralizá las llamadas en un módulo (`api.js`) con la base URL y los headers comunes, en vez de `fetch` suelto por componente.
3. **Estados de UI**: cargando / éxito / error / vacío. Toda petición asíncrona tiene los cuatro.
4. **Async/await + Promise.all**: paralelizá lecturas independientes:

```js
const [usuarios, config] = await Promise.all([
  fetch('/api/usuarios').then((r) => r.json()),
  fetch('/api/config').then((r) => r.json()),
]);
```

5. **Caché y revalidación**: lo que no cambia seguido, cachear (ETag, librerías de query).
6. **Errores globales**: un handler central para 401 (redirigir a login) evita repetirlo en cada llamada.

En [[Node JS]] el consumo es idéntico (`fetch` es nativo desde Node 18), y ahí **no existe CORS**.

## Conceptos de diseño de APIs (para leer APIs ajenas bien)

| Tema | Práctica estándar |
|---|---|
| **Versionado** | Prefijo en la URL: `/v1/usuarios`. Cuando se rompe un contrato se publica `/v2` y `/v1` sigue funcionando un tiempo |
| **Paginación** | `?page=2&limit=20` (offset) o *cursor-based* `?cursor=xyz` (estable con datos que cambian; lo usan GitHub/Twitter). Las listas grandes **nunca** devuelven todo |
| **Filtros y orden** | Query params: `?rol=admin&orden=-createdAt` |
| **HATEOAS** (avanzado) | Las respuestas incluyen links a las acciones disponibles; REST puro, raro en la práctica |
| **REST vs GraphQL vs RPC** | REST: recursos fijos, sobre/infra-fetching posible. GraphQL: el cliente pide exactamente los campos (`query`), un solo endpoint. RPC: acciones (`/calcular-total`), típico de APIs internas |

## Cómo probar una API (sin escribir JS)

La herramienta de diagnóstico #1 es ver **la petición y la respuesta crudas**:

```bash
# Lectura con cabeceras
curl -i https://api.ejemplo.com/v1/usuarios/42 \
  -H "Accept: application/json" \
  -H "Authorization: Bearer $TOKEN"

# Alta
curl -X POST https://api.ejemplo.com/v1/usuarios \
  -H "Content-Type: application/json" \
  -d '{"nombre": "Ada", "email": "ada@ejemplo.com"}' \
  -i   # -i incluye los códigos de estado y cabeceras en la salida

# Ver solo el preflight CORS
curl -i -X OPTIONS https://api.ejemplo.com/v1/usuarios \
  -H "Origin: https://tu-app.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type, authorization"
```

- `-i` muestra status + cabeceras (indispensable: ahí ves `Access-Control-*`, `Retry-After`...).
- **Postman** (o su alternativa open source **Insomnia/Bruno**) agrega colecciones, variables de entorno (`{{baseUrl}}`), historial y testeo automatizado de respuestas.
- En el navegador, la pestaña **Network** de DevTools muestra cada petición con su status real, headers y payload — muchas veces "la API falla" es en realidad "la API está bien y el cliente la llama mal".
- En la consola de JS del navegador también podés probar: `const r = await fetch('...'); await r.json()` (ver [[La Consola]]).

## Errores comunes

1. **Tratar 4xx/5xx como éxito**: `fetch` no lanza error con respuestas HTTP fallosas — siempre chequeá `res.ok`.
2. **Leer `.json()` en una `204`**: `No Content` no tiene cuerpo; `.json()` lanza `SyntaxError: Unexpected end of JSON input`.
3. **Usar GET para mutar** (borrar/actualizar): los prefetch, crawlers y reintentos del navegador lo "ejecutan" sin querer.
4. **Mezclar verbos**: `POST /actualizarUsuario` en vez de `PUT /usuarios/42` — rompe la interfaz uniforme y la cacheabilidad.
5. **Copiar la API key al frontend sin restricciones**: cualquiera la extrae del bundle. Las keys públicas deben restringirse por dominio y presupuesto.
6. **Pelearte con CORS desde el cliente**: es un permiso del servidor (o un proxy del dev server); el navegador no se "salta" con trucos de JS.
7. **`JSON.stringify` en el body sin `Content-Type: application/json`**: el servidor no parsea el cuerpo y responde 400 o lo ignora.
8. **Reintentos sin criterio**: duplican POSTs y amplifican una API caída (usá backoff y solo reintentá lo seguro).
9. **Pantalla vacía ante un error**: toda petición necesita estados visibles de carga y error — no solo el camino feliz.

## Cheatsheet

| Concepto | Regla |
|---|---|
| API | Contrato de comunicación que oculta la implementación |
| Librería vs framework | Vos llamás a la librería; el framework llama a vos |
| REST | Recursos por URL + verbos HTTP + representaciones + statelessness |
| `GET` | Leer. Seguro e idempotente |
| `POST` | Crear. Ni seguro ni idempotente |
| `PUT` | Reemplazar todo. Idempotente |
| `PATCH` | Cambiar parcial. |
| `DELETE` | Eliminar. Idempotente |
| `2xx` | Éxito · `4xx` | Error del cliente · `5xx` | Error del servidor |
| `201` + `Location` | Recurso creado |
| `204` | Éxito sin cuerpo (no llamar `.json()`) |
| `401` vs `403` | Sin autenticar vs sin permiso |
| `429` | Rate limit; respetar `Retry-After` |
| `fetch` no lanza error con HTTP 4xx/5xx | chequear `res.ok` siempre |
| CORS | Permiso del servidor; el navegador lo exige, no se "esquiva" |
| JWT | Token firmado y autocontenido; no revocable hasta expirar |
| API key | Identifica al proyecto, no al usuario |
| Versionado | `/v1/...`; cambios rotos → `/v2` |
| Paginación | Nunca devolver listas enteras: `page/limit` o cursor |

## Relacionadas

- [[Formato Json]] — el formato de los cuerpos que viajan por las APIs
- [[Acceder a Html]] — selección del DOM para pintar las respuestas
- [[Eventos]] — qué disparar cuando llegan (o fallan) los datos
- [[Java Script]] — fundamentos del lenguaje con el que se consume todo
- [[Recomendaciones]] — manejo de errores y seguridad al escribir el código cliente
- [[Uso de Java Script]] — MPA vs SPA: cómo cambia el consumo según el tipo de app
- [[Node JS]] — consumir APIs desde el servidor (sin CORS)
- [[La Consola]] — probar `fetch` rápido mientras desarrollás
