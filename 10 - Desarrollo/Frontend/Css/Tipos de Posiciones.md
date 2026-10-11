---
aliases: ["Position", "Posicionamiento CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Box Model]]"
  - "[[Flexbox]]"
  - "[[Efectos de Scroll]]"
  - "[[Tipos de Display]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/position"
---

# Tipos de Posiciones

La propiedad `position` define **cómo se sitúa un elemento** dentro del flujo de la página y respecto a qué punto se miden sus desplazamientos (`top`, `right`, `bottom`, `left`). Tiene cuatro valores principales — más el `static` por defecto — y son la base de menús pegajosos, botones flotantes y modales.

## Los valores de `position` de un vistazo

| Valor      | ¿Sale del flujo? | Referencia de `top/left/...`            | Uso típico                        |
| ---------- | ----------------- | --------------------------------------- | --------------------------------- |
| `static` (defecto) | No        | No admite desplazamientos              | Todo el contenido normal          |
| `relative` | No                | Su **posición original**                | Contexto de posicionamiento, corregir a mano |
| `absolute` | **Sí**            | El ancestor posicionado más cercano     | Overlays, badges, menús           |
| `fixed`    | **Sí**            | La **ventana** (viewport)               | Headers, botones flotantes, barras |
| `sticky`   | No (hasta pegarse) | La ventana, dentro de su contenedor     | Índices, encabezados de tabla     |

## `static`: el comportamiento por defecto

No necesita declararse. Los elementos estáticos fluyen normal (block apilados, inline en línea) y **ignoran `top`/`left`**: para poder usarlos hay que cambiar el `position`.

## `fixed`

Se mantiene **fijo en ese punto de la pantalla**, sin importar el scroll:

```css
h1 {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
}
```

Es muy útil para botones flotantes o barra de navegación anclados a un punto de la pantalla que no se mueven al hacer scroll. Como sale del flujo, deja un hueco donde estaba: casi siempre va acompañado de un margen en el contenido (por ejemplo `body { padding-top: 60px }`).

## `relative`

No saca al elemento del flujo (sigue ocupando su lugar), pero **desplaza su posición visual** respecto a donde debería estar, y — clave — **pasa a ser el punto de referencia** de sus hijos `absolute`:

```css
h1 {
  position: relative; /* Desbloquea el uso de los demás atributos */
  left: 10rem;        /* También admite %, rem, vw... */
  top: 2rem;
}
```

> [!tip] El patrón relative + absolute
> Un padre `position: relative` + un hijo `position: absolute` es el combo más usado del posicionamiento: el hijo se ubica **dentro de su padre**, sin afectar al resto del layout.

## `absolute`

Ubica el elemento a partir del **elemento padre posicionado** más cercano (el ancestor con `position` distinto de `static`). Si **ningún** ancestor está posicionado, la referencia es el `body`/documento inicial — ese es el requisito que más confunde:

```css
.contenedor {
  position: relative; /* origen de coordenadas para los hijos */
}

.contenedor .badge {
  position: absolute;
  top: 0;
  right: 0;           /* pegado a la esquina superior derecha del contenedor */
}
```

Como sale del flujo, los elementos siguientes **no reservan espacio** para él: se superpone.

## `sticky`

El híbrido: se comporta **como `relative` hasta que** llega el punto de anclaje durante el scroll, y entonces "se pega" a la ventana **dentro de los límites de su contenedor**:

```css
.indice {
  position: sticky;
  top: 1rem;   /* se pega cuando llegue a 1rem del borde superior */
}
```

Suele usarse en encabezados de sección, tablas largas o índices. Casos prácticos de pegado con scroll en [[Efectos de Scroll]].

## Desplazamientos: `top`, `right`, `bottom`, `left`

Funcionan con cualquier unidad (`px`, `rem`, `%`...) y miden **hacia dónde se empuja el elemento desde su referencia**:

- Con `absolute`/`fixed`: son la coordenada exacta respecto a la referencia.
- Con `relative`: son un desplazamiento sobre su posición original (el espacio original **sigue ocupado**).

Si definís `top` y `bottom` a la vez en un elemento no absoluto, gana `top`; en `absolute` con `height: auto`, ambos pueden estirarlo.

## `z-index`: quién va arriba

Los elementos posicionados se apilan; `z-index` (solo con `position` ≠ `static`) define el orden:

```css
.modal-overlay {
  position: fixed;
  inset: 0;
  z-index: 100; /* por encima de todo lo que no tenga z-index */
}
```

> [!warning] `z-index` no es infinito ni global
> Sin `position` no funciona. Y los valores gigantes (`99999`) son anticipo de problemas: en proyectos serios se usan escalas de z-index definidas en custom properties para no pelear después. El `inset` de arriba es el atajo moderno de `top/right/bottom/left: 0`.

## Errores comunes

- **Olvidar que `absolute` busca al ancestor posicionado**: sin un `relative` padre, el elemento se escapa hasta el `body`.
- **`fixed` sin dejar espacio**: el contenido de abajo queda tapado.
- **Usar `position` donde basta `flex`/`gap`**: centrar o alinear con Flexbox es más robusto que con desplazamientos — ver [[Flexbox]] y [[Tipos de Display]].

## Relacionadas

- [[Box Model]] — el espacio que cada elemento ocupa antes de moverlo
- [[Flexbox]] — alternativa declarativa a desplazar a mano
- [[Efectos de Scroll]] — efectos que aprovechan `sticky` y `fixed`
- [[Tipos de Display]] — cómo los elementos entran (o no) al flujo
