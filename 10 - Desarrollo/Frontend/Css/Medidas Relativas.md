---
tags:
  - desarrollo
  - frontend
  - medidas-relativas
tipo: concepto
estado: terminado
---

# Medidas Relativas

Las medidas relativas se calculan en función de algo que **no es el píxel fijo**: la ventana (viewport), el contenido de texto, la fuente raíz o el **contenedor** del componente. Las unidades de viewport (`vh`, `vw`...) son las que permiten que un elemento se adapte al tamaño de la pantalla sin media queries; las de contenedor (`cqw`...) dan ese mismo superpoder a un componente **aislado de la ventana**. Esta nota cubre ambos grupos y las recetas fluidas que las usan. Las unidades de fuente (`em`, `rem`, `ch`) viven en [[Medidas CSS]].

> [!info] El modelo mental: tres orígenes de referencia
> 1. **La ventana** (`vw/vh/...`): siempre el viewport entero, sin importar dónde esté el elemento. Perfecto para páginas; demasiado grueso para componentes.
> 2. **El contenedor de consulta** (`cq*`): la caja que declaró `container-type`. Perfecto para componentes reutilizables; la misma idea que `@media` pero al nivel del componente.
> 3. **La fuente** (`em/rem/ch`...): ver [[Medidas CSS]].
> Pregunta de siempre: *¿relativa a qué?* — y elegir el origen que el elemento realmente "conoce".

## Unidades de viewport

Imaginá la pantalla del navegador como el 100% de referencia:

| Unidad  | Relativa a                                   | Ejemplo práctico                            |
| ------- | -------------------------------------------- | ------------------------------------------- |
| `vh`    | **Alto** de la ventana (1vh = 1% del alto)   | Secciones a pantalla completa: `100vh`      |
| `vw`    | **Ancho** de la ventana (1vw = 1% del ancho) | Barras o texto que escala con la pantalla   |
| `vmin`  | El **menor** de los dos (ancho o alto)       | Elementos circulares adaptables             |
| `vmax`  | El **mayor** de los dos                      | Fondos o marcos que nunca deben cortarse    |

```css
.seccion-hero {
  height: 100vh;                /* ocupa exactamente la altura de la ventana */
  width: 50vw;                  /* la mitad del ancho de la ventana */
}

.logo-adaptable {
  /* Se ajusta al lado más pequeño de la pantalla */
  width: 30vmin;
  height: 30vmin;
}
```

Ejemplo clásico de centrado con estas medidas:

```css
.centrado-en-pantalla {
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
```

El 100% del viewport es el **initial containing block** (el rectángulo de referencia inicial de todo el documento); cuando scrolleás, ese rectángulo no se mueve — por eso los elementos con `position: fixed` respecto de la viewport se comportan como se comportan (ver [[Tipos de Posiciones]]).

### Variantes dinámicas: `svh`, `lvh`, `dvh`

> [!warning] `100vh` en móviles: el problema de la barra de direcciones
> En móviles, `100vh` mide la altura **incluyendo** la barra del navegador, que aparece y desaparece al hacer scroll: una sección `100vh` queda "más alta que la pantalla" y tapa contenido. La solución moderna son las unidades dinámicas:
>
> | Unidad  | Significado                                        |
> | ------- | -------------------------------------------------- |
> | `svh`   | Small: la altura mínima (con barra visible)        |
> | `lvh`   | Large: la altura máxima (con barra oculta)         |
> | `dvh`   | Dynamic: sigue el valor actual mientras scrolleás  |
>
> Para secciones a pantalla completa en móvil, usá **`100dvh`**.

La mecánica completa de la "toolbar saga" en iOS Safari (2018–2023): la barra de URL colapsa/expande con el scroll, y `100vh` se quedó con la **altura expandida** (la más grande) de ese momento — la sección tapa el contenido al cargar. Cómo usar cada una:

| Unidad | Cuándo ganar | Comportamiento |
|---|---|---|
| `svh` | Elemento con scroll propio dentro de la pantalla (evita que su scroll interno quede cortado) | Estable: no cambia con el scroll |
| `lvh` | Solo cuando **querés** aprovechar toda la pantalla expandida (hero full-bleed al inicio) | El más grande |
| `dvh` | Secciones "a pantalla completa" de app (la default moderna) | Sigue el valor real; causa reflow durante el scroll (los navegadores lo optimizan) |
| `vh` | Solo como fallback para navegadores sin soporte | Deprecated en la práctica |

```css
.hero {
  height: 100vh;              /* fallback razonable */
}

@supports (height: 100dvh) {
  .hero {
    height: 100dvh;           /* la versión buena */
  }
}
```

En la práctica 2026 el `@supports` casi no hace falta (soporte desde Chrome 108, Safari 15.4, Firefox 101), pero queda como patrón progresivo correcto.

### La familia completa: 18 unidades de viewport

A las cuatro clásicas se suman las versiones small/large/dynamic de **todas** ellas, más las orientadas a escritura:

| Base | Small | Large | Dynamic | Significado |
|---|---|---|---|---|
| `vw` | `svw` | `lvw` | `dvw` | % del ancho |
| `vh` | `svh` | `lvh` | `dvh` | % del alto |
| `vmin` | `svmin` | `lvmin` | `dvmin` | % del lado menor |
| `vmax` | `svmax` | `lvmax` | `dvmax` | % del lado mayor |
| `vi` | `svi` | `lvi` | `dvi` | % del lado **inline** (según writing-mode) |
| `vb` | `svb` | `lvb` | `dvb` | % del lado **block** (según writing-mode) |

`vi`/`vb` son las unidades de viewport **lógicas**: en un layout vertical (japonés clásico, `writing-mode: vertical-rl`) `1vi` apunta al alto y `1vb` al ancho — usalas en componentes multilingües donde el eje puede rotar (ver [[Html]] para i18n).

> [!tip] `vmin` para cuadrados y modales
> Un modal `width: 90vmin; height: 90vmin` es cuadrado y cabe siempre, en horizontal y en vertical, sin media query. Es el uso más limpio de `vmin`; `vmax` casi no tiene casos — los marcos que "nunca se cortan" se resuelven con `min()`/`max()` (ver [[Medidas CSS]]).

## Trampas del viewport que hay que conocer

### El scrollbar: `100vw` incluye la barra de scroll

`100vw` = ancho del viewport **incluida la barra de scroll vertical**; `width: 100%` no la incluye. En escritorio con scroll visible (~15px), un layout `width: 100vw` queda ~15px más ancho que el documento → **scroll horizontal fantasma**:

```css
/* El bug */
.ancho-total {
  width: 100vw;      /* desborda ~15px cuando hay scrollbar vertical */
}

/* Las soluciones */
.ancho-total {
  width: 100%;       /* relativo al contenedor, sin scrollbar — preferida */
}
body {
  overflow-x: clip;  /* contención defensiva si algo insiste en desbordar */
}
```

Casos donde el patrón sobrevive: elementos `position: fixed` (no viven dentro del flujo del documento, no generan doble scrollbar). Para todo lo demás, `100%` (o `min(100%, …)`) gana a `100vw`.

> [!warning] 100% ≠ 100vw
> `100%` es relativo al **contenedor**; `100vw` a la **ventana** — y si el contenedor tiene padding/border, el 100% del hijo ya no es el 100% del viewport. Con `box-sizing: border-box` y `width: 100%` todo cuadra; con `vw` hay que restar manualmente (ver [[Box Model]]).

### Unidades de viewport y texto: el límite de accesibilidad

Los `vw` **no responden al tamaño de fuente configurado por el usuario** — `font-size: 5vw` escala con la pantalla, no con la preferencia de lectura. Por eso la receta fluida correcta siempre tiene **piso en `rem`** (ver [[Medidas CSS]] y la fórmula de pendiente más abajo): el `rem` es la ancla de accesibilidad, el `vw` solo aporta la pendiente.

Zoom de página en móvil: las unidades de viewport se recalculan; con zoom alto, `dvh` es preferible a `lvh` para no perder contenido.

## Unidades de contenedor: `cqw`, `cqh`, `cqi`, `cqb`, `cqmin`, `cqmax`

Las container query units son al `@container` lo que `vw/vh` son al `@media`: 1 unidad = 1% de la caja que **declaró** el container query. Un componente que no sabe nada de la ventana puede dimensionarse contra **su propio** contenedor:

```css
.tarjeta {
  container-type: inline-size;   /* esta caja pasa a ser "contenedor de consulta" */
}

.tarjeta__titulo {
  font-size: clamp(1.25rem, 4cqw, 2rem); /* crece con la TARJETA, no con la ventana */
}

.tarjeta__grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
}

/* Y el query en sí: el layout cambia según el ancho del contenedor */
@container (min-width: 600px) {
  .tarjeta__grid { grid-template-columns: 1fr 1fr 1fr; }
}
```

| Unidad | 1 unidad = | Nota |
|---|---|---|
| `cqw` | 1% del **ancho** del contenedor | La más usada |
| `cqh` | 1% del alto | Requiere contenedor con altura definida |
| `cqi` | 1% del lado **inline** | Lógica, respeta writing-mode |
| `cqb` | 1% del lado **block** | Lógica |
| `cqmin` | 1% del lado menor | Como `vmin` |
| `cqmax` | 1% del lado mayor | Como `vmax` |

```css
/* El componente escalable completo: tipo según el CONTENEDOR */
.ficha {
  container-type: inline-size;
  container-name: ficha;
}

@container ficha (width > 400px) {
  .ficha__contenido { display: grid; grid-template-columns: 120px 1fr; }
}
```

Diferencia decisiva con `vw`: dos instancias de la misma `.ficha` en la página (una sidebar de 300px, una principal de 900px) reciben **estilos distintos** según su contenedor — con `@media` recibirían los mismos.

> [!info] Fallback si no hay contenedor
> Si una unidad `cq*` se usa sin un contenedor de consulta elegible, **se resuelve como la unidad `sv*` equivalente** (small viewport). Nunca es "inválida"; el navegador siempre tiene dónde anclarla. Por eso se pueden usar con `@supports` y sin romper nada en contextos huérfanos.

Soporte: Chrome 105+, Firefox 110+, Safari 16+ (2022–2023) — universal en 2026.

> [!tip] ¿`vw` o `cqw`? Regla de componente
> Si el elemento es **página** (secciones a sangre, heroes), `vw`. Si es **componente** (cards, widgets, modales reutilizables), `cqw` — el componente no debe saber cuánto mide la ventana. Los design systems modernos viven en `cqw` + `@container`.

## Recetas fluidas

### 1. Tipografía fluida con `clamp()` (la base)

```css
h1 { font-size: clamp(2rem, 1rem + 2.5vw, 3.5rem); }
p  { font-size: clamp(1rem, 0.95rem + 0.3vw, 1.125rem); }
```

### 2. La fórmula de pendiente: calcular dónde corta el `clamp`

La parte preferida `A + B·vw` es una recta; `clamp` corta la recta en dos puntos de la escala de viewport. Dados `clamp(MIN, A + B·vw, MAX)`, la función se activa (deja de ser plana) en:

```
vw mínimo donde alcanza MIN:  (MIN − A) / B
vw máximo donde choca con MAX: (MAX − A) / B
```

Ejemplo con `clamp(2rem, 1rem + 2.5vw, 3.5rem)` (suponiendo 1rem=16px → MIN=32px, MAX=56px):

| Corte | Cálculo | Viewport |
|---|---|---|
| Empieza a crecer en | (32 − 16) / 2.5 = 6.4 | 640px |
| Deja de crecer en | (56 − 16) / 2.5 = 16 | 1600px |

O sea: ≤640px queda en 2rem, entre 640–1600px crece lineal, ≥1600px se queda en 3.5rem. **Esa recta es el "media query invisible"** — si los cortes no coinciden con lo que el diseño necesita, se ajusta `B` (pendiente) o los extremos. Equivalencia útil: `B = (MAX − MIN) / (VMAX − VMIN)` viewport-por-viewport.

### 3. Espaciado que respira sin romper el ritmo

```css
.pagina {
  --separacion: clamp(1rem, 0.6rem + 2vw, 3rem);
  display: flex;
  flex-direction: column;
  gap: var(--separacion);
}
```

### 4. Componente con pendiente propia (contenedor-relativa)

```css
.widget {
  container-type: inline-size;
}

.widget__titulo {
  font-size: clamp(1.1rem, 2cqw + 0.5rem, 1.75rem); /* pendiente según el widget */
}
```

### 5. Media a pantalla completa bien

```css
.pantalla {
  min-height: 100svh;  /* nunca corta contenido (es el mínimo de las dos alturas) */
  /* o 100dvh si querés seguir la barra; o dvh + max-height para contener */
  display: grid;
  place-items: center;
}
```

## Accesibilidad

- **Nunca solo `vw` para texto**: el usuario que amplía la fuente en el navegador no recibe nada. Piso siempre en `rem` (recetas 1 y 2).
- **Alturas en `vh/dvh`**: contenido ampliado puede cortarse; `min-height` + flujo normal casi siempre > `height` fijo.
- El parallax y las secciones `100dvh` deben respetar `prefers-reduced-motion` cuando muevan fondo — ver [[Efectos de Scroll]].
- Zoom de texto (200%): las unidades de viewport se recalculan; verificá que `clamp` no asfixie el texto mínimo.

## Rendimiento

- Resolving de unidades: coste cero — no hay nada que optimizar aquí.
- `dvh` puede cambiar el layout durante el scroll (reflow de la altura de la sección); los motores lo optimizan, pero **animar** una propiedad en `dvh` en keyframes sí es costoso — animá con `transform` (ver [[Animaciones]]).
- Container queries re-evaluadas al redimensionar el contenedor: comportamiento barato y acotado al componente (a diferencia del reflow global de las media queries en layouts gigantes).

## Errores comunes

1. **`100vh` en móvil** sin fallback `dvh`: la sección tapa el contenido al cargar con la barra visible.
2. **`width: 100vw` en flujo**: scroll horizontal fantasma de ~15px por la scrollbar vertical — usar `100%`.
3. **Texto en `vw` puro**: inaccesible con fuente ampliada; siempre `clamp` con `rem`.
4. **`cq*` sin `container-type` en el ancestro**: se resuelven contra la small viewport — comportamiento "casi correcto" muy difícil de diagnosticar. Revisá el árbol de contenedores.
5. **`container-type: size` sin altura definida**: `size` exige alto determinado; para el 99% de los casos usá `inline-size`.
6. **Confundir `vmin` con `vmax`** en modales: `vmin` = lado menor (cabe siempre); `vmax` rara vez es lo que querés.
7. **`svh` donde importaba el movimiento**: `svh` no sigue la barra — correcto para layout estable, incorrecto si esperabas "llena la pantalla expandida" (eso es `lvh`/`dvh`).
8. **Unidades de viewport en `position: fixed` de móvil con teclado virtual**: el viewport visual se redimensiona al abrir el teclado; `dvh`/`lvh` ayudan, pero los inputs críticos deben quedar visibles con scroll (`scroll-margin`) — ver [[Efectos de Scroll]].

## Cheatsheet — unidades de viewport y contenedor

| Unidad | Referencia | Variante small/large/dynamic | Soporte (2026) | Uso típico |
|---|---|---|---|---|
| `vw` | % del ancho del viewport | `svw` `lvw` `dvw` | 2011+ / variantes 2021-22 | pendientes de tipografía, heroes |
| `vh` | % del alto del viewport | `svh` `lvh` `dvh` | 2011+ / variantes 2021-22 | secciones full-screen (`dvh`) |
| `vmin` | lado menor | `svmin` `lvmin` `dvmin` | 2011+ / variantes 2021-22 | modales, cuadrados |
| `vmax` | lado mayor | `svmax` `lvmax` `dvmax` | 2011+ / variantes 2021-22 | marcos que no cortan |
| `vi` / `vb` | lado inline / block (writing-mode) | `svi`… `dvb` | 2022+ | i18n, layouts verticales |
| `cqw` `cqh` | % ancho/alto del contenedor | — | Chrome 105 / FF 110 / Safari 16 | tipografía y spacing de componentes |
| `cqi` `cqb` | lado inline/block del contenedor | — | ídem | variantes lógicas |
| `cqmin` `cqmax` | lado menor/mayor del contenedor | — | ídem | cuadrados por contenedor |
| Sin contenedor válido | — | `cq*` ≡ `sv*` | — | fallback estricto por spec |

## Relacionadas

- [[Medidas CSS]] — `px`, `em`, `rem`, `%` y las funciones `calc/min/max/clamp`
- [[Responsive Web Design]] — media queries y `clamp()` en contexto
- [[Flexbox]] — modelos de layout que usan estas medidas en sus hijos
- [[CSS Grid]] — `fr`, `minmax()` y cómo encajan con unidades de viewport
- [[Efectos de Scroll]] — alturas, snap y teclado virtual en móvil
- [[Tipos de Posiciones]] — fixed respecto de la viewport vs del contenedor
- [[Animaciones]] — transform vs layout con unidades dinámicas
- [[Box Model]] — por qué `100%` y `100vw` no cuadran con padding/border
