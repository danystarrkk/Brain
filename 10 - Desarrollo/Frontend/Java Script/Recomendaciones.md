---
aliases: ["Recomendaciones", "Buenas prácticas JS", "Best Practices JavaScript", "Convenciones JavaScript"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Java Script]]"
  - "[[Sintaxis Moderna]]"
  - "[[Funciones]]"
  - "[[Document Object Model (DOM)]]"
  - "[[Qué es una API]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Guide"
---

# Recomendaciones de código JavaScript

Una recopilación de **buenas prácticas de JavaScript** con la lógica detrás de cada una — no reglas decorativas. El objetivo no es "código bonito": es código que **otra persona (o vos en seis meses)** entienda, depure y modifique sin miedo. Cada sección dice *qué*, *por qué* y *cómo se ve en la práctica*.

> [!info] La jerarquía de los estilos
> El estilo personal importa menos de lo que creés: lo que importa es la **consistencia dentro de un equipo** y la **automatización**. Por eso el punto final de esta nota son linters y formateadores: ellos convierten estas recomendaciones en reglas que el editor aplica solo.

## Formato y estilo base

- **Usá punto y coma `;` al final de cada instrucción.** Técnicamente es opcional: JavaScript tiene *Automatic Semicolon Insertion* (ASI), que inserta los `;` faltantes. Pero ASI tiene reglas traicioneras — ciertas sentencias que empiezan con `(`, `[` o `` ` `` en la línea siguiente se traga el punto y coma anterior y produce bugs surrealistas. Un `;` explícito es código, no opinión.
- **Una instrucción por línea.** `let a = 1; let b = 2;` en la misma línea hace que el diff de git sea ambiguo y la lectura, peor.
- **Indentación consistente** (tabulación o 2 espacios — elegir uno) y **llaves siempre**, incluso en una sola línea: `if (ok) { hacer(); }`. Evita el bug clásico de añadir una línea dentro de un `if` sin llaves.
- **Máximo ~100 caracteres por línea** y **espaciado legible**: `if (condición) {` con espacio, `{ x: 1, y: 2 }` con espacios, no `{x:1,y:2}`.

> [!tip] Casi todo esto se automatiza
> Prettier formatea; ESLint diagnostica. Configurarlos una vez y olvidarse del estilo es la recomendación #1 de cualquier guía moderna — ver "Linting y formateo" al final.

## Nombres: el 80% de la legibilidad

| Regla | Mal | Bien |
|---|---|---|
| `camelCase` para variables, funciones, métodos | `let user_name` | `let userName` |
| `PascalCase` para clases y constructores | `class usuario` | `class Usuario` |
| `SCREAMING_SNAKE_CASE` para constantes globales | | `const API_URL = '...'` |
| Verbos para funciones | `data`, `userList` | `fetchUser`, `renderUsers` |
| Booleanos con prefijo lógico | `active`, `visible` | `isActive`, `hasAccess`, `canEdit` |
| Nombres que digan **qué**, no **cómo** | `userArray`, `dataMap1` | `users`, `activeUsers` |
| Sin abreviaturas crípticas | `u`, `d`, `tmp` | `user`, `duration`, `tempConfig` |

```js
// El nombre debe responder "qué representa", no "cómo se calcula"
const totalConDescuento = subtotal - descuento;   // ✅ qué es
const x = a - b;                                  // ❌ qué es esto?
```

Regla práctica: si un nombre necesita un comentario para explicarse, cambia el nombre y borra el comentario.

## `const` por defecto, `let` con necesidad, `var` nunca

| Declaración | ¿Reasignable? | Alcance | ¿Zona muerta? | Uso |
|---|---|---|---|---|
| `const` | **No** | Bloque `{}` | Sí | El 90% del código: referencias, objetos, funciones |
| `let` | Sí | Bloque `{}` | Sí | Contadores, acumuladores, reasignaciones legítimas |
| `var` | Sí | Función (se escapa del bloque) | No | **Evitar**: legado (ver [[Sintaxis Antigua]]) |

```js
const usuarios = [];          // la referencia no cambia...
usuarios.push(nuevoUsuario);  // ...pero el contenido sí: perfecto

let total = 0;                // sí necesita reasignarse
for (const item of items) {   // cada iteración: const, es nueva variable
  total += item.precio;
}

// var se escapa del bloque: bug clásico
for (var i = 0; i < 3; i++) { /* ... */ }
console.log(i); // 3  ← i "fugó" del for (con let: ReferenceError)
```

> [!info] Por qué `const` primero
> `const` comunica intención: "esta referencia no va a cambiar". Cuando leés código y ves `let`, ya sabés que ahí hay estado mutable que conviene revisar. Además previene errores enteros de bugs de reasignación accidental.

## Comparaciones: `===` siempre

JavaScript tiene dos comparaciones. `==` intenta **convertir** tipos antes de comparar (coerción); `===` compara sin convertir.

```js
'1' == 1    // true  ← '1' se convierte a 1 (engaña)
'1' === 1   // false ← tipos distintos, y bien
0 == ''     // true  ← ¿cuándo eso sería correcto?
0 === ''    // false
null == undefined   // true  (la ÚNICA coerción "razonable" de ==)
null === undefined  // false
NaN === NaN  // false  ← única excepción de ===: usar Number.isNaN()
```

**Política: `===` y `!==` siempre.** Las únicas conversiones que aceptás son las explícitas y tempranas (`Number(x)`, `String(x)`, `Boolean(x)`). Si necesitás tratar `null` y `undefined` igual, escribí `x == null` conscientemente (es el único caso que `==` resuelve sin sorpresas) — o mejor: `x === null || x === undefined`.

> [!warning] Falsiness no basta a veces
> `0`, `''`, `null`, `undefined`, `NaN` y `false` son "falsy", pero `document.all` también. Para default parameters y checks explícitos, preferí `??` (nullish coalescing) que solo cae con `null`/`undefined`:

```js
const puntaje = 0;
puntaje || 10;   // 10  ← ¡el 0 desaparece!
puntaje ?? 10;   // 0   ← correcto: 0 es un dato válido
```

## Funciones: cortas, puras cuando se pueda, con un solo trabajo

- **Una responsabilidad**: si la función hace "y", "además" o "luego limpia", es dos funciones. Las funciones de más de ~20-30 líneas casi siempre esconden varias.
- **Puras cuando se pueda**: una función **pura** recibe entradas y devuelve un resultado sin tocar el mundo exterior (sin DOM, sin red, sin variables externas). Son triviales de testear y de razonar.

```js
// impura: depende del estado externo y del DOM
function mostrarTotal() {
  document.querySelector('#total').textContent =
    subtotal * IVA;                    // ¿qué subtotal? ¿qué IVA?
}

// pura: la lógica es testeable en una línea
function calcularTotal(subtotal, iva) {
  return subtotal * (1 + iva);
}
// y una capa "efecto" separada la pinta en el DOM
function pintarTotal(total) {
  document.querySelector('#total').textContent = total;
}
```

- **Nombres de acciones**: `getUserName()` (obtiene), `isAdult()` (pregunta → booleano), `calculateTotal()` (calcula). Nada de `doStuff()`, `data2()`.
- **Parametros con nombres claros y pocos**: >3-4 parámetros y el orden se olvida — pasá un objeto:

```js
// antes: ¿qué significa el 12?
crearNoticia('Hola', 'texto...', 12, true, null);

// después: auto-documentado
crearNoticia({ titulo: 'Hola', cuerpo: 'texto...', autorId: 12, publicada: true });
```

- **Early return** en vez de anidar: la regla de oro de la legibilidad

```js
// ❌ pirámide de indentación
function procesar(pedido) {
  if (pedido) {
    if (pedido.activo) {
      if (pedido.items.length > 0) {
        return cobrar(pedido);
      }
    }
  }
}

// ✅ guard clauses: el caso raro sale temprano
function procesar(pedido) {
  if (!pedido) return null;
  if (!pedido.activo) return null;
  if (pedido.items.length === 0) return null;
  return cobrar(pedido);
}
```

- **Funciones flecha** (`=>`) para callbacks cortos y cuando importa el `this` léxico; funciones `function` cuando necesitás *hoisting* o `this` dinámico. Detalle en [[Funciones]].

## Sintaxis moderna: usala, no la sufras

Destructuring, spread, template literals y optional chaining eliminan boilerplate entero. Repaso en [[Sintaxis Moderna]]:

```js
// Destructuring: extraer lo que necesitás
const { nombre, email = 'sin-email' } = usuario;      // default si es undefined
const [primero, ...resto] = numeros;

function mostrarPerfil({ nombre, email }) {            // params con nombre
  return `${nombre} <${email}>`;
}

// Spread / rest: copiar y combinar sin mutar
const copia = { ...original, activo: true };
const nums = [1, 2, ...otros];

// Template literals (nunca concatenar con +)
const saludo = `Hola ${nombre}, tenés ${count} mensajes`;

// Optional chaining y nullish: acceder sin encadenar ifs
const ciudad = usuario?.direccion?.ciudad ?? 'N/D';

// Object.entries para iterar objetos
for (const [clave, valor] of Object.entries(config)) {
  console.log(`${clave}: ${valor}`);
}

// Array methods encadenados (ver [[Array Methods]])
const activos = usuarios.filter((u) => u.activo).map((u) => u.nombre);
```

> [!warning] `forEach` no es `map`
> `map` **devuelve** un nuevo arreglo transformado; `forEach` devuelve `undefined` y es solo un bucle con efectos. Encadenar `map` y asignar a otra variable es correcto; no "chainees" `forEach`.

## Manejo de errores: `try`/`catch`/`finally` y errores propios

```js
try {
  const datos = JSON.parse(entrada);      // puede lanzar SyntaxError
  guardar(datos);
} catch (err) {
  console.error('No se pudo guardar:', err.message);
} finally {
  cerrarLoader();   // se ejecuta SIEMPRE: éxito o error
}
```

| Tipo de error | Cuándo | Ejemplo |
|---|---|---|
| `SyntaxError` | Código/imparseable | `JSON.parse('{')` |
| `TypeError` | Valor de tipo incorrecto | `null.x`, `f() pero f es undefined` |
| `ReferenceError` | Variable inexistente | typo en el nombre |
| `RangeError` | Fuera de rango | `new Array(-1)` |
| `EvalError` / `URIError` | Muy raros | legado |

Reglas:

- **Solo atrapá lo que podés manejar.** Un `catch` vacío (`catch {}` sin nada) es un agujero negro: el error desaparece y el bug se vuelve imposible de encontrar. Si no podés hacer nada, al menos logueá.
- **Lanzá errores con significado**: `throw new Error('Falta el email')` o, mejor, errores tipados:

```js
class ErrorValidacion extends Error {
  constructor(campo, mensaje) {
    super(mensaje);
    this.name = 'ErrorValidacion';
    this.campo = campo;
  }
}

function guardarUsuario(u) {
  if (!u.email?.includes('@')) {
    throw new ErrorValidacion('email', 'El email no es válido');
  }
}

// Uso: distinguir tipos de error sin parsear strings
try {
  guardarUsuario(datos);
} catch (err) {
  if (err instanceof ErrorValidacion) {
    resaltarCampo(err.campo, err.message);
  } else {
    throw err;   // los que no conozco, que suban
  }
}
```

- **`async/await` + try/catch**: una promesa rechazada dentro de `await` cae en el `catch` — pero **solo** si hay `try`; sin él, la promesa rechazada queda *unhandled* y el error se pierde. Ver [[Qué es una API]] para el patrón completo con `fetch`.

## Organización con módulos ES

JavaScript moderno se organiza en módulos (`import`/`export`) — ver [[Java Script]]:

```js
// utils/precios.js — nombres explícitos en los exports
export function calcularTotal(subtotal, iva = 0.21) {
  return subtotal * (1 + iva);
}
export const IVA_ES = 0.21;

// app.js
import { calcularTotal } from './utils/precios.js';
import calcularIva from './iva.js';   // default: UNA cosa por módulo
```

- Cada archivo = un tema. La función "helper" que usan tres módulos va a `utils/`, no duplicada.
- Preferí **named exports** (el buscador del editor los renombra bien); `default` solo para el export principal de un módulo-clase/componente.
- `import` se **eleva** (hoisted): siempre se resuelve al inicio, antes del resto del código.

## Rendimiento: hábitos que cuentan

- **No medir antes de tiempo.** El 95% del código de interfaz es perfectamente rápido. Mirá el perfil (DevTools → Performance) antes de optimizar.
- **No bloquear el hilo principal** con bucles gigantes: para datasets grandes, `setTimeout` por lotes, Web Workers, o procesar por chunks.
- **DOM batch**: leé y escribí el DOM en grupos separados. Leer-escribir-leer-escribir en bucle fuerza *reflow* en cada iteración:

```js
// ❌ un reflow por iteración
for (const item of items) {
  const alto = item.offsetHeight;   // lectura
  item.style.height = alto * 2 + 'px';  // escritura → reflow
}

// ✅ primero todas las lecturas, después todas las escrituras
const altos = items.map((el) => el.offsetHeight);
items.forEach((el, i) => { el.style.height = altos[i] * 2 + 'px'; });
```

- **Event delegation**: un solo listener por contenedor en vez de uno por cientos de hijos — menos memoria y funciona con elementos añadidos dinámicamente:

```js
// un listener para toda la lista, aunque se agreguen items después
lista.addEventListener('click', (e) => {
  const item = e.target.closest('.item');   // ¿quién fue?
  if (!item) return;
  item.classList.toggle('seleccionado');
});
```

- **Animá `transform` y `opacity`**, nunca `width`/`top` (ver [[Librería GSAP]] y [[Animaciones]]).
- **Deburré eventos de alta frecuencia** (`scroll`, `mousemove`, `resize`): los handlers corren decenas de veces por segundo; usá *throttle/debounce* o `requestAnimationFrame` — patrón completo en [[Librería Parallax]].

## Flujo de depuración

1. **`console.log` con contexto**, no suelto: `console.log('pedido:', pedido)` o mejor aún:
   - `console.table(usuarios)` → tabla legible de arreglos de objetos.
   - `console.dir(obj)` → árbol real del objeto (no su representación como string).
   - `console.group()`/`groupEnd()` para agrupar.
   - `console.trace()` → quién llamó a esta función.
   - *Trace points* / `debugger;` → pausa sin editar líneas (se puede insertar con clic izquierdo en el gutter de DevTools).
2. **`debugger` condicional**: clic derecho sobre el `debugger` → *Edit condition breakpoint* — solo pausa cuando se cumple.
3. **Breakpoints de DevTools**: pausar en un listener de evento, en una función por nombre (*Logpoints* para imprimir sin pausar).
4. **Lee el stack trace de verdad**: `TypeError: Cannot read properties of null (reading 'x')` te dice *qué propiedad* y *dónde* — la línea del error es el punto de partida, no "algo falló".
5. **Bisectar**: si algo funciona y de repente no, `git log`/`git diff` + comentar mitad del código es más rápido que mirar todo.

## Seguridad básica

- **Nunca `eval()`, `new Function()` con strings, ni `setTimeout('código string', t)`**: ejecutan texto como código — es una puerta de inyección directa y también más lento. Pasá funciones:

```js
setTimeout(() => hacerAlgo(x), 500);   // ✅
setTimeout('hacerAlgo(x)', 500);       // ❌ jamás
```

- **`innerHTML` con datos externos = XSS**. Cualquier string de usuario o API pintado como HTML puede contener `<script>` o `onerror=`. Regla: datos → `textContent`; HTML solo de fuentes de confianza:

```js
// ❌ XSS: el usuario puso <img src=x onerror=alert(1)> en su nombre
celda.innerHTML = `Hola ${usuario.nombre}`;

// ✅ datos como texto
celda.textContent = `Hola ${usuario.nombre}`;

// ✅ si necesitás HTML dinámico, construí nodos
const li = document.createElement('li');
li.className = 'usuario';
li.textContent = usuario.nombre;
```

- **Validá siempre en el servidor**: la validación del frontend es UX, no seguridad — se saltea con DevTools o `curl` (ver [[Qué es una API]]).
- **No confíes en `innerHTML` ni en la consola del usuario**: todo lo que el navegador puede hacer, un atacante también.
- **Credenciales**: ni API keys secretas en el bundle del navegador ni tokens en `localStorage` sin entender el riesgo — ver la sección de auth de [[Qué es una API]].

## Legibilidad: comentarios y estilo de código

- **El código se comenta solo con buenos nombres**: `calcularTotal()` necesita menos comentario que `calc2()`.
- **Comentá el *por qué*, no el *qué***:

```js
// ❌ comenta lo obvio
// suma IVA al total
total = subtotal * 1.21;

// ✅ comenta lo no obvio
// IVA 21% hardcodeado: la ley cambió en 2026, ver #142
total = subtotal * 1.21;
```

- **Código muerto se borra**: git lo recuerda. `if (false)`, funciones comentadas, imports sin usar — basura.
- **Una constante con nombre > un mágico**: `const MAX_INTENTOS = 3` en vez del `3` suelto repetido.

## Linting y formateo: la automatización que lo garantiza

| Herramienta | Qué hace | Rol |
|---|---|---|
| **Prettier** | Formatea el código al guardar (sangría, comas, comillas) | Estilo automático e innegociable |
| **ESLint** | Diagnostica patrones peligrosos: variables sin uso, `==`, `var`, errores comunes | Reglas del equipo + detección de bugs |
| **EditorConfig** | Unifica indentación/final de línea entre editores | Consistencia mínima |

```json
// .eslintrc.json — reglas representativas
{
  "rules": {
    "eqeqeq": ["error", "always"],          // prohíbe ==
    "no-var": "error",                       // prohíbe var
    "prefer-const": "error",
    "no-console": "warn",
    "no-unused-vars": "error"
  }
}
```

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2
}
```

- En VS Code: extensions de Prettier + ESLint + "Format on save".
- `no-console` en `warn` es un buen punto: la consola es para depurar, no para producción (ver [[La Consola]]).
- En CI, correr `eslint .` y `prettier --check .` evita discusiones de estilo en los reviews.

## Ruta de aprendizaje dentro del vault

Practicar estas recomendaciones en orden:

1. [[Java Script]] — fundamentos del lenguaje.
2. [[Sintaxis Moderna]] — la JS de hoy (const/let, arrows, spread, módulos).
3. [[Funciones]] — el corazón: callbacks, closures, `this`.
4. [[Arreglos]], [[Objetos]], [[Array Methods]] — datos.
5. [[Document Object Model (DOM)]] y [[Acceder a Html]] — pintar en pantalla.
6. [[Eventos]] — reaccionar al usuario.
7. [[Qué es una API]] — salir del navegador y consumir datos reales.
8. [[Recomendaciones|este note]] — consolidar el estilo mientras se escribe código grande.

## Errores comunes

1. **`=` donde iba `==`** — mitad de los bugs de condición: `if (x = 5)` *asigna* y siempre es truthy.
2. **`const` pensando que congela el contenido**: congela la referencia; el objeto/array de adentro sigue siendo mutable.
3. **`catch` vacío**: el error desaparece y nadie sabe por qué la app "no hace nada".
4. **`innerHTML` con datos de usuario**: XSS instantáneo.
5. **Listeners por elemento** en listas grandes (rendimiento + no alcanza a los elementos nuevos) → delegación.
6. **Optimizar sin medir**: horas ahorrando milisegundos que no molestaban.
7. **Comentar código muerto en vez de borrarlo**.
8. **Comparar con `==` "porque funciona"**: funciona hasta que `0 == ''` te cobra.

## Cheatsheet

| Tema | Regla |
|---|---|
| `;` | Siempre explícito (no confíes en ASI) |
| Sangría | 2 espacios (o tabs), consistente; llaves siempre |
| Variables | `const` → `let` → `var` (nunca) |
| Comparación | `===` / `!==` (excepto `x == null` consciente) |
| Defaults | `??` para null/undefined, no `\|\|` |
| Funciones | Una responsabilidad, puras cuando se pueda, early return |
| Parámetros | >3-4 → pasar objeto con nombres |
| Errores | `try/catch/finally`; errores propios con `instanceof`; nada de `catch` vacío |
| Módulos | ES `import`/`export`; named exports por defecto |
| DOM | Leer todo → escribir todo; delegación de eventos |
| Animación | `transform`/`opacity` solo |
| Eventos frecuentes | throttle/debounce/`requestAnimationFrame` |
| Datos al DOM | `textContent`, nunca `innerHTML` |
| `eval`/strings dinámicos | Prohibido |
| Comentarios | El *por qué*, no el *qué* |
| Estilo | Prettier + ESLint, no debates |

## Relacionadas

- [[Java Script]] — fundamentos del lenguaje
- [[Sintaxis Moderna]] — la sintaxis que estas recomendaciones asumen
- [[Funciones]] — profundidad en callbacks, closures y `this`
- [[Document Object Model (DOM)]] — sobre qué se aplican los hábitos de rendimiento
- [[Acceder a Html]] — selección segura de elementos
- [[Eventos]] — delegación y handlers
- [[La Consola]] — herramientas de depuración
- [[Qué es una API]] — seguridad y errores en el consumo de red
- [[Librería GSAP]] — animación con rendimiento correcto
