---
aliases: ["Forms", "Formularios HTML"]
tags:
  - dev/frontend/html
tipo: concepto
estado: terminado
creado: 2025-01-24
actualizado: 2026-10-11
relacionado:
  - "[[Html]]"
  - "[[Etiquetas]]"
  - "[[Eventos]]"
  - "[[Acceder a Html]]"
fuente: "https://developer.mozilla.org/es/docs/Web/HTML/Element/form"
---

# Formularios en HTML

Los formularios son el canal principal para **recibir datos del usuario**: registros, búsquedas, comentarios, pagos. En HTML se construyen con un puñado de etiquetas, siendo las centrales `<form>`, `<input>` y `<label>`.

## La base: `form`, `label` e `input`

Los formularios inician con la etiqueta `<form>` (buena práctica siempre, aunque el input funcione "suelto"). A partir de ahí usamos `<input>`, cuyo atributo `type` define el tipo de campo, y `<label>`, el texto descriptivo del input:

```html
<form>
  <label for="nombre">Nombre: </label>
  <input id="nombre" type="text" placeholder="Tu nombre">
</form>
```

- **`for` del label + `id` del input**: deben coincidir. Esto vincula el texto con el campo: al hacer clic en el label se enfoca el input, y los lectores de pantalla anuncian correctamente cada campo. Es la regla de oro de los formularios accesibles.
- **`placeholder`**: texto de ejemplo *dentro* del campo. **No reemplaza al `label`**: desaparece al escribir y no lo anuncian todos los lectores de pantalla.

> [!warning] Un `id` es único en la página
> Si dos campos comparten `id`, el `for` del label apunta al primero que encuentre y la validación/navegación se rompe.

## `type`: los diferentes tipos de `input`

El mismo `<input>` cambia por completo según su `type` — muchos tipos traen **validación y controles nativos** del navegador gratis:

| `type` | Comportamiento |
|---|---|
| `text` | Campo de texto libre (el por defecto) |
| `password` | Oculta lo que se escribe (con botón de mostrar/ocultar en los navegares modernos) |
| `email` | Valida formato de correo; en móviles muestra el teclado con `@` |
| `tel` | En móviles despliega el teclado numérico telefónico |
| `url` | Valida formato de dirección web |
| `number` | Solo números, con `min`, `max` y `step`; flechas incrementales |
| `date` / `time` / `datetime-local` | Selectores nativos de fecha y hora |
| `search` | Campo de búsqueda (estilo nativo y botón de limpiar) |
| `color` | Selector de color |
| `range` | Deslizador |
| `radio` | Opción múltiple **exclusiva**: los radios del mismo `name` se anulan entre sí |
| `checkbox` | Opción múltiple independiente |
| `file` | Selector de subida de archivos (`accept` limita el tipo) |
| `submit` | Botón que envía el formulario |
| `reset` | Botón que limpia todos los campos |

```html
<input type="radio" name="tamaño" id="chico">
<label for="chico">Chico</label>
<input type="radio" name="tamaño" id="grande">
<label for="grande">Grande</label>

<input type="submit" value="Enviar">
```

## Atributos de los campos

Los tipos configuran *qué es* el campo; los atributos configuran *cómo se comporta*:

- **`value`**: valor inicial del campo (y el dato que viaja al enviar).
- **`readonly`**: el campo no se puede modificar, pero **sí** se envía.
- **`disabled`**: el campo está inactivo y **no** se envía.
- **`required`**: validación nativa — el formulario no se envía si el campo está vacío.
- **`min` / `max` / `minlength` / `maxlength`**: límites numéricos o de longitud.
- **`pattern`**: expresión regular que el valor debe cumplir.
- **`autocomplete`**: autocompletado del navegador (`name`, `email`, `off`…).

```html
<input type="email" name="correo" required autocomplete="email"
       placeholder="tu@correo.com" maxlength="80">
```

## Agrupar con `fieldset` y `legend`

`<fieldset>` agrupa campos relacionados dentro de un recuadro, y `<legend>` le pone el título — clave en formularios largos para que el usuario (y los lectores de pantalla) entiendan qué contiene cada grupo:

```html
<form>
  <fieldset>
    <legend>Datos de contacto</legend>
    <label for="nombre">Nombre: </label>
    <input id="nombre" type="text" placeholder="Tu nombre">
  </fieldset>
</form>
```

> [!warning] Etiquetas siempre cerradas
> `<legend>`, `<fieldset>`, `<option>` y `<select>` son etiquetas **con cierre**: escribir `<legend/>` o cerrar con la propia etiqueta sin barra (`<option>…<option>`) es inválido y los navegares terminan "tragando" el resto del formulario.

## `<textarea>`

Para párrafos largos (comentarios, mensajes) usamos `<textarea>`, que sí tiene cierre, y su valor por defecto se escribe **entre las etiquetas**:

```html
<textarea rows="20" cols="40">Texto por defecto</textarea>
```

- **`rows`** (plural, no `row`): cantidad de líneas visibles.
- **`cols`**: ancho en columnas de caracteres.
- En la práctica hoy se redimensiona/construye con CSS — ver [[CSS Introducción]] — y `rows`/`cols` quedan como fallback de tamaño inicial.

## `<select>`

Crea un desplegable de opciones; a diferencia de los inputs, **tiene etiquetas internas**: cada opción es un `<option>`:

```html
<select name="cafe">
  <option value="0" selected>Elige un café</option>
  <option value="1">Espresso</option>
  <option value="2">Latte</option>
  <option value="3" disabled>Flat white (agotado)</option>
</select>
```

- **`value`**: el dato que se envía al servidor (el texto visible es solo para el usuario).
- **`selected`**: opción marcada por defecto.
- **`disabled`**: opción que no puede elegirse.
- **`<optgroup label="…">`**: agrupa opciones bajo un subtítulo.

## Enviar el formulario

`<form>` define a dónde y cómo van los datos:

```html
<form action="/api/registro" method="POST">
```

- **`action`**: URL que recibe los datos.
- **`method`**: `GET` (los datos viajan en la URL — para búsquedas) o `POST` (en el cuerpo — para crear/actualizar datos).

El envío dispara el evento `submit`, que se controla con JavaScript para validar o enviar por `fetch` sin recargar la página — ver [[Eventos]] y [[Acceder a Html]].

> [!tip] Validación nativa como red de seguridad
> `required`, `type="email"`, `pattern`, `min`/`max` te dan validación del navegador sin escribir JS. Es la primera línea de defensa; la validación con JS complementa, pero nunca debería ser la única.

## Relacionadas

- [[Html]] — base del lenguaje
- [[Etiquetas]] — catálogo general de etiquetas
- [[Eventos]] — capturar el `submit` y los cambios de cada campo
- [[Acceder a Html]] — leer valores de los inputs desde JavaScript
- [[CSS Introducción]] — estilar formularios con selectores y pseudo-clases (`:focus`, `:invalid`)
