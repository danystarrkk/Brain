---
aliases: ["Vite", "Bundler"]
tags:
  - dev/frontend/vite
tipo: concepto
estado: terminado
creado: 2026-03-21
actualizado: 2026-10-11
relacionado:
  - "[[ReactJS - Fundamentos]]"
  - "[[Node JS]]"
fuente: "https://vite.dev/guide/"
---

# ReactJS - Vite

**Vite** (del francés "rápido") es un **bundler** y servidor de desarrollo para proyectos de frontend. Es el estándar de facto para proyectos React nuevos (reemplazó a Create React App).

> [!info] ¿Qué hace un bundler?
> En desarrollo, el navegador trabaja con muchos archivos JS/JSX sueltos, módulos ES, CSS y assets. El **bundling** es empaquetar y **transpilar** todo eso en archivos finales que el navegador interpreta directamente. Vite automatiza ese proceso para que solo te dediques a escribir código.

![[Pasted image 20260321223709.png]]

## Características

- **Instant Server Start**: usa **ESM** (módulos nativos del navegador) y sirve solo el módulo que el navegador pide — no compila todo el proyecto para abrir la app.
- **Lightning Fast**: HMR (Hot Module Replacement): los cambios en un componente se reflejan al instante, preservando el estado de la app.
- **Rich Features**: soporta TypeScript, JSX, CSS, assets e íconos sin configuración extra.
- **Optimized Build**: `vite build` pre-compila y optimiza (minificación, code-splitting) para producción con **Rollup** por debajo.

## Crear un proyecto

Requisito: [Node.js](https://nodejs.org) instalado (`node -v` y `npm -v` para verificar).

```bash
npm create vite@latest
```

![[Pasted image 20260321224815.png]]

Elegimos la tecnología (React, Vue, etc.). También se puede crear directo desde la línea de comandos, sin asistente:

```bash
npm create vite@latest mi-app -- --template react
```

Luego se instalan las dependencias y se levanta el servidor de desarrollo:

```bash
cd mi-app
npm install
npm run dev
```

![[Pasted image 20260321225239.png]]

## Estructura del proyecto

![[Pasted image 20260321224941.png]]

| Archivo/Carpeta | Rol |
|---|---|
| `public/` | Assets estáticos (imágenes, favicon) que se sirven tal cual |
| `src/` | Todo tu código: componentes, estilos, lógica |
| `src/main.jsx` | **Punto de entrada**: monta la app en el DOM (`createRoot`) e importa a `App` |
| `src/App.jsx` | Componente raíz que unifica/rutea el resto de componentes |
| `package.json` | Dependencias y scripts |

## Scripts y dependencias

El `package.json` trae los scripts ya configurados:

![[Pasted image 20260321225357.png]]

| Script | Qué hace |
|---|---|
| `npm run dev` | Servidor de desarrollo con HMR (puerto 5173) |
| `npm run build` | Build de producción en `dist/` |
| `npm run preview` | Sirve localmente el build de `dist/` para verificar antes de desplegar |

Sobre las **dependencias**: las de `dependencies` llegan a producción; las de `devDependencies` (Vite, plugins de build) solo se usan en desarrollo y no se empaquetan. Muchas se pueden ajustar editando `package.json` o `vite.config.js`.

> [!tip] Integración con Tailwind
> En proyectos Vite con Tailwind v4 se integra con el plugin oficial: `npm install @tailwindcss/vite` y agregarlo en `vite.config.js`. Ver [[TailwindCSS - Instalación y Configuración]].

## Relacionadas

- [[ReactJS - Fundamentos]] — componentes, estado y props
- [[ReactJS - Efectos (useEffect)]] — efectos en componentes
- [[TailwindCSS - Instalación y Configuración]] — integración CSS con Vite
- [[Formato Json]] — formato del `package.json`
