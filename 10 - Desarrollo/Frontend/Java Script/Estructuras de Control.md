---
aliases: ["Control de flujo", "Condicionales en JavaScript"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Booleanos]]"
  - "[[Funciones]]"
  - "[[Iteradores]]"
fuente: "https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Statements/if...else"
---

# Estructuras de Control

Las **estructuras de control** deciden qué código se ejecuta y en qué orden. Las dos grandes familias son las **condicionales** (`if`, `switch`, ternario) y los **operadores lógicos** (`&&`, `||`, `!`) que combinan condiciones. Los bucles viven en [[Iteradores]].

## Comparadores: la base de toda condición

Antes del `if`, conviene dominar los comparadores, porque de ahí sale el booleano que la condición evalúa:

| Operador | Nombre | Ejemplo | Resultado |
| -------- | ------ | ------- | --------- |
| `==` | igual (con coerción de tipos) | `1000 == '1000'` | `true` |
| `===` | igual estricto (valor **y** tipo) | `1000 === '1000'` | `false` |
| `!=` | distinto (con coerción) | `1000 != 2000` | `true` |
| `!==` | distinto estricto | `1000 !== '1000'` | `true` |
| `>` `<` `>=` `<=` | mayor / menor | `300 >= 300` | `true` |

```js
const puntaje = 1000;

if (puntaje == '1000') {
  console.log('El puntaje sí es igual (con coerción)');
}

if (puntaje === '1000') {
  console.log('Nunca entra: number !== string');
} else {
  console.log('No es igual');
}
```

> [!warning] Usa `===` por defecto
> `==` convierte tipos antes de comparar y produce falsos positivos históricos (`0 == ''` es `true`). Con `===` lo que ves es lo que obtienes. Ver [[Booleanos]] para más sobre coerción.

## `if` / `else`

```js
const dinero = 300;
const totalApagar = 300;

if (dinero > totalApagar) {
  console.log('Sí podemos pagar');
} else if (dinero === totalApagar) {
  console.log('Pagamos exacto');
} else {
  console.log('Fondos insuficientes');
}
```

Sin llaves también funciona, pero solo para una línea: es válido y más corto, aunque las llaves explícitas evitan bugs famosos donde "se coló" una instrucción dentro del `if`.

```js
if (puntaje === 1000) console.log('¡Puntaje exacto!');
else console.log('Sigue intentando');
```

## Cadena de `else if`

Cuando hay varios rangos, se encadenan condiciones. Un caso típico: calificar un puntaje.

```js
const puntaje = 400;

if (puntaje >= 300 && puntaje < 400) {
  console.log('Buen puntaje... felicidades');
} else if (puntaje >= 400 && puntaje < 500) {
  console.log('Excelente');
} else {
  console.log('Puntaje fuera de rango');
}
```

## `switch`

`switch` compara **igualdad estricta** (`===`) de una misma expresión contra varios valores. Es más legible que una cadena enorme de `else if` cuando se compara contra constantes.

```js
function pagar() {
  console.log('Pagando....');
}

const metodoPago = 'efectivo';

switch (metodoPago) {
  case 'efectivo':
    pagar(); // se pueden invocar funciones
    console.log(`Pagaste con ${metodoPago}`);
    break; // sin break, "caería" al siguiente case (fall-through)
  case 'cheque':
    console.log(`Pagaste con ${metodoPago}`);
    break;
  case 'tarjeta':
    console.log(`Pagaste con ${metodoPago}`);
    break;
  default:
    console.log('Aún no se soporta este método de pago');
    break;
}
```

> [!warning] No olvides el `break`
> Sin `break`, la ejecución continúa al siguiente `case` (cascada). El único `fall-through` válido es el intencional: `case 'a': case 'b':` sin código intermedio para agrupar valores.

## Operadores lógicos

```js
// && (AND): ambas condiciones deben ser true
const usuario = true;
const puedePagar = false;

if (usuario && puedePagar) {
  console.log('Es usuario y sí puede pagar');
} else if (!usuario) {
  console.log('No estás registrado');
} else {
  console.log('No tienes fondos suficientes');
}

// || (OR): basta con una condición verdadera
const efectivo = 300;
const credito = 1000;

if (efectivo > 600 || credito > 600) {
  console.log('Sí podemos pagar');
}

// ! (NOT): niega un valor booleano
const autenticado = true;
if (!autenticado) {
  console.log('Inicia sesión primero');
}
```

Los operadores lógicos hacen **cortocircuito**: en `a && b`, si `a` es false no evalúa `b`; en `a || b`, si `a` es true tampoco. Eso permite patrones como `esUsuario && mostrarSaludo()` (ver [[Funciones]]).

## Ternario

El ternario es un `if/else` como **expresión** — se puede asignar o imprimir directamente:

```js
const autenticado = true;
const puedePagar = true;

// condición ? valorSiTrue : valorSiFalse
console.log(autenticado ? 'Sí está autenticado' : 'No está autenticado');

// Dos condiciones combinadas con &&
console.log(autenticado && puedePagar ? 'Sí puede pagar' : 'No puede pagar');
```

> [!warning] El ternario anidado es ilegible
> `a ? b ? c ? 'x' : 'y' : 'z' : 'w'` funciona, pero nadie lo entiende al leer. Si necesitas más de un nivel, usa `if/else` o una función. Regla práctica: **máximo un ternario por línea y que sea obvio**.

## `if` anidado

Un `if` dentro de otro aplica cuando las condiciones dependen entre sí:

```js
const autenticado = true;
const puedePagar = true;
const efectivo = 300;
const credito = 400;
const disponible = efectivo + credito;
const totalPagar = 700;

if (autenticado) {
  if (puedePagar) {
    if (disponible >= totalPagar) {
      console.log('Sí puede pagar');
    } else {
      console.log('No hay saldo suficiente');
    }
  } else {
    console.log('No puede pagar');
  }
}
```

Se puede simplificar combinando en una sola condición: `if (autenticado && puedePagar && disponible >= totalPagar)`. Anidar más de dos niveles suele ser señal de que conviene extraer el criterio a una función con nombre.

## Relacionadas

- [[Booleanos]] — truthy/falsy y el tipo booleano
- [[Iteradores]] — bucles: `for`, `while`, `for...of`
- [[Funciones]] — extraer condiciones complejas a funciones con nombre
- [[Operadores]] — operadores de JavaScript en detalle
