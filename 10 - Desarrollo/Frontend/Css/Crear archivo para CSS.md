---
aliases: ["Crear archivo para Css", "Crear archivo CSS", "Link a CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-05-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[CSS Introducción]]"
  - "[[Estilos para escribir CSS]]"
  - "[[Preload]]"
  - "[[Qué es Sass]]"
  - "[[Sintaxis de Sass]]"
  - "[[Normalize]]"
fuente: "https://developer.mozilla.org/es/docs/Learn_web_development/Core/Styling_basics/Getting_started"
---

# Crear archivo para CSS

El flujo básico: una **carpeta** que contenga los estilos, un **archivo `.css`** dentro, y un **enlace** desde el HTML. Sin ese último paso, el navegador no sabe que el archivo existe. Pero la etiqueta `<link>` tiene más control del que parece: prioridad de descarga, condición por media, aplicado diferido y hasta hojas "alternativas" para el usuario. Esta nota desarma cómo el navegador **descubre, descarga y aplica** tus hojas de estilo.

## 1. Estructura de carpetas

Creá una carpeta contenedora llamada `css`:

![[Pasted image 20250124160634.png]]

Dentro de ella, el archivo de estilos — convención habitual: `styles.css` o `style.css`:

![[Pasted image 20250124160740.png]]

Un proyecto mínimo queda así:

```
proyecto/
├── index.html
└── css/
    └── styles.css
```

Convenciones útiles:

- **Un solo archivo maestro** (`styles.css`) que importa el resto, o varios `<link>` — ver "Varios archivos CSS" más abajo y [[Estilos para escribir CSS]] para la organización.
- `index.html` y `css/` al mismo nivel: la ruta `css/styles.css` es relativa al HTML.
- Los archivos viven donde los sirva el servidor; en dev, cualquier servidor estático (`npx serve`, Live Server de VS Code) sirve.

> [!info] Rutas relativas: contra el documento, no contra la URL
> El `href` del `<link>` se resuelve contra la **URL del HTML**, no contra la del archivo. Si el HTML está en `/blog/entrada.html`, un `href="css/styles.css"` busca `/blog/css/styles.css`. Rutas que empiezan con `/` son desde la raíz del sitio; `../` sube un nivel. (Ojo: esto es al revés que dentro del CSS, donde `url()` es relativo al CSS — ver [[Anatomía de CSS]].)

## 2. Enlazar el archivo en el HTML

Para que los estilos se apliquen, el `<head>` del HTML necesita la etiqueta `<link>`:

```html
<link href="css/styles.css" rel="stylesheet">
```

- `rel="stylesheet"` le dice al navegador *qué es* el recurso (una hoja de estilo).
- El `href` es una **ruta relativa** al archivo HTML: `css/styles.css` significa "una carpeta `css` al mismo nivel que este HTML".
- **Sin `type`**: el atributo `type="text/css"` se escribía en HTML4/XHTML y hoy es redundante (HTML5 asume CSS en `.css` y `text/css` por MIME). En HTML antiguo había que envolver en CDATA; nada de eso aplica hoy.
- **Con `charset`**: `charset="utf-8"` es válido pero casi innecesario si el servidor manda `Content-Type: text/css; charset=utf-8` (lo hacen todos).

### Atributos del `<link rel="stylesheet">`

| Atributo | Ejemplo | Efecto |
|---|---|---|
| `media` | `media="screen and (min-width: 48rem)"` | la hoja aplica solo si matchea la condición |
| `media="print"` | hoja de impresión | se aplica solo al imprimir |
| `disabled` | `disabled` | hoja cargada pero **no aplicada** (el usuario/JS puede activarla) |
| `title` | `title="Alto contraste"` | convierte la hoja en **alternativa nombrada** (ver abajo) |

```html
<!-- Hoja condicional: desktop-only desde el HTML mismo -->
<link href="css/desktop.css" rel="stylesheet" media="(min-width: 60rem)">
```

El `media` del `<link>` es el **mecanismo barato** para CSS condicional: la hoja se descarga igual (es red, no CPU), pero el navegador solo la aplica — y lo re-evalúa al redimensionar. Para condicionar **dentro** de una hoja ya descargada, mejor `@media` adentro del CSS ([[Responsive Web Design]]).

> [!tip] Hojas alternativas (estándar, casi sin uso)
> Varios `<link rel="stylesheet">` con `title` y **sin** uno "sin título" crean un juego de hojas alternativas que el usuario elige desde el navegador. Sin `title`, todas se aplican a la vez. Es el atributo que hace la distinción — útil para modos de usuario (contraste, tema) sin JS.

## 3. Optimización: `preload`

Para que la hoja de estilos se descargue antes de que el HTML la pida, se puede precargar:

```html
<link rel="preload" href="css/styles.css" as="style">
<link href="css/styles.css" rel="stylesheet">
```

El `preload` dispara la descarga en alta prioridad al inicio de la página; el `<link rel="stylesheet">` posterior la aplica. Se mantiene **ambas líneas**: el preload acelera, pero solo el stylesheet aplica los estilos. Ver [[Preload]] para la estrategia completa de precargas.

> [!tip] Opción moderna: `onload` para CSS no crítico
> Para CSS que no debe bloquear el primer pintado (por ejemplo, hojas de terceros):

```html
<link rel="preload" href="css/secundaria.css" as="style" onload="this.rel='stylesheet'">
```

Dado que `onload` es inline, en sitios con CSP estricta necesitás nonce o hash en esa línea. La alternativa nativa sin JS: `rel="stylesheet" media="print" onload="this.media='all'"` — mismo truco, la hoja descarga con prioridad baja (como print) y al cargar pasa a all.

## 4. Varios archivos CSS

Nada impide enlazar varios archivos; el orden importa porque la cascada resuelve conflictos:

```html
<link href="css/normalize.css" rel="stylesheet">  <!-- 1º: reset/base -->
<link href="css/styles.css" rel="stylesheet">     <!-- 2º: estilos propios -->
```

Reglas del orden:

- Dos hojas con la misma especificidad: **gana la que aparece después** en el documento. Los `<link>` se ordenan como si fueran un solo stylesheet concatenado en ese orden — ver [[Especificidad]].
- Los `@layer` reorganizan este orden por capas sin depender de la posición de las etiquetas — ver [[Anatomía de CSS]].
- En proyectos grandes este trabajo lo automatiza el bundler (Vite, webpack) y en preprocesadores los archivos se dividen con `@use` — ver [[Sintaxis de Sass]] y [[Qué es Sass]]. Para decidir cómo organizar los estilos entre archivos, ver [[Estilos para escribir CSS]].

> [!warning] Evitá `@import` para encadenar CSS
> `@import url("otra.css");` dentro de un `.css` funciona pero **serializa la descarga**: el navegador descarga la hoja A, ve el import y recién ahí pide la B — cadena de waterfalls que cuesta RTTs. Varios `<link>` (o un bundler que los fusione) descargan **en paralelo**. El único `@import` aceptable hoy es el de `@import url() layer(...)` o para hojas condicionales legacy.

## 5. CSS inline vs externo vs `style` attribute

| Método | Ejemplo | Cuándo |
|---|---|---|
| Hoja externa | `<link rel="stylesheet" href="…">` | **default**: cacheable, mantenible |
| `<style>` en el head | `<style>.a{color:red}</style>` | **CSS crítico** (lo mínimo para el primer pintado) |
| Atributo `style` | `<div style="…">` | valores puntuales; gana en cascada, no escalable |

```html
<!-- Patrón "critical CSS": inlinás lo esencial, el resto async -->
<style>
  /* header, hero: lo que se ve sin scroll */
</style>
<link rel="preload" href="css/styles.css" as="style"
      onload="this.onload=null;this.rel='stylesheet'">
```

El atributo `style=""` tiene especificidad máxima (aunque por debajo de `!important`) — útil para JS que mide y escribe (`element.style.color`), malo para diseño ([[Especificidad]]).

## 6. Qué hace el navegador con el `<link>` (el pipeline real)

El ciclo de vida de una hoja de estilo tiene tres fases; entenderlas explica cada "¿por qué tarda en pintar?":

1. **Descubrimiento**: el *preload scanner* (parser especulativo que corre en paralelo al parser principal) barre el HTML y registra cada `<link href>` que ve — incluso si el parser principal todavía está en las primeras líneas. Por eso mover un `<link>` "más abajo" no lo hace invisible: lo hace **tarde**. Los `<style>` inline también se descubren aquí.
2. **Descarga**: cada hoja descubierta entra a la cola de red con su prioridad (CSS alto; con `preload as="style"` o `media="print"` diferido, más bajo).
3. **Aplicación**: el CSS se parsea, se inserta al CSSOM y la cascada resuelve qué reglas viven dónde. Hasta que una hoja que matchea (media="all") terminó de aplicarse, **no hay primer pintado** de nada dependiente de ella.

```
HTML → preload scanner → [network: css1, css2] → CSSOM → recorrido de estilos → layout → paint → composite
                                ↑ render-blocking hasta aquí (para el primer pintado)
```

> [!info] "Render-blocking" no es "todo o nada"
> Solo bloquea el primer pintado **la hoja que aplica a ese contexto**. Una hoja con `media="print"` o condición no matcheada no bloquea nada (se aplica/ignora después). De ahí el truco `media="print" onload`: descarga con prioridad de print y recién al cargarse se re-evalúa como `all`.

## 7. Producción: de `styles.scss` al CSS servido

En proyectos reales el `.css` que enlazás rara vez se escribe a mano — se **genera**:

| Herramienta | Entrada | Salida | Añade |
|---|---|---|---|
| Sass CLI / Vite | `.scss` con `@use` | `.css` compilado | imports, variables, mixins ([[Sintaxis de Sass]]) |
| Bundler (Vite, webpack) | múltiples CSS + JS imports | chunks con **hash en el nombre** | minificación, dedupe, code-splitting |
| PostCSS | `.css` | `.css` transformado | autoprefixer, minificación |
| Tailwind CLI | clases en HTML/JS | `.css` con solo lo usado | utilidades generadas ([[TailwindCSS - Sintaxis Básica]]) |

Salida típica de un build:

```
dist/
├── index.html          ← con <link href="/assets/styles.3f9a1c.css" …>
└── assets/
    └── styles.3f9a1c.css  ← hash de contenido: cambia el CSS, cambia el nombre
```

El hash en el nombre resuelve el problema de caché de arriba sin cabeceras raras: el navegador pide una URL **nueva** en cada deploy y la caché vieja queda huérfana (los assets con hash se cachean `immutable`). En dev, Vite sirve los módulos por separado con HMR; en build, todo se fusiona y minifica.

```html
<!-- Lo que genera el build en el <head> del HTML: un solo link con hash -->
<link rel="stylesheet" href="/assets/styles.3f9a1c.css">
```

## 8. Verificar que funciona (y depurar cuando no)

Flujo de diagnóstico en DevTools:

1. **Pestaña Network**: ¿la petición del `.css` aparece? ¿con `200`? Si es `404`, es ruta (mayúsculas, typo, ubicación).
2. **Content-Type**: debe ser `text/css`. Si el servidor manda `text/plain` u `application/octet-stream`, los navegadores la rechazan por MIME sniffing.
3. **Pestaña Sources/CSS o *Computed***: ¿la regla aparece en el CSSOM? Si no, la hoja no cargó. Si aparece pero tachada, error de sintaxis ([[Anatomía de CSS]]).
4. **Elements → Styles**: ¿la regla matchea? ¿quién la anula? ¿cuál es el origen (stylesheet vs inline vs UA)?
5. **Disable JS / cache off**: verificar con caché limpio; `Ctrl/Cmd+Shift+R` fuerza recarga sin caché.

```bash
# sanity check rápido desde terminal (ruta y tipo de contenido)
curl -sI http://localhost:5173/css/styles.css | grep -iE 'HTTP|content-type'
```

## Accesibilidad y rendimiento de la carga

- **El CSS es render-blocking por defecto**: el navegador no pinta nada hasta tener el CSS que matchea (pintar y luego repintar cuesta más que esperar). Por eso el CSS crítico va inline y lo demás async — el objetivo es *First Paint* con estilo, no cero CSS.
- **Autoflash temible (FOUC)**: si la hoja principal carga async y un instante no hay estilos, se ve el HTML "crudo". Mitigación: inline del CSS crítico + preload del resto.
- El orden de los `<link>` importa también para el **prioridad de red**: el navegador prioriza CSS descubiertos temprano; dejar CSS al final del `<body>` retrasa su descubrimiento (el parser lo encuentra tarde).
- `media="print"` como diferidor: descarga con prioridad baja, sin bloquear el primer pintado — la técnica más limpia sin JS (ver arriba).
- **Caché**: los `.css` se cachean por URL — los nombres versionados (`styles.v2.css`) o query (`?v=2`) rompen caché entre deploys; mejor aún, hash en el nombre (`styles.3f9a.css`) que hacen los bundlers.
- **CORS**: un CSS servido desde otro origen (CDN) se pide en modo CORS cuando va con `crossorigin`; si no, el navegador lo pide en modo no-CORS y aplica igual pero sin leer el contenido. Fuentes en CDN **sí** necesitan `crossorigin` explícito o fallan silenciosamente — ver [[Fuentes Externas]].
- `preconnect`/`preload` de fuentes externas evita el FOUT/FOUC de tipografía — ver [[Fuentes Externas]] y [[Preload]].

## Errores comunes

1. **Olvidar `rel="stylesheet"`** → el navegador no carga los estilos.
2. **Ruta mal escrita o mayúsculas distintas** → en servidores Linux las rutas distinguen mayúsculas (`CSS/styles.css` ≠ `css/styles.css`).
3. **Colocar el `<link>` fuera del `<head>`** funciona, pero retrasa el descubrimiento del CSS.
4. **`href` resuelto contra la URL del HTML**: rutas que funcionan en `/` rompen en `/blog/` — revisá rutas relativas con subcarpetas.
5. **`@import` al inicio de un archivo** como única forma de composición → waterfalls de descarga; usar `<link>` o bundler.
6. **MIME type incorrecto** del servidor → la hoja se rechaza silenciosamente.
7. **Esperar que `type="text/css"` arregle algo**: no carga nada si el `href` está mal; el atributo es redundante.
8. **Caché vieja tras deploy**: el navegador cachea agresivamente el CSS; versionar URLs.
9. **`disabled` vs `media` confundidos**: `disabled` oculta la hoja para siempre (hasta que algo la active); `media` la condiciona dinámicamente.

## Cheatsheet — cómo se enlaza CSS

| Pieza | Sintaxis | Rol |
|---|---|---|
| Enlace básico | `<link rel="stylesheet" href="ruta.css">` | aplica la hoja |
| Condicionada | `+ media="(…)"` | aplica si matchea |
| Diferida | `media="print" onload="this.media='all'"` | no bloquea el primer pintado |
| Alternativa | `rel="stylesheet" title="Tema"` | elegible por el usuario |
| Deshabilitada | `+ disabled` | cargada, no aplicada |
| Precarga | `<link rel="preload" href="…" as="style">` | descarga en alta prioridad |
| Estilo inline | `<style>…</style>` | CSS crítico |
| En línea | `style="…"` | valor puntual (máxima especificidad) |
| Import (evitar) | `@import url("…");` dentro de un CSS | serializa descargas |

| Atributo del `<link>` | Valores típicos | Nota |
|---|---|---|
| `rel` | `stylesheet`, `preload`, `alternate stylesheet` | define el rol del recurso |
| `href` | ruta relativa/absoluta | relativa al documento |
| `media` | condición media (`screen`, `(min-width: …)`) | re-evaluada en resize |
| `as` | `style` (en preload) | obligatorio en preload |
| `disabled` | booleano | hoja no aplicada |
| `title` | texto | la vuelve "alternativa" |
| `onload` | handler | truco de aplicación diferida |

## Relacionadas

- [[Html]] — estructura del documento donde se enlaza el CSS
- [[CSS Introducción]] — qué hace el CSS una vez enlazado
- [[Anatomía de CSS]] — qué pasa dentro del archivo una vez cargado
- [[Normalize]] — qué poner en el primer archivo CSS del proyecto
- [[Estilos para escribir CSS]] — cómo dividir el CSS entre varios archivos
- [[Qué es Sass]] — alternativa con preprocesador al CSS plano
- [[Sintaxis de Sass]] — `@use`/@import del preprocesador
- [[Preload]] — estrategia de precarga de recursos
- [[Responsive Web Design]] — condicionar estilos con media queries
- [[Fuentes Externas]] — el caso del `crossorigin` y las fuentes desde CDN
