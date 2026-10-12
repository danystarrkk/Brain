---
tags:
  - desarrollo
  - frontend
  - css
tipo: concepto
estado: terminado
---

# Sombras CSS

Las sombras dan **profundidad** a la interfaz: indican qué está elevado, qué es interactivo y qué está detrás. En CSS se aplican con `box-shadow` (a cajas) y `text-shadow` (a texto); para siluetas reales existe `filter: drop-shadow()`. Detrás de cada una hay una decisión de **pintura** (paint), no de layout: por eso una sombra nunca mueve a sus vecinos, pero tampoco sobrevive a un `overflow: hidden`.

> [!info] El modelo mental: luz y capas
> Una sombra es un **halo oscuro proyectado detrás de la caja**. El navegador calcula la silueta de la caja (con sus `border-radius`), la desplaza, la dilata (spread) y la aplica un desenfoque gaussiano, todo pintado **por debajo** del fondo del propio elemento y **por encima** del fondo del padre. "Elevación" en Material Design y similares es simplemente una escala de sombras más grandes según la jerarquía — ver la receta de tokens más abajo.

## `box-shadow`: anatomía de la sombra

Podemos aplicar sombras a cualquier caja — imágenes, botones, tarjetas:

```css
.button {
  box-shadow: 0.2rem 0.4rem 1rem rgba(0, 0, 0, 0.1);
  /*            X     Y     difuminado  color */
}
```

Los valores, en orden:

| Valor           | Significado                                              | Ejemplo            |
| --------------- | -------------------------------------------------------- | ------------------ |
| **X**           | Desplazamiento horizontal (negativo = a la izquierda)     | `0`                |
| **Y**           | Desplazamiento vertical (negativo = arriba)               | `0.4rem`           |
| **Difuminado**  | Radio de desenfoque (blur) — solo se puede agrandar      | `1rem`             |
| **Spread** (opcional) | Cuánto crece o encoge la sombra antes de difuminar | `0`                |
| **Color**       | Casi siempre con transparencia (`rgba` o `color-mix`)    | `rgba(0,0,0,.1)`   |
| **`inset`** (prefijo opcional) | Sombra **hacia adentro** (efecto hundido) | `inset 0 2px 4px …` |

> [!tip] Sombras sutiles, no llamativas
> La sombra moderna copia a la luz real: **poco desplazamiento, mucho desenfoque y mucha transparencia**. Ejemplo de escala de elevación tipo Material:
>
> ```css
> .elevacion-1 { box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06); }
> .elevacion-2 { box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08); }
> .elevacion-3 { box-shadow: 0 12px 32px rgba(0, 0, 0, 0.12); }
> ```

### Detalles de la mecánica que importan

- **Blur = desviación estándar × 2**: el valor del blur es el diámetro del área difuminada; el núcleo opaco de la sombra es el `box-shadow box + spread`, y el halo se difumina hacia afuera el radio indicado. Duplicar el blur duplica el halo, no lo hace "dos veces más suave" ópticamente.
- **Spread negativo**: encoge la sombra antes de difuminar; si supera el tamaño de la caja, la sombra **desaparece**. Sirve para sombras "cerradas" tipo borde suave.
- **Sombras por esquina**: `box-shadow` no admite valores por esquina, pero **sí** sigue el `border-radius` por esquina: con `border-radius: 8px 8px 0 0` la sombra sale solo de las esquinas redondeadas. Para una sombra en una sola esquina se puede recortar con `clip-path` en el elemento o pintarla con `background` (gradientes) — truco avanzado, no siempre vale la pena.
- **Pintura**: la sombra outer se pinta por debajo del fondo/borde del elemento y por encima del fondo del ancestro. La `inset` se pinta por encima del fondo del propio elemento, por debajo del contenido.
- **Colores**: siempre con transparencia (`rgb(... / 0.08)`, `hsl`, `color-mix()`). Sombras 100% opacas se ven como "pegatinas negras", no como sombras.

### Sombra inset: efectos hundidos

```css
/* Input hundido */
.input {
  background: #fff;
  box-shadow: inset 0 1px 3px rgb(0 0 0 / 0.15);
}

/* Ranura / canal */
.ranura { box-shadow: inset 0 2px 4px rgb(0 0 0 / 0.2); }

/* Borde interior luminoso (neumorfismo suave) */
.panel { box-shadow: inset 0 1px 0 rgb(255 255 255 / 0.6); }
```

La sombra inset se dibuja **dentro** del `padding-box`, respetando el `border-radius`; nunca sobresale del borde.

## Sombras múltiples

`box-shadow` acepta **varias sombras separadas por coma**, la primera pinta encima de las demás:

```css
.card {
  box-shadow:
    0 1px 2px rgba(0, 0, 0, 0.08),
    0 8px 24px rgba(0, 0, 0, 0.12);
}
```

Combinar una sombra dura y cercana con otra suave y amplia da un resultado mucho más natural que una sola. La física real proyecta sombras anidadas (penumbra + umbra), por eso **dos o tres capas** se perciben más creíbles que una gigante.

> [!warning] Interpolación de sombras múltiples
> Al transicionar, las sombras se interpolan **lista por lista, par a par**. Si en hover pasás de `1 sombra` a `3 sombras`, no hay pares y el navegador hace *interpolación discreta* (salto seco). Mantené la misma cantidad y orden de sombras en ambos estados — ver [[Animaciones]].

## `text-shadow`

Misma idea, sin spread ni `inset`:

```css
h1 {
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.4);
}
```

| Rasgo | `box-shadow` | `text-shadow` |
|---|---|---|
| Aplica a | la caja | el **glifo** (cada carácter) |
| Spread | ✅ | ❌ |
| `inset` | ✅ | ❌ |
| Sombras múltiples | ✅ | ✅ |
| Aplica a contenido generado | — | ✅ (hereda a `::before`/`::after` con texto) |

Usos reales: legibilidad de texto sobre imágenes (`text-shadow: 0 1px 3px rgb(0 0 0 / .6)` + `background` oscurecido), textos "energizados" con doble sombra dura (efecto neón: dos sombras en direcciones opuestas), capitulares editoriales.

> [!tip] Legibilidad sobre imagen: sombra + contraste
> Una sombra sola no garantiza contraste WCAG. Patrón robusto: `text-shadow` suave **más** un velo oscuro (`background: linear-gradient(rgb(0 0 0 / .55), rgb(0 0 0 / .55)))` en el contenedor) — ver [[Background]] y [[Colores CSS]].

## `box-shadow` vs `filter: drop-shadow`

| Propiedad               | Se dibuja sobre...                     | Característica                                   |
| ----------------------- | -------------------------------------- | ------------------------------------------------ |
| `box-shadow`            | El **rectángulo** de la caja           | Barato; ignora la forma real (p. ej. bordes redondeados con transparencia) |
| `filter: drop-shadow()` | La **silueta visible** del elemento    | Sigue la forma (PNG transparentes, `border-radius`); más costoso; sin spread |

```css
.foto-recortada {
  filter: drop-shadow(0 8px 16px rgba(0, 0, 0, 0.25));
}
```

Detalles de `drop-shadow()`:

- Sintaxis: `drop-shadow(offset-x offset-y blur-radius color)` — **sin spread** y sin `inset`. La sintaxis antigua con color primero ya no se acepta.
- Múltiples sombras: se **encadenan** `drop-shadow(...) drop-shadow(...)`, no se separan por coma. Cada cadena se aplica sobre el resultado de la anterior (por eso el efecto "doble contorno").
- Sigue el **canal alfa** del render: iconos SVG, PNGs recortados, texto, bordes redondeados — todo lo que tenga transparencia proyecta su silueta real.
- En **texto** es caro (se rasteriza el glifo completo); para texto usá `text-shadow`.
- El filtro se aplica al elemento **ya renderizado**, así que combina con `transform` y sobrevive a formas no rectangulares donde `box-shadow` recto delataría la caja.

> [!info] Las sombras no afectan el layout
> `box-shadow` **no ocupa espacio**: se dibuja fuera del flujo y puede sobrepasar los bordes vecinos. Para separar elementos seguí usando `gap`/márgenes — ver [[Box Model]].

## Color de sombra: del negro puro a los tonos

El negro puro (`#000`) sobre fondos claros produce sombras sucias. Escalas profesionales:

| Técnica | Ejemplo | Efecto |
|---|---|---|
| Negro transparente | `rgb(0 0 0 / 0.08)` | Neutra; el estándar mínimo |
| Tono del tema | `rgb(20 30 60 / 0.12)` | Sombra fría "de ambiente", más natural |
| `color-mix()` | `color-mix(in oklch, var(--marca) 12%, transparent)` | Sombra tintada por la marca |
| Sombras blancas (modo oscuro) | `rgb(255 255 255 / 0.06)` + borde | En fondos oscuros el negro "no se ve"; se usa luz inversa o bordes |

```css
:root {
  --sombra-color: 220 30% 20%;
}

.card {
  box-shadow:
    0 1px 2px hsl(var(--sombra-color) / 0.10),
    0 6px 16px hsl(var(--sombra-color) / 0.08);
}

/* Modo oscuro: la sombra se vuelve luz */
[data-tema="oscuro"] .card {
  --sombra-color: 0 0% 100%;
  box-shadow:
    0 1px 2px rgb(0 0 0 / 0.4),
    0 6px 16px hsl(var(--sombra-color) / 0.05);
}
```

## Escalas de elevación y patrones de diseño

### Tokens de sombra reutilizables

```css
:root {
  --shadow-1: 0 1px 2px rgb(0 0 0 / 0.06);
  --shadow-2: 0 2px 4px rgb(0 0 0 / 0.06), 0 4px 12px rgb(0 0 0 / 0.08);
  --shadow-3: 0 4px 8px rgb(0 0 0 / 0.06), 0 12px 28px rgb(0 0 0 / 0.12);
  --shadow-4: 0 8px 16px rgb(0 0 0 / 0.08), 0 24px 48px rgb(0 0 0 / 0.16);
}

.card { box-shadow: var(--shadow-2); }
.card:hover { box-shadow: var(--shadow-3); }
.dropdown { box-shadow: var(--shadow-4); }
```

Una escala (como en Material) evita el "shadow drift": 47 valores distintos de sombra en un sistema es deuda de diseño.

### Glassmorphism

```css
.vidrio {
  background: rgb(255 255 255 / 0.4);
  backdrop-filter: blur(12px) saturate(1.4);
  border: 1px solid rgb(255 255 255 / 0.6);
  box-shadow: 0 8px 32px rgb(31 38 135 / 0.15);
}
```

La sombra acá es secundaria: el efecto lo llevan `backdrop-filter` y el borde translúcido — ver [[Background]].

### Neumorfismo (con advertencia)

```css
.neumo {
  background: #e0e5ec;
  border-radius: 12px;
  box-shadow:
    6px 6px 12px rgb(0 0 0 / 0.15),
    -6px -6px 12px rgb(255 255 255 / 0.9);
}
```

Dos sombras en esquinas opuestas (una oscura, una clara) simulan relieve. **Advertencia de accesibilidad**: el contraste del texto sobre fondos neumórficos rara vez llega a 4.5:1 y los "botones hundidos" no comunican estado sin icono/texto — ver sección de accesibilidad.

## Animación de sombras

`box-shadow` y `text-shadow` **son interpolables**: entre dos listas con igual cantidad de sombras y mismas formas, el navegador interpola cada componente (offset, blur, spread, color). Técnicas:

```css
/* Hover simple: bien si la sombra es pequeña */
.tarjeta {
  box-shadow: var(--shadow-1);
  transition: box-shadow 0.2s ease;
}
.tarjeta:hover { box-shadow: var(--shadow-3); }
```

- Cada frame de la transición **repinta** la caja (la sombra no va a compositor). En elementos pequeños no se nota; en elementos a pantalla completa, sí (ver rendimiento abajo).
- Truco de rendimiento para superficies grandes: la sombra vive en un `::after` posicionado detrás; al hacer hover se **cambia de opacidad** entre dos capas (composición) en vez de interpolar:

```css
.gran-panel::after {
  content: "";
  position: absolute;
  inset: 0;
  box-shadow: var(--shadow-4);
  opacity: 0;
  transition: opacity 0.2s;
  pointer-events: none;
}
.gran-panel:hover::after { opacity: 1; }
```

- Con `prefers-reduced-motion: reduce`, cualquier transición de sombra debe desactivarse o acortarse — ver [[Animaciones]].

## Errores comunes

1. **Sombras recortadas por `overflow: hidden`**: un contenedor con `overflow: hidden` (scroll containers, `border-radius` con recorte) **corta** las sombras de sus hijos. Síntoma: la tarjeta "pierde" la sombra al entrar en un contenedor con scroll. Fix: `padding` en el scroller ≥ al radio de la sombra, o margen compensado.
2. **Sombras demasiado oscuras/opacas**: `rgba(0,0,0,.5)` con blur chico se ve como borde negro, no sombra. Regla: menos de 0.15 de alpha y blur generoso.
3. **Animar `box-shadow` en elementos grandes**: jank perceptible (repaint cada frame). Fix: transicionar solo en hovers pequeños; para grandes, el truco del `::after` con opacidad, o animar `transform` (ver [[Animaciones]]).
4. **Esperar que la sombra empuje a los vecinos**: no ocupa layout — si necesitás separación, es `gap`/`margin` (ver [[Box Model]]).
5. **Olvidar el spread negativo**: `inset 0 2px 4px -1px` cierra la sombra; con spreads mal firmados la sombra inset se ve como velo completo.
6. **`drop-shadow` con blur enorme en toda la página**: el filtro rasteriza y desenfoca la silueta completa — caro en scroll. Usar en elementos acotados (iconos, fotos), no en `body`.
7. **Interpolar listas de distinta cantidad de sombras**: salto seco en la transición — mantener el mismo número de capas en origen y destino.
8. **Sombras como única señal de jerarquía**: en `forced-colors` (modo alto contraste del SO) casi todas las sombras desaparecen; el diseño debe sostenerse sin ellas.

## Accesibilidad

- **Contraste primero**: la sombra no arregla texto con bajo contraste; verificar 4.5:1 sobre el fondo real (ver [[Colores CSS]]).
- **Modo forzado de colores**: `@media (forced-colors: active)` elimina o aplana sombras y bordes decorativos. No pongas información *solo* en la sombra (activo/inactivo, elevado/no elevado).
- **Estados no solo por sombra**: un botón "hundido" (`inset`) como único indicador de `:checked` falla si el usuario no distingue el relieve; acompañar con icono, color y `aria-pressed` (ver [[Formularios]]).
- **Neumorfismo con cuidado**: los contrastes de texto suelen quedar por debajo de WCAG; si se usa, fijar colores de texto con contraste verificado y estados con borde o icono adicional.
- **`text-shadow` sobre imagen**: verificar con la imagen real cargada (la sombra compensa, pero la verificación manda).

## Rendimiento

- Sombras **estáticas**: costo práctico nulo; se rasterizan una vez por repaint.
- **Animación**: pintura por frame — el costo crece con el área de la caja × blur. Regla: animar sombras solo en elementos pequeños (botones, filas); evitar en heroes o listas largas.
- `drop-shadow` es más caro que `box-shadow` (cálculo de silueta + filtro); encadenados, el costo se multiplica.
- Las sombras **no** disparan layout ni compositing: usarlas en vez de bordes `outline` animados es indiferente; lo que sí evita repaint es animar `transform`/`opacity` (ver [[Animaciones]]).
- `will-change` no ayuda a las sombras (no van al compositor); el truco del `::after` con opacidad es la vía.

## Generadores

Para experimentar con los valores existe [cssmatic](https://www.cssmatic.com/es/box-shadow) (también tiene generador de degradados — ver [[Degradados en CSS]]): solo copiá el código que genera y pegalo donde lo necesites. Útil para calibrar a ojo; después conviene normalizar los valores a una escala de sombras reutilizable.

## Cheatsheet

| Propiedad | Sintaxis | Valores | Default |
|---|---|---|---|
| `box-shadow` | `[inset]? x y blur spread color` | longitudes, `%`, colores; listas por coma | `none` |
| `box-shadow` inset | `inset x y blur spread color` | sombra interior al padding-box | — |
| `text-shadow` | `x y blur color` | sin spread ni inset; listas por coma | `none` |
| `filter: drop-shadow()` | `drop-shadow(x y blur color)` | sin spread; encadenable | `none` |
| `transition` de sombra | `transition: box-shadow .2s` | interpola listas de igual longitud | — |
| Blur | radio | mitad = desviación estándar de la gaussiana | — |
| Spread | radio | negativo encoge; < −tamaño anula la sombra | `0` |

| Capacidad | `box-shadow` | `text-shadow` | `drop-shadow` |
|---|---|---|---|
| Sigue `border-radius` | ✅ | — | ✅ |
| Sigue silueta alfa (PNG/SVG) | ❌ | — | ✅ |
| Spread | ✅ | ❌ | ❌ |
| `inset` | ✅ | ❌ | ❌ |
| Múltiples | coma | coma | encadenado |
| Costo | bajo | bajo-medio | medio-alto |

## Relacionadas

- [[Colores CSS]] — `rgba`, `hsl` y `color-mix()` para elegir el color de la sombra
- [[Degradados en CSS]] — el otro generador de cssmatic
- [[Box Model]] — por qué la sombra no suma a las medidas del elemento
- [[Tipos de Posiciones]] — capas y `z-index`, donde las sombras se nota el orden
- [[Animaciones]] — transiciones de sombra, costo de repaint y `prefers-reduced-motion`
- [[Background]] — glassmorphism, `backdrop-filter` y velos sobre imagen
- [[Formularios]] — estados de control hundido/levantado con `inset`
