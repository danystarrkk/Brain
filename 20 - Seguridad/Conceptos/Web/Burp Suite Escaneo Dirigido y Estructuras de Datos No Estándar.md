---
aliases: ["Burp Suite Targeted Scanning", "Burp Suite Non-Standard Data Structures", "Escaneo Dirigido Burp", "Targeted Scan"]
tags:
  - sec/web
  - sec/web/tools
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Cross-Site Scripting XSS]]"
  - "[[XML External Entity Injection]]"
  - "[[Server-Side Request Forgery (SSRF)]]"
  - "[[Vulnerabilidades de HTTP Request Smuggling]]"
  - "[[JSON Web Tokens Explotación y Bypass]]"
  - "[[Condiciones de Carrera]]"
fuente: "https://portswigger.net/burp/documentation/desktop/scan-configurations"
---

# Escaneo Dirigido en Burp Suite y Estructuras de Datos No Estándar

El **escaneo dirigido** (*Targeted Scan*) es una funcionalidad de Burp Suite Professional que permite concentrar el motor de escaneo en **peticiones, parámetros o rangos de bytes concretos**, en lugar de rastrear y auditar una aplicación completa. Es la forma correcta de validar una hipótesis concreta — "este parámetro refleja, probar XSS aquí", "este endpoint parsea XML, probar XXE" — y acelera drásticamente los laboratorios cuando ya sabes por dónde entrar.

> [!info] Escaneo completo vs. escaneo dirigido
> Un *scan* normal de Burp hace dos cosas: **crawl** (descubrir endpoints) y **audit** (probar vulnerabilidades en todo lo descubierto). El escaneo dirigido salta el crawl y audit únicamente lo que tú seleccionas. Cuando el tráfico ya está capturado (lo normal en un laboratorio o tras una fase manual), el crawl es redundante: solo añade ruido, tiempo y peticiones que el laboratorio puede penalizar.

## Mecánica: cómo Burp decide dónde inyectar

Antes de usar el escaneo dirigido conviene entender el modelo interno:

| Concepto | Significado |
|---|---|
| **Insertion point** (*punto de inserción*) | Posición exacta dentro de la petición donde Burp sustituye payloads: parámetros de query, body, cookies, cabeceras, XML/JSON, rangos de bytes seleccionados |
| **Audit** | Fase de prueba activa: envía payloads a los insertion points y analiza respuestas |
| **Crawl** | Descubrimiento de endpoints siguiendo enlaces, forms y scripts |
| **Passive scan** | Análisis de respuestas ya capturadas sin enviar nada extra |
| **Scan checks / audit checks** | Las pruebas individuales (XSS reflejado, SQLi por error, XXE, SSRF...) con sus *variants* |
| **Resource pool** | Límites de concurrencia y timeout del scan (clave en labs con rate limit) |
| **Custom insertion point** | Selección manual de bytes dentro de la petición para tratarlos como parámetro |

El motor **solo prueba lo que interpreta como insertion point**. Si tu dato vive dentro de una estructura que el parser no modela (JSON anidado raro, un campo codificado en Base64, HTTP/2 con cabeceras pseudo, un parámetro duplicado), el escaneo automático lo ignora — de ahí la segunda parte de esta nota.

> [!tip] Flujo mental del laboratorio
> 1. Capturar la petición que refleja/recibe tu dato → 2. Enviar a Repeater y confirmar manualmente que el dato llega al contexto → 3. **Repeater → Scan** (escaneo dirigido, audit only) → 4. Si Burp no encuentra nada, revisar *insertion points* y definir uno manual sobre tu selección → 5. Si el contexto es ciego (sin reflejo), usar **Burp Collaborator** en los payloads.

## Workflow del escaneo dirigido

Flujo aplicado en el laboratorio — capturar, seleccionar y auditar:

1. Capturamos la petición web en Proxy/Target:

![[Pasted image 20260223212852.png]]

2. Clic derecho en la petición y seleccionamos `Scan`:

![[Pasted image 20260223212958.png]]

3. Seleccionamos las opciones mostradas en la imagen (en la práctica: **audit** de la petición seleccionada, sin crawl, con la configuración de scan por defecto o una personalizada) y procedemos con el escaneo:

![[Pasted image 20260223213113.png]]

4. Confirmamos la selección y comenzamos el escaneo:

![[Pasted image 20260223213156.png]]

5. Burp Suite comienza a probar diferentes vectores de ataque en los insertion points detectados; es cuestión de esperar a que finalice:

![[Pasted image 20260223213542.png]]

6. Observamos una vulnerabilidad de tipo `External service interaction`, específicamente una **Inyección de Entidades Externas (XXE)**:

![[Pasted image 20260223213557.png]]

> [!warning] XXE (XML External Entity)
> Una vulnerabilidad XXE ocurre cuando un parser XML procesa una entrada que contiene referencias a **entidades externas**. Esto puede llevar a divulgación de archivos locales, ejecución remota de código (RCE), denegación de servicio (billion laughs) o interacciones con servicios externos (SSRF ciego). Ver [[XML External Entity Injection]].

### Explotación manual del hallazgo

Dado que se están manipulando entidades externas, y recordando los conceptos de XXE, utilizamos un payload específico para forzar la lectura de recursos internos del sistema:

> [!tip] Payload clásico de divulgación de archivos en XXE
> `<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>` seguido de una referencia a la entidad `&xxe;` dentro del XML. El parser lee `/etc/passwd` y su contenido se devuelve en la respuesta. Variante ciega: `SYSTEM "http://<collaborator>"` para confirmación por out-of-band.

![[Pasted image 20260223214102.png]]

## Estructuras de datos no estándar: por qué los escáneres fallan

> [!info] Qué es una "estructura no estándar"
> Formatos que el escáner no modela bien: JSON anidado profundo o con arrays, JSON dentro de JSON, XML/SOAP, multipart con nombres raros, datos codificados (Base64, hex, URL-encoded múltiple), serializaciones propietarias (JWT, Java serialization, pickle), o tráfico **HTTP/2** con cabeceras pseudo. Todas son pasadas por alto por el fuzzing ingenuo y por eso suelen contener vulnerabilidades que ninguna herramienta detecta sin ayuda.

### Categorías y cómo tratarlas

| Estructura | Por qué rompe el escáner | Cómo probarla |
|---|---|---|
| **JSON anidado** (`{"a":{"b":{"c":"val"}}}`) | El parser usa solo claves de primer nivel como parámetros | Definir insertion point manual sobre el valor interno; probar payloads escapados para JSON: `\",\"x\":\"` |
| **Arrays JSON** (`{"ids":[1,2,3]}`) | Los índices no se tratan como parámetros | Seleccionar el elemento como insertion point; probar `1 OR 1=1` si el backend itera |
| **JSON dentro de string** (`{"data":"{\"k\":\"val\"}"}`) | El dato es un string "escapado" — doble contexto | Cerrar el escape primero: `\"}; malicious; //` |
| **Base64/Hex en parámetro** | El valor no es texto plano; payloads literales no llegan vivos | Decodificar → inyectar → recodificar manualmente (Burp Decoder) |
| **HTTP/2 y HTTP/2 prioridad** | Cabeceras pseudo (`:method`, `:path`), HPACK comprime cabeceras; el scanner histórico fallaba | Usar Repeater con HTTP/2; editar cabeceras pseudo directamente; probar request smuggling H2→H1 (ver [[Vulnerabilidades de HTTP Request Smuggling]]) |
| **Parameter pollution** (`a=1&a=2` o `{"a":1,"a":2}`) | Frameworks toman la primera, la última o la concatenación | Enviar duplicados con valores distintos y observar cuál interpreta el backend |
| **JWT / tokens serializados** | Campo con estructura propia dentro de la cabecera Cookie/Authorization | Decodificar en Decoder, atacar el algoritmo (ver [[JSON Web Tokens Explotación y Bypass]]) |
| **XML con prefijos/entidades** | Atributos, namespaces y DTDs no son "parámetros" | Insertion point sobre el valor + DTD externa propia |
| **Multipart con filename** | El nombre del archivo es un insertion point olvidado | Inyectar en `filename=` (ver [[Abuso de Subida de Archivos]]) |
| **JSON doblemente anidado en JS** | El dato viaja como string dentro de JSON y se `JSON.parse` en cliente | Contexto DOM: escapar de la cadena y de la estructura (ver [[Cross-Site Scripting XSS]]) |

> [!tip] Regla de oro
> Cuando Burp no marca nada, pero tú **ves** tu dato reflejado en la respuesta, el fallo es del parser, no de la app. Selecciona a mano los bytes exactos donde entra tu dato → *Define custom insertion point* → y ahí es donde vuelve a escanear. La combinación "humano identifica contexto, Burp fuerza payloads" es la más eficiente del escaneo dirigido.

### Workflow con estructuras no estándar

Comenzamos capturando la petición — contiene el valor `wiener` y otra estructura de datos cuyo formato exacto desconocemos:

![[Pasted image 20260223214732.png]]

Seleccionamos las opciones mostradas, asegurándonos de tener seleccionado el valor `wiener` (así Burp define el insertion point sobre ese rango exacto de bytes):

![[Pasted image 20260223215209.png]]

![[Pasted image 20260223215222.png]]

Iniciamos el `Scan` y esperamos a que Burp Suite complete su análisis:

![[Pasted image 20260223215741.png]]

Al enviar la petición al Repeater observamos que el sistema está intentando escapar de algún contexto, pero nos permite realizar peticiones externas:

![[Pasted image 20260223215904.png]]

> [!info] External service interaction
> La detección de `External service interaction` indica una vulnerabilidad **ciega**: SSRF, XXE ciego, XSS ciego o inyección de comandos que hace que el servidor contacte un recurso externo. El **Burp Collaborator** (o [[Server-Side Request Forgery (SSRF)|SSRF]] out-of-band con interactsh) registra la interacción (DNS/HTTP/SMTP) y confirma la vulnerabilidad aunque no haya respuesta visible.

![[Pasted image 20260223220137.png]]

### Exfiltración vía Collaborator

Modificamos la petición para que interactúe con nuestro Burp Collaborator, incluyendo la cookie del usuario objetivo codificada en Base64:

> [!tip] Burp Collaborator
> Genera un dominio único (p. ej. `abc123.oastify.com`) que, al ser contactado por la aplicación vulnerable, registra la interacción y revela la existencia y detalles de la vulnerabilidad. La exfiltración en Base64 evita que caracteres especiales de los datos rompan el transporte (query string, headers).

![[Pasted image 20260223220356.png]]

Hemos recibido la interacción en Collaborator, confirmando la exfiltración del identificador deseado. Esto sugiere que el usuario administrador, a nivel de sesión, fue afectado por una inyección en algún panel administrativo — posiblemente un **visor de logs** donde los comandos inyectados fueron interpretados, enviándonos sus cookies.

Para finalizar, iniciamos sesión como `administrator` utilizando la cookie obtenida:

![[Pasted image 20260223220713.png]]

Una vez dentro, confirmamos la existencia de un apartado de Logs donde se interpretan las entradas. Hacemos una prueba adicional inyectando una alerta JavaScript para verificar la vulnerabilidad de XSS:

![[Pasted image 20260223220852.png]]

Enviamos la petición y verificamos si la alerta se genera al acceder a la sección de logs:

![[Pasted image 20260223220924.png]]

Efectivamente, este es el punto vulnerable. Finalmente, eliminamos al usuario `carlos` para completar el laboratorio:

![[Pasted image 20260223220957.png]]

> [!warning] El patrón descubierto
> El hallazgo encadena: **estructura no estándar → inyección ciega → panel que interpreta datos de logs → XSS administrativo → robo de sesión**. Los visores de logs son un clásico "second-order sink": los datos guardados por el sistema (user-agent, campos de formulario, cabeceras) se re-renderizan en el panel de administración sin escapar. Relacionado: [[Log Poisoning]].

## Detección y validación manual vs automatizada

| Método | Ventaja | Cuándo usarlo |
|---|---|---|
| **Escaneo dirigido (audit)** | Rápido, cubre variants de payloads, deja informe | Hipótesis clara sobre un endpoint/parámetro conocido |
| **Repeater manual** | Control total del contexto y la codificación | Contextos codificados/exóticos; confirmación de hallazgos |
| **Intruder + wordlists** | Fuzzing de etiquetas/eventos permitidos, campos ocultos | Blacklists de tags (labs de XSS), parámetros desconocidos |
| **Collaborator/interactsh** | Detecta ceguera total (sin reflejo, sin error) | XXE/SSRF/XSS ciego, callbacks en payloads |
| **Turbo Intruder** | Cientos de peticiones en una conexión | Carreras, rate limits (ver [[Condiciones de Carrera]]) |

Señales que indican que el dato llega "vivo" aunque el escáner guarde silencio:
- **Reflejo byte a byte** en la respuesta (Repeater con highlight).
- **Retardo** proporcional al payload (time-based) — el parser lo evaluó.
- **Error diferencial** ante comillas/llaves: romper la estructura produce 500/stacktrace = contexto descubierto.
- **Interacción out-of-band** sin ninguna respuesta visible.

## Errores comunes

**Atacante:**
1. Lanzar un scan completo de la app cuando la petición vulnerable ya está capturada — pierde tiempo y satura el resource pool.
2. Confiar en que el escáner interpreta el contexto: **JSON profundo, Base64 y HTTP/2 requieren insertion point manual**.
3. Ignorar los "External service interaction" como falsos positivos de laboratorio: en la vida real suelen ser el único rastro de SSRF/XXE ciego.
4. No verificar el efecto en Repeater antes de automatizar: el scanner prueba variants genéricas; tu contexto puede necesitar un escape concreto.
5. Escanear con la app autenticada como usuario equivocado — muchos checks solo tienen sentido con la sesión adecuada.

**Defensor (falsas soluciones):**
1. "El escáner de seguridad no marcó nada" — los escáneres fallan sistemáticamente con estructuras no estándar y ceguera; es un input, no una garantía.
2. Filtrar payloads literales (`<script>`) sin escapar por contexto: con inserción en JSON/atributos/JS el bypass es trivial (ver [[Cross-Site Scripting XSS]]).
3. Bloquear el reflejo pero dejar la interacción saliente (SSRF/XXE out-of-band sigue funcionando).

## Cheatsheet

| Acción | Dónde |
|---|---|
| Escaneo dirigido de petición | Clic derecho en Repeater/Proxy → **Scan** |
| Audit sin crawl | Scan configuration → *Audit only* / deseleccionar crawl |
| Insertion point manual | Seleccionar bytes → **Define custom insertion point** |
| Revisar insertion points detectados | Panel *Scan task → Audit → Insertion points* |
| Payload out-of-band | Insertar dominio Collaborator en el punto (URL/header/body) |
| Codificar/decodificar estructuras | Decoder (Base64, URL, hex) — recodificar antes de reenviar |
| HTTP/2 | Selector de protocolo en Repeater; editar cabeceras pseudo |

| Hallazgo típico | Estructura que lo esconde | Vector |
|---|---|---|
| XXE | XML/JSON con XML embebido | DTD externa + entidad |
| SSRF ciego | Parámetro de URL en estructura anidada | Collaborator en el valor |
| XSS | Dato dentro de JSON/JS embebido | Escape de comillas + contexto |
| Smuggling | HTTP/2 front-end → HTTP/1.1 back-end | `Transfer-Encoding` oculto |

## Laboratorios y referencias

- PortSwigger Academy — Scanner: https://portswigger.net/burp/documentation/scanner
- PortSwigger Academy — XXE: https://portswigger.net/web-security/xxe
- PortSwigger Academy — SSRF: https://portswigger.net/web-security/ssrf
- Burp Collaborator: https://portswigger.net/burp/documentation/collaborator
- interactsh (alternativa open-source): https://github.com/projectdiscovery/interactsh

## Relacionadas

- [[XML External Entity Injection]] — el hallazgo del primer laboratorio en profundidad
- [[Server-Side Request Forgery (SSRF)]] — la otra mitad del "external service interaction"
- [[Cross-Site Scripting XSS]] — el XSS administrativo descubierto vía logs
- [[Vulnerabilidades de HTTP Request Smuggling]] — la estructura más hostil para los escáneres (HTTP/2)
- [[JSON Web Tokens Explotación y Bypass]] — tokens como estructura no estándar dentro de Authorization
- [[Condiciones de Carrera]] — cuándo el escáner debe complementarse con Turbo Intruder
