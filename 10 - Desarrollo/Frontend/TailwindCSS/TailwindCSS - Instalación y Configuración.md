---
aliases: ["Instalación de Tailwindcss", "Tailwind CLI"]
tags:
  - dev/frontend/tailwind
tipo: concepto
estado: terminado
creado: 2026-06-03
actualizado: 2026-10-11
relacionado:
  - "[[TailwindCSS - Sintaxis Básica]]"
  - "[[CSS Introducción]]"
fuente: "https://tailwindcss.com/docs/installation/tailwind-cli"
---

# TailwindCSS - Instalación y Configuración

**Tailwind CSS** es un framework CSS *utility-first*: en lugar de escribir clases propias, compones la interfaz usando utilidades predefinidas (`bg-red-500`, `p-4`, `text-xl`) directamente en el HTML. Esta nota describe el setup completo con **Tailwind v4** usando la **Tailwind CLI**, la vía más directa para empezar desde cero.

> [!info] Tailwind v4 es CSS-first
> A diferencia de la v3, en la v4 **no existe `tailwind.config.js` por defecto**: la configuración y las personalizaciones se hacen desde el propio CSS con `@import "tailwindcss"` y `@theme`. También cambió el paquete del ejecutable: ahora es `@tailwindcss/cli`.

## Requisitos

Necesitamos **Node.js completo** instalado (no solo el runtime), porque vamos a usar `npm` y `npx`:

```bash
node -v   # verificar instalación
npm -v
```

## 1. Iniciar el proyecto

Creamos el `package.json` que va a gestionar las dependencias:

```bash
npm init -y
```

El flag `-y` acepta los valores por defecto sin pasar por el asistente interactivo. Se genera el archivo `package.json` en la raíz del proyecto.

![[Pasted image 20260603094619.png]]

![[Pasted image 20260603094740.png]]

## 2. Instalar Tailwind

```bash
npm install tailwindcss @tailwindcss/cli
```

- `tailwindcss`: el motor del framework.
- `@tailwindcss/cli`: el ejecutable de línea de comandos que escanea nuestro código y genera el CSS final.

## 3. Estructura de archivos

Creamos la carpeta `src/` con el archivo de entrada CSS:

```css
/* src/input.css */
@import "tailwindcss";
```

Esa única línea importa todo el framework. Luego creamos nuestro `index.html` apuntando al CSS **compilado** (`style.css`, que todavía no existe):

```html
<!doctype html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link href="/src/style.css" rel="stylesheet">
</head>
<body>
  <h1 class="text-3xl font-bold underline">
    Hello world!
  </h1>
</body>
</html>
```

![[Pasted image 20260603095821.png]]

## 4. Compilar

En este punto **no se ve nada estilizado**, y es esperado: Tailwind no funciona en tiempo de ejecución, hay que generar el archivo CSS:

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/style.css
```

> [!tip] ¿Por qué no se aplicaban los estilos?
> El HTML carga `style.css`, pero ese archivo lo genera el compilador de Tailwind. El CLI escanea nuestros archivos en busca de clases utilitarias (como `text-3xl`, `font-bold`, `underline`) y escribe **solo esas** en el CSS de salida. Sin compilar, el archivo no existe y no hay estilos.

### Modo desarrollo (watch)

Para que recompile automáticamente al guardar cambios:

```bash
npx @tailwindcss/cli -i ./src/input.css -o ./src/style.css --watch
```

Recargá la página y el `<h1>` debería verse grande, en negrita y subrayado.

## Otras vías de instalación

La CLI es una opción; Tailwind v4 también se puede integrar directo con el build tool del proyecto:

| Vía | Cuándo usarla |
|---|---|
| **CLI** | Proyectos vanilla, aprendizaje, setups a medida |
| **Vite** (`@tailwindcss/vite`) | Proyectos con Vite (React, Vue, etc.) |
| **PostCSS** (`@tailwindcss/postcss`) | Cuando ya existe un pipeline PostCSS |
| **Browser** (`@tailwindcss/browser@4`) | Prototipos rápidos por CDN — ver [[TailwindCSS - Sintaxis Básica]] |

## Relacionadas

- [[TailwindCSS - Sintaxis Básica]] — cómo se escriben las utilidades
- [[CSS Introducción]] — fundamentos de CSS que Tailwind abstrae
- [[Flexbox]] — los layouts que componés con utilidades
- [[CSS Grid]] — grid layout con utilidades `grid-*`
- [[Responsive Web Design]] — variantes responsive (`md:`, `lg:`) en Tailwind
- [[Vite]] — integración con el plugin `@tailwindcss/vite`
