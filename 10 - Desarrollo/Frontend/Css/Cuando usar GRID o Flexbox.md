---
tags:
  - desarrollo
  - frontend
  - grid-flexbox
tipo: concepto
estado: terminado
---

# Cuando usar GRID o Flexbox

No hay respuesta única: **no existe un modelo "mejor" que el otro**. Resuelven problemas distintos con filosofías distintas, y la mayoría de sitios reales usan **los dos, anidados**. Esta nota es una guía de decisión: ambos modelos lado a lado, el mismo layout resuelto de las dos formas, señales concretas de cuándo cambiar de modelo, y cómo combinarlos.

## Los dos modelos de un vistazo

| | [[Flexbox]] | [[CSS Grid]] |
|---|---|---|
| **Dimensión** | Una (fila *o* columna) | Dos (filas y columnas a la vez) |
| **Filosofía** | El **contenido negocia**: el contenedor reparte espacio entre items | El **contenedor diseña**: define la retícula y los items se colocan |
| **Pregunta típica** | "¿Alineo o reparto **estos ítems**?" | "¿Cómo reparto **el espacio total** de la página?" |
| **Quién controla el tamaño** | El item (`flex-basis`, contenido) | El contenedor (las pistas) |
| **Relación entre items** | Items se alinean solo **en su línea** | Items se alinean **entre todas** las pistas |
| **Responsive por defecto** | `flex-wrap` + `flex-basis` | `repeat(auto-fit, minmax())` |
| **Costo de layout complejo** | Alto (wrappers, hacks de porcentaje) | Bajo (declarativo) |

> [!tip] La pregunta de decisión
> Preguntate: *¿estoy alineando un grupo de cosas que ya tienen su tamaño, o estoy definiendo la estructura del espacio?* Si es lo primero → Flexbox. Si es lo segundo → Grid. Y si es "la estructura de la página **y** la alineación dentro de cada zona" → Grid afuera, Flexbox adentro.

## Utilizá Flexbox cuando…

Flexbox organiza elementos en **una sola dimensión**. Es la herramienta para **alinear y distribuir** lo que está *dentro* de un contenedor, como una barra de navegación:

![[Pasted image 20250124170846.png]]

Casos típicos:

- Navbar con logo a la izquierda y enlaces a la derecha (`justify-content: space-between`).
- Botones centrados vertical y horizontalmente.
- Cards de igual altura en una fila.
- Apilar ítems con `gap` en una columna.
- Cualquier componente "de una línea": toolbars, tags, breadcrumbs, paginación, form actions.

```css
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
}
```

### Ejemplo profundo: media object (patrón de redes sociales)

```css
.media { display: flex; gap: 1rem; }
.media__avatar { flex: 0 0 48px; }        /* siempre 48px: no crece ni encoge */
.media__body   { flex: 1; min-width: 0; } /* el texto fluye, scrollea si es largo */
```

**Por qué Flexbox acá**: el avatar tiene tamaño fijo, el cuerpo se adapta, y *no hay segunda dimensión* que planificar. Meter Grid acá sería ruido.

### Ejemplo profundo: toolbar responsive con wrap

```css
.toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.5rem;
}
.toolbar .spacer { flex: 1; } /* empuja los botones restantes a la derecha */
```

## Utilizá CSS Grid cuando…

Grid define el **layout general** en dos dimensiones: columnas, filas y áreas de la página.

![[Pasted image 20250124170917.png]]

Casos típicos:

- Estructura header / nav / contenido / sidebar / footer.
- Galerías de imágenes con columnas fluidas.
- Cualquier diseño donde un ítem debe cruzar varias filas o columnas.
- Dashboards, layouts de app, revistas, formularios de dos columnas.

```css
.pagina {
  display: grid;
  grid-template-areas:
    "header header"
    "nav    main"
    "footer footer";
  grid-template-columns: 240px 1fr;
  gap: 1.5rem;
}
```

### Ejemplo profundo: galería sin breakpoints

```css
.galeria {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: 1rem;
}
```

**Por qué Grid acá**: el contador de columnas lo resuelve el navegador según el ancho. Equivalente en flex requiere `flex-wrap` + `flex: 1 1 220px` **y** un centinela `::after` para que el último item no se estire deforme (ver [[Flexbox]]).

## El mismo layout, resuelto de las dos formas

Comparación real de un "holy grail" (header, sidebar fija, main, footer). La versión Flexbox es legítima, pero fijate cuántas fragilidades aparecen:

**Versión Flexbox (posible, con defensas):**

```css
.layout {
  display: flex;
  flex-wrap: wrap;
  min-height: 100vh;
}
.header  { flex: 1 0 100%; }      /* ocupa todo el ancho: truco del 100% */
.sidebar { flex: 0 0 240px; }
.main    { flex: 1; min-width: 0; }
.footer  { flex: 1 0 100%; }      /* repetimos el truco */
```

**Versión Grid (declarativa):**

```css
.layout {
  display: grid;
  min-height: 100vh;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";
  grid-template-columns: 240px minmax(0, 1fr);
  grid-template-rows: auto 1fr auto;
}
.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }
.main    { grid-area: main; }
.footer  { grid-area: footer; }
```

Lectura de la diferencia:

- En flex, las **posiciones** (header arriba, footer abajo) se codifican con `flex-basis: 100%` y el orden del DOM. Cambiar el layout = reordenar el HTML o sumar wrappers.
- En grid, las posiciones se leen del mapa `grid-template-areas`. Cambiar el layout = cambiar el mapa (pantalla chica: un mapa distinto en el `@media`, **sin tocar el HTML**).
- `grid-template-rows: auto 1fr` da la fila central elasticidad; en flex equivaldría a jugar con `flex-grow` de hijos de un contenedor columna… anidado.

> [!info] No se trata de líneas de código
> Se trata de **dónde vive la complejidad**: en flex vive en los hacks (`100%` de basis, wrappers, centinelas, porcentajes que suman >100% con gaps). En grid vive en un mapa declarativo que podés leer de un vistazo. Cuando la complejidad crece, el modelo que la **declara** escala mejor que el que la **negocia**.

## Tabla de decisión (criterio por criterio)

| Criterio | Elegí **Flexbox** | Elegí **Grid** |
|---|---|---|
| Cantidad de dimensiones | 1 (fila o columna) | 2 (filas y columnas) |
| Tamaños del contenido | Definen el layout (contenido primero) | Se estiran a pistas definidas (layout primero) |
| Alineación **entre** filas | No posible sin hacks | Nativo (`align-items`, pistas) |
| Items que cruzan varias pistas | `flex-basis` + porcentajes, frágil | `grid-column: span 2` |
| Elementos superpuestos | Requiere `position`/`z-index` + wrapper | Mismo `grid-area` + `z-index` |
| Contador de columnas responsive | `flex-wrap` + basis + centinela | `repeat(auto-fit, minmax())` |
| Reordenar al hacer responsive | `order` (con cuidado a11y) o reordenar DOM | Cambiar `grid-template-areas` |
| Componente de una línea (nav, toolbar) | **Ideal** | Sobredimensionado |
| Página completa con zonas | Posible, frágil | **Ideal** |
| Fila/columna de botones, tags | **Ideal** | Innecesario |
| Formulario de varias columnas alineadas | Limitado | **Ideal** (`grid-template-columns`) |
| Layout con many items de tamaños variables | `flex-wrap` razonable | `masonry` aún limitado; ver nota |
| Depende del orden del DOM | Fuertemente (flex order es visual) | Menos (colocación explícita) |
| Cantidad de propiedades nuevas | 3–4 esenciales | 8–10 esenciales |

> [!note] `masonry`
> El "grid de Pinterest" (items de alto variable sin huecos) es el único layout que **ninguno** de los dos resuelve bien nativamente: existe `grid-template-rows: masonry` en propuesta, y en la práctica se resuelve con JS (librerías) o aceptando los huecos de Grid.

## El flujo de decisión

```text
¿El grupo de elementos es "una línea" (nav, toolbar, botones, tags)?
  └─ SÍ → Flexbox
  └─ NO ↓
¿Estoy definiendo zonas/columnas/filas de una página o sección?
  └─ SÍ → Grid
  └─ NO ↓
¿Los items deben alinearse con items de OTRAS filas (columnas de tabla visual)?
  └─ SÍ → Grid
  └─ NO ↓
¿Los items tienen tamaños de contenido que definen el layout?
  └─ SÍ → Flexbox
  └─ DUDAS → empezá con Grid: los casos flex siguen siendo flex adentro
```

## Híbrido: Grid afuera, Flexbox adentro

La combinación real: **Grid para el esqueleto** (dónde van las zonas) y **Flexbox dentro** de cada zona (alineación de listas, botones, etc.). No compiten: cada uno resuelve su nivel del diseño. Un item de Grid es perfectamente un contenedor flex:

```css
/* esqueleto: grid */
.layout {
  display: grid;
  grid-template-columns: 240px 1fr;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
  gap: 1rem;
}

/* cada zona: flex o grid según su contenido */
.header        { display: flex; align-items: center; justify-content: space-between; }
.main__cards   { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 1rem; }
.card          { display: flex; flex-direction: column; }
.card__acciones{ display: flex; gap: 0.5rem; margin-top: auto; }
```

Regla práctica de anidación: **cada nivel elige el modelo que le corresponde**. El layout de página es grid; una card es flex (columna de contenido + acciones al fondo con `margin-top: auto`); los tags de la card son flex otra vez. `display` no es hereditario — cada contenedor decide solo.

> [!tip] La capa intermedia que muchos olvidan
> Un grid item con `display: flex` convive con ambas naturalezas sin conflicto: como **item** de la grilla recibe área, alineación de celda y `z-index`; como **contenedor flex** organiza a sus hijos. No hay que elegir "el modelo del documento", hay que elegir el modelo **de cada contenedor**.

## Heurísticas de migración: "tu flex está pidiendo grid"

Señales de que el código flex acumuló hacks y conviene migrar ese bloque:

1. **Centinela `::after` con flex solo para contener al último item** → `repeat(auto-fit, minmax())` en grid.
2. **`flex-basis: 33.33%` + cálculos con gaps** → `1fr` o `minmax()` (los fr restan gaps solos).
3. **Wrappers vacíos en el HTML cuyo único propósito es la posición** (un `<div>` solo para que el header quede arriba) → `grid-template-areas`.
4. **`flex-basis: 100%` repetido para "romper línea"** → filas explícitas en grid.
5. **Dos contenedores flex paralelos que intentan alinearse entre sí** (columna de títulos + columna de valores) → una sola grilla con columnas; la alineación entre filas es exactamente lo que flex no puede.
6. **`order` usado para reordenar más de 1-2 items** → colocación con `grid-column`/`grid-row`.

Y la migración inversa: si tu "grid" tiene **una sola fila/columna de items auto colocados** y nunca usás filas… probablemente es flex semánticamente (no funcionalmente: grid de una dimensión funciona, pero comunica peor la intención y trae propiedades que no vas a usar).

## ¿Y los floats y el posicionamiento?

Los `float` fueron el **único** mecanismo de layout multi-columna antes de Flexbox y Grid, y arrastran problemas (el contenedor colapsa, hay que limpiarlos con `clear` o clearfix). Hoy solo se justifican para envolver texto alrededor de imágenes — el caso original delimitado para el que fueron inventados.

- `position: absolute` para layouts → solo para superposiciones puntuales (badges, modales); sale del flujo y no responde.
- `position: sticky` → complemento útil **dentro** de ambos modelos (sidebars, headers de tabla) — ver [[Tipos de Posiciones]] y [[Efectos de Scroll]].
- `display: table` → legado puro.

> [!warning] Nunca más floats para layout
> Si ves `float: left` en un layout de columnas, es código heredado: reemplazalo por [[Flexbox]] o [[CSS Grid]].

## Responsive: estrategias lado a lado

| Estrategia | Flexbox | Grid |
|---|---|---|
| Items que fluyen y se acomodan | `flex-wrap: wrap` + `flex: 1 1 220px` | `repeat(auto-fit, minmax(220px, 1fr))` |
| Cambiar estructura por breakpoint | `@media` + `order`/basis (limitado) | Otro `grid-template-areas` completo |
| Adaptar al contenedor (no a la pantalla) | Container queries + `flex-basis` | Container queries + `@container` |
| Alturas iguales en fila | `align-items: stretch` (nativo) | `stretch` por defecto |

Ambos responden bien a `gap`, ambos conviven con container queries, y ambos se benefician de `minmax(0, 1fr)` / `min-width: 0` para el contenido largo — ver [[Responsive Web Design]] para el marco completo.

## Rendimiento

No hay diferencia práctica entre modelos para layouts de interfaz: los dos son *layout normal* del navegador. Donde **sí** cuenta la elección es en complejidad indirecta:

- Los hacks flex con wrappers extra = **más nodos DOM** = más trabajo de style recalc y layout.
- Los media queries de breakpoints disparan reescrituras de reglas; `auto-fit/minmax` la resuelve el motor de layout sin cambiar reglas.
- Animar propiedades de tamaño de ambos modelos es *layout* cada frame — animar `transform`/`opacity` en su lugar (ver [[Animaciones]]).

## Errores típicos de elección

1. **Flex para página completa**: se puede, pero cada zona nueva obliga a reordenar HTML o wrappers. Elegí Grid para esqueletos.
2. **Grid para componentes de una línea**: funciona, pero `justify-items`, `grid-auto-flow` y el resto quedan sin uso — ruido que confunde al equipo.
3. **Ignorar el orden del DOM en ambos**: `order` y `dense` cambian lo visual sin cambiar la lectura accesible del teclado.
4. **Esperar de flex lo que solo da grid**: alineación entre filas distintas (columnas de tabla visual) es el caso imposible de flex puro.
5. **Mezclar modelos en el mismo contenedor**: `display: flex` y `display: grid` en el mismo elemento — gana el último declarado; no es "doble modelo".

## Cheatsheet de decisión

| Pregunta | Respuesta |
|---|---|
| ¿Una fila o columna de cosas? | Flexbox |
| ¿Zonas de página, filas y columnas? | Grid |
| ¿Contenido define tamaño? | Flexbox |
| ¿Contenedor define retícula? | Grid |
| ¿Items alineados entre sí en varias filas? | Grid |
| ¿Componente autocontenido (nav, card, toolbar)? | Flexbox (grid adentro si tiene columnas) |
| ¿Responsive sin breakpoints? | Grid (`auto-fit`) o flex (`wrap` + centinela) |
| ¿Layout completo de sitio? | Grid afuera + flex adentro |

## Relacionadas

- [[Flexbox]] — sintaxis y propiedades de una dimensión
- [[CSS Grid]] — sintaxis y propiedades de dos dimensiones
- [[Responsive Web Design]] — cómo se adaptan ambos modelos a pantallas chicas
- [[Tipos de Display]] — cómo activar cada contexto de formato
- [[Tipos de Posiciones]] — el tercer modelo: superposiciones puntuales
- [[Box Model]] — lo que cada caja aporta a cualquier modelo de layout
- [[Efectos de Scroll]] — carruseles horizontales con snap en ambos modelos
- [[TailwindCSS - Sintaxis Básica]] — las utilidades `flex-*`, `grid-cols-*`, `col-span-*`
- [[Estilos para escribir CSS]] — cómo nombrar estas capas de layout a escala
