---
aliases: ["Selectores CSS", "Tipos de selectores"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Especificidad]]"
  - "[[Anatomía de CSS]]"
  - "[[Pseudo Código]]"
  - "[[Estilos para escribir CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Learn/CSS/Building_blocks/Selectors"
---

# Selectores en CSS

Un selector define **a qué elementos aplica** una regla. Existen muchas formas de escribirlos; este repaso cubre los tipos básicos, cómo se combinan y qué conviene usar en cada caso.

## Selector de elemento (tag)

Selecciona por nombre de etiqueta. Es lo más básico: todos los elementos de ese tipo:

```css
p {
  color: blue;
}
```

![[Pasted image 20250124161224.png]]

Con lo que vemos en la imagen, **todos** los párrafos pasan a ser azules: es tan abstracto que toma todas las etiquetas `p`, y lo mismo con cualquier otra etiqueta.

> [!warning] Máxima alcance, mínima especificidad
> Un selector de elemento llega a todos los `<p>` del documento (incluidos los que no querías) y puntúa 0–0–1 — la especificidad más baja útil. Ver [[Especificidad]].

## Selector de clase

Selecciona por el atributo `class`, con punto adelante. Las clases son **reutilizables**: se pueden usar las veces que queramos, en la misma página o en varias:

```css
.destacado {
  background: yellow;
}
```

![[Pasted image 20250124161246.png]]

Una clase puede estar en muchos elementos, y un elemento puede tener varias clases (`<p class="destacado urgente">`). Es el selector **por defecto** del CSS moderno y la base de metodologías como BEM — ver [[Estilos para escribir CSS]].

## Selector de ID

Funciona igual que la clase pero con `#` adelante, y **no es reutilizable**: cada id debe ser único en la página, por lo que solo puede aplicar a un elemento:

```css
#principal {
  max-width: 1200px;
}
```

![[Pasted image 20250124161315.png]]

> [!warning] No uses IDs para estilos
> Su especificidad (1–0–0) es tan alta que pisarlos desde otras hojas es un calvario, y la unicidad impide reutilizarlos. Regla práctica: **ids para JavaScript/`label`, clases para CSS**.

## Selector de atributo

Selecciona elementos según el valor (o presencia) de un atributo:

```css
input[type="text"] { border: 1px solid #ccc; }
a[target="_blank"] { color: #7f5af0; }
a[href^="https"] { padding-right: 1em; }   /* ^ = empieza con */
a[href$=".pdf"] { color: red; }            /* $ = termina con */
a[href*="github"] { font-weight: bold; }   /* * = contiene */
```

![[Pasted image 20250124161348.png]]

Al principio parece extraño, pero con tiempo y práctica se comprende y se usa bastante (especialmente con formularios y enlaces externos).

## Combinadores: cómo se relacionan los selectores

### Descendiente (espacio)

Selecciona cualquier descendiente de un elemento, a cualquier profundidad — por ejemplo, la clase `nombre` **dentro de** la clase `cliente`:

```css
.cliente .nombre {
  font-weight: bold;
}
```

![[Pasted image 20250124161421.png]]

Busca la clase `cliente` y, dentro de todo su árbol, aplica a los `.nombre`.

### Hijo directo (`>`)

Solo modifica los hijos **inmediatos** de una clase, id o etiqueta:

```css
.cliente > p {
  margin: 0;
}
```

![[Pasted image 20250124161442.png]]

Dice: modificará a todos los párrafos que sean **hijos directos** de `.cliente` — los nietos quedan afuera. Se puede usar con cualquier etiqueta.

### Hermanos

```css
h2 + p { margin-top: 0; }  /* + : el hermano inmediato siguiente */
h2 ~ p { color: #555; }    /* ~ : todos los hermanos siguientes */
```

| Combinador | Nombre        | Relación                              |
| ---------- | ------------- | ------------------------------------- |
| ` `        | Descendiente  | Cualquier nivel por dentro            |
| `>`        | Hijo          | Nivel inmediato                       |
| `+`        | Hermano adyacente | El siguiente directo               |
| `~`        | Hermanos      | Todos los que siguen                  |

## Selector universal y pseudo-selectores

```css
* { box-sizing: border-box; }   /* universal: todo elemento (0 puntos) */
```

Para estados (`:hover`, `:nth-child`) y partes virtuales (`::before`) ver [[Pseudo Código]].

## Recomendaciones de uso

1. **Preferí clases**: reutilizables, especificidad plana (0–1–0) y fáciles de mantener.
2. **Evitá IDs para CSS** y selectores largísimos: cuantos más elementos encadenás, más difícil es reutilizar y sobreescribir.
3. **No estilizas desde JavaScript**: agregá/quitar clases y dejá el estilo en el CSS.
4. Conocé la puntación de cada selector — [[Especificidad]] decide qué regla gana.

## Relacionadas

- [[Especificidad]] — la puntuación que decide qué selector gana
- [[Anatomía de CSS]] — cómo se estructura selector + declaración
- [[Pseudo Código]] — pseudo-clases y pseudo-elementos
- [[Estilos para escribir CSS]] — BEM y naming para selectores mantenibles
