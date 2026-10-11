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
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Attributes/rel/prefetch"
---

# Prefetch

**Prefetch** es una *hint* (pista) al navegador para que **descargue durante su tiempo libre un recurso que probablemente se usará en una futura navegación**. A diferencia del [[Preload]] —que anticipa la página actual—, el prefetch apunta al **siguiente paso del usuario**: la página que casi seguro va a abrir después.

> [!info] La idea en una frase
> Preload = "voy a necesitar esto **ahora**". Prefetch = "voy a necesitar esto **después**, si hay tiempo".

## Cómo se declara

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

- El navegador lo descarga con **prioridad baja**, normalmente cuando la página actual ya está ociosa (más allá del *critical rendering path*).
- El recurso va a la **caché HTTP**: cuando el usuario navega efectivamente a esa página, se sirve desde caché y la navegación es casi instantánea.
- Si el usuario nunca llega, la descarga fue desperdicio — el único riesgo real de esta técnica.

## ¿Cómo saber qué prefetchear?

La intuición ("precargo lo que creo que va a visitar") hay que sustentarla con datos:

- **Analítica de navegación**: en Google Analytics (GA4) o Google Search Console revisá las páginas más visitadas y los recorridos habituales (qué página sigue a la que está el usuario).
- **Flujos de usuario conocidos**: en un e-commerce, después del carrito casi siempre viene el checkout; en una app de eventos, después del listado viene el detalle.
- **Enlaces dominantes**: si el 80% del tráfico entra al home y de ahí va a "Servicios", prefetch de "Servicios" desde el home.

> [!warning] No prefetchees a ciegas
> Cada prefetch compite por ancho de banda con los recursos de la página actual (especialmente en móvil y redes lentas). Prefetchear "por las dudas" varias páginas puede **empeorar** la experiencia que intentás mejorar. Medí, elegí una o dos páginas, y revisá el efecto.

## Cómo lo detecta el servidor

Las peticiones de prefetch son peticiones HTTP normales con semántica especial, que el servidor puede identificar:

```http
GET /nosotros.html HTTP/1.1
Sec-Purpose: prefetch
```

También puede declararse por cabecera HTTP en lugar de etiqueta HTML:

```http
Link: </nosotros.html>; rel=prefetch
```

Esa semántica (`Sec-Purpose: prefetch`) permite al servidor distinguirlas de una navegación real: por ejemplo, no contarlas como tráfico en analítica, o servirlas desde caché con políticas propias. Ver [[Etiquetas Meta]] para más configuración del lado del documento.

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

- `where`: patrón de URLs a anticipar (acá, cualquier enlace del sitio — es decir, todos los enlaces internos se prefetchean cuando el usuario *interactúa* con ellos, según `eagerness`).
- `eagerness`: cuán agresivo es el anticipado — `conservative` (solo al hover/focus/touch), `moderate` (al interactuar), `eager` (de entrada), o un número en milisegundos de retardo.
- Existe también el bloque `"prerender"`, que precarga **la página completa con ejecución**: navegaciones instantáneas de verdad.

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

> [!info] Soporte
> Las Speculation Rules se implementan primero en Chromium; Firefox y Safari avanzan por su cuenta. El enfoque progresivo es empezar con ellas donde haya soporte — para el resto de navegadores simplemente no ocurre nada y se comportan como antes. Se puede declarar igualmente por cabecera: `Speculation-Rules: /js/rules.json`.

## Buenas prácticas

- **Una o dos páginas candidatas**, elegidas con datos de analítica, no el sitio entero.
- Anticipá desde la página en la que el usuario *está*, hacia el destino más probable de su flujo.
- En sitios con muchos enlaces internos, las Speculation Rules con `eagerness: "conservative"` dan lo mejor de ambos mundos: anticipan solo cuando el usuario ya muestra intención (hover/focus).
- No confundir con caché de Service Worker: el prefetch usa la caché HTTP estándar y no sobrevive a la recarga; un Service Worker puede dar persistencia y control total — eso ya es otra capa.
- Verificá en DevTools → Network: las peticiones prefetcheadas aparecen con la pestaña y el propósito identificables.

## Relacionadas

- [[Preload]] — lo contrario: recurso crítico de la página actual con prioridad alta
- [[Lazy Loading]] — diferir en lugar de anticipar
- [[Html]] — el elemento `<link>` y sus `rel`
- [[Etiquetas Meta]] — otras configuraciones del `<head>` con efecto en carga y SEO
