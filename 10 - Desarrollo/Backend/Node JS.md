---
aliases: ["Node", "NodeJS", "Node.js"]
tags:
  - dev/backend/node
tipo: concepto
estado: borrador
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[ReactJS - Vite]]"
  - "[[Formato Json]]"
fuente: "https://nodejs.org/docs/latest/api/"
---

# Node JS

**Node.js** es un entorno de ejecución de JavaScript **fuera del navegador**: permite correr JS en el sistema operativo usando **V8**, el motor de Chrome. Antes de Node, JS solo existía en el navegador; con él nació el JavaScript de servidor y de herramientamiento.

> [!info] Por qué importa para el frontend
> Todas las herramientas de desarrollo modernas (npm, Vite, los CLIs de Tailwind, React itself en su toolchain) corren sobre Node. Tenerlo instalado es prerrequisito de casi todo el ecosistema de [[ReactJS - Vite]] y de la instalación por CLI de [[TailwindCSS - Instalación y Configuración]].

## Qué aporta

- **npm** (`npm`, `npx`): gestor de paquetes — el mayor ecosistema de librerías del mundo. Ver [[Formato Json]] para el formato de `package.json`.
- **Módulos ES**: mismos `import`/`export` que en el navegador moderno.
- **APIs de sistema**: archivos, procesos, redes, streams — lo que lo hace apto para backends.
- **Sin navegador**: no hay `document`, `window` ni DOM — eso es labor de [[Document Object Model (DOM)]] en el frontend.

## Verificar la instalación

```bash
node -v   # versión del runtime
npm -v    # versión del gestor de paquetes
```

## Ejecutar JS con Node

```bash
node archivo.js
```

Node también tiene un REPL interactivo: escribí `node` en la terminal y empezás a ejecutar JS directo.

## Versión LTS

Para proyectos usá siempre la versión **LTS** (Long Term Support) desde nodejs.org — es la que garantizan las herramientas del ecosistema (Vite, los CLIs de frameworks, etc.).

## Relacionadas

- [[ReactJS - Vite]] — bundler que corre sobre Node
- [[TailwindCSS - Instalación y Configuración]] — CLI vía npm/npx
- [[Formato Json]] — formato de `package.json`
- [[Bash Scripting - Fundamentos Básicos]] — la shell desde donde se invoca Node
