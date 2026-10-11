---
aliases: ["useEffect", "Efectos React", "Hooks useEffect"]
tags:
  - dev/frontend/react
tipo: concepto
estado: borrador
creado: 2026-10-11
actualizado: 2026-10-11
relacionado:
  - "[[ReactJS - Fundamentos]]"
fuente: "https://react.dev/learn/synchronizing-with-effects"
---

# ReactJS - Efectos (useEffect)

`useEffect` es el hook para ejecutar **código que queda fuera del render**: timers, peticiones a APIs, suscripciones a eventos del navegador, etc. Se usa solo para eso — **no** para calcular valores derivados, que van directo en el cuerpo del componente.

> [!warning] ¿Por qué no directamente en el cuerpo del render?
> El cuerpo del componente se ejecuta en **cada** render. Un `setTimeout` o `subscribe` escrito ahí se re-crea indefinidamente (bucle infinito de renders o fugas de memoria). `useEffect` controla **cuándo** se ejecuta ese código.

## Sintaxis básica

```jsx
import { useState, useEffect } from 'react';

function Contador() {
  const [valor, setValor] = useState(0);

  useEffect(() => {
    // Este bloque se ejecuta DESPUÉS del render
    const timer = setTimeout(() => {
      setValor((v) => v + 1);
    }, 3000);

    // Cleanup: se ejecuta antes del siguiente effect (y al desmontar)
    return () => clearTimeout(timer);
  }, []); // [] = dependencias: solo se ejecuta al montar

  return <div>Contador: {valor}</div>;
}
```

Anatomía:

1. **Primer argumento**: la función con el efecto. Corre *después* de que React pinta en el DOM.
2. **Segundo argumento**: el arreglo de **dependencias**.
3. **Return opcional (cleanup)**: se ejecuta antes de correr el effect de nuevo y al desmontar el componente — el lugar correcto para limpiar timers, conexiones y suscripciones.

## El arreglo de dependencias

| Dependencias | Cuándo corre el efecto |
|---|---|
| `[]` | Solo al montar el componente (una vez) |
| `[valor]` | Al montar y **cada vez que `valor` cambia** |
| Sin arreglo | En **cada** render (casi siempre un error) |

```jsx
useEffect(() => {
  document.title = `Contador: ${valor}`;
}, [valor]); // reacciona a cambios de `valor`
```

## Ejemplo práctico: obtener datos de una API

```jsx
function Usuarios() {
  const [usuarios, setUsuarios] = useState([]);
  const [cargando, setCargando] = useState(true);

  useEffect(() => {
    let activo = true; // bandera para ignorar respuestas tardías

    fetch('/api/usuarios')
      .then((res) => res.json())
      .then((datos) => {
        if (activo) {
          setUsuarios(datos);
          setCargando(false);
        }
      });

    return () => { activo = false; };
  }, []);

  if (cargando) return <p>Cargando....</p>;
  return <ul>{usuarios.map((u) => <li key={u.id}>{u.nombre}</li>)}</ul>;
}
```

El renderizado condicional con `cargando` sigue el patrón de [[ReactJS - Fundamentos]], y el listado usa `key` único por elemento.

## Reglas de oro

- **Efectos lo mínimo posible**: todo lo que pueda calcularse en cada render va en el cuerpo del componente, no en un effect.
- **Siempre limpiar** lo que tenga cleanup (`setTimeout`, `setInterval`, `addEventListener`, subscriptions, AbortController en `fetch`).
- **Dependencias completas**: si el effect usa una variable del scope, agrégala al arreglo — omitirla genera closures viejas y bugs difíciles de cazar.
- Dos effects **no** se ejecutan en orden garantizado si son asincrónicos.

## Relacionadas

- [[ReactJS - Fundamentos]] — componentes, estado y props
- [[Funciones]] — closures y scope (por qué las dependencias importan)
