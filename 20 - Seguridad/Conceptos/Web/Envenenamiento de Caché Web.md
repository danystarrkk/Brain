---
tags:
  - ciberseguridad
  - web
  - envenenamiento-web
tipo: concepto
estado: terminado
---

# Envenenamiento de Caché Web

El **envenenamiento de caché web** (*Web Cache Poisoning*) es un ataque en el que el atacante hace que una **caché intermedia almacene una respuesta maliciosa** bajo una clave que la víctima también usará; a partir de ahí, la caché **sirve el veneno a todos los usuarios** sin que ellos envíen nada sospechoso. La potencia del ataque no está en la vulnerabilidad inyectada (normalmente un XSS reflejado o un Host header attack), sino en la **distribución**: una sola petición del atacante envenena a miles.

> [!info] Clave de caché (cache key)
> La clave de caché es el identificador con el que la caché almacena y recupera una respuesta. Generalmente se deriva de la URL (host + path + query), pero **puede incluir o ignorar** cabeceras, cookies u otros componentes según la configuración. La vulnerabilidad central es: si un componente **no forma parte de la clave** (*unkeyed input*) pero **influye en la respuesta**, el atacante puede modificar la respuesta sin cambiar la clave — la caché guarda su respuesta venenosa y la sirve a víctimas cuyas peticiones, sin ese componente, generan la misma clave.

> [!tip] La intuición completa del ataque
> ```
> 1. Descubrir un input que cambia la respuesta pero NO está en la clave de caché
> 2. Inyectar en ese input un payload que quede reflejado en la respuesta (XSS, redirect, script src...)
> 3. El servidor de origen responde con la página "normal + veneno"
> 4. La caché ALMACENA esa respuesta bajo la clave de la víctima
> 5. Toda víctima con la misma clave recibe la respuesta venenada
> ```

## Mecánica: capas de caché, frescura y cabeceras

| Capa | Quién la sirve | Comportamiento relevante |
|---|---|---|
| **Caché del navegador** | El propio navegador | `Cache-Control`, `Vary`; distinta por usuario — difícil de explotar de forma masiva |
| **CDN / caché compartida** | Cloudflare, Fastly, Akamai… | Clave definida por el CDN; **puede ignorar** headers/cookies → terreno clásico del poisoning |
| **Reverse proxy (Nginx/Varnish)** | Infraestructura propia | Configurada con `proxy_cache_key`; unkeyed inputs posibles |
| **Caché interna de la app** | La aplicación (Redis, framework) | Dos niveles (externa + interna) pueden envenenarse en cadena (lab abajo) |

Cabeceras clave para leer una respuesta cacheable:

| Cabecera | Significado | Uso ofensivo/defensivo |
|---|---|---|
| `Cache-Control: max-age=N` | Frescura de la respuesta en segundos | Si hay `max-age` > 0, la respuesta se cachea |
| `Age: N` | Segundos que la respuesta lleva en la caché | `Age: 0` tras tu petición = se refrescó tu clave; `Age` alto = respuesta vieja |
| `Vary: header1, header2` | La clave incluye esos headers | `Vary: User-Agent` = solo los UAs iguales reciben esa respuesta (lab dirigido) |
| `X-Cache: hit/miss` | (CDN) si vino de caché | Confirmar hits/misses |
| `Surrogate-Key` / `Cache-Key` | (CDN) etiquetas y clave interna | Exponen la composición de la clave |

**Metodología de laboratorio (probe → clave → veneno):**

1. **Probe de clave**: añadir un parámetro/valor único a la petición (p. ej. `?cb=12345`) y observar si `Age` vuelve a `0`. Si vuelve, ese componente **está en la clave** (refresca); si no, es candidato a unkeyed.
2. **Descubrir unkeyed inputs**: Param Miner (guess headers/params/cookies) — la extensión automatiza la búsqueda de entradas que alteran la respuesta sin alterar la clave.
3. **Evaluar el reflejo**: con la entrada unkeyed descubierta, ver en Repeater cómo cambia la respuesta (¿refleja en HTML? ¿cambia un `src`? ¿altera un redirect?).
4. **Explotar**: inyectar payload (XSS, script apuntando al exploit server) y enviar la petición para que la caché la almacene.
5. **Confirmar**: petición limpia desde el navegador (sin tus headers) → recibes la respuesta venenada.

## Catálogo de técnicas (clases de Web Cache Poisoning)

| Técnica | Mecanismo | Input unkeyed típico |
|---|---|---|
| **Unkeyed header** | Header ignorado en la clave pero reflejado en la respuesta | `X-Forwarded-Host`, `X-Forwarded-Scheme`, `X-Host` |
| **Unkeyed cookie** | Cookie no entra en la clave pero se usa en el render | Cookie de preferencia/feature (`fehost`) |
| **Unkeyed query string/param** | Parámetro ignorado por la clave pero procesado por el origen | `utm_*`, tracking params |
| **Multiple headers** | Encadenar headers (scheme + host) para controlar un redirect y su destino | `X-Forwarded-Scheme: http` + `X-Forwarded-Host: evil` |
| **Parameter cloaking** | Parámetro oculto tras `;` que el origin ignora pero la clave sí contempla (o viceversa) | `?utm_content=x;callback=malicioso` |
| **Fat GET** | Cuerpo en GET: la clave ignora el body pero el origin lo procesa | `GET / con body callback=evil` |
| **URL normalization** | Diferente normalización entre navegador/origin/caché: el origin ve una forma, la clave otra | `%3C` vs `<`, doble encoding |
| **Cache key injection** | Controlar/componer la clave con un input (host, path extra) para que una respuesta venenada se guarde bajo la clave de la víctima | `X-Original-URL`, delimitadores `$$` |
| **Targeted / Vary** | La respuesta se cachea con `Vary: User-Agent` — solo el UA elegido la recibe | Robar el UA de la víctima (img en post) y envenenar con ese UA |
| **Caché interna** | El origin tiene su propia caché interna tras la externa; envenenar la interna hace persistir el veneno aunque se purgue la externa | Repetir la petición hasta colar la respuesta interna |

> [!warning] Parameter cloaking y el punto y coma
> En algunas configuraciones, `;` separa parámetros para el origin (o lo ignora tras `;`) mientras la caché la trata como parte del valor/clave. Ejemplo de lab: `?utm_content=x;callback=alert(1)` — el origin (o su JS embebido) procesa el `callback` malicioso, la clave de caché lo considera parte del recurso, y la respuesta venenada se cachea. Ver nota en el lab "Ocultamiento de Parámetros".

## PortSwigger

### Lab: Web Cache Poisoning con una Cabecera No Clave (Unkeyed Header)

Envenenar mediante una cabecera que la caché no usa para la clave, inyectando un `alert(0)` que se ejecute en otros usuarios al visitar la raíz. Capturamos la petición principal:

![[Pasted image 20250926094028.png]]

Vemos `Cache-Control` y `Age` en la respuesta:

> [!info] Cabeceras `Cache-Control` y `Age`
> *   **`Cache-Control`**: directivas de caché; `max-age=3600` = recurso fresco 3600 s.
> *   **`Age`**: segundos que el objeto lleva en la caché; `0` = respuesta fresca o refrescada.

Probamos con parámetros para ver si `Age` regresa a 0:

![[Pasted image 20250926094327.png]]

`Age` cambió a 0 — el parámetro forma parte de la clave. El objetivo es encontrar una cabecera que permita inyectar sin entrar en la clave. Usamos **Param Miner**:

![[Pasted image 20250926094658.png]]

En Repeater, Param Miner → *guess headers*:

![[Pasted image 20250926094824.png]]

Panel siguiente:

![[Pasted image 20250926094845.png]]

Damos OK y vamos a `All Issues`:

![[Pasted image 20250926094940.png]]

![[Pasted image 20250926095023.png]]

Reporta envenenamiento de caché usando `x-forwarded-host`. Probamos:

![[Pasted image 20250926095158.png]]

Funciona: el dominio que pasamos se refleja en la respuesta. Cualquier usuario que consulte la raíz solicitará un script de la ruta que controlemos. Aprovechamos el exploit server:

![[Pasted image 20250926095427.png]]

Modificamos para que coincida con el resto de la petición y enviamos el `alert(0)`:

![[Pasted image 20250926095533.png]]

Funcionó. Recargamos desde el navegador y vemos la inyección en acción:

![[Pasted image 20250926095724.png]]

![[Pasted image 20250926095811.png]]
Lab Terminado.

### Lab: Web Cache Poisoning con una Cookie No Clave (Unkeyed Cookie)

Envenenar la caché mediante la cookie `fehost`. Capturamos la raíz:

![[Pasted image 20250926100042.png]]

La cookie parece insertarse dentro de una tupla en el JS embebido. Probamos insertar algo:

![[Pasted image 20250926100208.png]]

Cerramos la tupla (`";`) e insertamos el alert, comentando lo que queda colgado:

![[Pasted image 20250926100604.png]]

![[Pasted image 20250926100653.png]]
Lab Terminado.

### Lab: Web Cache Poisoning con Múltiples Cabeceras

Ejecutar un `alert` encadenando varias cabeceras. Capturamos la raíz:

![[Pasted image 20250926101231.png]]

Vamos a volver a emplear Param Miner con el objetivo de encontrar cabeceras vulnerables:

![[Pasted image 20250926101335.png]]

Reporta `x-forwarded-scheme` (define el protocolo de conexión). Probamos con `HTTP`:

> [!info] Cabecera `X-Forwarded-Scheme`
> Cabecera de facto que identifica el protocolo (HTTP/HTTPS) que usó el cliente para llegar al proxy, cuando el proxy termina TLS y reenvía HTTP al origin.

![[Pasted image 20250926101815.png]]

Realiza un redirect a la raíz cambiando a `https`. Buscamos alterar ese redirect: Param Miner reporta `x-forwarded-host`, que puede controlar el dominio del redirect:

![[Pasted image 20250926102002.png]]

![[Pasted image 20250926102116.png]]

Controla el host. Buscamos consultas alternas que se hagan al cargar la raíz:

![[Pasted image 20250926102346.png]]

Hay una petición a `tracking.js`; si el host controlado aplica a cualquier parte de la web, podemos redirigir esa carga a nuestro recurso malicioso y cachearlo:

![[Pasted image 20250926102502.png]]

![[Pasted image 20250926102746.png]]

Configuramos el exploit server y enviamos la petición:

![[Pasted image 20250926102818.png]]

![[Pasted image 20250926103322.png]]

![[Pasted image 20250926103549.png]]
Lab Terminado.

### Lab: Web Cache Poisoning Dirigido Usando una Cabecera Desconocida

Envenenar con una cabecera desconocida. Capturamos la raíz:

![[Pasted image 20250926105152.png]]

Param Miner detecta `X-host`:

![[Pasted image 20250926105817.png]]

Probamos qué edita en la respuesta:

![[Pasted image 20250926110001.png]]

Cambia el dominio de una solicitud:

![[Pasted image 20250926110427.png]]
![[Pasted image 20250926110444.png]]

Se queda en la caché:

![[Pasted image 20250926110558.png]]

Pero el lab no se completa: hay una cabecera `vary`:

![[Pasted image 20250926110821.png]]

`Vary: User-Agent` hace que la respuesta envenenada solo se sirva a los usuarios con nuestro User-Agent:

> [!info] Cabecera `Vary`
> Le indica a los proxies y navegadores que la respuesta varía según cabeceras de la solicitud. `Vary: User-Agent` = la caché guarda versiones distintas por UA; el veneno solo llega a quien tiene ese UA.

![[Pasted image 20250926110928.png]]

La solución: obtener el `User-Agent` real de la víctima por otros medios. La descripción dice que los usuarios visitan los posts — probamos si podemos dejar un `<img>` apuntando a nuestro servidor:

![[Pasted image 20250926111306.png]]

![[Pasted image 20250926111412.png]]

![[Pasted image 20250926111440.png]]

Vemos el UA específico. Envenenamos de nuevo usando ese UA:

![[Pasted image 20250926111555.png]]

Esperamos y ahora sí:

![[Pasted image 20250926111607.png]]
Lab Terminado.

### Lab: Web Cache Poisoning a Través de una Cadena de Consulta No Clave (Unkeyed Query String)

La cadena de consulta se excluye de la clave de caché. Capturamos la raíz:

![[Pasted image 20250926112233.png]]

Probamos con un parámetro de prueba: si `Age` no vuelve a 0, no está en la clave:

![[Pasted image 20250926112409.png]]

![[Pasted image 20250926112618.png]]

La caché no cambia con el parámetro — no crea clave nueva. Inyectamos una XSS saliendo del link e ingresando `img`:

![[Pasted image 20250926113113.png]]

Perfecto, se inserta la etiqueta `img` y genera el `alert(0)`. Veamos si se completa el laboratorio:

![[Pasted image 20250926113153.png]]

![[Pasted image 20250926113217.png]]
Lab Terminado.

### Lab: Web Cache Poisoning a Través de un Parámetro de Consulta No Clave (Unkeyed Query Parameter)

A diferencia del lab anterior, los parámetros **sí** entran en la clave por defecto — hay que descubrir un parámetro que la ignora. Capturamos la raíz:

![[Pasted image 20250926151142.png]]

Vemos que `Age` pasa a 0 con un parámetro cualquiera (está en la clave):

![[Pasted image 20250926151258.png]]

Param Miner descubre el parámetro `utm_content`:

![[Pasted image 20250926151502.png]]

![[Pasted image 20250926151724.png]]

Probamos — se inserta en la respuesta sin refrescar clave:

![[Pasted image 20250926151940.png]]

Inyectamos la XSS:

![[Pasted image 20250926152217.png]]

![[Pasted image 20250926152251.png]]
Lab Terminado.

### Lab: Ocultamiento de Parámetros (Parameter Cloaking)

> [!info] Parameter Cloaking
> Ocurre cuando un servidor ignora ciertos parámetros (por un parser distinto o una lista blanca), pero la caché **sí** los considera — o viceversa. El atacante inyecta un parámetro que una capa ignora y otra procesa/cachea, generando una respuesta venenada con clave única.

Capturamos la petición principal:

![[Pasted image 20250926152741.png]]

Hay una cookie `country` interesante. Param Miner encuentra `utm_content`:

![[Pasted image 20250926153154.png]]

![[Pasted image 20250926153248.png]]

La app llama a un recurso externo `geolocate` cuyo parámetro se refleja en la respuesta — y es código JavaScript:

![[Pasted image 20250926153502.png]]

![[Pasted image 20250926153547.png]]

Por defecto llama a `setCountryCookie`; necesitamos inyectar sin cambiar eso. Probamos con `utm_content`:

![[Pasted image 20250926153937.png]]

No cambia el valor. El truco: en algunos parsers, lo que va tras un `;` se interpreta como nuevo parámetro:

> [!tip] Uso del punto y coma (`;`) en URLs
> Algunos parsers antiguos tratan `;` como separador, como `&`. El origin puede ignorar el parámetro tras `;` mientras la caché lo incluye en la clave (o al revés), creando la discrepancia explotable.

![[Pasted image 20250926154141.png]]

Se cambia el `callback`. Inyectamos:

![[Pasted image 20250926154427.png]]

![[Pasted image 20250926154443.png]]

![[Pasted image 20250926154459.png]]
Lab Terminado.

### Lab: Web Cache Poisoning a Través de una Solicitud GET "Fat"

> [!info] Solicitudes GET "Fat"
> Un `GET` con cuerpo: no estándar en HTTP/1.1, pero algunos servidores lo procesan. Si la caché ignora el body para la clave, pero el origin lo lee para la respuesta, el body se convierte en un unkeyed input de cabo a rabo.

Capturamos la petición:

![[Pasted image 20250926155020.png]]

Vemos el `geolocate` de nuevo:

![[Pasted image 20250926155135.png]]

Como la clave ignora el body y el GET puede llevar body, inyectamos ahí:

![[Pasted image 20250926155254.png]]

Se queda en caché. Realizamos la inyección:

![[Pasted image 20250926155419.png]]

![[Pasted image 20250926155428.png]]

![[Pasted image 20250926155451.png]]
Lab Terminado.

### Lab: Normalización de URL

> [!info] Normalización de URL
> La caché y el origin pueden aplicar reglas distintas de normalización (decodificación, orden de parámetros, minúsculas, barras finales). La discrepancia permite que el atacante "pase" una inyección que el navegador enmascara con URL-encoding.

Analizamos la petición principal:

![[Pasted image 20250926155924.png]]

Provocamos un error que refleja el path:

![[Pasted image 20250926160223.png]]

Cerramos el párrafo e insertamos `script`:

![[Pasted image 20250926160325.png]]

Se inserta. Pero si enviamos desde el navegador, el payload se URL-encodea y la caché guarda la versión codificada:

![[Pasted image 20250926160406.png]]

![[Pasted image 20250926160551.png]]

La solución: enviar el payload **sin URL-encodear** desde Burp (que no normaliza igual que el navegador) para envenenar la caché, y luego dar a la víctima un link que, tras la normalización del navegador, caiga en la misma clave:

![[Pasted image 20250926160649.png]]

![[Pasted image 20250926160732.png]]

![[Pasted image 20250926160815.png]]
Lab Terminado.

### Lab: Web Cache Poisoning para Explotar una Vulnerabilidad DOM a Través de una Caché con Criterios de Cacheabilidad Estrictos

La caché tiene criterios estrictos de cacheabilidad; hay una vulnerabilidad DOM. Capturamos la raíz:

![[Pasted image 20250926161753.png]]

Hay `script` que almacenan host y path; buscan su uso:

![[Pasted image 20250926162102.png]]

Concatena el host con una ruta. Probamos con `X-Forwarded-Host`:

![[Pasted image 20250926162455.png]]

Cambia el host — combinado con la carga posterior podemos servir desde nuestro exploit server. Primero vemos qué hace ese script:

![[Pasted image 20250926162623.png]]

Devuelve un JSON. Ahora, si revisamos nuestra web, esto aparece también — y ahí es donde se da el vector del ataque:

![[Pasted image 20250926162707.png]]

Configuramos el exploit server:

![[Pasted image 20250926162820.png]]

Cambiamos el host de la petición al del exploit server:

![[Pasted image 20250926163003.png]]

Puede que no interprete por el formato — devolvemos JSON:

![[Pasted image 20250926163328.png]]

![[Pasted image 20250926163512.png]]

La consola reclama `Access-Control-Allow-Origin`:

> [!info] Cabecera `Access-Control-Allow-Origin` (CORS)
> Permite indicar desde qué orígenes puede cargarse el recurso. Sin ella, el navegador bloquea la carga cross-origin del script (ver [[Vulnerabilidades CORS y Laboratorios]]).

Agregamos la cabecera en el exploit server:

![[Pasted image 20250926163754.png]]

![[Pasted image 20250926163905.png]]

Configuramos la inyección:

![[Pasted image 20250926164003.png]]

![[Pasted image 20250926164328.png]]

Vemos si con esto completamos el lab:

![[Pasted image 20250926164354.png]]

Vemos que se ejecuta:

![[Pasted image 20250926164408.png]]
Lab Terminado.

### Lab: Combinando Vulnerabilidades de Web Cache Poisoning

Varios tipos de ataque encadenados; objetivo: la alerta. Capturamos el tráfico:

![[Pasted image 20250926192921.png]]

Hay una petición de traducción:

![[Pasted image 20250926193008.png]]

Devuelve todos los idiomas. Seleccionamos un idioma en la web:

![[Pasted image 20250926193105.png]]

Vemos que se tramitó:

![[Pasted image 20250926193157.png]]

Lo primero que se tramita es `setlang/es` — define el idioma y redirige. La cadena de ataque: cacheear una inyección dentro de `translations.js` y luego redirigir al idioma español para que se inserte.

Primero, la traducción:

![[Pasted image 20250926230211.png]]

Param Miner descubre `x-original-url`:

![[Pasted image 20250926230500.png]]

![[Pasted image 20250926230732.png]]

Genera un *not found* (altera la ruta pedida) — no sirve directamente para alterar la respuesta. Volvemos a analizar la petición a la raíz:

![[Pasted image 20250926231116.png]]

La petición es igual a la del lab anterior. Probamos `X-Forwarded-Host`:

![[Pasted image 20250926231404.png]]

Cambia el host; como la app llama a `translation.json`, podemos servir ese archivo desde el exploit. Configuramos y cacheamos:

![[Pasted image 20250926231833.png]]

![[Pasted image 20250926231939.png]]

Con lo cacheado, recargamos la web en español:

![[Pasted image 20250926232846.png]]

Revisamos el exploit server:

![[Pasted image 20250926232952.png]]

![[Pasted image 20250926233158.png]]

La inyección no quedó bien aplicada — la reajustamos:

![[Pasted image 20250926233326.png]]

![[Pasted image 20250926233408.png]]

Ahora falta redirigir la web al español. `x-original-url` redirige:

![[Pasted image 20250926235005.png]]

Apuntamos a español y cacheamos ambas peticiones en bucle para que todo se ejecute cuando la víctima entre al home:

![[Pasted image 20250927000041.png]]
Lab Terminado.

### Lab: Inyección de Clave de Caché (Cache Key Injection)

Cadena de vulnerabilidades para ejecutar `alert(1)` controlando/componiendo la clave de caché. Analizamos las peticiones: la primera a `/` es un redirect; otra también redirige. Donde podemos manipular es en los puntos donde se generan estas redirecciones; probamos con `/login?lang=en`:

![[Pasted image 20250928124331.png]]

La página de destino ya carga contenido:

![[Pasted image 20250928124731.png]]

`lang` se aplica directamente en la respuesta, pero se URL-encodea al insertarse:

![[Pasted image 20250928125111.png]]

Hay un recurso dentro de un script que se queda en caché y donde `lang=en` se refleja. Lo alteramos:

![[Pasted image 20250928125230.png]]

![[Pasted image 20250928125519.png]]

Enviamos la petición que el lab indica para obtener datos extra: una cabecera nos permite ver la **clave de caché** usada, y al final vemos un delimitador `$$`:

![[Pasted image 20250928125801.png]]

![[Pasted image 20250928130048.png]]

Probamos `lang=test` en la URL:

![[Pasted image 20250928130451.png]]

Modifica el contenido — y viaja a la otra petición como `lang=test`:

![[Pasted image 20250928130746.png]]

![[Pasted image 20250928135317.png]]

Hay un `vary: origin` que limita la caché al mismo origen/UA. Jugamos con la cabecera `origin`:

![[Pasted image 20250928135720.png]]

Pasa a ser parte de la clave. Probamos valores de parámetros `cors`:

![[Pasted image 20250928140739.png]]

Logramos que se considere. Hacemos una **header injection**:

![[Pasted image 20250928141240.png]]

Y a partir del `Content-Length` sobrescribimos la respuesta:

![[Pasted image 20250928141646.png]]

El problema es hacer que la consulta use esa clave de caché. Como conocemos cómo se consulta el documento, el propósito es representarlo en la primera solicitud. Param Miner encuentra `utm_content` (ignora la caché):

![[Pasted image 20250928142358.png]]

![[Pasted image 20250928142553.png]]

![[Pasted image 20250928142833.png]]

Con la siguiente inyección logramos redirigir y forzar el código:

![[Pasted image 20250928151542.png]]

La otra solicitud queda así:

![[Pasted image 20250928151629.png]]

Debemos almacenar en caché un código malicioso con el **mismo nombre de clave**: las dos claves difieren (los `$$`). Agregamos los `$$` a la petición:

![[Pasted image 20250928152233.png]]

![[Pasted image 20250928152453.png]]

Y por URL:

![[Pasted image 20250928152522.png]]

Las claves ya coinciden. Cacheamos ambas:

![[Pasted image 20250928153001.png]]

![[Pasted image 20250928153031.png]]

Resumen: generamos una petición con valores exactos que produce otra solicitud con clave exacta; una petición alterna con la misma clave pero inyectable envenena la caché; cuando el home genera esa segunda solicitud, agarra la caché infectada y ejecuta el XSS.

![[Pasted image 20250928153031.png]]
Lab Terminado.

### Lab: Envenenamiento de Caché Interna

Envenenar la **caché interna** del origin (detrás de la externa). Objetivo: alerta. Capturamos la raíz:

![[{F68BDB58-E8BB-4E96-9F77-9DFA375F4E2B}.png]]

El link canónico no da paso a inyecciones directas. Param Miner encuentra `x-forwarded-host`:

![[{280C8418-096B-4353-9290-BC8FD792C0FC}.png]]

![[{BEEE3D35-2493-43EC-B64B-80BBA141D89F}.png]]

Cambia el link canónico — pero se manipula la caché **externa** (la clave también cambia con la cabecera, así que la víctima sin la cabecera no la recibe). La meta: conseguir un match **sin** la cabecera. Hay dos puntos de inyección; enviamos de forma continua hasta ver un **tercer** punto en la respuesta — señal de que estamos entrando en la caché interna:

![[{FEF2D0BE-357B-48FE-82E8-CC3982CF0BC3}.png]]

Tras un tiempo, una nueva cabecera cambia: el `test.com` se fija sin nuestra cabecera. Dejamos el exploit server apuntando al dominio malicioso:

![[{77061D41-DA9C-467D-B994-9A9B549BAA1A}.png]]

Fijamos el dominio en la caché interna:

![[{E3A2DDC4-FD6F-4C8D-A484-ECA8B3A0215D}.png]]

El tercer link quedó fijado y se cachea de forma interna. Verificamos:

![[{97FCCC82-F1D2-41F0-983C-989C46413175}.png]]
Lab Terminado.

> [!warning] La lección de la caché interna
> Con dos capas de caché, envenenar la externa no basta si la clave depende de tu cabecera: la víctima nunca la envía. La técnica es inducir al origin a que **su propia caché interna** almacene la respuesta venenada bajo una clave que la víctima sí genere — el veneno entonces sobrevive a purgas de la capa externa y no depende de headers del atacante.

## Detección (defensor)

- Alertas por volumen anómalo de peticiones con headers `X-Forwarded-*` o parámetros `utm_*` hacia la misma URL (firma del probe/miner).
- Mismos patrones de payload (`<script>`, `alert(`) en headers o query en logs del CDN.
- Respuestas cacheadas que contienen dominios no propios en URLs de scripts/enlaces (`src="http://evil"`).
- Monitorear `Age`/`X-Cache`: respuestas que cambian de miss a hit tras una petición con headers extra.

## Impacto

1. **XSS distribuido**: el veneno llega a todos los usuarios cacheados — robo de sesión masivo, keyloggers, defacement.
2. **Redirección masiva**: phishing servido "desde" tu propio dominio.
3. **Exfiltración vía script cargado desde el origin del atacante** (con CORS abierto en el exploit server).
4. **Persistencia**: el veneno sobrevive a la sesión del atacante y a veces a purgas; en cachés internas, se re-almacena solo.
5. La vulnerabilidad subyacente puede ser "solo" un reflejo inofensivo — el impacto real lo da la caché.

## Prevención (con configuración)

```nginx
# Nginx: clave de caché EXPLÍCITA y normalizada — nunca heredar headers del cliente
proxy_cache_key "$scheme$request_method$host$uri$args";
proxy_ignore_headers X-Forwarded-Host X-Forwarded-Scheme;   # no reflejar headers de proxy
proxy_cache_valid 200 10m;

# Varnish: definir la clave y filtrar headers peligrosos
sub vcl_recv {
    if (req.http.X-Forwarded-Host) { unset req.http.X-Forwarded-Host; }
    if (req.url ~ "utm_") { set req.url = regsub(req.url, "&utm_[^&]+", ""); }
}
sub vcl_backend_response {
    set beresp.http.Cache-Control = "public, max-age=0, s-maxage=300";
}
```

```http
# Respuestas personalizadas o sensibles: no cachear jamás
Cache-Control: no-store, no-cache, private
Vary: *
```

Reglas de diseño:
- **Clave explícita**: definir exactamente qué componentes forman la clave (host canónico + path + query normalizada); nunca incluir headers del cliente salvo los estrictamente necesarios, y **normalizar antes de cachear**.
- **No reflejar** `X-Forwarded-Host`/`X-Original-URL` salvo detrás de un proxy propio que los fije y sane.
- **No cachear** páginas con contenido de usuario o personalizado (`private`/`no-store`).
- **Vary correcto**: si la respuesta varía por algún input, ese input debe estar en `Vary` **y** en la clave.
- Sanitizar/reflejar con escaping antes de componer respuestas cacheables (el XSS reflejado + caché = XSS masivo; ver [[Cross-Site Scripting XSS]]).
- En CDN, activar las protecciones de "unkeyed header" del proveedor y auditar la configuración de cache key con periodicidad.

## Errores comunes

**Atacante:**
1. No hacer el probe de clave con `Age` antes de nada: sin saber qué refresca la clave, envenenas a ciegas.
2. Envenenar con un UA/Origin propio cuando la respuesta tiene `Vary`: la víctima no la recibe (lab dirigido).
3. Olvidar la normalización: el navegador URL-encodea tu payload y el veneno no se reproduce (lab normalización).
4. No distinguir caché externa de interna: la cabecera que envenena la externa puede cambiar la clave y hacer invisible el veneno a la víctima (lab caché interna).
5. Creer que "Age: 0" significa "no cacheada" — puede significar "se refrescó TU clave".

**Defensor (falsas soluciones):**
1. "Solo cacheamos GET de la home" — basta un reflejo en un header para envenenar la home.
2. Cache key por defecto del CDN que incluye headers del cliente o los refleja.
3. Ignorar los `utm_*` como "métricas inofensivas" — son el unkeyed input preferido.
4. Cache-control laxo en páginas con XSS reflejado latente: la combinación multiplica el impacto.
5. Purgar la caché como respuesta permanente: sin cerrar el unkeyed input, el veneno regresa.

## Cheatsheet

| Paso | Acción |
|---|---|
| Probe de clave | Añadir parámetro único; observar `Age: 0` |
| Headers unkeyed | Param Miner → guess headers (`X-Forwarded-Host`, `X-Host`, `X-Forwarded-Scheme`) |
| Parámetros unkeyed | Param Miner → guess params (`utm_content`) |
| Cookies unkeyed | Quitar/probar cookies de preferencia en la respuesta |
| Fat GET | Body en GET con parámetro inyectado |
| Cloaking | `?utm=x;callback=payload` |
| Normalización | Envenenar con Burp (sin encode), link a víctima con encoding |
| Vary | Robar UA/Origin de la víctima (`<img>` en post) y envenenar con ese valor |
| Caché interna | Repetir hasta que la respuesta sin headers extra cambie (tercer punto de inyección) |

| Señal | Lectura |
|---|---|
| `Age` vuelve a 0 con tu input | El input está en la clave |
| `Age` no cambia con header que altera respuesta | Unkeyed input explotable |
| `Vary: User-Agent` + veneno no llega | Envenenar con el UA de la víctima |
| Dos capas: veneno externo invisible | Caché interna — persistir internamente |

## Laboratorios y referencias

- PortSwigger Academy — Web cache poisoning (todos los labs de arriba): https://portswigger.net/web-security/web-cache-poisoning
- PortSwigger Research — Web cache deception attac (relacionado): https://portswigger.net/research/web-cache-deception-attack
- RFC 9111 — HTTP Caching: https://datatracker.ietf.org/doc/html/rfc9111
- HackTricks — Web cache poisoning: https://book.hacktricks.xyz/pentesting-web/methods-for-vulnerabilities/web-cache-poisoning
- Param Miner: https://portswigger.net/burp/documentation/scanner/using-burp/extensions/param-miner

## Relacionadas

- [[Web cache deception]] — el primo del ataque: engañar a la caché para que cachee páginas privadas en lugar de envenenar respuestas
- [[HTTP Host header attacks]] — el Host header inyectado es el unkeyed input clásico que alimenta el poisoning
- [[Cross-Site Scripting XSS]] — el payload subyacente que convierte el poisoning en impacto directo
- [[Vulnerabilidades CORS y Laboratorios]] — necesario en los labs para servir el exploit cross-origin
- [[Vulnerabilidades de HTTP Request Smuggling]] — otra forma de confundir capas (smuggling + poisoning es la cadena definitiva)
