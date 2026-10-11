---
aliases: ["Colores Css", "Valores de color", "Custom properties de color"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Degradados en CSS]]"
  - "[[Background]]"
  - "[[Animaciones]]"
  - "[[Sombras CSS]]"
  - "[[Box Model]]"
  - "[[Especificidad]]"
fuente: "https://developer.mozilla.org/es/docs/Web/CSS/color_value"
---

# Colores CSS

CSS puede representar un mismo color de varias formas. Elegir el formato correcto afecta a la legibilidad, al tamaño del CSS, a qué tan fácil es mantener el diseño y —desde 2023— a si tu sitio aprovecha las pantallas **wide-gamut** (P3). Esta nota cubre todos los formatos de valor color, la gestión de paletas con custom properties, los temas claro/oscuro y las reglas de contraste.

> [!info] Un color en CSS es un valor, no "un hex"
> `#3498db`, `rgb(52 152 219)`, `hsl(204 70% 53%)` y `oklch(0.62 0.15 235)` pueden describir **el mismo color** en distintos espacios. CSS convierte entre espacios automáticamente (con pérdida de gama al recortar); por eso conviene trabajar en un solo espacio por decisión de diseño y no mezclar sin criterio.

## Formatos de color

### 1. Por nombre (keyword)

![[Pasted image 20250124161916.png]]

Son palabras clave reservadas (`blue`, `rebeccapurple`, `tomato`...). Solo hay ~148 colores con nombre y la mayoría no sirve para una paleta de marca. **No se recomienda salvo para `black`, `white` y `transparent`**, donde son claros e innegociables.

Datos útiles sobre keywords:

- `transparent` = `rgb(0 0 0 / 0)` — no es "sin color", es negro 100% transparente: al mezclarlo (`color-mix`) tiende a oscurecer.
- `currentcolor` = el valor de `color` **del propio elemento** (heredable). Es la keyword que hace componentes reutilizables: un `border: 1px solid currentcolor` se tiñe solo con el texto.
- Historia: `rebeccapurple` se añadió en 2014 en memoria de Rebecca Meyer, hija de Eric Meyer, autor de CSS. Los keywords **sí distinguen mayúsculas en origen pero se tratan case-insensitive**; `aqua` y `cyan` son el mismo color, igual que `gray`/`grey` y `magenta`/`fuchsia`.
- Las keywords de sistema (`Canvas`, `CanvasText`, `AccentColor`, `ButtonFace`…) existen para el modo de color forzado del usuario — ver "Accesibilidad".

### 2. Hexadecimal

![[Pasted image 20250124162011.png]]

Formato `#RGB` (3 dígitos) o `#RRGGBB` (6 dígitos), donde cada par es el valor 00–FF de rojo, verde y azul. Desde 2016 también existen las variantes con alfa: `#RGBA` y `#RRGGBBAA`:

```css
.negro { color: #000000; }
.negro-corto { color: #000; }      /* equivalente a lo anterior */
.acento { color: #3498db; }
.acento-80 { color: #3498dbcc; }   /* alfa cc ≈ 80% */
```

- La forma corta expande duplicando cada dígito: `#349` = `#334499`.
- Hex es **RGB puro**: no expresa intención de diseño (no sabés cuánta luz tiene el color).
- El navegador detecta hex con alfa moderno por longitud (8 o 4 dígitos) — no hay riesgo de confusión con sintaxis vieja.

### 3. Funciones: `rgb()`, `hsl()`, `hwb()`

```css
.rgb   { color: rgb(52 152 219 / 0.8); }   /* moderna: canales separados por espacio, alfa con / */
.rgba  { color: rgba(52, 152, 219, 0.8); } /* sintaxis antigua por comas, todavía válida */
.rgba2  { color: rgb(52, 152, 219, 0.8); } /* rgb() acepta comas legacy con 4 args */
.hsl   { color: hsl(204 70% 53%); }        /* matiz, saturación, luminosidad */
.hwb   { color: hwb(204 20% 14%); }        /* matiz + blanco + negro (más intuitivo que HSL) */
```

| Formato     | Ventaja                                      | Uso recomendado                          |
| ----------- | -------------------------------------------- | ---------------------------------------- |
| `#hex`      | Compacto, universal                          | Valores fijos conocidos                  |
| `rgb()`     | Cada canal es intuitivo con un picker        | Colores derivados de diseño              |
| `hsl()`     | Fácil variar luz/saturación para tonos       | Paletas, hovers, estados                 |
| `hwb()`     | Agregar blanco/negro es directo              | Ajustes finos de tintes                  |

Sintaxis de HSL, en profundidad:

- **Hue** `0–360` (círculo cromático: 0 = rojo, 120 = verde, 240 = azul); acepta grados, `grad`, `rad`, `turn`.
- **Saturation** `0–100%` (0 = gris) y **Lightness** `0–100%` (0 = negro, 100 = blanco).
- Limitación clásica de HSL: mantener la saturación no mantiene la **percepción** — dos tonos con `50%` de luz se ven de brillo distinto. Por eso existen los espacios modernos.

> [!tip] El canal alfa va con `/`
> La sintaxis moderna de todas las funciones acepta el canal alfa con barra: `rgb(0 0 0 / 50%)`. Es más legible que la vieja coma final de `rgba()`. Los porcentajes o números 0–1 funcionan en ambos.

### 4. Espacios modernos: `lab()`, `lch()`, `oklab()`, `oklch()`

```css
.lch   { color: lch(55 90 230); }        /* perceptual vía CIE */
.oklch { color: oklch(0.65 0.17 237); }  /* espacio moderno, gama más amplia */
.oklab { color: oklab(0.6 0.1 -0.1); }   /* sus coordenadas rectangulares */
```

- **`oklch()`** (2021, browsers desde 2023) es el espacio recomendado para diseño: sus ejes son **luminosidad L (0–1)**, **croma C (0≈gris, máximo≈neón)** y **hue H (0–360°)**.
- La gran ventaja: **la L es perceptual**. `oklch(0.7 …)` se ve igual de brillante en cualquier matiz — para generar paletas (tonos claros/oscuros de una marca) es trivial y consistente.
- **Gama más amplia**: oklch puede describir colores fuera del sRGB que los monitores P3 sí pueden mostrar. Si el navegador no llega, **mapea el color hacia adentro** (no hace fallback a sRGB a menos que lo pidas).
- `lch()`/`lab()` son el modelo CIE clásico; para CSS moderno se prefiere `oklch()` por su manejo del croma.

### 5. `color()`: el espacio con nombre explícito

```css
.p3 {
  color: color(display-p3 1 0.5 0);        /* rojo-dorado P3 */
  /* fallback donde no hay soporte (declaración anterior descartada): */
  color: rgb(255 128 0);
}
```

| Espacio en `color()` | Alcance |
|---|---|
| `srgb` | lo que CSS siempre pudo mostrar |
| `srgb-linear` | sRGB sin compand (para cálculos) |
| `display-p3` | pantallas Apple/modernas: ~25% más de gama |
| `a98-rgb`, `prophoto-rgb` | espacios Adobe/Foto |
| `xyz-d65` | base absoluta CIE |

### 6. `color-mix()`: mezclar sin salir de CSS

```css
.hover     { background: color-mix(in oklch, var(--marca) 85%, white); }
.sombra-suave { box-shadow: 0 8px 24px color-mix(in srgb, black 20%, transparent); }
```

- `color-mix(espacio, colorA peso%, colorB peso%)` — los pesos deben sumar 100 (o se normalizan).
- El `in <espacio>` decide en qué modelo se mezcla: mezclar **en oklch** da resultados perceptualmente parejos; en `srgb`, tienden a apagarse.
- Reemplaza la costumbre de tener 50 variables "primario-10…primario-90" a mano: los tonos se generan.

> [!warning] `transparent` y la mezcla premultiplicada
> `color-mix(in srgb, red 50%, transparent)` da `rgb(255 0 0 / 0.5)` correcto, pero `color-mix(in oklch, …, transparent)` puede producir oscurecimiento indeseado si el espacio no premultiplica bien. Para transparencias simples, preferí alfa explícito: `rgb(255 0 0 / 50%)`.

### 7. `light-dark()`: dos temas en una declaración

```css
.texto { color: light-dark(#111827, #e5e7eb); }
body { color-scheme: light dark; }  /* necesario: define qué cuenta como "light" */
```

Requiere `color-scheme` (o la media query de tema) y devuelve el primer valor en tema claro, el segundo en oscuro. Combinado con `@media (prefers-color-scheme: dark)` para custom properties, resuelve el modo oscuro completo.

## Paleta con custom properties

Para crear una paleta reutilizable se define un selector raíz `:root` (el elemento `<html>`, el más alto del árbol) y ahí viven las **custom properties** (variables CSS), que siempre empiezan con doble guion:

```css
:root {
  --blanco: #ffffff;
  --negro: #111827;
  --primario: #3498db;
  --exito: #2ecc71;
  --error: #e74c3c;
}
```

> [!warning] El punto y coma es obligatorio
> Un custom property sin `;` antes de `}` rompe **todas** las declaraciones posteriores del bloque. El ejemplo original de la nota tenía `--blanco: #ffffff` sin punto y coma — corregido arriba.

Luego se consumen con `var()`:

```css
.boton {
  background-color: var(--primario);
  color: var(--blanco);
  border-radius: 6px;
}
.boton:hover {
  background-color: var(--exito);
}
```

> [!info] Custom properties vs colores fijos
> Las variables viven en el navegador y se pueden cambiar en **runtime** (por JS, o por una clase distinta como `.tema-oscuro`). Eso las hace el mecanismo natural para **temas**: definí la paleta en `:root` y redefinila en otro scope.

```css
.tema-oscuro {
  --blanco: #e5e7eb;  /* redefinición: todos var(--blanco) cambian */
  --negro: #0b1120;
}
```

### Paleta generada: semillas + `color-mix()`

La técnica moderna de sistemas de diseño: pocas semillas y tonos derivados en el navegador:

```css
:root {
  --marca: oklch(0.62 0.15 235);
  --marca-suave:  color-mix(in oklch, var(--marca), white 70%);
  --marca-fuerte: color-mix(in oklch, var(--marca), black 25%);
}
```

### Variables tipificadas con `@property`

```css
@property --color-marca {
  syntax: "<color>";
  inherits: false;
  initial-value: oklch(0.62 0.15 235);
}
/* ahora --color-marca INTERPOLA en @keyframes/transiciones:
   se puede animar el color de un tema entero */
```

Sin `@property`, las custom properties son *tokens*: un `transition: color 1s` sobre `var(--color-marca)` cambia de golpe, no interpola — ver [[Animaciones]].

## Tema oscuro (los tres caminos)

```css
/* 1. Media query: el sistema operativo decide */
@media (prefers-color-scheme: dark) {
  :root { --blanco: #e5e7eb; --negro: #0b1120; }
}

/* 2. Clase en <html>: lo decide el usuario en tu UI */
.tema-oscuro { --blanco: #e5e7eb; --negro: #0b1120; }

/* 3. light-dark() por propiedad (mínimo boilerplate) */
.caja { background: light-dark(#ffffff, #111827); color: light-dark(#111827, #e5e7eb); }
```

Recomendación: **sembrá con `prefers-color-scheme`** (respeta el sistema sin que el usuario toque nada) y ofrecé toggle con clase que sobreescriba las mismas variables. `color-scheme` en el root también alinea los controles nativos (inputs, scrollbars) con el tema.

## Accesibilidad: contraste y modo forzado

- **Contraste texto/fondo mínimo 4.5:1** para texto normal; **3:1** para texto grande (≥24px o ≥19px bold) y para bordes/gráficos de interfaz — criterios WCAG 2.2 AA. DevTools muestra el ratio al inspeccionar cualquier color.
- HSL/oklch hacen trivial derivar variantes de contraste: bajar la L hasta pasar el ratio en vez de adivinar hex.
- **`prefers-contrast: more`** y **`forced-colors: active`** (Windows high contrast): el usuario puede imponer paletas propias y el navegador **elimina `background-image` y muchos colores author**. Soportá `forced-colors` dejando bordes/outline visibles y usando keywords de sistema (`CanvasText`, `AccentColor`) donde importe el estado — ver [[Box Model]] para outlines.
- El color **nunca debe ser el único canal** de un estado (error, éxito, foco): acompañar con ícono, texto o forma.
- Paletas con `oklch` y gama P3 tienen colores que en pantallas sRGB se **recortan**; verificá el ratio con el fallback sRGB, no con el color ideal.

> [!tip] Validar gama moderna con `@supports`
> `@supports (color: oklch(0.7 0.2 40))` permite servir la versión sRGB a navegadores viejos sin que el color se rompa — el patrón correcto de *progressive enhancement* de color.

## Errores comunes

1. **Mezclar formatos sin sistema**: `#3498db` en un lado, `hsl(204 70% 53%)` en otro para el mismo token — imposible de mantener. Un token = un formato.
2. **Alpha legacy vs moderno**: `rgba(52, 152, 219, 0.8)` está bien, pero en la sintaxis moderna la coma antes del alfa es inválida: `rgb(52 152 219, 0.8)` no existe.
3. **`transparent` como "sin color" en mezclas** → tiende a oscurecer; usar alfa explícito del mismo color.
4. **Hue que "se pasa de 360"**: 400 = 400−360 = 40 (es un círculo, es válido y explicable), pero conviene normalizar para legibilidad.
5. **Confianza ciega en el hex visto en tu monitor**: sRGB vs P3 — lo que ves tú no es lo universal; los contrastes se validan con números, no con ojo.
6. **Variables de color sin `@property` que "no interpolan"** en animaciones — ver `@property` arriba.
7. **Oscurecer con `black` en vez de bajar la L en oklch**: los degradados con negro "ensucian" los colores; `color-mix` en oklch con L menor los mantiene vibrantes.
8. **Olvidar `color-scheme`** al usar `light-dark()` → el navegador no sabe qué tema está activo y todos los valores devuelven el claro.

## Cheatsheet — formatos y valores de color

| Formato | Ejemplo | Alfa | Notas |
|---|---|---|---|
| keyword | `tomato`, `black` | `transparent` | ~148 nombres; `currentcolor` = color heredado |
| hex | `#3498db`, `#349`, `#3498dbcc` | 4 y 8 dígitos | RGB puro, compacto |
| `rgb()` moderna | `rgb(52 152 219 / 80%)` | con `/` | sintaxis de espacios |
| `rgb()` legacy | `rgba(52, 152, 219, 0.8)` | 4º argumento | comas, todavía válida |
| `hsl()` | `hsl(204 70% 53% / 0.8)` | con `/` | matiz 0–360, S y L en % |
| `hwb()` | `hwb(204 20% 14%)` | con `/` | matiz + blanco + negro |
| `lab()`/`lch()` | `lch(55 90 230)` | con `/` | espacio CIE perceptual |
| `oklch()` | `oklch(0.62 0.15 235)` | con `/` | recomendado: L perceptual, gama amplia |
| `oklab()` | `oklab(0.6 0.1 -0.1)` | con `/` | ejes rectangulares de oklch |
| `color()` | `color(display-p3 1 0.5 0)` | con `/` | espacio con nombre explícito |
| `color-mix()` | `color-mix(in oklch, A 70%, B)` | hereda | mezcla en el espacio indicado |
| `light-dark()` | `light-dark(#111, #eee)` | — | dos temas en una declaración |

| Keyword global | Efecto sobre un color |
|---|---|
| `inherit` | toma el color del padre |
| `initial` | negro (`#000`) |
| `unset` | `inherit` (color es heredable) |
| `currentcolor` | el `color` computado del elemento |

## Relacionadas

- [[Degradados en CSS]] — combinar colores en degradados
- [[Background]] — colores aplicados a fondos
- [[Animaciones]] — interpolar colores en keyframes (y por qué `@property` desbloquea variables)
- [[Sombras CSS]] — colores de sombra con alfa
- [[Especificidad]] — cómo ganan las reglas que pintan el color
- [[Box Model]] — `currentColor` y estados de borde/foco
