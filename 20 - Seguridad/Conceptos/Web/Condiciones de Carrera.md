---
aliases: ["Race Conditions", "Condiciones de Carrera", "TOCTOU", "Time of Check Time of Use"]
tags:
  - sec/web
  - sec/web/logica
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Abuso de Subida de Archivos]]"
  - "[[Laboratorios de Vulnerabilidades Lógicas y de Autenticación]]"
  - "[[Insecure Direct Object References IDOR]]"
  - "[[Session Puzzling Session Fixation y Session Variable Overloading]]"
  - "[[Cross-Site Request Forgery (CSRF)]]"
fuente: "https://portswigger.net/web-security/race-conditions"
---

# Condiciones de Carrera (Race Conditions)

Una **condición de carrera** (*race condition*) ocurre cuando el resultado correcto de una operación depende del **orden no garantizado** en que dos o más operaciones concurrentes se ejecutan, y la aplicación carece de sincronización que lo impida. En seguridad web, la explotación clásica es **TOCTOU** (*Time-Of-Check to Time-Of-Use*): el servidor **verifica** un estado (saldo, cupón sin usar, permiso, ausencia de registro) en un instante y **usa** ese estado en otro; un atacante inserta peticiones en la ventana entre ambos momentos y engaña al servidor para que aplique una acción pensada para un estado que ya no existe.

> [!info] Secciones críticas y primitivas de sincronización
> Una "sección crítica" es el código que accede a un recurso compartido. Para evitar carreras se usan **mutexes** (exclusión mutua), **semáforos**, **locks** y **monitores**, que garantizan que solo un hilo/proceso esté en la sección crítica a la vez. En bases de datos, el equivalente son las **transacciones** con bloqueo (`SELECT ... FOR UPDATE`) o las **constraints únicas**. La ausencia de estas primitivas en una operación sensible es la vulnerabilidad.

## Mecánica: por qué el web es un terreno de carreras

En una aplicación web, cada petición HTTP es potencialmente un hilo distinto. El modelo de concurrencia del backend decide cuántas peticiones se procesan **a la vez**, y eso define si una carrera es viable:

| Factor | Efecto sobre la explotación |
|---|---|
| **Servidor async** (Node, Go, event loops) | Procesa miles de peticiones en paralelo → carreras fáciles con 2 peticiones |
| **Servidor sync con workers** (PHP-FPM, WSGI con N workers) | Paralelismo limitado al número de workers → disparar muchas peticiones a la vez para superar la serialización aparente |
| **Lockeo de sesión** (p. ej. PHP por defecto bloquea la sesión por petición) | **Serializa** las peticiones de la misma sesión: la carrera "no funciona"... hasta que envías peticiones desde **sesiones distintas** (lab time-sensitive: dos cookies de sesión diferentes en paralelo) |
| **HTTP/1.1** | Cada petición puede usar una conexión; el navegador abre varias → paralelismo real |
| **HTTP/2** | Todas las peticiones viajan **multiplexadas en UNA conexión TCP** → el ataque "single packet" puede soltar decenas de peticiones casi simultáneas (el mismo frame de red) |
| **Keep-alive + Repeater** | Burp envía grupos en paralelo sobre conexiones separadas — el clásico "Group of parallel requests" |
| **Load balancer / WAF** | Puede serializar o rate-limiter; parte del trabajo del atacante es esquivarlo (sesiones distintas, timing) |

> [!tip] La intuición del laboratorio
> Cuando una operación "no se puede repetir" (cupón único, cambio de email con token, registro duplicado) **funciona en secuencial pero no en paralelo**, no concluyas "está bien protegido": el servidor puede estar simplemente serializando por sesión o por worker. El test correcto es **paralelo con muchas repeticiones**, no una sola vez.

### Patrones de ataque (taxonomía)

| Patrón | Mecánica | Ejemplo típico |
|---|---|---|
| **Overrun de límite** | Aplicar una operación restringida N veces antes de que el contador se registre | Cupón de descuento aplicado 50 veces en paralelo |
| **Bypass de rate limit** | El contador de intentos fallidos se incrementa tras validar, no antes | Fuerza bruta de login sin que el lockout se dispare |
| **Carrera multi-endpoint** | Dos endpoints con tiempos de respuesta distintos; el lento corre tras la validación del rápido | Validar precio (rápido) mientras añades producto (lento) al carrito |
| **Carrera single-endpoint** | El mismo endpoint procesa peticiones paralelas con estado compartido | Cambio de email: el cuerpo del correo (destino) y la petición (token) se desincronizan |
| **Doble gasto de token** | Un token de un solo uso se consume tras la verificación, no atómicamente | Tokens de invitación, reset de contraseña, cupones |
| **Construcción parcial** | La entidad se crea incompleta; una operación paralela la aprovecha antes del relleno | Registro con email sin confirmar: `token=` vacío confirmado a la vez que se crea la cuenta |
| **TOCTOU de archivo** | Validar archivo temporal y moverlo en momentos distintos | Upload validado → carrera → ejecución antes del borrado (ver [[Abuso de Subida de Archivos]]) |

### Secuencia TOCTOU del overflow de cupón (lab 1)

```
T0  POST /apply-coupon  ─┐
T1  POST /apply-coupon   │  50 peticiones idénticas EN PARALELO
...                       │
T49 POST /apply-coupon  ─┘
     → El servidor valida "¿ya usó el cupón?" (todavía NO registrado)
     → 50 veces pasa la validación, 50 veces aplica el 20%
     → El registro "usado" llega DESPUÉS: ya es tarde
Resultado: descuento acumulado ~1000% → producto gratis
```

## Metodología de laboratorio

### Técnica 1 — Burp Repeater: grupo de peticiones paralelas

1. Capturar la petición sensible en Proxy.
2. Enviar a Repeater → **Create group** (o clic derecho → *Group of parallel requests*).
3. Duplicar la petición dentro del grupo N veces (30–50 para servidores sync).
4. Modificar los valores que deben diferir (cupones distintos, emails distintos) según el patrón.
5. **Send group in parallel** (no "sequentially") y observar respuestas: no todas tienen por qué "entrar" — interesa que **alguna** supere la validación.
6. Verificar el efecto (descuento aplicado, correo llegado, contador).

### Técnica 2 — Turbo Intruder con `gate` (la robusta)

Para carreras que requieren precisión milimétrica (el registro y la confirmación deben coincidir), se usa el plugin **Turbo Intruder** de PortSwigger con el patrón `gate`:

```python
# Turbo Intruder — race con gate: encolar N peticiones y liberarlas a la vez
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2)   # HTTP/2: single-packet attack
    for i in range(50):
        engine.queue(target.req, target.baseRequest.split('username=')[0] + 'username=test%d' % i,
                     gate='race')
    engine.openGate('race')   # suelta las 50 en el mismo frame

def handleResponse(req, interesting):
    table.add(req)
```

El template `race-single-packet-attack.py` viene con Turbo Intruder y hace exactamente esto: todas las peticiones del gate viajan en un **único paquete HTTP/2**, superando cualquier serialización de aplicación. También existe la opción de cargar la *bambda* [ProbeForRaceCondition](https://github.com/PortSwigger/bambdas/blob/main/CustomAction/ProbeForRaceCondition.bambda) como custom action de Burp para disparar carreras desde Repeater directamente.

### Técnica 3 — Análisis de tiempos

Antes de explotar, **mide** la diferencia de latencia entre endpoints (multi-endpoint) o entre sesiones (lockeo de sesión). Si dos peticiones del mismo grupo responden en tiempos muy distintas, el paralelismo no es real: probar con **dos cookies de sesión diferentes** para sortear el lockeo por sesión (lab "time-sensitive").

## PortSwigger

### Limit overrun race conditions

El propósito es comprar un producto pagando un valor no esperado. Comenzamos ingresando en la web:

![[Pasted image 20260311190918.png]]

Tenemos 50 dólares y un cupón de descuento del 20%; veamos qué hay en la tienda:

![[Pasted image 20260311191020.png]]

Mi objetivo es comprar la chaqueta, no me alcanza, así que primero la agregamos al carrito:

![[Pasted image 20260311191102.png]]

Tenemos una opción para aplicar el cupón de forma única — la usamos:

![[Pasted image 20260311191154.png]]

Bueno, no podemos aplicarlo más de una vez. Nuestro pensamiento: hay dos fases, la aplicación del cupón y la validación/registro. ¿Qué sucede si aplicamos el cupón muchas veces **antes** de llegar a la fase de verificación? Mientras el sistema no registre el uso en la cuenta, seguimos entrando.

Primera forma: capturar la petición de aplicación del cupón:

![[Pasted image 20260311191710.png]]

Desde Repeater creamos un grupo y duplicamos la consulta dentro del grupo:

![[Pasted image 20260311191907.png]]

Enviamos las 50 peticiones en paralelo; no todas serán válidas, pero algunas entrarán:

![[Pasted image 20260311192016.png]]

Algunas aplican y otras no — revisamos el descuento:

![[Pasted image 20260311192054.png]]

El descuento ya es mayor al 20% original. Para bajar más el precio, basta con aumentar el número de peticiones.

Segunda forma: cargar la *bambda* ProbeForRaceCondition en Burp ([link](https://github.com/PortSwigger/bambdas/blob/main/CustomAction/ProbeForRaceCondition.bambda)), capturar la petición y enviarla a Repeater:

![[Pasted image 20260311192717.png]]

Entramos en Custom actions → Load → *Create new template*:

![[{840B214E-273C-48CD-9284-4A0E109D1D2D}.png]]

Se despliega la ventana:

![[Pasted image 20260311193000.png]]

Nos desplazamos a *Trigger race condition*: el código ya está implementado; damos en *create using this template* (si hay que cambiarlo, se puede pegar el código de la web):

![[Pasted image 20260311193301.png]]

Cambiamos el número de peticiones al deseado (50) y damos OK:

![[Pasted image 20260311193434.png]]

Es cuestión de dar al botón de play y ver qué sucede:

![[Pasted image 20260311193558.png]]

Tenemos ya un descuento enorme — esto ya se puede pagar:

![[Pasted image 20260311193638.png]]

Lab Terminado.

### Bypassing rate limits via race conditions

Laboratorio donde sobrepasar los límite de intentos de login para fuerza bruta. Vamos directamente: la petición de inicio de sesión ya está capturada:

![[Pasted image 20260311194147.png]]

La condición de carrera se produce con la extensión **Turbo Intruder** (ya instalada): clic derecho → extensiones → Turbo Intruder sobre la petición → ventana siguiente:

![[Pasted image 20260311194502.png]]

Para ejecutar la carrera modificamos la parte de *Last code used* por la de `race-single-packet-attack.py`:

![[Pasted image 20260311194557.png]]

Podemos observar el código; hay que modificar la petición donde el valor de password se representa como `%s` en Python:

![[Pasted image 20260311194724.png]]

Agregamos nuestra lista de contraseñas al código:

![[Pasted image 20260311195125.png]]

Copiamos las contraseñas, modificamos el bucle y enviamos:

![[Pasted image 20260311200939.png]]

Solo una respuesta es `302` (redirección de login exitoso) — esa es la válida. Verificamos:

![[Pasted image 20260311201050.png]]

Lab Terminado.

> [!warning] Por qué funciona aquí
> El rate limit cuenta los intentos **fallidos** y la validación de credenciales ocurre en paralelo: si el contador se incrementa *tras* confirmar el fallo, las 50 peticiones del gate se procesan antes de que el lockout se active. La defensa correcta (ver prevención) es bloquear **antes** de validar, o con constraints atómicas.

### Multi-endpoint race conditions

Tenemos un mecanismo de compra que permite pagar productos por un precio no intencionado; el objetivo es pagar un producto para el que no tenemos dinero suficiente:

![[Pasted image 20260311202219.png]]

![[Pasted image 20260311202232.png]]

Hay dos peticiones: la primera agrega un producto al carrito y la segunda paga. Pensando en fases: el pago valida si tenemos dinero suficiente y luego compra. ¿Qué pasa si, tras la validación, logramos incorporar otro producto que ya no se paga? Hay que verificar los tiempos: la de agregar producto tarda **más** que la de validar+y-comprar — si enviamos en paralelo, la lenta se inserta en la ventana de la rápida:

![[Pasted image 20260311214400.png]]

Creamos el grupo con las dos peticiones y enviamos en paralelo:

![[Pasted image 20260311214335.png]]

Lab Terminado.

### Single-endpoint race conditions

Alguien con una dirección de correo tiene una invitación pendiente para ser administrador: si alguien crea la cuenta con ese correo, hereda las propiedades de administrador. Tenemos un cliente de email:

![[Pasted image 20260312092810.png]]

Hay un apartado de cambio de email; probamos cambiándolo registrando el flujo en Burp:

![[Pasted image 20260312092920.png]]

Habla de un link de confirmación; en el cliente de email encontramos dicho email:

![[Pasted image 20260312093000.png]]

Confirmamos y el correo se cambia en el dashboard:

![[Pasted image 20260312093027.png]]

Esto lo registramos en Burp; vemos la petición del cambio:

![[Pasted image 20260312093122.png]]

Creamos un grupo con la petición duplicada unas 10 veces, cambiando en cada una el correo para intentar enviar por individual cada uno:

![[Pasted image 20260312093414.png]]

Enviamos todas por conexiones separadas:

![[Pasted image 20260312093442.png]]

En el cliente, donde deberíamos ver todos los correos:

![[Pasted image 20260312093546.png]]

No vemos nada inusual en secuencial. ¿Y si mandamos ahora en paralelo?

![[Pasted image 20260312093633.png]]

Verificamos el correo:

![[Pasted image 20260312093732.png]]

Atención: el correo va a `test10` pero **el contenido de la confirmación es para `test2`**. Detectada la condición de carrera: el servidor redacta el mensaje (con el token del usuario pendiente en ese momento) y luego envía a la dirección pendiente actual — dos operaciones desincronizadas, y el mensaje correcto llega a un buzón equivocado.

La explotación: registrar un correo que sí nos llegue primero, y duplicar la consulta con el correo **víctima** en el mismo grupo:

![[Pasted image 20260312094338.png]]

![[Pasted image 20260312094526.png]]

Grupo con las dos peticiones en paralelo, intentando que el correo se genere para carlos pero nos llegue a nosotros:

![[Pasted image 20260312094620.png]]

Enviamos:

![[Pasted image 20260312094920.png]]

No sucede a la primera: se necesitan varias repeticiones — paciencia. Cuando confirmamos, el correo se vincula a la cuenta:

![[Pasted image 20260312100113.png]]

Tenemos el panel de administrador; borramos al usuario Carlos para terminar:

![[Pasted image 20260312100157.png]]

Lab Terminado.

### Exploiting time-sensitive vulnerabilities

Laboratorio con un mecanismo de reset de contraseña roto: criptografía débil en la creación de tokens:

![[Pasted image 20260312100648.png]]

Enviamos el reset y en el cliente de email encontramos una ruta con un token:

![[Pasted image 20260312100822.png]]

Lo copiamos e identificamos la encriptación:

![[Pasted image 20260312101242.png]]

Parece SHA1. El token puede estar creado con tiempo, fecha o un valor que envía el servidor — muchas posibilidades. Vemos cómo se tramita al entrar a la URL:

![[Pasted image 20260312101545.png]]

Se tramita un `csrf` que puede o no estar incluido.

Enviamos dos consultas en paralelo para observar el comportamiento:

![[Pasted image 20260312101811.png]]

Revisamos los tiempos — difieren:

![[Pasted image 20260312101853.png]]
![[Pasted image 20260312101909.png]]

Uno mayor que otro: **no se envían en paralelo sino secuencial**, probablemente una restricción por cookie de sesión para evitar problemas (lockeo de sesión). Y si nos fijamos en los correos:

![[Pasted image 20260312101956.png]]

no cambian.

Refrescamos la cookie para enviar dos peticiones en paralelo **desde diferentes sesiones** en Burp:

![[Pasted image 20260312102427.png]]

![[Pasted image 20260312102437.png]]

Tenemos dos sesiones; lo intentamos:

![[Pasted image 20260312102829.png]]

Ahora sí las peticiones se enviaron en paralelo. En el panel de email:

![[Pasted image 20260312102938.png]]

**El token de ambas es igual**: el generador usa el tiempo actual + un valor secreto ( SHA1(timestamp + secret) ). No conocemos el secreto, pero seguimos jugando: enviar en paralelo una petición para `carlos` y otra para `wiener`:

![[Pasted image 20260312104048.png]]

Vemos la URL en el email:

![[Pasted image 20260312104203.png]]

La abrimos cambiando `user=carlos`:

![[Pasted image 20260312104243.png]]

Es válido — cambiamos la contraseña:

![[Pasted image 20260312104324.png]]

Logramos cambiarla e ingresar a su cuenta; eliminamos al usuario `carlos`:

![[Pasted image 20260312104411.png]]

Lab Terminado.

### Partial construction race conditions

Podemos sobrepasar el registro de un email mediante una condición de carrera, usando un correo que no poseemos (dominio `@ginanjuice.shop` sin acceso). Sin credenciales, nos registramos primero:

![[Pasted image 20260312105215.png]]

![[{C210CB6B-DBDD-4D0C-B6EA-4D3E438E49F9}.png]]

Intentamos registrar algo válido mientras capturamos todo:

![[Pasted image 20260312105553.png]]

La petición de registro:

![[Pasted image 20260312105644.png]]

Reenviarla no funciona: está "en espera de confirmación":

![[Pasted image 20260312105752.png]]

Cambiando el nombre de usuario tampoco es válido.

Navegando encontramos un script de JavaScript interesante:

![[Pasted image 20260312110150.png]]

En el código de respuesta encontramos la función `confirmEmail`:

![[Pasted image 20260312110227.png]]

La confirmación es un `POST` a `/confirm?` + `action` donde `action` es un token y su valor. Lo reproduzcimos en Burp:

![[Pasted image 20260312110510.png]]

Responde "token invalid": el token se crea tras el registro y se envía al correo en un enlace.

La idea: si al crear la cuenta enviamos la confirmación del token **a la vez**, podríamos encontrar el token aún sin asignar — es decir, vacío (`token=`). Probamos:

![[Pasted image 20260312111816.png]]

Eso está prohibido — parece que esa vulnerabilidad ya se parcheó. Pero "vacío" se puede representar de otras formas: `token[]=` o `token[]`:

![[Pasted image 20260312111946.png]]

¡Pasa! Tomando esto, creamos el grupo con la petición de registro lista:

![[Pasted image 20260312112152.png]]

![[Pasted image 20260312112204.png]]

No es válido todavía: mirando los tiempos, no fue del todo en paralelo, y el servidor responde más rápido a la verificación que al registro — al no tener registrado el usuario, no confirma nada. Jugamos con **Turbo Intruder**: para un registro, armamos un `gate` con 50 peticiones de confirmación de token que se liberan al mismo tiempo que el registro, para que alguna coincida con la ventana en que el usuario existe pero su token aún no está asignado.

Petición de registro → Turbo Intruder:

![[Pasted image 20260312113038.png]]

Generamos usernames `test1,test2,...` y modificamos el código:

![[Pasted image 20260312113714.png]]

Para la petición de registro armamos un `gate` y, con un bucle anidado en el mismo gate, encolamos las 50 peticiones de confirmación; al `openGate` se liberan juntas:

![[Pasted image 20260312114127.png]]

No funcionó a la primera: solo se registraron correctamente. Aumentamos a 60, cambiamos usuario y correo, y probamos con distintos valores:

![[Pasted image 20260312114946.png]]

Funcionó: activamos la cuenta de `beater9`. Intentamos iniciar sesión:

![[Pasted image 20260312115056.png]]

Laboratorio terminado.

## Detección (defensor)

- Patrones de peticiones idénticas en ráfaga desde una IP/sesión (NGINX/WAF logs con misma URI repetida en <1s).
- Multiples `POST /apply-coupon`, `/confirm`, `/change-email` con outcomes exitosos duplicados para el mismo usuario.
- Auditoría de código: toda operación "check-then-act" sobre saldo/contador/token sin transacción o constraint.
- Tests de estrés con concurrencia en CI (Gatling/JMeter con 100 hilos contra los endpoints sensibles) — si el resultado no es idempotente, hay bug.

## Impacto

1. **Pérdida económica**: precios manipulados, límites de cupones/descuentos superados, retiros dobles.
2. **Bypass de seguridad**: rate limits (fuerza bruta), límites de email/two-step flows, confirmaciones de doble paso.
3. **Escalada de privilegios**: heredar roles (invitaciones), confirmar cuentas con tokens ajenos/parciales.
4. **Robo de datos**: mensajes con tokens enviados al buzón equivocado (session hijacking indirecta).
5. **Integridad**: corrupción de estado en balanceadores, colas y sistemas concurrentes.

## Prevención (con código)

```sql
-- 1. Constraint atómica: el cupón solo puede aplicarse UNA vez
-- (el INSERT falla si ya existe — no hay ventana TOCTOU)
INSERT INTO coupon_redemptions (user_id, coupon_id) VALUES (?, ?);
-- Sin tabla intermedia: UPDATE atómico con condición:
UPDATE accounts SET balance = balance - 500
 WHERE id = ? AND balance >= 500;   -- el WHERE condicional hace el check atómicamente
-- 2. Bloqueo pesimista en flujos de lectura-modificación:
SELECT * FROM accounts WHERE id = ? FOR UPDATE;  -- dentro de BEGIN/COMMIT
```

```python
# 2. Lockeo de aplicación (Redis) alrededor de la sección crítica
import redis, time
r = redis.Redis()

def apply_coupon(user_id, coupon):
    lock = r.lock(f"coupon:{user_id}", timeout=5, blocking_timeout=2)
    if not lock.acquire():
        return "Intenta más tarde"        # rechazar, no encolar ciegamente
    try:
        if r.sismember("coupons_used", f"{user_id}:{coupon}"):
            return "Ya usado"             # check Y mark dentro del lock
        r.sadd("coupons_used", f"{user_id}:{coupon}")
        apply_discount(user_id, 0.20)
    finally:
        lock.release()
```

```javascript
// 3. Rate limit ANTES de validar (no después del fallo)
app.post('/login', rateLimit({ windowMs: 15*60*1000, max: 5 }), authMiddleware);
// 4. Tokens de un solo uso: consume atómicamente
const ok = db.consumeToken(token);  // DELETE FROM tokens WHERE token=? AND used=0 → affected rows == 1
```

Reglas de diseño:
- **Check-then-act → act-then-check**: condición en el `WHERE` del `UPDATE`, constraint UNIQUE, o lock distribuido. Nunca dos queries sin transacción.
- **Idempotencia**: claves idempotentes en pagos/acciones para que la repetición no multiplique efectos.
- **Rate limit previo** a la autenticación y a acciones sensibles.
- Desconfiar del "no puedo reproducirlo": las carreras son intermitentes por naturaleza — probar con miles de repeticiones antes de cerrar un hallazgo.

## Errores comunes

**Atacante:**
1. Enviar el grupo de Repeater en modo **sequential** por error — no hay carrera, todo pasa en orden.
2. Una sola ronda de 2 peticiones: los servidores con workers serializan — subir a 30–50 peticiones y repetir rondas.
3. Ignorar el lockeo de sesión: si todo se serializa, repetir con **dos cookies de sesión** distintas (lab time-sensitive).
4. No medir tiempos de respuesta entre endpoints antes del ataque multi-endpoint.
5. Esperar que funcione a la primera: en labs como "partial construction" hacen falta varias rondas con usuarios distintos.
6. Olvidar que los parámetros "vacíos" tienen formas alternativas (`token[]=`) cuando el framework no valida tipos.

**Defensor (falsas soluciones):**
1. "El botón ya está deshabilitado en el front-end" — el front no controla la concurrencia; todo se replica en Burp.
2. Token de un solo uso verificado **después** de usarlo: la ventana sigue abierta.
3. Contadores incrementados tras la validación (rate limit inútil contra el gate).
4. Confianza en que "las sesiones están bloqueadas": el lockeo serializa tu sesión, no la de los atacantes con N cuentas.
5. Validar y escribir en dos transacciones distintas.

## Cheatsheet

| Escenario | Herramienta | Truco clave |
|---|---|---|
| Overrun (cupón, límite) | Repeater → group → parallel | 30–50 repeticiones |
| Rate limit bypass | Turbo Intruder `race-single-packet-attack.py` | HTTP/2 single packet |
| Multi-endpoint | Repeater group con 2 peticiones distintas | Medir latencias primero |
| Token de un solo uso | Turbo Intruder gate | N confirmaciones encoladas |
| Serialización por sesión | Burp + 2 cookies de sesión | Paralelo desde sesiones distintas |
| Construcción parcial | Gate + `token[]=` (bypass de validación) | Muchas rondas, cambiar usuario |

| Señal | Lectura |
|---|---|
| Funciona en secuencial, no en paralelo | Lockeo de sesión o worker — replantear |
| 200 en todas pero un solo efecto | Serialización efectiva: más peticiones/rondas |
| Estados intermedios en respuestas | Ventana TOCTOU confirmada |
| Un `302` entre muchos `200` | Éxito puntual de fuerza bruta por carrera |

## Laboratorios y referencias

- PortSwigger Academy — Race conditions (todos los labs de arriba): https://portswigger.net/web-security/race-conditions
- Turbo Intruder: https://portswigger.net/turbo-intruder
- Bambdas (ProbeForRaceCondition): https://github.com/PortSwigger/bambdas
- PortSwigger Research — Advanced race conditions (single-packet attack): https://portswigger.net/research/racing-against-the-clock
- OWASP — Time-of-check to time-of-use: https://owasp.org/www-community/attacks/TOCTOU

## Relacionadas

- [[Abuso de Subida de Archivos]] — TOCTOU clásico de archivo temporal (lab de carrera incluido)
- [[Laboratorios de Vulnerabilidades Lógicas y de Autenticación]] — el terreno lógico donde viven las carreras
- [[Session Puzzling Session Fixation y Session Variable Overloading]] — manipulación del estado de sesión compartido
- [[Cross-Site Request Forgery (CSRF)]] — combinable: una carrera disparada desde el navegador de la víctima
