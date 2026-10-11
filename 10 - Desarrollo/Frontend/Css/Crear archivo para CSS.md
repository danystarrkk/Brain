---
aliases: ["Crear archivo para Css", "Crear archivo CSS", "Link a CSS"]
tags:
  - dev/frontend/css
tipo: concepto
estado: terminado
creado: 2025-05-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[CSS Introducción]]"
  - "[[Estilos para escribir CSS]]"
fuente: "https://developer.mozilla.org/es/docs/Learn_web_development/Core/Styling_basics/Getting_started"
---

# Crear archivo para CSS

El flujo básico: una **carpeta** que contenga los estilos, un **archivo `.css`** dentro, y un **enlace** desde el HTML. Sin ese último paso, el navegador no sabe que el archivo existe.

## 1. Estructura de carpetas

Creá una carpeta contenedora llamada `css`:

![[Pasted image 20250124160634.png]]

Dentro de ella, el archivo de estilos — convención habitual: `styles.css` o `style.css`:

![[Pasted image 20250124160740.png]]

Un proyecto mínimo queda así:

```
proyecto/
├── index.html
└── css/
    └── styles.css
```

## 2. Enlazar el archivo en el HTML

Para que los estilos se apliquen, el `<head>` del HTML necesita la etiqueta `<link>`:

```html
<link href="css/styles.css" rel="stylesheet">
```

- `rel="stylesheet"` le dice al navegador *qué es* el recurso (una hoja de estilo).
- El `href` es una **ruta relativa** al archivo HTML: `css/styles.css` significa "una carpeta `css` al mismo nivel que este HTML".

> [!warning] Errores comunes
> - Olvidar `rel="stylesheet"` → el navegador no carga los estilos.
> - Ruta mal escrita o mayúsculas distintas → en servidores Linux las rutas distinguen mayúsculas (`CSS/styles.css` ≠ `css/styles.css`).
> - Colocar el `<link>` fuera del `<head>` funciona, pero retrasa el descubrimiento del CSS.

## 3. Optimización: `preload`

Para que la hoja de estilos se descargue antes de que el HTML la pida, se puede precargar:

```html
<link rel="preload" href="css/styles.css" as="style">
<link href="css/styles.css" rel="stylesheet">
```

El `preload` dispara la descarga en alta prioridad al inicio de la página; el `<link rel="stylesheet">` posterior la aplica. Se mantiene **ambas líneas**: el preload acelera, pero solo el stylesheet aplica los estilos.

> [!tip] Opción moderna: `onload` para CSS no crítico
> Para CSS que no debe bloquear el primer pintado (por ejemplo, hojas de terceros):

```html
<link rel="preload" href="css/secundaria.css" as="style" onload="this.rel='stylesheet'">
```

## 4. Varios archivos CSS

Nada impide enlazar varios archivos; el orden importa porque la cascada resuelve conflictos:

```html
<link href="css/normalize.css" rel="stylesheet">  <!-- 1º: reset/base -->
<link href="css/styles.css" rel="stylesheet">     <!-- 2º: estilos propios -->
```

En proyectos grandes este trabajo lo automatiza el bundler (Vite, webpack) y en preprocesadores los archivos se dividen con `@use` — ver [[Sintaxis de Sass]] y [[Qué es Sass]]. Para decidir cómo organizar los estilos entre archivos, ver [[Estilos para escribir CSS]].

## Relacionadas

- [[Html]] — estructura del documento donde se enlaza el CSS
- [[CSS Introducción]] — qué hace el CSS una vez enlazado
- [[Normalize]] — qué poner en el primer archivo CSS del proyecto
- [[Qué es Sass]] — alternativa con preprocesador al CSS plano
