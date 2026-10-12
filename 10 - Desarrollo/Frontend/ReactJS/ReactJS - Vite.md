---
tags:
  - desarrollo
  - frontend
  - reactjs
tipo: concepto
estado: terminado
---

# ReactJS - Vite

**Vite** (del francés "rápido") es un **bundler y servidor de desarrollo** para proyectos de frontend. Es el estándar de facto para proyectos React nuevos (reemplazó a Create React App y webpack para la mayoría de casos). Esta nota cubre por qué es rápido, su configuración completa y cómo se despliega.

> [!info] ¿Qué hace un bundler?
> En el navegador trabajás con decenas/miles de archivos: módulos JS/JSX, CSS, assets. El **bundling** es transpilar (JSX→JS, TS→JS) y empaquetar todo eso en pocos archivos optimizados. Vite automatiza eso para que solo te dediques a escribir código. Ver imagen del proceso:

![[Pasted image 20260321223709.png]]

## Por qué Vite es rápido: el modelo ESM en desarrollo

La diferencia histórica explica todo. **webpack + dev server** (y CRA) arrancaban **todo** el grafo de módulos de la app antes de poder servir la primera página — con apps grandes eso son 30–60 s de arranque. **Vite** invierte el orden:

| | Bundler tradicional (webpack/CRA) | Vite (dev) |
|---|---|---|
| Arranque | Empaqueta **toda** la app antes de servir | Servidor arriba en milisegundos |
| Qué sirve | Bundle único generado | **Módulos ES nativos** (`<script type="module">`), uno por archivo, bajo demanda |
| Transformación | Todo upfront | **On-demand**: convierte (TSX→JS) solo el módulo que el navegador pidió |
| Escala del arranque | Lineal con el tamaño del proyecto | Casi constante |

Dos pilares:

1. **ESM nativo**: el navegador importa módulos directamente (`import App from './App.jsx'`); el dev server solo intercepta cada módulo y lo sirve transformado. El navegador resuelve el grafo, no el bundler.
2. **Pre-bundling con esbuild** (escrito en Go): dependencias de `node_modules` (React, lodash, librerías CommonJS) se pre-empaquetan **una sola vez** y se cachean en `node_modules/.vite`. Resuelve dos problemas: convertir CJS→ESM y evitar el "million requests" de importar paquetes con miles de archivos internos. esbuild es ~10–100× más rápido que Babel para compilar.

Para **producción** (`vite build`) Vite usa **Rollup** (maduro, con tree-shaking de alta calidad) y desde Vite 5 permite alternar a Rolldown. En resumen: **esbuild para la velocidad de desarrollo, Rollup para la calidad del bundle final**.

> [!tip] HMR instantáneo
> Hot Module Replacement: al guardar un archivo, Vite envía **solo ese módulo** al navegador, que lo intercambia en caliente — sin recargar la página y **preservando el estado** de React (árbol, hooks, inputs abiertos). La velocidad del HMR no depende del tamaño del proyecto: es constante porque solo viaja el archivo cambiado.

## Crear un proyecto

Requisito: [Node.js](https://nodejs.org) instalado (`node -v` y `npm -v` para verificar — ver [[Node JS]]).

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
| `public/` | Assets estáticos (imágenes, favicon) que se sirven **tal cual**, sin procesar ni hashear |
| `src/` | Todo tu código: componentes, estilos, lógica |
| `src/main.jsx` | **Punto de entrada**: monta la app en el DOM (`createRoot`) e importa a `App` |
| `src/App.jsx` | Componente raíz que unifica/rutea el resto de componentes |
| `index.html` | Plantilla real de la app (Vite la usa como entry, a diferencia de webpack) |
| `vite.config.js` | Configuración: plugins, aliases, proxy, `base` |
| `package.json` | Dependencias y scripts |

```jsx
// src/main.jsx — el "bootstrap" de toda app React con Vite
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

`<StrictMode>` habilita verificaciones extra en desarrollo (doble invocación de efectos — ver [[ReactJS - Efectos (useEffect)]]); no afecta a producción.

## Scripts y dependencias

![[Pasted image 20260321225357.png]]

| Script | Qué hace | Resultado |
|---|---|---|
| `npm run dev` | Servidor de desarrollo con HMR (puerto **5173**) | Servido en memoria, no escribe archivos |
| `npm run build` | Build de producción con Rollup | Carpeta `dist/` lista para hosting estático |
| `npm run preview` | Sirve localmente el build de `dist/` | Verificación final **antes** de desplegar (puerto 4173) |

> [!warning] `preview` ≠ `dev`
> `dev` corre con transforms on-demand y HMR (puede esconder problemas de bundle). `preview` sirve los archivos **reales** de producción: es el sanity check de que el build funciona (rutas relativas, assets, code splitting).

Sobre las **dependencias** (gestión en [[Formato Json]]): las de `dependencies` llegan a producción; las de `devDependencies` (Vite, plugins de build) solo se usan en build/desarrollo y no se empaquetan en el bundle final. Nota: React va en `dependencies`; Vite y `@vitejs/plugin-react` en `devDependencies`.

## Configuración de referencia: `vite.config.js`

```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'node:path';

export default defineConfig({
  plugins: [react()],                    // JSX + Fast Refresh
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'), // alias de import: import X from '@/components/X'
    },
  },
  server: {
    port: 3000,                           // puerto del dev server
    open: true,                           // abre el navegador al arrancar
    proxy: {                              // ver sección Proxy
      '/api': { target: 'http://localhost:8080', changeOrigin: true },
    },
  },
  build: {
    outDir: 'dist',                       // salida del build
    sourcemap: true,                      // mapas para depurar producción
  },
});
```

- **`defineConfig`** es solo tipado/autocompletado; devolver la config plana funciona igual.
- **`plugins`**: el único obligatorio en React es `@vitejs/plugin-react` (Babel/SWC para JSX y Fast Refresh). `@vitejs/plugin-react-swc` usa SWC (aún más rápido).
- **`resolve.alias`**: atajos de import que evitan rutas relativas profundas (`../../../utils`). Convención: `@` = `src/`.

### Variables de entorno: `import.meta.env`

Vite expone variables con prefijo **`VITE_`** (las demás se descartan por seguridad):

```bash
# .env (no commitear; usar .env.example)
VITE_API_URL=https://api.misitio.com
VITE_ENABLE_MOCKS=false
```

```js
// En el código (sustitución en build-time, NO en runtime)
const url = import.meta.env.VITE_API_URL;
const esDev = import.meta.env.DEV;        // true en dev, false en build
const esProd = import.meta.env.PROD;
const base = import.meta.env.BASE_URL;     // el `base` configurado
```

| Archivo | Cuándo aplica |
|---|---|
| `.env` | Siempre |
| `.env.local` | Siempre, sobreescribe `.env` (no git) |
| `.env.development` | Solo `vite dev` |
| `.env.production` | Solo `vite build` |

> [!warning] Las env vars se inyectan **en tiempo de build**
> No hay servidor que las cambie en producción: cambiar `VITE_API_URL` requiere **reconstruir**. Y cualquier cosa pública: lo que empiece con `VITE_` viaja en el bundle JS al navegador — jamás claves secretas.

## Assets: `public/` vs import

| Vía | Cómo se usa | Procesamiento | Nombre en build |
|---|---|---|---|
| `public/logo.png` | `<img src="/logo.png">` | Ninguno, se copia tal cual | El mismo (`/logo.png`) |
| `import logo from './logo.png'` | `import` + `{logo}` en el src | Hash de contenido + optimización opcional | `logo-a1b2c3d4.png` |

```jsx
import logo from './assets/logo.png';   // Vite retorna la URL final
<img src={logo} alt="Logo" width={100} />;
```

Ventajas del import: el hash rompe caché al cambiar el archivo y Rollup lo puede convertir a **base64 inline** si es pequeño (< 4 KB por defecto). Para SVG "importable como componente" se necesita un plugin (SVGR).

## Proxy: llamar a una API sin CORS

En desarrollo, el front corre en `localhost:5173` y la API en otro puerto — el navegador bloquearía la petición por CORS. El dev server de Vite actúa de **intermediario**:

```js
// vite.config.js
server: {
  proxy: {
    '/api': {
      target: 'http://localhost:8080',
      changeOrigin: true,        // reescribe el Host al del target
      rewrite: (p) => p.replace(/^\/api/, ''), // opcional: quitar el prefijo
    },
  },
},
```

```js
// En el código: mismo origen, sin CORS
const res = await fetch('/api/usuarios');
```

En producción el proxy no existe: se resuelve con un reverse proxy real (nginx, Cloudflare) o con la config de CORS del backend.

## Code splitting con import dinámico

Las **dynamic imports** (`import()`) dividen el bundle: el módulo se carga (y descarga) **bajo demanda**:

```jsx
import { useState, Suspense, lazy } from 'react';

// React: lazy requiere <Suspense>
const Dashboard = lazy(() => import('./pages/Dashboard.jsx'));

function App() {
  const [mostrar, setMostrar] = useState(false);
  return (
    <Suspense fallback={<p>Cargando....</p>}>
      {mostrar && <Dashboard />}
    </Suspense>
  );
}
```

```js
// Fuera de React: el clásico "cargar bajo demanda"
boton.on('click', async () => {
  const { initCharts } = await import('./charts.js'); // chunk separado
  initCharts();
});
```

Cada destino de `import()` crea un **chunk** con su propio archivo hash en `dist/`. Con rutas por páginas (react-router), lo normal es un lazy por ruta: el usuario solo descarga JS de la página que visita.

## Tailwind y PostCSS

Tailwind v4 se integra como **plugin de Vite** (sin config PostCSS aparte):

```bash
npm install tailwindcss @tailwindcss/vite
```

```js
// vite.config.js
import tailwindcss from '@tailwindcss/vite';
export default defineConfig({ plugins: [react(), tailwindcss()] });
```

```css
/* src/index.css */
@import "tailwindcss";
```

Ver [[TailwindCSS - Instalación y Configuración]] para `@theme` y tokens. Para Sass u otros preprocesadores no hace falta plugin: Vite soporta `.scss` nativamente (instalando `sass` como devDependency — ver [[Qué es Sass]]). Cualquier PostCSS (autoprefixer, etc.) se configura en `postcss.config.js` y Vite lo detecta solo.

## Despliegue

`vite build` produce `dist/`: HTML + JS/CSS hasheados + assets. Es un sitio **estático** — sirve en cualquier host (Netlify, Vercel, GitHub Pages, nginx, S3).

- **Rutas SPA**: al no haber archivos reales para `/dashboard`, el host debe reescribir todo a `index.html` (config "SPA fallback": `netlify.toml` con `/* → /index.html`, `404.html` en GitHub Pages, etc.).
- **Subcarpeta**: si el sitio vive en `misitio.com/app/`, configurar `base: '/app/'` en `vite.config.js` (o `base: './Build/'`) para que los assets se resuelvan relativos. Es el error #1 de despliegue.
- **Verificación**: `npm run preview` local replica cómo se comportará el sitio servido.

## TypeScript

Vite compila TS **sin chequeo de tipos** (eso lo delega a `tsc --noEmit`): el template `react-ts` incluye `"build": "tsc -b && vite build"`. Tipos de `import.meta.env`, aliases y configs se declaran en `vite-env.d.ts` (`/// <reference types="vite/client" />`). Vite y TS se complementan: Vite transpila (rápido), `tsc` valida (riguroso).

## Contexto histórico: webpack, CRA y por qué Vite ganó

Create React App (2016) escondía webpack detrás de `react-scripts`: cero config, pero arranques lentos y ecosistema rígido. En 2023 CRA se de-facto abandonó (el propio equipo de React recomienda frameworks o Vite). La ventaja de Vite no es "menos config" sino la **arquitectura ESM on-demand**: el tiempo de arranque no crece con el proyecto. Next.js/Remix siguen siendo la opción cuando se necesita SSR/rutas del lado servidor; Vite es la base de los SPAs puros y de herramientas como Astro (dev server) o Storybook.

## Errores comunes

1. **`Failed to resolve import "./Component"** — typo en el path o falta la extensión en archivos no-jsx; con alias, olvidar configurarlo en `resolve.alias` (y en `jsconfig.json`/`tsconfig.json` para el editor).
2. **Assets 404 tras el build** — usar `/imagenes/x.png` (raíz) sin `base` correcto al subcarpeta, o meter en `public/` algo que se importaba. Regla: rutas **relativas al HTML** o imports; nada de rutas absolutas arbitrarias.
3. **`window is not defined` / `document is not defined`** — código de navegador ejecutado en SSR; Vite es un bundler de cliente, para SSR usar un framework o `vite-plugin-ssr`.
4. **Env var `undefined`** — no tiene prefijo `VITE_`, o se leyó con `process.env.X` (eso es webpack/CRA; en Vite es `import.meta.env.X`).
5. **API no responde en dev** — falta el proxy o se llama con URL absoluta al backend (CORS). Ver sección Proxy.
6. **Cambio de config ignorado** — `vite.config.js` modificado requiere **reiniciar** el dev server (no hay HMR de la config).
7. **`npm run preview` muestra un sitio roto** — rutas absolutas/base mal configurada; revisar `base` y SPA fallback.

## Cheatsheet

| Comando / Concepto | Valor |
|---|---|
| Crear proyecto | `npm create vite@latest mi-app -- --template react` |
| Dev server | `npm run dev` → `localhost:5173`, HMR instantáneo |
| Build | `npm run build` → `dist/` (Rollup, minificado, hasheado) |
| Preview prod | `npm run preview` → `localhost:4173` |
| Env vars | `VITE_X` en `.env` → `import.meta.env.VITE_X` (build-time) |
| Alias | `resolve.alias` en config |
| Proxy dev | `server.proxy['/api']` |
| Lazy load | `lazy(() => import('./X'))` + `<Suspense>` |
| Assets | `public/` tal cual · `import` con hash |
| Tailwind v4 | plugin `@tailwindcss/vite` |
| Sass | `npm i -D sass` + importar `.scss` (nativo) |
| Motor dev | esbuild (prebundle + transforms) |
| Motor build | Rollup |

## Compatibilidad

- Requiere navegador con ESM (todos los actuales). IE11 no.
- Node ≥ 18 para Vite 5/6.
- SSR: fuera del alcance de Vite vanilla; se necesita framework (Next, Astro, SvelteKit).

## Relacionadas

- [[ReactJS - Fundamentos]] — componentes, estado y props (el código que Vite sirve)
- [[ReactJS - Efectos (useEffect)]] — efectos y StrictMode (el doble invoke en dev)
- [[TailwindCSS - Instalación y Configuración]] — integración con el plugin oficial
- [[Qué es Sass]] — soporte nativo de SCSS en Vite
- [[Node JS]] — el runtime que ejecuta Vite y npm
- [[Formato Json]] — estructura del `package.json` y sus scripts
