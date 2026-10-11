---
aliases: ["Lo Básico", "Tailwind Básico"]
tags:
  - dev/frontend/tailwind
tipo: concepto
estado: terminado
creado: 2026-06-03
actualizado: 2026-10-11
relacionado:
  - "[[TailwindCSS - Instalación y Configuración]]"
  - "[[Especificidad]]"
fuente: "https://tailwindcss.com/docs"
---

# TailwindCSS - Sintaxis Básica

Una vez visto el setup en [[TailwindCSS - Instalación y Configuración]], esta nota cubre cómo se **escriben** las utilidades de Tailwind y las formas de usarlo.

## Uso mediante CDN (script)

Podemos usar Tailwind directamente desde un `<script>` en el HTML, sin instalar nada:

```html
<!doctype html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
</head>
<body>
  <h1 class="text-3xl font-bold underline">
    Hello world!
  </h1>
</body>
</html>
```

> [!warning] Limitaciones de la versión browser
> `@tailwindcss/browser` compila en el navegador: sirve para prototipos y pruebas rápidas, pero **no es óptimo para producción** — no hay caché de compilación, el CSS se genera en cada carga y se pierde todo el pipeline de build. Para proyectos reales usá la vía CLI/Vite de [[TailwindCSS - Instalación y Configuración]].

## Anatomía de una utility

Cada clase sigue el patrón **Propiedad → Valor → Intensidad**:

| Clase | Propiedad | Valor | Intensidad |
|---|---|---|---|
| `bg-red-500` | background-color | red | 500 (media) |
| `text-white` | color | white | — |
| `p-4` | padding | 4 (1rem) | — |
| `text-3xl` | font-size | 3xl | — |

La escala de intensidades va de `50` (más clara) a `950` (más oscura); los valores intermedios como `400`, `500` y `600` son los más usados.

## Ejemplos

Aplicando color de fondo y texto:

```html
<div class="bg-orange-500">CSS</div>
<div class="text-violet-400">CSS 2</div>
```

Tipografías de la escala `font-sans` / `font-serif`:

```html
<p class="font-sans">
  Lorem, ipsum dolor sit amet consectetur adipisicing elit...
</p>
<p class="font-serif">
  Lorem ipsum dolor sit amet, consectetur adipisicing elit...
</p>
```

> [!tip] Variantes: responsive, hover y más
> Las utilidades aceptan **variantes** que agregan contexto: `md:text-2xl` aplica en pantallas medianas en adelante, `hover:bg-red-600` al pasar el mouse. Ver [[Responsive Web Design]] para el patrón responsive completo.

## Relacionadas

- [[TailwindCSS - Instalación y Configuración]] — setup con CLI y compilación
- [[Especificidad]] — cómo resuelve Tailwind la cascada sin `!important`
- [[Flexbox]] — utilidades `flex-*`, `justify-*`, `items-*`
- [[CSS Grid]] — utilidades `grid-cols-*`, `gap-*`
