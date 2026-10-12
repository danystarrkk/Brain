---
aliases: ["Node", "NodeJS", "Node.js"]
tags:
  - dev/backend/node
tipo: concepto
estado: terminado
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[ReactJS - Vite]]"
  - "[[Formato Json]]"
  - "[[Qué es una API]]"
  - "[[TailwindCSS - Instalación y Configuración]]"
  - "[[Document Object Model (DOM)]]"
  - "[[Bash Scripting - Fundamentos Básicos]]"
  - "[[Python - Redes y HTTP]]"
  - "[[Python - Concurrencia Threading Multiprocessing Asyncio]]"
fuente: "https://nodejs.org/docs/latest/api/"
---

# Node JS

Referencia profunda de **Node.js**: arquitectura V8 + libuv, las fases del event loop y las microtareas, el thread pool, gestión de paquetes (npm/npx/corepack/yarn/pnpm), módulos CJS vs ESM con su tabla de interoperabilidad, `fs` en sus tres estilos, streams, servidores HTTP desde cero, el proceso (env, argv, códigos de salida, señales con receta de apagado ordenado), manejo de errores globales, `package.json` campo por campo, semver, rendimiento, debugging y seguridad de supply chain. Verificado con **Node v24.21.0** (V8 13.6) salvo indicación.

> [!info] Qué es Node por dentro
> Node.js no es "JavaScript con filesystem": es **V8** (el motor JIT de Chrome, escrito en C++) ejecutando tu JS, envuelto en **libuv** (biblioteca en C de event loop, I/O no bloqueante, thread pool, señales, child processes) más los bindings de la stdlib (`fs`, `net`, `dns`, `http`). Tu código corre en **un solo hilo** (el main thread); libuv gestiona las partes que esperan. Es la misma arquitectura de concurrencia cooperativa que asyncio en Python (ver [[Python - Concurrencia Threading Multiprocessing Asyncio]]): un hilo, muchas tareas esperando.

## El event loop: fases y microtareas

Cuando tu código termina de ejecutarse, el proceso **no termina** si hay handles vivos: el loop de libuv sigue. Cada vuelta recorre fases, en este orden:

| Fase | Qué ejecuta | API |
|---|---|---|
| **timers** | Callbacks vencidos | `setTimeout`, `setInterval` |
| **pending callbacks** | I/O con retraso (algunas) | — |
| **idle/prepare** | Internas de libuv | — |
| **poll** | I/O: completar callbacks de sockets/archivos | `fs` callbacks, `net` |
| **check** | Inmediatos después del poll | `setImmediate` |
| **close** | Cierres | `socket.on("close")` |

**Microtareas** (NO son fases): `process.nextTick` (primero) y la cola de promesas — se drenan **entre** fases y callbacks, hasta vaciarse. Verificado con CJS en Node 24:

```js
// orden OBSERVADO (corrido en Node 24):
// process.nextTick -> Promise.then -> queueMicrotask -> setImmediate -> setTimeout(0)
```

> [!warning] `setTimeout(0)` vs `setImmediate`: orden no garantizado
> Ambos agendados desde el **módulo principal**: la carrera la decide si el loop ya entró en poll (documentado como no determinista; en las 40 corridas de verificación aquí ganó `setImmediate`). Lo **sí** determinista: dentro de un callback de timer, `setTimeout(0)` corre primero; dentro de `setImmediate`, `setImmediate` primero. Regla práctica: no los mezcles para ordear — usá promesas.

```js
// La lección operativa de las microtareas:
setTimeout(() => console.log("1: timer (macrotarea)"), 0);
setImmediate(() => console.log("2: immediate (macrotarea)"));
Promise.resolve().then(() => console.log("3: microtarea — ANTES de todo timer"));
process.nextTick(() => console.log("4: nextTick — antes incluso que las promesas"));
queueMicrotask(() => console.log("5: queueMicrotask (estándar web, igual que Promise)"));
```

**Por qué importa**: un `while` infinito o un `JSON.parse` de 2 GB en el main thread congela **todo** — timers, respuestas, todo (mismo problema que `time.sleep` en un event loop de asyncio). Trabajo pesado → `worker_threads` o proceso externo.

### El thread pool de libuv

Varias APIs del módulo `fs` (todo el sync no-blocked), `dns.lookup`, `zlib`, `crypto` y `child_process.exec` ejecutan su syscall real en **un pool de 4 hilos** (`UV_THREADPOOL_SIZE`, ajustable hasta 1024). Consecuencias: 5 operaciones de disco simultáneas con el pool default = 4 corren y la 5ª espera. Si tu app es intensiva en `fs`/`crypto` con CPU, subir `UV_THREADPOOL_SIZE=16` ayuda (por variable de entorno, no en runtime).

## Instalación y versiones

```bash
node -v          # runtime (p.ej. v24.21.0)
npm -v           # gestor
```

- **LTS vs Current**: para proyectos usá siempre **LTS** (soporte extendido, la que garantizan las herramientas); Current trae features nuevas sin promesa de estabilidad.
- Gestores de versión: `nvm` (bash), `fnm` (rápido, rust), `volta` (multi-lenguaje). `.nvmrc` fija la versión del proyecto:
  ```
  # .nvmrc
  24
  ```
  `nvm use` lo lee; CI y Vercel/Netlify también. Mejor aún: `"engines"` en `package.json` (ver tabla de campos).

## npm, npx, corepack, yarn, pnpm

| Herramienta | Qué es | Cuándo |
|---|---|---|
| **npm** | Gestor de paquetes por defecto; el mayor registro del mundo | Siempre disponible; suficiente para la mayoría |
| **npx** | Ejecuta un paquete **sin instalarlo globalmente** | `npx create-vite`, `npx tsc --init`, CLIs puntuales |
| **corepack** | Habilita yarn/pnpm **versionados** vía `packageManager` | Proyectos que estandarizan yarn/pnpm |
| **yarn** (classic/berry) | Alternativa de npm; berry añade PnP | Ecosistemas que ya lo usan |
| **pnpm** | Instalación en store global + hardlinks: rápido, disco ahorrado, estricto | Monorepos y CI con caché |

```bash
npm init -y                     # package.json mínimo
npm install express             # dependency (producción)
npm install -D vitest           # devDependency (solo desarrollo)
npm install -g npm@latest       # global: CLIs de máquina (no libs)
npx vitest run                  # ejecuta sin instalar en el proyecto
npm ci                          # CI: instala EXACTO del lockfile, borra node_modules
npm outdated && npm update
npm audit                       # CVEs; npm audit fix
```

> [!tip] `npm install` vs `npm ci`
> `install` actualiza `package-lock.json` si hace falta; `ci` es determinista: **falla** si el lockfile no coincide con `package.json`. En CI siempre `ci` — builds reproducibles o nada.

## Módulos: CommonJS vs ES Modules

Node soporta dos sistemas. La interop cambió con el tiempo — estado verificado en Node 24:

| Aspecto | CommonJS (`require`) | ESM (`import`) |
|---|---|---|
| Archivo | `.cjs` o `.js` sin `"type": "module"` | `.mjs` o `.js` con `"type": "module"` |
| Declarar | `module.exports = X` / `exports.X = X` | `export const X = ...` / `export default X` |
| Consumir | `const X = require("./x")` | `import X from "./x.js"` (**extensión obligatoria**) |
| `__dirname` / `__filename` | ✅ disponibles | ❌ **`ReferenceError: __dirname is not defined`** (verificado) → `import.meta.dirname` (21.2+) o `fileURLToPath(import.meta.url)` |
| `require()` del otro lado | `require()` de un ESM puro fallaba (ERR_REQUIRE_ESM) | — |
| Cargar ESM desde CJS | ✅ **`require(esm)` funciona desde Node 22.12/23 sin flag** (verificado en 24: `[Module: null prototype] { x: 1, default: [Function] }`) | — |
| Síncrono | ✅ todo | `import` top-level sí; `await import()` dinámico |
| Resolución | `node_modules` + `paths` | ESM resolution: `exports`/`imports` de package.json |
| Hot-reload/eval | `require.cache` | Módulos inmutables |

```js
// CJS (math.cjs)
exports.sumar = (a, b) => a + b;
// consumidor CJS
const { sumar } = require("./math.cjs");
```

```js
// ESM — archivo math.mjs (el módulo)
export const sumar = (a, b) => a + b;
export default function ejecutar() {}
```

```js
// ESM — archivo consumidor.mjs (el import; notá la extensión obligatoria)
import principal, { sumar } from "./math.mjs";
```

> [!warning] La confusión número uno: paquetes duales
> Un paquete publica `require` (CJS) e `import` (ESM) en `exports`. El **default export** de un módulo ESM requerido desde CJS llega como `m.default` (verificado); los named exports quedan como propiedades. Por eso `const express = require("express")` a veces trae `{ default: fn }` y hay que hacer `.default`. Regla: en código nuevo, `"type": "module"` desde el día uno y olvidarte de la transición.

## `fs`: sincrónico, callback, promesa

```js
import fs from "node:fs";
import fsp from "node:fs/promises";

// 1) Sync — bloquea el loop COMPLETO. Solo en: arranque, CLI, módulos top-level
const datos = fs.readFileSync("config.json", "utf8");

// 2) Callback — estilo legacy, callback-first (err, result)
fs.readFile("config.json", "utf8", (err, datos) => {
  if (err) return manejar(err);     // el error SIEMPRE es el primer argumento
  console.log(datos);
});

// 3) Promesas — el estándar moderno (verificado: node:fs/promises)
const datos2 = await fsp.readFile("config.json", "utf8");
await fsp.writeFile("salida.txt", "hola", "utf8");
await fsp.mkdir("dir/sub", { recursive: true });
const st = await fsp.stat("config.json");
console.log(st.isFile(), st.size, st.mtime);
for await (const entrada of await fsp.opendir("src")) console.log(entrada.name);
```

| Estilo | Bloquea | Uso legítimo |
|---|---|---|
| `*Sync` | **Sí**, el event loop entero | Arranque (`loadConfig`), scripts one-shot |
| Callback | No (thread pool para disco) | Código legado |
| Promesas/`await` | No | **Default en código nuevo** |

`fs.existsSync()` está deprecado (usá `try/catch` con `access`); escribir archivos grandes: `createReadStream`/`createWriteStream` (streams abajo) — un `readFileSync` de 8 GB mata el proceso.

## Streams: procesar datos sin cargarlos en memoria

Un stream es una secuencia de datos disponibles por chunks. Cuatro tipos: `Readable` (fuente), `Writable` (sumidero), `Duplex` (ambas), `Transform` (modifica y emite). Verificado con `pipeline`:

```js
import { pipeline } from "node:stream/promises";
import { createReadStream, createWriteStream } from "node:fs";
import { Readable, Writable } from "node:stream";

// Copia de archivo con memoria constante + manejo de errores + backpressure automático
await pipeline(createReadStream("grande.zip"), createWriteStream("copia.zip"));

// Streams propios
const fuente = Readable.from(["a\n", "b\n", "c\n"]);
const sumidero = new Writable({
  write(chunk, encoding, callback) {
    process.stdout.write(chunk);
    callback();                       // llamar callback libera el siguiente chunk
  },
});
await pipeline(fuente, sumidero);

// Transform (pipeline: entrada → transform → salida)
import { Transform } from "node:stream";
const mayusculas = new Transform({
  transform(chunk, enc, cb) { cb(null, chunk.toString().toUpperCase()); },
});
```

`pipeline` reemplaza al viejo `.pipe()`: **propaga errores** y libera los streams al terminar (con `pipe()` un error en la fuente dejaba el proceso colgado). Backpressure: si el sumidero es lento, la fuente se pausa sola — no buffers infinitos (análogo a los `chunk` de requests de [[Python - Redes y HTTP]]).

## Servidor HTTP desde cero (sin frameworks)

```js
import http from "node:http";

const server = http.createServer(async (req, res) => {
  if (req.method === "GET" && req.url === "/json") {
    res.writeHead(200, { "Content-Type": "application/json" });   // ver [[Formato Json]]
    res.end(JSON.stringify({ mensaje: "Hola" }));
  } else if (req.method === "POST" && req.url === "/echo") {
    const chunks = [];
    for await (const c of req) chunks.push(c);          // req es un Readable
    const body = Buffer.concat(chunks).toString();
    res.writeHead(201, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ recibido: JSON.parse(body || "{}") }));
  } else {
    res.writeHead(404); res.end("no encontrado");
  }
});

server.listen(3000, "127.0.0.1", () => console.log("http://127.0.0.1:3000"));
```

Verificado en Node 24 contra un cliente real: `GET /json` → `200 {"mensaje":"Hola"}`, `GET /otra` → `404`. `res` es un `Writable`; `req` es un `Readable` de chunks (por eso el body se concatena, igual que el framing de TCP en la nota de Redes). Frameworks (Express/Fastify/Hono) añaden routing, body parsing, middleware — pero entender esta capa explica cada error 431/timeout que veas.

## El proceso: `process`

```js
process.env.NODE_ENV            // variables de entorno (siempre strings)
process.env.PORT ?? 3000
process.argv                    // [execPath, scriptPath, ...args] — args desde índice 2
process.exitCode = 1            // salida ordenada: termina cuando el loop se vacía
process.exit(1)                 // salida INMEDIATA: skip de finally/cleanup → evitar
process.cwd(), process.chdir("/")
process.pid, process.versions.node, process.versions.v8
process.uptime(), process.memoryUsage()   // rss/heapUsed/external
process.hrtime.bigint()         // nanosegundos: medir duraciones
```

### Señales y apagado ordenado (receta verificada)

En contenedores/PM2/systemd, el shutdown llega por **SIGTERM** (y SIGINT por Ctrl+C). Receta:

```js
process.on("SIGTERM", () => {
  console.log("SIGTERM recibido: cerrando ordenadamente...");
  server.close(() => process.exit(0));        // corta conexiones nuevas, deja terminar las vivas
  setTimeout(() => process.exit(0), 10_000).unref();  // red de seguridad si algo se cuelga
});
```

Verificado con `kill -TERM`: el handler corre, `exitCode`/`process.exit(0)` → **código de salida 0**. Sin handler, Node muere con el código de señal (128+15 = 143) y las conexiones en vuelo se cortan. Siempre distinguí: `SIGTERM` (apagá bien), `SIGINT` (usuario), `SIGHUP` (terminal cerrada; en algunos setups = "recargá config").

### Códigos de salida

| Código | Significado |
|---|---|
| `0` | Éxito |
| `1` | Error genérico / `process.exitCode` no seteado en crash |
| `2` | Uso incorrecto de la CLI (argumentos mal) |
| `78` (EX_CONFIG) | Configuración inválida — convención en CLIs |

Para un CLI: `process.exitCode = 2` + mensaje de uso; para un servidor: `1` al crash irrecuperable.

## Manejo de errores globales

```js
// 1) Unhandled rejection: promesa rechazada sin catch (Node ≥15: CRASHEA por defecto)
process.on("unhandledRejection", (reason) => {
  logger.fatal({ reason }, "promesa rechazada sin manejar");
  process.exit(1);            // en servidores: morí ruidoso; un proceso con estado desconocido es peor
});

// 2) Errores NO recuperables (fallos de C++, JSON corrupto en sync, OOM)
process.on("uncaughtException", (err) => {
  logger.fatal(err, "excepción no capturada");
  process.exit(1);            // la política segura: NO seguir corriendo
});

// 3) Errores en handlers de señal: a veces se ignoran; acá también
```

> [!warning] `uncaughtException` NO es un salvavidas
> Documentado por el propio equipo de Node: si ignorás la excepción (`process.exit(0)` o seguir), el **estado interno del proceso es desconocido** (pool de buffers a medias, locks de sqlite, transacciones abiertas). Política correcta: loguear y `exit(1)`; el orquestador (systemd, Docker, PM2) lo reinicia. Recuperación real solo con `express`-level error middleware y `try/catch` donde corresponde — el handler global es la red de seguridad, no el manejo.

`try/catch` **no** captura errores de callbacks asíncronos ni rechazos de promesas sin await — la causa del 90% de los "errores invisibles" heredados. Por eso `--unhandled-rejections=strict` es el default moderno.

## `package.json`: campo por campo

```jsonc
{
  "name": "mi-app",              // minúsculas, sin espacios; el identificador del registro
  "version": "1.4.2",            // semver — ver siguiente sección
  "description": "Una app",
  "keywords": ["cli", "util"],
  "license": "MIT",
  "type": "module",              // .js = ESM (sin esto: CJS)
  "main": "dist/index.cjs",      // legacy: entrypoint para require() viejo
  "module": "dist/index.mjs",    // no estándar; bundlers lo prefieren
  "exports": {                   // entrypoints MODERNOS (mapa de rutas de import)
    ".": { "import": "./dist/index.mjs", "require": "./dist/index.cjs" },
    "./cli": "./dist/cli.js"
  },
  "bin": { "mi-cli": "./bin/cli.js" },   // al instalar: comando `mi-cli` en el PATH
  "files": ["dist"],             // whitelist de publicación (npm pack --dry-run para verlo)
  "engines": { "node": ">=22" }, // advertencia/fallo según engine-strict
  "packageManager": "pnpm@9.12.0", // corepack fija el gestor y su versión
  "scripts": {
    "dev": "node --watch src/index.js",
    "test": "vitest run",
    "lint": "eslint .",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/index.js",
    "prepublishOnly": "npm run build && npm test"
  },
  "dependencies": { "express": "^4.19.0" },      // runtime
  "devDependencies": { "vitest": "^2.0.0" },     // solo desarrollo (no en prod)
  "optionalDependencies": { "fsevents": "^2.3.3" } // si falla la instalación: sigue
}
```

Ganchos de lifecycle: `pre<script>`/`post<script>` corren antes/después (`npm test` dispara `pretest` → `test` → `posttest`); `prepare` corre en install local y antes de publicar — el lugar de `husky install`. Ver [[Formato Json]] para la sintaxis.

### Scripts npm: el "Makefile de JavaScript"

```jsonc
{
  "scripts": {
    "dev": "node --watch src/index.js",          // reinicia al guardar (nativo desde 18.11)
    "dev:api": "npm run dev --workspace=api",    // --workspace repasa argumentos
    "test": "vitest",
    "test:ci": "vitest run --coverage",
    "ci": "npm run lint && npm run test:ci && npm run build"
  }
}
```

npm **encadena** con `&&` y hereda `node_modules/.bin` en el PATH del script (por eso `vitest` funciona sin ruta). En scripts, usá el binario local; los `npx` dentro de scripts son redundantes.

## Semver

`MAJOR.MINOR.PATCH` — rompiste compatibilidad / agregaste features / arreglasto bug. Los rangos del `package.json`:

| Rango | Significado | Ejemplo |
|---|---|---|
| `^1.2.3` | ≥1.2.3 <2.0.0 | cambios menores/patch automáticos |
| `~1.2.3` | ≥1.2.3 <1.3.0 | solo patch |
| `1.2.3` | Exacto | pin total |
| `>=1.2.3` | Sin techo | **peligroso**: rompe sin avisar |
| `*` / `latest` | Cualquiera | registros |
| `1.x` / `1.2.x` | Cualquier menor/patch | — |
| `git+https://...` | Directo del repo | forks, builds tempranos |

Regla de oro: antes de `1.0.0`, MINOR puede romper (semver lo permite). Y `npm ci` + lockfile es lo que hace reproducible el `^`.

## Node vs Deno vs Bun

| Criterio | Node | Deno | Bun |
|---|---|---|---|
| Runtime | V8 + libuv | V8 + Tokio (Rust) | JavaScriptCore (Safari) |
| Lenguaje | JS/TS vía tooling | **TS nativo**, JS, WASM | JS/TS nativo |
| Seguridad | Todo acceso por defecto | **Sandbox explícito** (`--allow-net`...) | Sin sandbox como Node |
| Gestor | npm/pnpm/yarn | `deno install` (URLs + npm compat) | `bun install` (rápido) |
| Stdlib | npm ecosystem | std deno.land + compat npm creciente | API compatible con Node |
| Maduro | Muy (producción desde 2009) | Creciente | Rápido; APIs aún moving |
| Elegir cuando… | Producción, ecosistema, hosting | Scripts/CLIs seguros, TS-first | Speed, tooling integrado (bundler/test) |

Para [[ReactJS - Vite]] y la mayoría del ecosistema npm hoy: Node es el terreno seguro; Deno/Bun son opciones reales para CLIs y servicios nuevos, con la salvedad de compatibilidad de dependencias nativas.

## Debugging

```bash
node --inspect src/index.js            # inspector en ws://127.0.0.1:9229
node --inspect-brk src/index.js        # pausa en la PRIMERA línea (antes de tu código)
node --watch src/index.js              # reload en cambios
# desde Chrome: chrome://inspect → tu proceso; desde VS Code: launch.json "type": "node"
```

En vivo: breakpoints, `console.table`, `console.time/timeEnd`, `node:trace` (`node --trace-exit` explica quién mató el proceso), `NODE_OPTIONS="--trace-warnings"`. Para rendimiento: **Performance tab** del inspector (event loop, HTTP, GC) y `perf_hooks`:

```js
import { performance, monitorEventLoopDelay } from "node:perf_hooks";
const h = monitorEventLoopDelay({ resolution: 10 }); h.enable();
const t0 = performance.now(); await trabajo();
console.log(`duración ${performance.now() - t0}ms, lag medio ${h.mean / 1e6}ms`); // verificable
```

**Event loop lag**: si `h.mean` supera decenas de ms, algo pesado corre en el main thread. **Cluster**: `node:cluster` forkea el proceso por núcleo (el loop es de un hilo; multi-core requiere procesos — mismo dilema GIL/threads de [[Python - Concurrencia Threading Multiprocessing Asyncio]]). En la práctica: contenedores con 1 vCPU + réplicas horizontales > cluster, salvo servicios Node crudos de alto throughput.

## Seguridad

```bash
npm audit                 # CVEs en el árbol de dependencias
npm audit fix             # parches compatibles
npm audit fix --force     # ⚠ puede saltar major versions
npm ls --depth=0          # qué tenés instalado realmente
```

Supply chain — el riesgo real del ecosistema npm:

1. **`package.json` malicioso**: scripts `preinstall` que corren al **instalar** (no solo al publicar). Paquetes typosquatted (`lodahs` en vez de `lodash`) — revisá el nombre antes de aceptar.
2. **Lockfile es tu contrato**: commiteá `package-lock.json`; en CI usá `npm ci`; revisá diffs del lockfile en PRs (una dependencia nueva = un diff enorme si te lo muestran).
3. **Menos deps = menos superficie**: `npm ls` de un proyecto Vite típico supera los 300 paquetes transitivos; no agregues una librería por 5 líneas.
4. **Herramientas**: `npm audit` + Dependabot/Renovate + `socket.dev`/`osv-scanner` para detección de comportamiento sospechoso.
5. **Secrets**: `process.env` + `.env` en `.gitignore` (el módulo `dotenv` lo carga en dev); jamás en el bundle. El historial de git es eterno — ver [[Python Ofensivo]] para el mismo vector en Python.

## Errores comunes

1. **Trabajo pesado en el main thread** → el loop se congela entero; `worker_threads`, proceso, o servicio aparte.
2. **`readFileSync`/`JSON.parse` gigantes en runtime** → memoria + latencia; streams y parseo incremental.
3. **`require` de un ESM o rutas sin extensión en ESM** → `ERR_MODULE_NOT_FOUND`/`ERR_REQUIRE_ESM` (y recuerda: `require(esm)` recién funciona desde 22.12).
4. **Usar `__dirname` en ESM** → `ReferenceError` (verificado); `import.meta.dirname`.
5. **Body sin `for await (const c of req)`** → body vacío; `req` es un stream.
6. **`uncaughtException` + seguir corriendo** → estado interno desconocido; log + exit(1).
7. **`process.exit()` en el onclose de un servidor** → corta respuestas en vuelo; `server.close()` primero.
8. **`npm install` en CI en vez de `npm ci`** → builds no deterministas.
9. **Lockfile sin commitear** → "en mi máquina sí" infinito.
10. **Global installs para libs** → las libs van en `dependencies`; global solo CLIs (o mejor `npx`/`packageManager`).
11. **`^` en dependencias con releases diarios** → surprises en cada deploy; lockfile + auditar updates.

## Cheatsheet

| Tema | Comando/API | Una línea |
|---|---|---|
| Versiones | `node -v`, `npm -v`, `.nvmrc` + `nvm use` | LTS en proyectos |
| Ejecutar | `node archivo.js`, `node -e "..."`, REPL: `node` | — |
| Watch | `node --watch src/index.js` | Reload nativo |
| Paquetes | `npm i`, `npm i -D x`, `npm ci`, `npx x` | `ci` en CI siempre |
| ESM | `"type": "module"` + `import x from "./y.js"` | Extensión obligatoria |
| dirname en ESM | `import.meta.dirname` | `__dirname` no existe |
| fs moderno | `await fsp.readFile(...)` | Sync solo en arranque |
| Streams | `pipeline(src, dst)` | Errores + backpressure |
| HTTP | `http.createServer((req, res) => ...)` | req/res son streams |
| Env | `process.env.X ?? default` | Siempre string |
| Args | `process.argv.slice(2)` | Índice 2 en adelante |
| Salida | `process.exitCode = 1` | Mejor que `exit()` |
| Shutdown | handler SIGTERM → `server.close()` | Receta verificada |
| Errores globales | `unhandledRejection` + `uncaughtException` → log + exit(1) | No seguir tras crash |
| Perf | `monitorEventLoopDelay` | Lag medio > 50ms = problema |
| Debug | `node --inspect-brk` + chrome://inspect | Pause en línea 1 |
| Seguridad | `npm audit`, lockfile revisado | Supply chain es el riesgo |

## Relacionadas

- [[ReactJS - Vite]] — el toolchain de frontend que corre sobre Node (npm, npx, el dev server)
- [[TailwindCSS - Instalación y Configuración]] — CLIs vía npm/npx
- [[Formato Json]] — el formato de `package.json` y los lockfiles
- [[Qué es una API]] — el contrato HTTP que los servidores Node implementan
- [[Document Object Model (DOM)]] — lo que Node **no** tiene: el DOM vive en el navegador
- [[Bash Scripting - Fundamentos Básicos]] — la shell desde donde se invocan node/npm y los scripts de CI
- [[Python - Concurrencia Threading Multiprocessing Asyncio]] — la misma semántica de event loop, comparada
- [[Python - Redes y HTTP]] — `http.server`/`requests` frente a `http.createServer`/fetch
