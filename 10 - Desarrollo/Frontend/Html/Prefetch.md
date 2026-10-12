---
aliases: ["rel=prefetch", "Prefetching"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Preload]]"
  - "[[Lazy Loading]]"
  - "[[Html]]"
  - "[[Etiquetas Meta]]"
  - "[[Formularios]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Attributes/rel/prefetch"
---

# Prefetch

**Prefetch** es una *hint* (pista) al navegador para que **descargue durante su tiempo libre un recurso que probablemente se usará en una futura navegación**. A diferencia del [[Preload]] —que anticipa la página actual—, el prefetch apunta al **siguiente paso del usuario**: la página que casi seguro va a abrir después.

> [!info] La idea en una frase
> Preload = "voy a necesitar esto **ahora**". Prefetch = "voy a necesitar esto **después**, si hay tiempo".

## Cómo se declara (forma clásica)

Un `<link>` con `rel="prefetch"` en el `<head>`:

```html
<link rel="prefetch" href="nosotros.html">
```

También funciona con cualquier recurso de una futura página:

```html
<link rel="prefetch" href="nosotros.html">
<link rel="prefetch" href="css/nosotros.css">
<link rel="prefetch" href="img/nosotros-hero.webp">
```

Comportamiento clave:

- El navegador lo descarga con **prioridad baja** (la más baja de la cola, "Lowest" en Chromium), normalmente cuando la página actual ya está ociosa (tras o casi tras el `load`).
- El recurso va a la **caché HTTP**: cuando el usuario navega efectivamente a esa página, se sirve desde caché y la navegación es casi instantánea.
- La caché de prefetch es **breve** (del orden de minutos en Chromium): si el usuario navega mucho después, el recurso se re-descarga.
- Si el usuario nunca llega, la descarga fue desperdicio — el único riesgo real de esta técnica.
- `as` es opcional en prefetch (a diferencia de preload); por defecto se asume `document`. Para recursos no-documento (`as="fetch"`, `as="style"`, `as="image"`) declararlo mejora la clasificación.

### Soporte de `rel="prefetch"` por navegador (forma clásica)

| Navegador | Soporte |
|---|---|
| Chromium (Chrome, Edge) | ✔ (y es donde mejor se documenta la prioridad) |
| Firefox | ✔ — descarga el recurso en background y lo deja en caché |
| **Safari** | **✖ nunca lo soportó** — el `<link rel="prefetch">` se ignora en silencio |

> [!warning] Safari ignorá el prefetch clásico
> En iPhone/iPad el prefetch HTML simple no hace nada. La estrategia multi-motor moderna es la **Speculation Rules API** (con soporte que va creciendo) más el prefetch *nativo* de Safari (su heurística interna de seguir enlaces, no controlable). Nunca hagas que una mejora crítica dependa de que el prefetch ocurra: es un *bonus*, no una base.

## ¿Cómo saber qué prefetchear?

La intuición ("precargo lo que creo que va a visitar") hay que sustentarla con datos:

- **Analítica de navegación**: en Google Analytics (GA4) o Google Search Console revisá las páginas más visitadas y los recorridos habituales (qué página sigue a la que está el usuario). Las métricas de flujo ("página siguiente más frecuente") son literalmente la lista de candidatos.
- **Flujos de usuario conocidos**: en un e-commerce, después del carrito casi siempre viene el checkout; en una app de eventos, después del listado viene el detalle.
- **Enlaces dominantes**: si el 80% del tráfico entra al home y de ahí va a "Servicios", prefetch de "Servicios" desde el home.
- **Búsqueda interna**: lo que la gente busca (registros del buscador del sitio) son páginas candidatas de destino.
- **Medí el resultado**: en DevTools → Network, las peticiones anticipadas aparecen con pestaña *(prefetch)* o initiator correspondiente; en GA, comprobá que las páginas prefetcheadas mueven su *Tiempo de carga* tras la implementación.

> [!warning] No prefetchees a ciegas
> Cada prefetch compite por ancho de banda con los recursos de la página actual (especialmente en móvil y redes lentas). Prefetchear "por las dudas" varias páginas puede **empeorar** la experiencia que intentás mejorar. Medí, elegí una o dos páginas, y revisá el efecto.

## Modelo de coste: cuándo el prefetch juega a favor y cuándo en contra

| Factor | Efecto |
|---|---|
| Ancho de banda | Descarga en prioridad mínima, pero **son bytes reales** en la misma conexión; en 3G compiten con imágenes del fold |
| CPU/RAM | Cachear un documento es barato; **cachear un prerender completo no** (se ejecuta JS, se pinta) |
| Probabilidad de uso | Si P(uso) < ~50%, el desperdicio esperado supera el beneficio en UX percibida |
| Momento | Después del `load`, cuando el presupuesto de red está libre; nunca durante el critical rendering path |
| Eviction | Caché de prefetch de vida corta: navegar tarde no aprovecha |
| Datos del usuario | Con redes medidas/`Save-Data`, el ahorro de no anticipar suele ser preferible |

## La forma moderna: Speculation Rules API

Desde 2023-2024, el estándar que reemplaza a `rel="prefetch"` y `rel="prerender"` es la **Speculation Rules API**: un JSON declarativo con mucho más control (patrones de URL, cuándo disparar, prefetch vs prerender):

```html
<script type="speculationrules">
{
  "prefetch": [
    {
      "where": { "href_matches": "/*" },
      "eagerness": "moderate"
    }
  ]
}
</script>
```

### Anatomía del JSON

| Clave | Valores | Significado |
|---|---|---|
| Bloques raíz: `"prefetch"` / `"prerender"` | arrays de reglas | Qué hacer: solo descargar bytes (`prefetch`) o montar la página entera con ejecución (`prerender`) |
| `where` | `{ "href_matches": "/ruta/*" }` | Patrón de URLs (glob; `"/*"` = todos los enlaces internos del documento) |
| `where` | `{ "selector_matches": "a.premium" }` | Solo enlaces cuyo `<a>` coincida con el selector |
| `where` | `{ "document_has_focus": true }` | Solo si el documento tiene foco (evita anticipar en pestañas en background) |
| `where` | `{ "and": [...] }`, `{ "not": {...} }` | Combinadores para refinar los patrones |
| `eagerness` | `"conservative"` | Solo al **interactuar con el enlace concreto**: hover, focus, tap |
| `eagerness` | `"moderate"` | Tras la carga de la página, en cuanto hay interacción del usuario (o unos segundos después) — balance por defecto recomendado |
| `eagerness` | `"eager"` (alias histórico: `"immediate"`) | Tan pronto como se pueda — agresivo |
| `eagerness` | número (ms) | Retardo tras la carga antes de anticipar |

Ejemplo de prerender con criterio fino:

```html
<script type="speculationrules">
{
  "prerender": [
    {
      "where": { "href_matches": "/checkout/*" },
      "eagerness": "moderate"
    }
  ]
}
</script>
```

Ejemplo con exclusiones:

```html
<script type="speculationrules">
{
  "prefetch": [
    {
      "where": {
        "and": [
          { "href_matches": "/*" },
          { "not": { "href_matches": "/admin/*" } },
          { "not": { "selector_matches": "a[data-no-prefetch]" } }
        ]
      },
      "eagerness": "conservative"
    }
  ]
}
</script>
```

### Reglas por cabecera HTTP

Las mismas reglas pueden servirse por cabecera (útil sin tocar el HTML, p. ej. desde el servidor o CDN):

```http
Speculation-Rules: "/js/rules.json"
```

- Aplica a la página que trae la cabecera; el archivo JSON debe estar en el mismo origen (o traer cabecera `Cross-Origin-Resource-Policy` adecuada si es cross-origin).
- Ventaja: se centraliza para todo el sitio sin duplicar el `<script>` en cada plantilla.

### Guardrails de prerender (Chromium)

`prerender` carga la página **completa con ejecución de JS**: peticiones de red, analítica, ads, workers. Por eso el motor pone límites:

- **Same-origin** por defecto; no se prerenderan páginas con flujos que requieren interacción (permisos, pagos) hasta que se resuelven.
- **Un prerender a la vez** por defecto (el spec permite configurar más, pero el límite práctico es 1): si el usuario navega a otra parte, se descarta.
- Si la página prerenderizada no se usa en un tiempo razonable, se libera (coste controlado, pero no gratis).
- Efecto colateral: analítica y ads de la página prerenderizada **pueden ejecutarse aunque el usuario nunca llegue** — medir con `Sec-Purpose: prefetch;prerender` del lado del servidor y filtrar.

> [!info] Soporte
> Las Speculation Rules se implementaron primero en Chromium; Firefox y Safari avanzan por su cuenta (implementación en curso). El enfoque progresivo es empezar con ellas donde haya soporte — para el resto de navegadores simplemente no ocurre nada y se comportan como antes. Se puede declarar igualmente por cabecera: `Speculation-Rules: /js/rules.json`.

## Cómo lo detecta el servidor

Las peticiones de prefetch son peticiones HTTP normales con semántica especial, que el servidor puede identificar:

```http
GET /nosotros.html HTTP/1.1
Sec-Purpose: prefetch
```

- En prerender, la cabecera combina propósitos: `Sec-Purpose: prefetch;prerender`.
- Cabecera legacy que todavía emiten algunos flujos: `Purpose: prefetch` (algunos CDNs y CDNs móviles la usaban para transcoding).
- Chromium **limita el acceso desde JS** a `Sec-Purpose` (presupuesto de privacidad/fingerprinting), pero del lado del servidor se lee sin problema.

También puede declararse por cabecera HTTP en lugar de etiqueta HTML:

```http
Link: </nosotros.html>; rel=prefetch
```

Esa semántica permite al servidor distinguirlas de una navegación real: por ejemplo, no contarlas como tráfico en analítica, servirlas con caché agresiva, o no generar HTML pesado para bots de prefetch. Ver [[Etiquetas Meta]] para más configuración del lado del documento.

## Preload vs Prefetch vs Prerender

| | **Preload** | **Prefetch** | **Prerender** |
|---|---|---|---|
| `rel` | `preload` | `prefetch` | `prerender` (legacy) |
| ¿Para qué página? | La **actual** | Una **futura** | Una futura (completa) |
| Prioridad | **Alta** | Baja (tiempo ocioso) | Baja-media |
| ¿Ejecuta scripts? | No (es solo descarga) | No (es solo descarga) | **Sí**: carga, ejecuta JS, dispara peticiones |
| Riesgo | Desperdicio si no se usa | Desperdicio si no se navega | Muy costoso si no se navega |
| Ejemplo típico | Fuente crítica, hero LCP | Siguiente página de un flujo | Página siguiente en sitios con recorridos fijos |

El preload completo vive en su propia nota: [[Preload]]. El lazy loading es el extremo opuesto (diferir en lugar de anticipar): [[Lazy Loading]].

### La familia completa de hints de carga

| Hint | Resuelve | Coste | Cuándo |
|---|---|---|---|
| `dns-prefetch` | Solo resolución DNS de un origen | Muy bajo (una query) | Terceros heredados; en Chromium limitado por defecto para hosts de terceros — preferir `preconnect` |
| `preconnect` | DNS + TCP + TLS listos | Bajo (100–300 ms ahorrados luego) | Origen del que *seguro* vendrán recursos (fuentes, CDN de imágenes) |
| `prefetch` | **Bytes** de una futura navegación | Medio (ancho de banda real) | Siguiente página probable del flujo |
| `preload` | **Bytes** de la página actual, prioridad alta | Alto si no se usa | Recurso crítico descubierto tarde — ver [[Preload]] |
| `prerender` | Página futura **entera, ejecutada** | Muy alto (red + CPU + RAM) | Flujos con destino casi seguro (checkout) |

## Interacción con bfcache (caché de ida y vuelta)

El navegador mantiene páginas completas en memoria al navegar «atrás» (*back/forward cache*). Eso cambia el cálculo del prefetch:

- Si la página destino está en bfcache, la navegación atrás **ya es instantánea**: el prefetch no aportó nada ahí.
- El prefetch brilla cuando el usuario llega por un **enlace nuevo** (no por «atrás»): home → detalle, card → página de producto. Esos destinos no están en bfcache.
- Por eso el candidato ideal es el **enlace interno más clickeado de la página actual**, no «la página que el usuario visitó antes».
- Los motores pueden descartar bfcache por cabeceras (`Cache-Control: no-store`): en sitios así, el prefetch gana peso porque cada vuelta atrás re-descarga.

## Errores comunes

1. **Prefetchear el sitio entero** (`href_matches: "/*"` + `eagerness: "eager"`): convierte el tiempo ocioso en saturación; el usuario nota *peor* rendimiento. Usá `conservative`/`moderate`.
2. **Usar prefetch para recursos de la página actual**: esa es la semántica de [[Preload]]; con prefetch la prioridad es mínima y llega tarde.
3. **No medir el destino**: sin analítica previa, el prefetch es una apuesta. Si la página candidata no concentra navegaciones, se desperdicia.
4. **Ignorar Safari**: el prefetch clásico no existe ahí; no prometas "carga instantánea" en iPhone basándote solo en esto.
5. **Doble conteo de analítica**: las páginas prefetcheadas/prerenderizadas pueden generar hits sin visita real; filtrar por `Sec-Purpose` en el servidor o marcar con `document_has_focus`.
6. **Confundir con caché de Service Worker**: el prefetch usa la caché HTTP estándar y no sobrevive a la recarga; un Service Worker puede dar persistencia y control total — eso ya es otra capa.
7. **Olvidar `document_has_focus`**: con `"/*"` y pestañas en background se anticipan páginas que nadie mira, quemando datos.

## Accesibilidad

- El prefetch no afecta el árbol de accesibilidad, pero su *eagerness* interactúa bien con usuarios de teclado: `conservative` dispara justo en el `focus` por teclado — la anticipación alineada con la intención real.
- No dependas del prefetch para contenido que deba estar disponible offline o inmediatamente: eso es trabajo de Service Worker, no de hints.

## Rendimiento

- Beneficio: navegación siguiente casi instantánea (hitos de TTFB/LCP de la *siguiente* página), porque el documento ya está en caché HTTP.
- Coste: si el usuario navega en 20s, la caché pudo expirar; si no navega, bytes perdidos. La métrica que gobierna es la **probabilidad de transición**.
- Conexiones lentas: en 2G/3G el prefetch compite con el render actual; en Chromium con `Save-Data` activo conviene abstenerse.
- Verificación: DevTools → Network → columna *Priority* (Lowest) y pestaña *(prefetch)*; en la pestaña *Priority* de la request, "Lowest" confirma la semántica.

## Compatibilidad

- `rel="prefetch"`: Chromium ✔, Firefox ✔, Safari ✖.
- Speculation Rules: Chromium ✔ (2023+), Firefox/Safari en despliegue progresivo — degradación silenciosa donde no hay soporte.
- `Sec-Purpose`: emisivo por Chromium y Firefox; el servidor no debe asumir que siempre está presente.

## Cheatsheet

| Mecanismo | Sintaxis | Página | Prioridad |
|---|---|---|---|
| Prefetch clásico | `<link rel="prefetch" href="…">` | Futura | Mínima |
| Prefetch por cabecera | `Link: </p>; rel=prefetch` | Futura | Mínima |
| Spec rules prefetch | `<script type="speculationrules">` + `"prefetch"` | Futura | Mínima |
| Spec rules prerender | igual + `"prerender"` | Futura, ejecutada | Baja-media |
| `eagerness` | `conservative` / `moderate` / `eager` / nº ms | — | Cuándo disparar |
| Cabecera del servidor | `Sec-Purpose: prefetch` / `…;prerender` | — | Filtrado en backend |

## Relacionadas

- [[Preload]] — lo contrario: recurso crítico de la página actual con prioridad alta
- [[Lazy Loading]] — diferir en lugar de anticipar
- [[Html]] — el elemento `<link>` y sus `rel`
- [[Etiquetas Meta]] — otras configuraciones del `<head>` con efecto en carga y SEO
- [[Formularios]] — flujos con destino casi seguro (checkout) como candidatos a prerender
