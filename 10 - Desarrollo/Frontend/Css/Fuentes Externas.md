---
tags:
  - desarrollo
  - frontend
  - fuentes-externas
tipo: concepto
estado: terminado
---

# Fuentes Externas

CSS permite usar cualquier tipografía, no solo las que el usuario tiene instaladas. Hay dos caminos: usar una **fuente del sistema** (ya presente en el equipo) o cargar una **fuente externa** desde un servidor. Esta nota cubre el pipeline completo: cómo el navegador elige una cara de fuente, cómo declararlas con `@font-face`, cómo servirlas (Google Fonts vs self-hosting) y cómo minimizar el costo de performance.

> [!info] El modelo mental: el navegador no "descarga una fuente"
> El navegador ejecuta un **algoritmo de matching**: 1) busca la familia en las caras disponibles (sistema + las declaradas con `@font-face`); 2) dentro de la familia, filtra por `font-style`, `font-weight`, `font-stretch` y `font-variation-settings`; 3) si la red no respondió a tiempo, decide qué hacer con `font-display`. Entender ese flujo explica casi todos los bugs: por qué "Roboto" no carga, por qué el bold sintético se ve mal, y por qué el texto desaparece 3 segundos.

## `font-family`: la pila de fuentes

`font-family` acepta una **lista** de fuentes separada por comas: el navegador usa la primera que encuentre disponible en el sistema y, si no hay ninguna, cae en el tipo genérico final:

```css
body {
  font-family: "Segoe UI", Roboto, Arial, Helvetica, sans-serif;
  /*           ↑ primero      ↑ si no está  ↑ fallbacks ↑ último recurso */
}
```

Con esto ya cambiamos a las fuentes predeterminadas que todos los navegadores/ sistemas tienen (Arial, Helvetica, Times...). Los tipos genéricos válidos son: `serif`, `sans-serif`, `mono`, `cursive`, `fantasy` y `system-ui`. Además existen genéricos **reservados al sistema** (mismos en todos los navegadores de una plataforma):

| Genérico | Para qué |
|---|---|
| `system-ui` | La fuente de UI del sistema (San Francisco, Segoe UI, Roboto…) — el fallback moderno por defecto |
| `ui-monospace` | Monoespaciada del sistema (SF Mono, Consolas, Menlo…) — mejor que `monospace` genérico |
| `ui-serif`, `ui-sans-serif`, `ui-rounded` | Variantes serif/sans/redondeadas del sistema |
| `emoji`, `math`, `fangsong` | Glifos matemáticos, emoji y kai china |

> [!tip] Aplicá la fuente en `body`
> Si vas a usar el mismo tipo de fuente en toda la web, declaralo en `body` y se propagará por **herencia** a todo el documento; solo marcarás las excepciones.

> [!warning] Los formularios no heredan la fuente
> Inputs, textareas, selects y botones **no** heredan `font-family` (usan la fuente del sistema por defecto del navegador). Sin `input, textarea, select, button { font-family: inherit }` en el CSS base, el formulario se ve de otro mundo. Es uno de los primeros pecados del reset (ver [[Normalize]]).

### Comillas y comas: sintaxis

- Nombres con espacios **entre comillas** (`"Times New Roman"`) o con guiones bajos (`Times_New_Roman`).
- La última entrada debe ser un **genérico** — es la red de seguridad cuando ninguna familia existe.
- El orden importa: el navegador toma la **primera disponible**; por eso el pila `"Segoe UI", Roboto, system-ui, sans-serif` usa la nativa de cada plataforma y solo cae a genérico si no hay nada.

### `font-weight`, `font-style` y síntesis

```css
h1 { font-weight: 700; }   /* 100–900, o keywords (bold=700, normal=400) */
cite { font-style: italic; }

/* Prohibir falsos negros/cursivas cuando la cara no existe */
.titulo {
  font-synthesis-weight: none;
  font-synthesis-style: none;
}
```

Si pedís `font-weight: 700` y la familia solo cargó la cara 400, el navegador **fabrica** un bold sintético (engorda los glifos): se ve plástico y rompe el espaciado. Con `font-synthesis: none` muestra la 400 real; con `font-display: swap` y cargas parciales, evitar pedir pesos que no serviste.

## `@font-face`: la ruta manual

Detrás de Google Fonts está el at-rule `@font-face`, que también podés usar con tus propios archivos. Es **el** mecanismo de las web fonts — todo lo demás es azúcar encima:

```css
@font-face {
  font-family: "MiFuente";
  src: url("../fuentes/mifuente.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

h1 {
  font-family: "MiFuente", sans-serif;
}
```

### Descriptores completos

| Descriptor | Valores | Para qué sirve |
|---|---|---|
| `font-family` | nombre | Cómo se invoca después en `font-family` |
| `src` | `url(...)`, `local(...)`, `format()`, `tech()` | De dónde sale el archivo; **WOFF2 primero** |
| `font-weight` | número, rango `100 900` | Qué pesos cubre esta cara (rango = fuente variable) |
| `font-style` | `normal`, `italic`, `oblique 10deg` | Cara romana vs cursiva |
| `font-stretch` | rango en % (`75% 125%`) | Anchos semi-condensados/condensados |
| `font-display` | `auto`, `block`, `swap`, `fallback`, `optional` | Qué pasa mientras carga (ver tabla abajo) |
| `unicode-range` | lista de rangos U+0041-005A | Esta cara cubre **solo** esos caracteres → subsetting por capas |
| `size-adjust` | % | Escala la cara para igualar al fallback (anti-salto de layout) |
| `ascent-override` / `descent-override` / `line-gap-override` | % | Fuerzan métricas de línea — fallback métricamente igualado |
| `font-feature-settings` | `"liga" 1`, `"kern" 1` | Activa features OpenType en la cara |
| `font-variation-settings` | `"wght" 400` | Fija ejes de fuente variable para esta cara |
| `font-named-instance` | nombre | Elige una instancia nombrada de la fuente variable |
| `local()` | nombre instalado | Usa la versión local si existe (¡con cuidado, difiere entre SO) |

```css
/* Cadena completa: local de referencia → woff2 → woff, con rango de variable font */
@font-face {
  font-family: "Inter";
  src:
    local("Inter Variable"),
    url("/f/inter.woff2") format("woff2-variations"),
    url("/f/inter.woff2") format("woff2");
  font-weight: 100 900;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153;
}
```

> [!warning] `local()` puede traer versiones distintas
> `local("Arial")` en Windows y en macOS son fuentes **diferentes** (métricas distintas). Solo sirve para variantes exactas de tu propia fuente (p. ej. `local("MiFuente Regular")`); con nombres de sistema, mejor saltá `local()` y serví el archivo siempre igual.

### Formatos de archivo

| Formato | Compresión | Soporte | Estado |
|---|---|---|---|
| **WOFF2** | Brotli, ~30% mejor que WOFF | Todos los modernos (2016+) | **Estándar actual** |
| WOFF | Zlib | Universal, incluido IE11 | Fallback legacy |
| OTF / TTF | Ninguna (lo sirve el servidor) | Universal | Solo como origen de conversión |
| EOT | — | Solo IE | Muerto |
| SVG font | — | iOS Safari antiguo | Muerto |

Regla: serví **WOFF2**; solo si necesitás IE9/10 (no lo necesitás) agregá WOFF. Ver [[Formatos Avif y Webp]] para el paralelo con imágenes.

### `font-display`: el contrato de la carga

Cada `@font-face` define qué ve el usuario mientras el archivo viaja:

| Valor | Antes de cargar | Espera | Tras el timeout | Uso |
|---|---|---|---|---|
| `auto` | comportamiento del navegador (~ como `block`) | ~3s | invisible | evitar |
| `block` | **texto invisible** (FOIT) | ~3s | fallback | logos, un titular puntual |
| `swap` | **fallback inmediato** (FOUT) | 0s | parpadeo al llegar | texto de lectura (default de Google Fonts) |
| `fallback` | fallback inmediato | ~100ms | fallback si llega tarde | buen balance |
| `optional` | fallback | ~100ms | fallback **definitivo** (la sesión no la usa) | cero parpadeo, offline-first |

```css
@font-face {
  font-family: "MiFuente";
  src: url("mifuente.woff2") format("woff2");
  font-display: swap; /* texto visible con fallback mientras carga */
}
```

**FOIT** (Flash of Invisible Text) = texto no visible hasta que la fuente carga o expira el timeout. **FOUT** (Flash of Unstyled Text) = fallback visible primero, salto a la fuente real. `swap` y `fallback` eligen FOUT a propósito: **texto siempre legible > tipografía perfecta**.

### Fallback métricamente igualado: matar el salto de layout

El parpadeo duele menos si el fallback **mide igual** que la fuente real (mismas alturas de línea, mismo ancho aproximado). La técnica moderna: un `@font-face` de fallback con overrides de métricas + `size-adjust`:

```css
/* 1. El fallback ajustado (nombre local, sin archivo) */
@font-face {
  font-family: "Inter Fallback";
  src: local("Arial");
  size-adjust: 107.4%;
  ascent-override: 90%;
  descent-override: 22.5%;
  line-gap-override: 0%;
}

/* 2. La fuente real, y la pila que la usa */
h1 {
  font-family: "Inter", "Inter Fallback", sans-serif;
}
```

Con esto, el swap de Arial→Inter mueve el texto poquísimo: layout shift ≈ 0. Herramientas como `fontaine` o `next/font` (Next.js) generan estos overrides automáticamente.

La propiedad nativa `font-size-adjust` (2023+) resuelve un ángulo similar escalando la fuente fallback para igualar la altura de x:

```css
h1 {
  font-family: "Inter", sans-serif;
  font-size-adjust: 0.52; /* altura-x / font-size de la fuente preferida */
}
```

## Fuentes variables: un archivo, todos los pesos

Las **variable fonts** empaquetan todos los pesos/anchos de una familia en **un solo archivo** con ejes interactivos registrados: `wght` (100–900), `wdth` (75–125%), `slnt`, `ital`, `opsz` (tamaño óptico), más ejes personalizados (`CRSV`, `MONO`, `CASL` en fuentes tipo Recursive).

```css
/* Una sola cara cubre todo el rango */
@font-face {
  font-family: "Inter";
  src: url("inter-var.woff2") format("woff2");
  font-weight: 100 900; /* rango: la cara es variable */
  font-display: swap;
}

/* Después se usa como siempre */
h1 { font-weight: 800; }
.nav { font-weight: 450; } /* pesos intermediarios reales, no interpolados */
```

| Fuentes estáticas | Fuentes variables |
|---|---|
| 1 archivo por peso (400, 500, 700 = 3 descargas) | 1 archivo para todo el rango |
| Peso intermediario = sintético | Cualquier peso real del eje |
| Cache por archivo | Cache única, reutilizada en todo el sitio |
| Menor si usás 2 pesos de 9 | Mayor si usás 2 pesos (pero casi nunca es el caso) |

> [!tip] Regla práctica
> Si el diseño usa ≥ 3 pesos, la variable font gana siempre. Serví un **subset estático instanciado** solo si el tráfico justifica recortar bytes (p. ej. `opsz` fijo en Material Symbols, ver [[Material Icons]]).

## Cargar fuentes externas: Google Fonts

Para fuentes que nadie tiene instalada hay que descargarlas de algún lado. La fuente más usada es [Google Fonts](https://fonts.google.com/), con miles de tipografías gratuitas. El flujo:

1. Buscá la fuente (por ejemplo **Roboto**) y elegí los pesos/estilos que necesites.

   ![[Pasted image 20250124162423.png]]

2. La página te da el `<link>` para el HTML y el `font-family` para el CSS.

   ![[Pasted image 20250124162505.png]]

3. Agregá el `<link>` en el `<head>` **antes de tu CSS** (para no tener conflictos de cascada):

```html
<head>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link
    href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap"
    rel="stylesheet"
  />
  <link rel="stylesheet" href="css/estilos.css" />
</head>
```

4. Y usala en el CSS:

```css
body {
  font-family: "Roboto", sans-serif;
}
```

### Anatomía de la URL `css2`

```
https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&family=Inter:ital,wght@0,400;1,700&display=swap
                                      ↑ familia       ↑ pesos       ↑ segunda familia con cursivas
```

| Parámetro | Ejemplo | Significado |
|---|---|---|
| `family=Nombre` | `Roboto` | Una familia por cada `family=` (URL-encodear espacios como `+`) |
| `:wght@400;700` | lista con `;` | Pesos estáticos pedidos |
| `:wght@100..900` | rango | Pide la **variable font** con ese eje abierto |
| `ital,wght@0,400;1,700` | ejes en lista | Cursiva + peso; los ejes van en orden alfabético, mayúsculas primero |
| `display=swap` | `swap` | **Siempre ponerlo**: sin él el default es `block` (texto invisible) |

El CSS que devuelve es una hoja con `@font-face` + `unicode-range` por **subsetting automático**: Google parte la familia en trozos (`latin`, `latin-ext`, `cyrillic`…) y el navegador descarga **solo los trozos con caracteres que la página usa**. Es subsetting gratis, del lado del servidor.

### Self-hosting: servirlo vos

Bajar la familia de Google Fonts (o de la fuente original) y servirla desde tu dominio:

```css
@font-face {
  font-family: "Roboto";
  src: url("/fonts/roboto-v30-latin.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC;
}
```

| Criterio | Google Fonts (CDN) | Self-hosting |
|---|---|---|
| Peticiones | 2 orígenes distintos (CSS + fuente) | 1 origen, HTTP/2 multiplexed |
| Cache | Depende de su CDN | Tus headers, immutable por hash de versión |
| Privacidad | El usuario contacta a Google (GDPR: sentencias alemanas 2022 contra hotlink) | Cero terceros |
| Subsetting | Automático por unicode-range | Manual: `pyftsubset` o `fonttools` |
| Mantenimiento | Cero | Actualizar versiones a mano |
| Offline/dev | Requiere red | Funciona en localhost sin red |

Veredicto 2026: para producción, **self-hosting WOFF2 subseteado** es el default recomendado; Google Fonts queda bien para prototipos. Si self-hosteás, subseteá por el alfabeto que usás (español = `latin` + `latin-ext`): `pyftsubset roboto.ttf --unicodes="U+0000-00FF,U+0100-017F" --flavor=woff2 --output-file=roboto.woff2`.

## Rendimiento: cada fuente externa es una ida y vuelta

> [!warning] Las fuentes externas cuestan tiempo de carga
> Cada fuente agrega una petición de red y puede provocar **FOIT/FOUT** (texto invisible o texto con fuente provisional parpadeando). Tres medidas obligatorias:
> - **`preconnect`** a `fonts.googleapis.com` y `fonts.gstatic.com` (abre la conexión TLS antes).
> - **`display=swap`** en la URL: muestra un fallback mientras carga la real.
> - Pedí **solo los pesos que uses** (400 y 700, no toda la familia): cada peso es un archivo aparte. Ver también [[Preload]].

Checklist de rendimiento completo:

| Acción | Por qué |
|---|---|
| 1 familia, 2–3 pesos (o 1 variable) | Cada archivo pesa decenas de KB y es render-blocking en el primer texto |
| `preconnect` a los 2 orígenes de Google | Ahorra el handshake TLS (~100–300 ms) |
| `display=swap` | El texto nunca desaparece |
| `preload` **solo** de la fuente crítica de la portada | `rel="preload" as="font" type="font/woff2" crossorigin` — ver [[Preload]] |
| Subsetting al alfabeto usado | −50% a −80% de peso típicos |
| Fallback métrico (`size-adjust`) | Layout shift ~ 0 en el swap |

> [!warning] `preload` sin `crossorigin` descarga dos veces
> Las fuentes siempre se fetchan en modo crossorigin (incluso same-origin). Un `<link rel="preload" as="font">` sin el atributo `crossorigin` provoca una **segunda** petición normal — el preload se desperdicia.

Cálculo de presupuesto: cada peso estático WOFF2 subseteado al latin ronda los 10–25 KB; una variable completa 40–80 KB. Un sitio con 2 familias × 2 pesos ≈ 60–100 KB comprimidos — razonable. Todo lo que se acerque a 300 KB de fuentes es señal de subsetear o recortar pesos.

## Accesibilidad

- **Tamaños en `rem`**: la fuente externa no debe romper el zoom ni el tamaño configurado por el usuario (WCAG 1.4.4 — ver [[Medidas CSS]]).
- **`font-display: swap` o `fallback`** en texto de lectura: contenido legible desde el primer frame, aunque la red falle.
- **Contraste del fallback**: el parpadeo de fuente no puede ocultar texto crítico; elegí fallbacks de la misma familia óptica.
- El usuario puede instalar **fuentes propias** (hojas de estilo del usuario) — no neutralices `font-family` con `!important` en elementos de lectura.
- Fuentes con `local()`: si la versión local del usuario es ilegible o distorsionada, tu diseño se rompe para esa persona; serví vuestro archivo.

## Errores comunes

1. **Familia con espacios sin comillas** en `font-family: Mi Fuente` → la declaración se descarta entera.
2. **Olvidar `display=swap`**: el default de Google es `block` → texto invisible hasta ~3 s en conexiones lentas.
3. **`preload` sin `crossorigin`** (o con `crossorigin` mal escrito): doble fetch, cero aceleración.
4. **Pedir 900 y declarar solo 400**: bold sintético feo. Pedí el peso real o usá la variable.
5. **Ruta de `url()` relativa al HTML**: como con las imágenes, la ruta de `@font-face` se resuelve **desde la hoja CSS** (ver [[Imágenes CSS]]).
6. **Mezclar Google Fonts + fuente local homónima**: `local()` no está en el snippet de Google, pero si lo agregás, Windows/macOS darán métricas distintas.
7. **Fuentes de terceros sin `preconnect`**: se paga el TLS en el momento crítico del FCP.
8. **Cargar fuentes completas del sistema operativo de diseño** (pega la TTF de 2 MB): subseteá siempre; una familia .ttf sin comprimir no es un recurso web.

## Cheatsheet — fuentes

| Concepto | Valores / formatos | Defecto | Nota |
|---|---|---|---|
| `font-family` | pila separada por comas + genérico final | la del agente | comillas si hay espacios |
| Genéricos | `serif`, `sans-serif`, `monospace`, `cursive`, `fantasy`, `system-ui`, `ui-*` | — | `system-ui` = fuente nativa de UI |
| `font-weight` | 100–900, `bold`, `normal` | `400` | rango `100 900` en @font-face = variable |
| `font-style` | `normal`, `italic`, `oblique` | `normal` | — |
| `font-synthesis` | `none`, `weight`, `style`, `small-caps`, `position` | todo permitido | `none` prohíbe falsos bold/cursivas |
| Formato WOFF2 | `format("woff2")` | — | estándar, Brotli |
| `font-display` | `auto`, `block`, `swap`, `fallback`, `optional` | `auto` (~block) | `swap` para texto |
| `unicode-range` | `U+0041-005A, U+0100-017F` | todo el set | subsetting en el navegador |
| `size-adjust` | % | `normal` | iguala fallback al peso real |
| `ascent/descent/line-gap-override` | % | métricas de la fuente | fallback métrico |
| FOIT / FOUT | — | — | invisible vs parpadeo; `swap` elige FOUT |
| Ejes variables | `wght`, `wdth`, `slnt`, `ital`, `opsz` | — | 1 archivo para toda la familia |
| Google Fonts | `css2?family=…:wght@…&display=swap` | `block` sin param | **poné `display=swap`** |
| Subsetting | `pyftsubset`, unicode-range | — | español: latin + latin-ext |

## Compatibilidad

`@font-face` es viejo (IE9+); WOFF2 es universal desde 2016; las fuentes variables, desde 2018 (Chrome 62, Firefox 62, Safari 11); `size-adjust`/overrides de métricas llegaron en 2020–2023 (Firefox y Chrome tempranos, Safari 16.4+); `font-size-adjust` con sintaxis de número se estabilizó en 2023–2024. Sin soporte, el navegador ignora el descriptor y el comportamiento degrada al `font-display` normal — nunca a ausencia de texto.

## Relacionadas

- [[Html]] — dónde va el `<link>` dentro del `<head>`
- [[Preload]] — acelerar la llegada de recursos críticos como fuentes
- [[Prefetch]] — el siguiente nivel de la especulación de red
- [[Imágenes CSS]] — el otro recurso externo típico de un diseño (y las mismas reglas de ruta relativa)
- [[Material Icons]] — fuentes usadas como iconos: ligaduras y ejes variables
- [[Medidas CSS]] — `em`/`rem` y por qué los tamaños deben ser relativos
- [[Responsive Web Design]] — peticiones de fuente por breakpoint: evitadas con variables
- [[Normalize]] — `font-family: inherit` en formularios, el primer fix
