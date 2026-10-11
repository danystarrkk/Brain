---
aliases: ["Cómo se ejecuta JavaScript", "Etiqueta script"]
tags:
  - dev/frontend/js
tipo: concepto
estado: terminado
creado: 2025-01-25
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Document Object Model (DOM)]]"
  - "[[Acceder a Html]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Element/script"
---

# Java Script en una página HTML

JavaScript es el lenguaje que da **comportamiento** a la página: mientras [[Html]] define la estructura y CSS el estilo, JS decide qué pasa cuando el usuario interactúa. Para que el navegador ejecute JavaScript, el código debe estar dentro de la página de alguna de estas dos formas: **inline** (dentro del HTML) o en un **archivo externo**. En ambos casos, la buena práctica es cargarlo **antes del cierre de `</body>`**.

## Dentro del HTML (inline)

```html
<script>
  alert('Hola Mundo');
</script>
```

La etiqueta `<script>` le dice al navegador que el contenido es JavaScript y debe interpretarse. `alert` es una función propia del navegador que muestra un cuadro de diálogo.

> [!warning] Con `alert` basta para aprender, no para producir
> `alert` **bloquea** el hilo de ejecución hasta que el usuario acepta. En código real se usa `console.log` (ver [[La Consola]]) o el DOM.

## Archivo externo (forma recomendada)

Lo recomendable es separar el JavaScript en su propio archivo, por ejemplo en una carpeta `js/` con un archivo llamado `app.js`:

![[Pasted image 20250125080131.png]]

El archivo contiene solo JavaScript, **sin** etiqueta `<script>`:

```js
// js/app.js
alert('Hola Mundo');
```

Y se enlaza desde el HTML:

```html
<script src="js/app.js"></script>
```

Ventajas de la forma externa:

- **Cacheo**: el navegador guarda `app.js` y no lo vuelve a descargar en cada página.
- **Mantenimiento**: el HTML queda limpio; el JS se edita en su propio archivo.
- **Reutilización**: el mismo archivo puede usarse en varias páginas.

## ¿Dónde colocar el `<script>`?

El orden importa: el navegador descarga y ejecuta el JS mientras construye el HTML. Si el script usa `document.querySelector(...)` y corre antes de que ese elemento exista, obtiene `null` y falla. Por eso la regla histórica es **colocar el `<script>` justo antes del cierre de `</body>`**: todo el HTML ya está parseado.

```html
<body>
  ...contenido...
  <script src="js/app.js"></script>
</body>
```

### `defer` y `async`

El atributo `defer` moderniza ese consejo: permite poner el `<script>` en el `<head>` sin bloquear el parseo del HTML:

```html
<script src="js/app.js" defer></script>
```

| Atributo | Descarga | Ejecución |
| -------- | -------- | --------- |
| *(ninguno)* | bloquea el parseo | en cuanto baja, detiene el HTML |
| `defer` | en paralelo | **tras** terminar el parseo, en orden |
| `async` | en paralelo | en cuanto baja, donde esté el parseo |

> [!tip] Regla práctica
> Usa `defer` para tus scripts (se ejecutan con el DOM listo y en el orden correcto). Reserva `async` para scripts independientes que no dependen del DOM ni de otros scripts, como un widget de terceros.

## El puente entre el script y la página

Una vez cargado, el script accede a la página a través del objeto `document` y su árbol [[Document Object Model (DOM)|DOM]]:

```js
document.querySelector('h1').textContent = 'Hola desde JavaScript';
document.addEventListener('DOMContentLoaded', () => {
  // el HTML ya está listo: seguro trabajar con el DOM
});
```

Ver [[Acceder a Html]] para seleccionar y modificar elementos, y [[Eventos]] para reaccionar a las interacciones del usuario.

## Relacionadas

- [[Html]] — el lenguaje donde se incrusta el script
- [[Document Object Model (DOM)]] — la estructura que el script manipula
- [[Acceder a Html]] — seleccionar y modificar elementos del DOM
- [[Eventos]] — hacer la página interactiva
- [[La Consola]] — `console.log`, la herramienta de depuración básica
