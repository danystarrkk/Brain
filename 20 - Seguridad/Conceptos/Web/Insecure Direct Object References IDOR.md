---
aliases: ["IDOR", "Insecure Direct Object References", "Referencias Directas Inseguras"]
tags:
  - sec/web
  - sec/web/idor
  - sec/web/access-control
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[GraphQL Introspección y Mutaciones en Ataques IDOR]]"
  - "[[Labs - Control de Acceso]]"
  - "[[Abuso de APIs]]"
  - "[[Ataque de Asignación Masiva]]"
  - "[[Gestión de Autenticación]]"
  - "[[Information disclosure]]"
fuente: "https://portswigger.net/web-security/access-control/idor"
---

# Insecure Direct Object References (IDOR)

Las **Insecure Direct Object References** (**IDOR**) son un tipo de vulnerabilidad de seguridad que se produce cuando una aplicación web utiliza **identificadores internos** (como números o nombres) para identificar y acceder a recursos (como archivos o datos) y no se valida adecuadamente la autorización del usuario para acceder a ellos.

> [!info]
> Las IDOR son una forma de control de acceso roto (Broken Access Control), donde la aplicación no verifica si el usuario autenticado tiene permiso para acceder al recurso solicitado, basándose únicamente en la existencia del identificador. Esto puede llevar a la divulgación de información sensible o a la ejecución de acciones no autorizadas.

## Mecánica: por qué existe IDOR

El servidor necesita un *identificador* para saber qué objeto devolver o modificar: `?pdf_id=1487`, `GET /orders/124`, `{"userId": 3}`. El error no está en usar el identificador — es inevitable — sino en **asumir que conocer el identificador implica tener derecho sobre el objeto**. La autorización correcta es una relación `(usuario, acción, objeto)`:

```
Petición:   GET /download   con cookie de USER_A   y pdf_id=1489
Correcto:   server.check(A, READ, pdf_1489)  ->  ¿A es dueño o admin?
Vulnerable: server.get(pdf_id)               ->  si existe, lo devuelve
```

| Dimensión | Tipos | Ejemplo |
|---|---|---|
| **Horizontal** | Usuario A → objetos de usuario B (mismo rol) | Leer el pedido 124 de otro cliente |
| **Vertical** | Usuario → funciones de rol superior | `?role=admin`, acceso a `/admin` por ID |
| **Metadata** | Cambiar el *tipo* de referencia | `document.pdf` → `../../etc/passwd` (ver [[Path Traversal]]) |

### El mito del UUID

Usar UUIDs hace la enumeración *casi imposible*, pero **no arregla un IDOR**: si el servidor no valida propiedad, basta un UUID filtrado (URL compartida, log, referencia en otra API, [[Information disclosure]]) para explotarlo igual. Los UUIDs son defensa de profundidad, no sustituto de la comprobación de autorización.

### Blind IDOR

Cuando el servidor responde igual (mismo status, misma longitud) exista o no el objeto —o directamente no responde (204, imagen vacía)— hablamos de **IDOR ciego**: no puedes enumerar con respuestas, pero sí con **oráculos laterales**: diferencias de tiempo (objeto grande vs. inexistente), efectos secundarios (el correo del dueño se envía, el balance se modifica), o disparadores (webhook, notificación push). Ver [[Condiciones de Carrera]] para los casos con efectos asíncronos.

## Metodología de testing de autorización

1. **Mapea referencias**: en cada petición, marca todo valor que parezca un identificador: path params, query, body JSON/XML, headers (`X-Id`, `Referer`), cookies, IDs en respuestas (hasta los `userId` de payloads grandes).
2. **Consigue dos cuentas** (A = víctima atacante, B = objetivo) y repite la operación legítima con cada una: define los objetos "propios" de cada cuenta.
3. **Intercambia**: toma la petición de A y pon el ID de B (Repeater). Éxito → horizontal. Repite con endpoints de gestión (update/delete) → impacto de escritura.
4. **Prueba variantes del valor**: decimal, hex, octal, negativo, `%41`, double-encoding (`%2561`), path traversal (`../`), UUID vs numérico, JSON number vs string, arrays (`[1,2]`).
5. **Prueba cabeceras de contexto**: `X-Original-URL`, `X-Rewrite-URL`, cambio de método (GET→POST/PUT/PATCH) — muchos controles solo protegen un método (ver [[Labs - Control de Acceso]]).
6. **Prueba la vertical**: IDs de objetos admin, funciones de rol (`?admin=1`), rutas `/admin`.
7. **Automatiza**: Intruder/wfuzz sobre rangos (como el lab de `pdf_id`) o sobre una lista de IDs extraídos; filtra por longitud/contenido distinto al "no encontrado".
8. **Cadena**: IDOR de lectura → [[Information disclosure]] → hallas IDs de objetos de alto valor (admin, facturas, tokens) → IDOR de escritura → escalada.

| Payload de prueba | Cuándo |
|---|---|
| `1` → `2`, `3`... | IDs secuenciales (el caso clásico del lab de cafés) |
| `0`, `-1`, `999999`, `2147483647` | Rangos y límites de tipo |
| `%00`, `%0a` | Truncado/injection en el lookup |
| `1000`, `0x3E8`, `03570` | Codificaciones alternativas |
| UUID ajeno filtrado | IDs no enumerables |
| `../` sobre la referencia | Metadatos / [[Path Traversal]] |

> [!info]
> Las IDOR son una forma de control de acceso roto (Broken Access Control), donde la aplicación no verifica si el usuario autenticado tiene permiso para acceder al recurso solicitado, basándose únicamente en la existencia del identificador. Esto puede llevar a la divulgación de información sensible o a la ejecución de acciones no autorizadas.

Por ejemplo, si una aplicación web utiliza un identificador numérico para identificar un registro en una base de datos, un atacante puede intentar adivinar este identificador y acceder a los registros sin la debida autorización. Esto puede permitir a los atacantes acceder a información confidencial, modificar datos, crear cuentas de usuario no autorizadas y realizar otras acciones maliciosas.

> [!tip]
> Los vectores comunes para IDOR incluyen parámetros en la URL (query strings o path parameters), campos ocultos en formularios, parámetros en cuerpos de peticiones JSON/XML, y encabezados HTTP personalizados. La clave es identificar dónde se utilizan identificadores para referenciar objetos.

La vulnerabilidad IDOR se puede producir cuando una aplicación web **no implementa controles de acceso adecuados** para los recursos que maneja. Por ejemplo, una aplicación puede validar el acceso a través de autenticación y autorización para los recursos que se muestran en la interfaz de usuario, pero no aplicar la misma validación para los recursos que se acceden directamente a través de una URL.

Para explotar una vulnerabilidad IDOR, un atacante puede intentar modificar manualmente el identificador de un objeto en la URL o utilizar una herramienta automatizada para probar diferentes valores. Si el atacante encuentra un identificador que le permite acceder a un recurso que no debería estar disponible, entonces la vulnerabilidad IDOR se ha explotado con éxito.

Por ejemplo, supongamos que un usuario 'A' tiene un pedido con el identificador numérico **123** y el usuario 'B' tiene un pedido con el identificador numérico **124**. Si el atacante intenta acceder a través de la URL `https://example.com/orders/124`, la aplicación web podría permitir el acceso a ese pedido sin validar si el usuario tiene permiso para acceder a él. De esta manera, el atacante podría acceder al pedido del usuario 'B' sin la debida autorización.

Para prevenir la vulnerabilidad IDOR, es importante validar adecuadamente la autorización del usuario para acceder a los recursos, tanto en la interfaz de usuario como en las solicitudes directas a través de URL. Además, se recomienda restringir los permisos de acceso a los recursos y mantener actualizado el software y los sistemas operativos.

> [!warning]
> La prevención de IDOR requiere implementar controles de acceso robustos en el lado del servidor para cada solicitud que involucre un identificador de objeto. Esto significa verificar no solo la autenticación del usuario, sino también su autorización específica para acceder a *ese* objeto en particular. El uso de identificadores impredecibles (UUIDs) puede dificultar la enumeración, pero no reemplaza una validación de autorización adecuada.

## Notas Prácticas

Estas son las referencias inseguras. A continuación, exploraremos esta vulnerabilidad en la siguiente aplicación web:

![[Pasted image 20260227154748.png]]

Tenemos 5 opciones diferentes de cafés a seleccionar, pero ¿son solo 5?

Si nos fijamos en la `URL`, le pasamos el mismo valor de `item` cuando seleccionamos uno y hacemos `submit`:

![[Pasted image 20260227154930.png]]

Al parecer solo son 5, pero ¿qué sucede si intentamos con el identificador `6`?

![[Pasted image 20260227154950.png]]

Como vemos, sí podemos acceder a otro café. Mediante el cambio de los identificadores, podemos verificar en ocasiones información extra que no debería ser accesible, como es este el caso.

> [!tip]
> Este es un ejemplo clásico de IDOR secuencial o enumerable. La aplicación utiliza identificadores numéricos predecibles, lo que permite a un atacante simplemente incrementar o decrementar el valor para acceder a otros recursos. La enumeración puede ser manual o automatizada.

En otro laboratorio, tenemos la siguiente situación:

![[Pasted image 20260227155541.png]]

Tenemos una aplicación web mediante la cual podemos crear un PDF. Veamos qué sucede al hacer clic en 'Crear':

![[Pasted image 20260227155625.png]]

Observamos que nos indica que se creó y nos proporciona un identificador, en este caso `1487`.

Intentemos descargarlo usando este identificador:

![[Pasted image 20260227155710.png]]

En efecto, esto funciona con identificadores directos.

En este caso, ¿qué pasa si ponemos el valor de `1` o cualquier otro identificador:

![[Pasted image 20260227155902.png]]

Cuando hacemos esto, al parecer nos arroja un PDF con el mensaje de 'volver a intentarlo':

![[{6C3C60BE-6B2E-4AC6-A79F-6A8EA4C7B780}.png]]

Al ingresar el valor de `3`, al parecer no existe, y se nos indica que los valores válidos están en el rango del `1` al `1500`.

> [!info]
> Los mensajes de error detallados, como el que indica el rango de IDs válidos (1 al 1500), son extremadamente útiles para los atacantes. Proporcionan información valiosa que facilita la enumeración y el fuzzing, revelando el espacio de búsqueda para encontrar objetos válidos.

Con esta información, podemos intentar realizar `fuzzing` para encontrar archivos válidos en ese rango. Primero, debemos analizar cómo son las peticiones utilizando Burp Suite:

![[Pasted image 20260227160241.png]]

Como observamos, la petición es de tipo `POST` y el parámetro de datos es `pdf_id`. Por lo tanto, nuestro comando en `wfuzz` quedaría de la siguiente manera:

> [!info]
> **Explicación del comando `wfuzz`:**
> - `-c`: Muestra la salida con colores para una mejor legibilidad.
> - `--hh=10703,1233`: Oculta las respuestas con un tamaño de contenido (content-length) de 10703 o 1233 bytes. Esto es útil para filtrar respuestas de error o no válidas y centrarse en las que tienen un tamaño diferente, indicando un contenido válido.
> - `-t 200`: Establece el número de hilos concurrentes a 200, acelerando el proceso de fuzzing.
> - `-z range,1-1500`: Define el payload como un rango numérico desde 1 hasta 1500. `wfuzz` sustituirá `FUZZ` con cada número en este rango.
> - `-d 'pdf_id=FUZZ'`: Especifica que el payload (`FUZZ`) debe ser insertado en el cuerpo de la petición POST, en el parámetro `pdf_id`.
> - `-u http://172.17.0.1:5000/download`: Define la URL objetivo para las peticiones.

```bash
wfuzz -c --hh=10703,1233 -t 200 -z range,1-1500 -d 'pdf_id=FUZZ' -u http://172.17.0.1:5000/download
```

![[Pasted image 20260227160635.png]]

Como podemos observar, hemos encontrado un identificador único válido:

![[Pasted image 20260227160704.png]]

Como vemos, las vulnerabilidades IDOR son relativamente simples de explotar, donde se manipulan identificadores con el objetivo de ver u obtener información extra que no está disponible por defecto, pero tampoco ha sido eliminada o protegida adecuadamente.

> [!warning]
> El impacto de una vulnerabilidad IDOR puede variar desde la divulgación de información confidencial (datos de usuarios, documentos privados) hasta la modificación o eliminación de datos, la toma de control de cuentas de usuario o la ejecución de funciones administrativas. Es crucial tratarlas con la misma seriedad que otras vulnerabilidades de seguridad críticas.

## Patrones de cadena (IDOR como eslabón)

| Cadena | Pasos |
|---|---|
| IDOR de lectura → toma de cuenta | Filtrar `password_reset_token`/API key de otro usuario → usarlo (ver [[Gestión de Autenticación]]) |
| Mass assignment → IDOR | Asignar `userId` en el update → modificar objeto ajeno ([[Ataque de Asignación Masiva]]) |
| IDOR → GraphQL | Mismo campo `user(id:)` en query y mutation ([[GraphQL Introspección y Mutaciones en Ataques IDOR]]) |
| IDOR ciego → race | Efecto asíncrono con doble redeem de un token ([[Condiciones de Carrera]]) |
| IDOR en API | Headers `X-User-Id`, filtros de tenant mal validados ([[Abuso de APIs]]) |

## Defensa

### 1. Autorización explícita en cada objeto (el control correcto)

```python
# ❌ El ID manda
def get_order(order_id):
    return Order.get(order_id)

# ✅ Filtro de propiedad en la query + re-chequeo en el objeto
@app.route("/orders/<order_id>")
def get_order(order_id):
    order = Order.query.filter_by(
        id=order_id,
        tenant_id=session["tenant_id"]    # el scope viaja en la query
    ).first_or_404()
    require(order, action="read", user=session["user"])  # re-chequeo (defensa en profundidad)
    return order
```

```java
// ✅ Spring: control por anotación + criterio de propiedad
@PreAuthorize("#id == authentication.principal.owns(#id)")
@GetMapping("/orders/{id}")
public Order get(@PathVariable long id) { ... }
```

### 2. Indirect object references (referencias del cliente)

Devolver al cliente un **token opaco** (URL firmada de corta vida) en lugar del ID interno:

```python
# ✅ El cliente nunca ve el ID real
token = itsdangerous.URLSafeSerializer(secret).dumps({"pdf": 1489, "uid": user.id, "exp": t+300})
return redirect(f"/download/{token}")
```

### 3. Modelos multi-tenant: el tenant en TODAS las queries

```python
# ❌ SELECT * FROM docs WHERE id = ?
# ✅ SELECT * FROM docs WHERE id = ? AND tenant_id = ?   (con tenant del token, nunca del body)
```

Middlewares/RLS (PostgreSQL Row-Level Security) como última línea de defensa.

### 4. Higienización de identificadores

Validar tipo y rango (`id` entero positivo, UUID con `parse()` estricto) — reduce superficie de fuzzing y encadenamientos con [[Path Traversal]]/SQLi aunque no sustituye la autorización.

### 5. Sin feedback distintivo

Respuestas uniformes para "no existe" y "no autorizado" (404 genérico) para no dar oráculo de enumeración.

## Errores comunes

**Atacante:**
- Probar solo el objeto de lectura y dar por terminado el test: los IDOR de **escritura** (update/delete/admin) son los que escalan a reporte crítico.
- No obtener una segunda cuenta: sin baseline "objeto propio vs ajeno" no hay comparación.
- Rendirse con UUIDs sin buscar filtraciones (misma app, otras APIs, referer, respuestas).
- Ignorar parámetros "no numéricos" como `tenant`, `org`, `user` en bodies JSON.

**Defensor:**
- Validar autorización en la capa de UI ("el botón solo se ve si eres dueño") y olvidar el backend.
- Filtrar por ID sin el scope de tenant/usuario en la misma query.
- Creer que "los IDs son UUIDs, está seguro".
- Devolver 403 vs 404 distintos y regalar el oráculo.

## Cheatsheet

```
# Intercambio básico (Repeater)
#  petición propia (A) -> sustituir id por el de B
GET /download?pdf_id=1489   Cookie: session=<user_A>

# Enumeración (lab pdf_id 1-1500)
wfuzz -c --hh=10703,1233 -t 200 -z range,1-1500 -d 'pdf_id=FUZZ' -u http://target/download

# Variantes del identificador
1  0  -1  999999  0x2  02  %31  %2531  [1]  "1"  1.0  ../  %00

# Variantes de contexto
X-Original-URL: /admin      X-Rewrite-URL: /admin
GET /x -> POST /x (método no protegido)

# Blind IDOR: oráculos
#  tiempo: objeto grande vs id inexistente
#  efecto: notificación/correo del dueño, balance modificado
```

## Referencias

- PortSwigger Academy — IDOR: https://portswigger.net/web-security/access-control/idor
- OWASP — Broken Access Control: https://owasp.org/Top10/A01_2021-Broken_Access_Control/
- HackTricks — IDOR: https://book.hacktricks.wiki/en/pentesting-web/idor-insecure-direct-object-reference.md.html
- Related internals: [[GraphQL Introspección y Mutaciones en Ataques IDOR]] · [[Labs - Control de Acceso]] · [[Abuso de APIs]] · [[Ataque de Asignación Masiva]] · [[Gestión de Autenticación]] · [[Information disclosure]] · [[Nmap Guía Completa]]