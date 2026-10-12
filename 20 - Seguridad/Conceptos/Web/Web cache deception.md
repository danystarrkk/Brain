---
tags:
  - ciberseguridad
  - web
  - web-deception
tipo: concepto
estado: terminado
---

# Web Cache Deception

**Web Cache Deception (WCD)** es una vulnerabilidad en la que un **proxy de caché** (CDN, reverse proxy) almacena una **respuesta personalizada** — típicamente la página "Mi cuenta" de una víctima — bajo una **clave** que después puede consultar **cualquier usuario**. La víctima solo tiene que visitar una URL aparentemente inocua (sufijada con una extensión de archivo); la caché guarda su contenido privado y el atacante lo recupera sin autenticarse. La clave del ataque es un desajuste entre **cómo la caché construye su cache key** (la ruta pedida, literal) y **cómo el origen decide qué devolver** (ignora el sufijo extra y sirve la página de sesión).

> [!warning] No confundir con Web Cache Poisoning
> | | **Web Cache Deception** | **Web Cache Poisoning** (ver [[Envenenamiento de Caché Web]]) |
> |---|---|---|
> | Qué se envenena | La **respuesta legítima de una víctima** se cachea bajo una URL que yo consulto | Una **respuesta maliciosa** (con mi payload) se cachea bajo la URL de otros |
> | Requiere | Un usuario autenticado que visite mi URL (o que yo lo fuerce) | Un input no-clave que altere la respuesta |
> | Bug central | Desajuste cache-key ↔ ruta/normalización | Entrada no incluida en la cache key |
> | Resultado | Leo datos de la víctima (CSRF tokens, API keys, sesiones) | Ejecuto JS/acción en contexto de otros |
> | Sin víctimas | No: hay que poblar la caché con una respuesta autenticada | Sí: puedo envenenar yo mismo |

## Mecánica: cómo una caché decide almacenar

Una caché web opera con tres piezas:

1. **Cache key**: normalmente `host + ruta pedida` (a veces + ciertos parámetros/headers). Todo lo demás — sufijos basura, headers, cookies — queda **fuera** de la clave si no está configurado explícitamente.
2. **Cacheability**: qué respuestas puede guardar (`Cache-Control`, `Vary`, status, método). Un cache-control laxo (`max-age` sin `private`/`no-store`) hace cacheable incluso contenido personalizado.
3. **Serving**: la próxima petición con la **misma clave** recibe la respuesta guardada **sin tocar el origen** y **sin mirar cookies**.

El bug WCD nace cuando **la caché y el origen "leen" la URL de forma distinta**:

- La caché usa **toda la ruta literal** como clave: `/my-account/aaa.css` es una clave nueva y única.
- El origen **ignora el sufijo** (routing por prefijo, normalización, manejo de extensiones) y devuelve `/my-account` — **personalizado con la cookie de sesión**.
- Resultado: la respuesta privada de la víctima queda publicada bajo una clave pública y predecible.

## Vectores de desajuste (path mismatch)

| Clase | Ejemplo sobre `https://app.com/my-account` | Mecanismo |
|---|---|---|
| **Extensión de archivo** | `/my-account/aaa.css`, `/my-account.js` | La caché key incluye el sufijo; el origen sirve la ruta base |
| **Delimitador de path** | `/my-account;.css`, `/my-account;a=1.css` | El origen trata `;...` como parámetro de path y lo ignora; la caché lo ve como clave nueva |
| **`?` y query** | `/my-account?.css`, `/my-account?x=.css` | La query no forma parte de la clave configurable, pero altera la normalización |
| **Normalización del origin** | `/resources/../my-account.css`, `/my-account/..;/aaa.css` | El origen normaliza o redirige a una ruta cacheable; hay que usar encoding (`%2f`) para que la caché vea otra clave |
| **Normalización de la caché** | `/my-account%2f..%2fmy-account.css` | Al revés: la caché normaliza antes de componer la clave y el origen no |
| **Reglas exact-match** | `/resources/../my-account` dentro de una regla exacta `/resources/...` | La política de cacheado solo aplica a rutas según la caché; hay que caer dentro de la regla sin que el origen redirija y rompa la cadena |

## Catálogo de payloads (sufijos sobre la ruta privada)

```
GET /my-account.css
GET /my-account/aaa.css
GET /my-account;.css
GET /my-account/.css
GET /my-account%23.css          (# urlencoded)
GET /my-account%3b.css          (; urlencoded)
GET /my-account%2f..%2fmy-account.css
GET /my-account/..;/aaa.css
GET /my-account?.css
GET /my-account/?aaa.css
GET /aaa.css/../my-account
```

> [!tip] Workflow rápido con Intruder
> 1. Captura `GET /my-account` y confirma que **no** tiene cabeceras de caché.
> 2. Intruder sobre la ruta con un payloadlist de sufijos (`;.css`, `/x.css`, `%2f..%2fmy-account.css`, etc. — ver catálogo).
> 3. Filtros: `Cache-Control` presente + `X-Cache: miss` + status 200 + longitud idéntica a la respuesta legítima.
> 4. Confirmación manual: repite la petición y mira `Age`/`X-Cache: hit` **con otra cookie o sin cookie** — si sigue viniendo el contenido privado, la clave no incluye tu sesión.
> 5. Explotación: manda la URL al exploit server → la víctima autenticada la carga (poblando la caché) → vos la recargas desde tu navegador sin su cookie y leés su contenido.
> 6. Si la respuesta contiene tokens `csrf`, formulario de cambio de email o API keys, el impacto es account takeover (último lab).

## Metodología de testing

1. **Identifica rutas privadas** relevantes: `/my-account`, `/admin`, `/dashboard`, perfiles, endpoints JSON con datos de usuario.
2. **Busca evidencia de caché**: headers `Age`, `X-Cache`, `CF-Cache-Status`, `Via`, `X-Saved-By`; repite peticiones y observa tiempos de respuesta.
3. **Descubre qué es cacheable**: prueba rutas estáticas (`/resources/...js`) — si esas sí se cachean, la política existente te da la pista del directorio/sufijo que la caché acepta.
4. **Aplica el catálogo de sufijos** y filtra por diferencias (Intruder).
5. **Valida el desajuste**: contenido personalizado servido bajo clave + posibilidad de servirlo sin tu cookie = vulnerable.
6. **Explota con víctima**: exploit server con `<script>new Image().src='https://APP/my-account/aaa.css'</script>`; esperá al pop; recargá la URL cacheada desde tu navegador.

## Laboratorios y notas originales de clase

Mapa lab → clase de desajuste, seguido de los walkthroughs originales:

| Lab | Clase de desajuste |
|---|---|
| Exploiting path mapping for web cache deception | Extensión de archivo sobre la ruta privada |
| Exploiting path delimiters for web cache deception | Delimitadores `;` y `?` |
| Exploiting origin server normalization for web cache deception | Normalización del origin + encoding |
| Exploiting cache server normalization for web cache deception | Normalización de la caché + parámetros |
| Exploiting exact-match cache rules for web cache deception | Reglas exactas de cacheado → CSRF del admin |

## Laboratorios PortSwigger (notas originales de clase)

### Exploiting path mapping for web cache deception

En este laboratorio vamos a ver como la web cache puede filtrar mediante el mal gestionado de la misma y tomando la ruta como un key de la cache información de otros usuario:

Comenzamos iniciando sesión y capturando la petición de la pagina principal:

![[Pasted image 20260317103423.png]]

estando en esa pagina vamos a capturar la petición:

![[Pasted image 20260317103502.png]]

Podemos ver como en la petición no tenemos en realidad nada sobre cache, vamos a hacer pruebas donde luego de `/my-account` intentamos poner cualquier cosa:

![[Pasted image 20260317103623.png]]

Sin importar lo que pongamos vamos a ver que la  web carga pero si revisamos el código de la respuesta vamos a observar como en realidad no vemos nada aun:

![[Pasted image 20260317103730.png]]

Como podemos observar pues nada, ahora vamos a intentar con extensiones a ver si esto cambia en algo:

![[Pasted image 20260317103821.png]]

Como podemos observar ya se implementa la cache con un tiempo de vida de 30 segundos, ahora aquí viene el problema debido a que esta cache toma la ruta como su key por lo que cualquier persona que la use cargara la información, vamos a aprovecharnos de esto y forzando que la victima mediante el servidor atacante que tenemos genere esa cache y nosotros al recargar veamos la información:

![[Pasted image 20260317104351.png]]

Como vemos esto lo almacenamos y lo enviamos a la victima, luego cargamos la misma ruta y veamos si obtenemos lo que queriamos:

![[Pasted image 20260317104338.png]]

Como vemos lo obtenemos sin problemas y con esto podemos terminar el laboratorio:

![[Pasted image 20260317104427.png]]


### Exploiting path delimiters for web cache deception

La idea es la misma con la diferencia de que la web cache se hace presente y toma a la ruta como una nueva clave cache al hacer uso de caracteres especiales.

![[Pasted image 20260317104831.png]]

como observamos esta es la pagina que vamos a capturar la petición nuevamente:

![[Pasted image 20260317104920.png]]

Como observamos no logramos tenemos nada respecto a web cache en la respuesta así que vamos a usar un diccionario que nos ayude a identificar cuales caracteres permiten desatar el web cache:

![[Pasted image 20260317105312.png]]

bueno vamos a comenzar el ataque:

![[Pasted image 20260317105510.png]]

Podemos ver que dos opciones que son `? y ;` son validas y aunque no detonan el cache web, estas si permiten cargar correctamente, en este punto lo que podemos verificar es cual mediante una extensión talvez genere y cree una nueva key cache:

![[Pasted image 20260317105645.png]]

Como podemos observar la valida es `;` por lo tanto ya con esto identificado vamos a realizar el mismo ataque que en el laboratorio anterior:

![[Pasted image 20260317105903.png]]

Lo almacenamos y lo enviamos a la victima a ver si funciona:

![[Pasted image 20260317105930.png]]

Perfecto con esto terminamos el laboratorio:

### Exploiting origin server normalization for web cache deception

Este laboratorio es prácticamente similar y una posible combinación de los dos anteriores por lo que vamos a ver como se resuelve:

Comenzamos como siempre capturando la petición de la web principal para ver como se esta tramitando:

![[Pasted image 20260317110441.png]]

Con esto vamos a ver la petición:

![[Pasted image 20260317110516.png]]

Las pruebas que vamos a hacer son las mismas y vamos directo en este caso al intruder:

![[Pasted image 20260317110648.png]]

Como logramos observar tenemos en este caso que el `?` es valido, pero no vemos aun que detone la web cache por lo tanto queda a medias vamos a intentar con extensiones a ver:

![[Pasted image 20260317110812.png]]

Observamos como tampoco funciona.

Vamos a intentar algo nuevo y es retroceder un directorio:

![[Pasted image 20260317110919.png]]

como vemos esto nos resuelve a la raíz, podemos volver a entrar al directorio y ver si la cache explota:

![[Pasted image 20260317111006.png]]

Observamos que no funciona.

Otra cosa es intentar ver en el historial de peticiones si encontramos múltiples peticiones que si haga uso de la cache a ver si algo podemos intentar:

![[Pasted image 20260317111231.png]]

observamos otra petición que si lo hace vamos a llevarla al repeater y la prueba es ver de que directorio se trata puede que sea por la extensión el recurso o el directorio `/resource`:
![[Pasted image 20260317111357.png]]

como podemos observar esto esta bajo el directorio `resources` que la cache guarda información, podríamos intentar mediante la recursion que ya intentamos anteriormente cargar `my-account` :

![[Pasted image 20260317114114.png]]

Como podemos observar esto es valido ya tenemos pero vamos a hacer que esto sea un poco mas especifico y a demás lo cargue la victima:
![[Pasted image 20260317114326.png]]

vamos a enviarlo y a cargarlo ahora desde nuestro navegador a ver si funciona:

![[Pasted image 20260317114451.png]]

vemos que no lo aplica. Puede que la web de alguna forma realiza una normalización y redirige directo a ruta lo que genera una nueva cache key y rompe con la cadena pero podemos intentar en la url-encode de `/` en ella:

![[Pasted image 20260317114647.png]]

volvamos a intentarlo de esta manera:
![[Pasted image 20260317114740.png]]

como observamos esto ya es valido, y podemos terminar el laboratorio:

![[Pasted image 20260317114814.png]]

### Exploiting cache server normalization for web cache deception

En este laboratorio tenemos que obtener la key de `carlos` y completar el laboratorio.

Comenzamos como de costumbre con la petición de nuestro home:

![[Pasted image 20260317115123.png]]

Todo lo que tenemos que probar es lo de los demás laboratorios pero vamos de igual forma directo en este caso al intruder:

![[Pasted image 20260317115311.png]]

Podemos observar 2 valido pero tenemos que tener mucho cuidado porque aunque son valido en esta ocasión están url-encode por lo tanto tomarlo en cuenta.

A partir de aquí podemos intentar con extensiones al final pero no encontramos nada en especifico, si intentamos normalización pero tampoco aplica.

Vamos nuevamente en el historial a buscar por alguna petición que si aplique la cache a ver lo gramos aprovecharnos de esto:

![[Pasted image 20260317115658.png]]

encontramos una ruta que si lo permite, vamos a intentar de igual forma ver si es resourceso el recurso quien lo aplica:

![[Pasted image 20260317115915.png]]

vemos que nos dice que nos es valida pero sigue aplicándose la web cache por lo tanto vamos a intentar solo como prueba la normalización:

![[Pasted image 20260317120020.png]]

No solo no es valido sino que la cache ya no funciona.

Ahora una cosa que podemos intentar ver directamente es si la web cache esta asociada no solo al recurso sino al echo de que `resources` se use como directorio, parámetro y mas:

![[Pasted image 20260317120502.png]]

Podemos observar como mediante el uso de url-encode y ademas los parámetros logramos ya cargar la web cache, en este caso vamos a darle algo mas especifico y veamos que sucede si lo cargamos a la victima:

![[Pasted image 20260317120654.png]]

vemos si logramos cargar o no la cache:

![[Pasted image 20260317121152.png]]

lab terminado.

### Exploiting exact-match cache rules for web cache deception

Bueno ahora es tratar de cambiar el correo de administrador.

Vamos a ver la web primero:

![[Pasted image 20260317121416.png]]

Como observamos tenemos el formulario para cambiar el correo por lo tanto vamos a realizar un prueba para interceptar el como se realiza la petición:

![[Pasted image 20260317121530.png]]

al parecer pues si podemos hacer el cambio mediante el valor del email y un valor de `csrf`.

En este punto lo primero que podemos intentar ver es en que punto entra en juego la cache mediante todas las pruebas que ya hemos hecho en los otros laboratorios como son parámetros, normalización, combinación de parámetros, vamos directo con los delimitadores en el intruder:

![[Pasted image 20260317122537.png]]

como observamos tenemos dos delimitadores validos pero la cache no se hace presente.

Bueno vamos a buscar por peticiones que hagan uso de la cache, pero no vamos a encontrar nada por lo que mediante el Burp Suite vamos a realizar un reconocimiento:

![[Pasted image 20260317123219.png]]

como vemos desde la pestaña de target y haciendo uso de eso vamos a comenzar un reconocimiento:

![[Pasted image 20260317124939.png]]

como vemos encontramos `robots.txt` y pues bueno si admite cache, por lo tanto vamos a comenzar a hacer pruebas para representarla dentro de `my-account`:

![[Pasted image 20260317125119.png]]

como vemos ahora ya logramos crear una clave cache pero nosotros tenemos que cambiar el correo y el como lo vamos a hacer es sencillo ya que vamos a obtener el unico dato que nos falta siendo el `csrf` que esta almacenada en la web de forma oculta en el mismo home entonces lo que vamos a hacer es hacer que la vitima cargue `my-account`, cargar la cache de la victima obtenerlo nosotros y de eso obtener ya el `csrf` entonces la petición a la victima quedara de la siguiente manera:

![[Pasted image 20260317125447.png]]

Veamos si logramos cargarlo:

![[Pasted image 20260317125521.png]]

vemos que si se cambio porque su tiempo es diferentes y si hacemos render vamos a ver el panel de administrador, ahora saquemos el csrf:

![[Pasted image 20260317125609.png]]

con este valor vamos a cambiar su correo de la siguiente manera:

![[Pasted image 20260317125653.png]]

lo que hacemos ahora es generar un CSRF con ayuda del BurpSuite:

![[Pasted image 20260317125940.png]]

copiamos y pegamos y nos dejamos listo en el exploit server:

![[Pasted image 20260317130012.png]]

enviamos y esto debería terminar con el laboratorio:

![[Pasted image 20260317130035.png]]

## Detección (defensor)

| Señal | Riesgo |
|---|---|
| `Cache-Control` cacheable en rutas que usan cookie de sesión | Personalizado cacheable |
| Respuesta 200 idéntica sin cookie tras pedir una ruta con sufijo extra | Desajuste ruta/clave |
| `X-Cache: hit` + `Age > 0` en páginas "Mi cuenta" | Fuga directa |
| Logs con rutas `/my-account/;x.css`, `/admin/.js` | Exploración activa de WCD |
| Redirect301/308 internos seguidos de cacheado | Trampa de normalización |
| `Set-Cacheable`/`Cache-Control` heredado globalmente | Todo el sitio cacheable por defecto |

## Impacto y riesgo

- Lectura de páginas privadas de cualquier usuario que pase por la URL: account takeover si la respuesta contiene tokens `csrf` (el último lab usa el `csrf` cacheado para cambiar el correo del admin) o datos de sesión.
- Fuga de API keys, perfiles, PII en HTML cacheado.
- Escalada: si la ruta cacheada es un JSON de API, robo masivo de datos sin necesidad de víctima por recurso.

## Defensa (con configuración y código)

### 1. Cacheado por allowlist: solo rutas explícitamente públicas

```nginx
# ❌ Todo cacheable — cualquier respuesta 200 (incluida la personalizada) se cachea
# proxy_cache_valid 200 1m;

# ✅ Nginx: cachea solo estáticos; todo lo demás, sin caché
location ~* ^/(resources|static|assets)/ {
    proxy_cache_valid 200 10m;
    proxy_ignore_headers Set-Cookie;
}
location / {
    proxy_cache off;
    add_header Cache-Control "no-store, private";
}
```

### 2. Clave de caché estricta y normalizada

```nginx
# ❌ proxy_cache_key "$request_uri";          # literal del cliente, con basura incluida
# ✅ clave = host + ruta NORMALIZADA ($uri ya viene normalizada por Nginx)
proxy_cache_key "$scheme$request_method$host$uri";
# No cachear respuestas con sesión:
proxy_no_cache "$cookie_session $upstream_http_set_cookie";
```

```vcl
# Varnish: allowlist en vcl_recv + no cachear con Set-Cookie
sub vcl_recv {
    if (req.url !~ "^/static/") { return (pass); }
}
sub vcl_backend_response {
    if (beresp.http.Set-Cookie) { set beresp.uncacheable = true; }
}
```

### 3. El origen no debe ignorar basura en la ruta

- Routing con rutas **exactas**: `/my-account/xxx` debe devolver 404, no la página base (evita el "path mapping" del primer lab).
- Rechazar (no redirigir) paths con `;`, `%2f`, `..` — o normalizarlos **antes** de cualquier política de caché.
- Marcar toda respuesta personalizada `Cache-Control: no-store, private` en la app:

```python
# Flask
@app.after_request
def no_store(resp):
    if session.get("user_id"):
        resp.headers["Cache-Control"] = "no-store, private"
    return resp
```

### 4. Detección en WAF/CDN

No cachear y alertar sobre requests que combinen rutas privadas conocidas (`my-account`, `admin`, `account`, `dashboard`) con sufijos de extensión estática (`;.css`, `.js`, `.png`) o delimitadores.

## Errores comunes

**Atacante:**
1. Probar solo extensiones y no delimitadores (`;`, `%23`) — varios labs exigen `;` o encoding.
2. Dar por hecho que `X-Cache: hit` implica éxito: verifica **con otra cookie** que el contenido servido es el de la víctima, no el tuyo.
3. Enviar el exploit sin que la víctima esté autenticada: la caché se puebla con una respuesta anónima y no hay fuga.
4. Ignorar la normalización: si al recargar te redirigen a la ruta limpia, la clave cambia y perdés el hit.

**Defensor:**
1. Cacheado global ("cachea todo lo 200") incluyendo páginas de sesión.
2. Usar `$request_uri` (literal del cliente) como cache key en vez de `$uri` normalizado.
3. Devolver 200 con contenido privado en rutas que tragan sufijos.
4. No monitorear rutas privadas con sufijos sospechosos en logs.

## Cheatsheet

```http
# ¿Hay caché? repite y mira las cabeceras
GET /my-account HTTP/1.1               -> sin Age/X-Cache = no cacheada
GET /resources/main.js HTTP/1.1        -> Age > 0 = existe política de caché

# Sondeo de desajuste (Intruder sobre la ruta)
GET /my-account;.css HTTP/1.1          # delimitador
GET /my-account/aaa.css HTTP/1.1       # extensión
GET /my-account%2f..%2fmy-account.css  # normalización
# Filtro: Cache-Control presente && X-Cache: miss && len == respuesta legítima

# Confirmación (sin cookie)
GET /my-account/aaa.css HTTP/1.1       -> X-Cache: hit = fuga

# Explotación (exploit server)
<script>new Image().src="https://APP/my-account/aaa.css";</script>
```

## Laboratorios y referencias

- PortSwigger Academy — Web Cache Deception: https://portswigger.net/web-security/web-cache-deception
- PortSwigger Academy — Web Cache Poisoning: https://portswigger.net/web-security/web-cache-poisoning
- Artículo original (Kafeine, 2017): https://blog.sucuri.net/2017/10/web-cache-deception-attack.html
- Nginx — A Guide to Caching: https://nginx.org/en/docs/http/ngx_http_proxy_module.html

## Relacionadas

- [[Envenenamiento de Caché Web]] — el primo peligroso: envenenar respuestas en vez de robarlas
- [[Cross-Site Request Forgery (CSRF)]] — el último lab encadena WCD → robo de token CSRF → cambio de email del admin
- [[Insecure Direct Object References IDOR]] — el patrón de "acceder al objeto de otro" aplicado a rutas cacheadas
- [[Information disclosure]] — el WCD es, en esencia, una fuga de información por caché
- [[HTTP Host header attacks]] — otra forma de confundir a la capa de caché con la identidad del host
