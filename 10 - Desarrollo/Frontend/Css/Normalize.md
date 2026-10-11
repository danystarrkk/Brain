---
aliases: ["Normalize CSS", "Normalizar CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Box Model]]"
  - "[[TailwindCSS - Sintaxis Básica]]"
fuente: "https://necolas.github.io/normalize.css/"
---

# Normalize

Los navegadores no arrancan de cero: cada uno trae su propia **hoja de estilos por defecto** (user-agent stylesheet) y, por eso, un mismo HTML se ve distinto en Chrome, Firefox y Safari (márgenes de `h1`, tamaños de `ul`, espaciados de `formularios`...). **Normalize.css** resuelve esto aplicando una capa inicial de estilos que **unifica los puntos de partida** entre navegadores, sin borrar el diseño de cada uno.

> [!info] Normalize vs Reset
> - **Reset** (reseteo agresivo): borra *todo* estilo por defecto (`margin: 0` en todo, sin listas, sin headings especiales) y te obliga a reconstruir cada elemento desde cero.
> - **Normalize** (lo que usamos acá): preserva lo útil de los estilos por defecto y **solo corrige inconsistencias y bugs conocidos**. Es menos destructivo y parte de un terreno conocido.

## Paso 1: descargarlo

Vamos a la página oficial de [normalize](https://necolas.github.io/normalize.css/):

![[Pasted image 20250124162622.png]]

Damos en *Download* y veremos el código fuente:

![[Pasted image 20250124162653.png]]

> [!tip] Desactivá el traductor automático
> Si tenés el traductor de Chrome activo, desactivalo en esta página: puede corromper los comentarios dentro del CSS al copiarlo.

## Paso 2: guardarlo en nuestro proyecto

Copiamos todo el contenido y lo pegamos en un archivo llamado `normalize.css` dentro de nuestra carpeta de estilos.

## Paso 3: cargarlo antes que nada

Lo enlazamos en el HTML **como la primera hoja de estilos**: todo lo que venga después debe poder sobreescribirlo sin pelearse con la cascada (ver [[Especificidad]] para por qué el orden importa):

```html
<head>
  <link rel="stylesheet" href="css/normalize.css" />
  <link rel="stylesheet" href="css/estilos.css" />
</head>
```

![[Pasted image 20250124162743.png]]

> [!warning] Orden siempre fijo
> Si normalize va después de tu CSS, sus reglas (de especificidad baja, casi todas de elemento) pueden pisar tus estilos. Primero normalize, después lo tuyo.

## Alternativas modernas

| Opción                        | Enfoque                                                       |
| ----------------------------- | ------------------------------------------------------------- |
| **normalize.css** (v8)        | Unificación cuidadosa — el clásico de este apunte              |
| **Modern CSS Reset**          | Resets cortos y modernos (usar `margin` en vez de `margin-block`, etc.) |
| **Preflight de Tailwind**     | El reset que trae Tailwind por defecto — ver [[TailwindCSS - Sintaxis Básica]] |
| **Reinventar el reset**       | Poquitas reglas propias con propiedades lógicas modernas       |

Si el proyecto ya usa Tailwind, no necesitás normalize aparte: Preflight ya lo hace.

## Relacionadas

- [[Html]] — dónde van los `<link>` de hojas de estilos
- [[Box Model]] — un normalize moderno también unifica `box-sizing`
- [[TailwindCSS - Sintaxis Básica]] — Preflight, el reset incluido en Tailwind
