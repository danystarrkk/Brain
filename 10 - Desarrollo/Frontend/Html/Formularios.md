---
tags:
  - desarrollo
  - frontend
  - formularios
tipo: concepto
estado: terminado
---

# Formularios en HTML

Los formularios son el canal principal para **recibir datos del usuario**: registros, búsquedas, comentarios, pagos. En HTML se construyen con un puñado de etiquetas, siendo las centrales `<form>`, `<input>` y `<label>`. Y son el lugar donde HTML deja de ser "solo contenido" para convertirse en **interacción**: aquí el navegador aporta teclados especiales, validación, autocompletado y envío nativos que ningún framework debería reimplementar desde cero.

## La base: `form`, `label` e `input`

Los formularios inician con la etiqueta `<form>` (buena práctica siempre, aunque el input funcione "suelto"). A partir de ahí usamos `<input>`, cuyo atributo `type` define el tipo de campo, y `<label>`, el texto descriptivo del input:

```html
<form>
  <label for="nombre">Nombre: </label>
  <input id="nombre" name="nombre" type="text" placeholder="Tu nombre">
</form>
```

### Asociar label e input: los tres caminos

1. **`for` + `id`** (el canónico): deben coincidir. Al hacer clic en el label se enfoca el input, y los lectores de pantalla anuncian el label como nombre accesible del campo.

```html
<label for="correo">Correo:</label>
<input id="correo" name="correo" type="email">
```

2. **Envolver el input dentro del label** (no necesita `for` ni `id`):

```html
<label>Correo: <input name="correo" type="email"></label>
```

3. **ARIA de último recurso**: cuando el diseño no permite label visible, `aria-label="Correo"` (texto solo para tecnologías asistivas) o `aria-labelledby` apuntando a otro elemento. **Nunca** como primera opción: no hay texto visible para quien necesita ampliación de tamaño.

> [!warning] Un `id` es único en la página
> Si dos campos comparten `id`, el `for` del label apunta al primero que encuentre y la validación/navegación se rompe. Los `id` duplicados son el bug #1 de formularios copiados y pegados.

### `placeholder` no es un label

`placeholder` es texto de ejemplo **dentro** del campo: desaparece al escribir, no lo anuncian todos los lectores de pantalla consistentemente, y su contraste bajo es un fallo WCAG recurrente. Sirve para **formato de ejemplo** (`dd/mm/aaaa`, `tu@correo.com`), nunca como sustituto del label.

## `type`: los tipos de `input` con comportamiento completo

El mismo `<input>` cambia por completo según su `type`: muchos tipos traen **validación y controles nativos** del navegador gratis. El tipo por defecto, si se omite, equivale a `text` (y un valor desconocido cae a `text` también).

| `type` | Comportamiento | Validación nativa | Notas |
|---|---|---|---|
| `text` | Campo de texto libre | — | El por defecto |
| `password` | Oculta lo escrito (botón mostrar/ocultar en navegadores modernos) | — | `autocomplete="current-password"` / `"new-password"` lo potencian |
| `email` | Teclado con `@` en móvil | Formato de correo (multi-valores con `multiple`) | `multiple` acepta `a@b.com, c@d.com` |
| `tel` | Teclado numérico telefónico | Ninguna (formatos varían por país) | Los `+`, espacios y guiones son válidos |
| `url` | Teclado con `/` | Formato URL (requiere esquema: `https://…`) | |
| `number` | Flechas incrementales | Rango con `min`/`max`/`step` | **Admite `e` y signos**; `inputmode="decimal"` suele ser mejor |
| `search` | Estilo nativo + botón de limpiar | — | `inputtype=search` en Chrome |
| `date` | Selector de fecha nativo | `min`/`max`/`step` (días) | Valor interno ISO `2026-03-12` |
| `time` | Selector de hora | `min`/`max`/`step` (segundos) | Valor `14:30` |
| `datetime-local` | Fecha + hora sin zona | `min`/`max` | No existe `type="datetime"` (fue removido) |
| `month` | Año-mes | `min`/`max` | `2026-03` |
| `week` | Semana ISO | `min`/`max` | `2026-W10` |
| `range` | Deslizador | `min`/`max`/`step` (no bloquea, solo limita el slider) | Usar con `<output>` para mostrar el valor |
| `color` | Selector de color nativo | — | Solo hex simple `#rrggbb`, sin alfa |
| `checkbox` | Opción múltiple **independiente** | — | `indeterminate` (solo vía JS) para estado mixto |
| `radio` | Opción múltiple **exclusiva** | — | Los del mismo `name` se anulan entre sí |
| `file` | Selector de subida | `accept`/`multiple` (solo UX, no validación fuerte) | Ver sección de subida |
| `submit` | Botón que envía el formulario | — | Puede ir fuera del `<form>` con `form="id"` |
| `image` | Botón-envío como imagen | — | Legado; `alt` obligatorio |
| `reset` | Botón que limpia todos los campos | — | Rara vez conviene (rompe "undo") |
| `hidden` | Campo oculto, **sí se envía** | — | Tokens CSRF, IDs; no sirve para datos sensibles |

```html
<input type="radio" name="tamaño" id="chico">
<label for="chico">Chico</label>
<input type="radio" name="tamaño" id="grande">
<label for="grande">Grande</label>

<input type="submit" value="Enviar">
```

> [!info] El teclado virtual es un control de UX real
> En móvil, `email`, `tel`, `url`, `number` y `search` cambian el teclado; `inputmode="numeric" | "decimal" | "text"` aplica ese efecto a elementos no-input (como un `div` editable). Elegir el tipo correcto recorta errores antes de que existan.

## Atributos de los campos

Los tipos configuran *qué es* el campo; los atributos configuran *cómo se comporta*:

| Atributo | Efecto |
|---|---|
| `name` | **Clave del dato al enviar** — sin `name`, el campo no viaja en el formulario |
| `value` | Valor inicial (y el dato que viaja al enviar) |
| `readonly` | No se puede modificar, pero **sí** se envía y se puede copiar/focus |
| `disabled` | Inactivo, **no** se envía, no recibe foco, se ve apagado |
| `required` | Validación nativa: sin valor, el formulario no se envía |
| `min` / `max` | Límites numéricos o de fecha (según `type`) |
| `minlength` / `maxlength` | Longitud mínima/máxima en caracteres |
| `step` | Granularidad de incremento (`number`, `range`, fechas) |
| `pattern` | Expresión regular que el valor debe cumplir (solo `text` y afines) |
| `placeholder` | Texto de ejemplo interno |
| `autocomplete` | Tokens de autocompletado del navegador |
| `autofocus` | Recibe el foco al cargar (uno solo por página) |
| `inputmode` | Teclado virtual: `numeric`, `decimal`, `email`, `tel`, `search`, `none` |
| `spellcheck` | Autocomprobación ortográfica (`true`/`false`) |
| `tabindex` | Orden de tabulado (mejor no tocarlo) |
| `form` | Asocia el campo a un `<form>` por `id`, aunque esté fuera |
| `list` | Vincula un `<datalist>` de sugerencias |

```html
<input type="email" name="correo" required autocomplete="email"
       placeholder="tu@correo.com" maxlength="80">
```

### `readonly` vs `disabled`: la diferencia que pierde datos

Ambos "apagan" el campo, pero solo `disabled` lo excluye del envío. Patrón típico: un campo calculado se muestra `readonly` (el usuario lo ve y viaja al servidor); un campo inaplicable en el momento se `disabled` (no viaja). **Cuidado**: si tu lógica deshabilita campos, el backend no los recibirá — no los uses para "proteger" datos.

## Validación nativa (Constraint Validation)

HTML trae un motor de validación completo, gratis y accesible:

### Atributos de validación

```html
<form>
  <label for="usuario">Usuario (3–12 letras/números):</label>
  <input id="usuario" name="usuario" required minlength="3" maxlength="12"
         pattern="[A-Za-z0-9]+" title="Solo letras y números">
  <button type="submit">Registrarse</button>
</form>
```

| Atributo | Aplica a | Comportamiento |
|---|---|---|
| `required` | todos | Vacío = inválido |
| `minlength`/`maxlength` | `text`, `search`, `url`, `tel`, `email`, `password` | Longitud del valor |
| `min`/`max` | `number`, `range`, fechas | Rango del valor |
| `step` | igual que min/max | Múltiplo de step (con `min` como base) |
| `pattern` | `text` y afines (no `email`/`url`, que ya validan) | Regex **implícitamente anclada** (`^(?:…)$`); validar todo el valor |
| `type="email"`/`url` | — | Formato según spec |
| `novalidate` | `<form>` | Desactiva toda la validación nativa al enviar |

- El envío **nativo** cancela si hay campos inválidos y enfoca el primero con un mensaje burbuja.
- `pattern` usa la sintaxis de JS regex sin barras; añadí `title` o `aria-describedby` con la regla: el mensaje nativo es genérico ("usa el formato…").
- `setCustomValidity("mensaje")` (JS) inyecta mensajes propios; `setCustomValidity("")` los limpia.

### La API de Constraint Validation (desde JS)

Para validación en vivo (al escribir, no al enviar) el navegador expone propiedades y métodos en cada campo:

```js
const campo = document.querySelector('#usuario');

campo.validity;        // objeto ValidityState: valueMissing, tooShort, patternMismatch,
                       // rangeUnderflow, rangeOverflow, stepMismatch, badInput, typeMismatch, valid
campo.validationMessage; // mensaje nativo del navegador (i18n gratis)
campo.checkValidity(); // true/false — lanza evento 'invalid' si es false
campo.reportValidity(); // valida Y muestra la burbuja nativa

// Patrón de validación en vivo
campo.addEventListener('input', () => {
  campo.setCustomValidity(
    campo.value.includes(' ') ? 'Sin espacios, por favor.' : ''
  );
});
```

| Propiedad `validity` | Se activa cuando… |
|---|---|
| `valueMissing` | `required` y vacío |
| `typeMismatch` | formato de `email`/`url` mal |
| `patternMismatch` | no cumple `pattern` |
| `tooShort` / `tooLong` | `minlength`/`maxlength` (tooLong solo si el usuario escribe de más; el límite de `maxlength` lo previene) |
| `rangeUnderflow` / `rangeOverflow` | fuera de `min`/`max` |
| `stepMismatch` | no es múltiplo de `step` |
| `badInput` | entrada ilegible (por ejemplo texto en `type="number"`) |
| `valid` | `true` si todo lo anterior está en `false` |

> [!tip] `:invalid` y `:valid` en CSS
> El navegador expone el estado a estilos: `input:invalid` (tras intentar enviar), `input:user-invalid` (solo tras interactuar — mejor UX), `input:required`, `input:valid`. Con `:has()` puedes ocultar/mostrar mensajes de error por campo. Ver [[CSS Introducción]].

## `<textarea>`

Para párrafos largos (comentarios, mensajes) usamos `<textarea>`, que sí tiene cierre, y su valor por defecto se escribe **entre las etiquetas**:

```html
<textarea name="mensaje" rows="6" cols="40" maxlength="500"
          placeholder="¿Qué te gustó del café?" required>Texto por defecto</textarea>
```

- **`rows`** (plural, no `row`): líneas visibles. **`cols`**: ancho en caracteres — fallback de tamaño inicial; en la práctica se redimensiona con CSS (ver [[CSS Introducción]]).
- A diferencia de `<input>`, **sí puede contener salto de línea en el markup** y el contenido interno es su valor inicial; el valor escrito se lee con `.value`, nunca con `textContent`.
- `required` valida que no esté vacío (espacios cuentan); no tiene `minlength` implícito por `rows`.

## `<select>`, `<option>` y `<optgroup>`

Crea un desplegable de opciones; a diferencia de los inputs, **tiene etiquetas internas**: cada opción es un `<option>`:

```html
<label for="cafe">Café:</label>
<select id="cafe" name="cafe" required>
  <option value="" disabled selected>Elige un café</option>
  <optgroup label="Clásicos">
    <option value="espresso">Espresso</option>
    <option value="latte">Latte</option>
  </optgroup>
  <optgroup label="Especiales">
    <option value="flatwhite" disabled>Flat white (agotado)</option>
    <option value="cortado">Cortado</option>
  </optgroup>
</select>
```

- **`value`**: el dato que se envía; el texto visible es solo para el usuario (el `label` vacío inicial evita mandar "Elige un café" como dato — ojo con eso en el backend).
- **`selected`**: marcada por defecto; **`disabled`**: no elegible; **`multiple`**: selección múltiple (Ctrl/Cmd-click, difícil en móvil — en ese caso mejor checkboxes).
- **`<optgroup label="…">`**: agrupa opciones bajo un subtítulo no seleccionable.
- El `size` la convierte en lista visible de N filas en vez de desplegable.
- **`<datalist>`** ofrece **sugerencias** bajo un `<input>` (autocompletado con opciones, pero el usuario puede escribir lo que quiera):

```html
<label for="pais">País:</label>
<input id="pais" name="pais" list="paises">
<datalist id="paises">
  <option value="Argentina">
  <option value="Brasil">
  <option value="Chile">
</datalist>
```

  Ojo: en algunos navegadores el datalist no muestra las etiquetas, solo los `value`.

## `<button>`

Un `<button>` sin `type` es **`type="submit"` por defecto** — la sorpresa más frecuente al meter botones dentro de un formulario:

```html
<form>
  <button type="button" onclick="...">No envía nada</button>
  <button>Envía el formulario (submit implícito)</button>
  <button type="reset">Limpia los campos</button>
</form>
```

Regla: **todo `<button>` dentro de un form necesita `type="button"` explícito** si no es de envío. `<button>` admite contenido HTML (iconos, `<span>`), a diferencia de `<input type="submit">`.

## Agrupar con `fieldset` y `legend`

`<fieldset>` agrupa campos relacionados dentro de un recuadro, y `<legend>` le pone el título — clave en formularios largos para que el usuario (y los lectores de pantalla) entienda qué contiene cada grupo:

```html
<form>
  <fieldset>
    <legend>Datos de contacto</legend>
    <label for="nombre">Nombre: </label>
    <input id="nombre" name="nombre" type="text" placeholder="Tu nombre">
    <label for="tel">Teléfono: </label>
    <input id="tel" name="tel" type="tel">
  </fieldset>

  <fieldset>
    <legend>Preferencias</legend>
    <input type="radio" name="tostado" id="t-claro"><label for="t-claro">Claro</label>
    <input type="radio" name="tostado" id="t-oscuro"><label for="t-oscuro">Oscuro</label>
  </fieldset>
</form>
```

- El `legend` es el **nombre accesible del grupo**: los lectores de pantalla anuncian "Preferencias, grupo" al entrar.
- `disabled` en el fieldset desactiva todos sus campos (que además no se envían).
- `disabled` en el fieldset desactiva todos sus campos (y no se envían).

> [!warning] Etiquetas siempre cerradas
> `<legend>`, `<fieldset>`, `<option>` y `<select>` son etiquetas **con cierre**: escribir `<legend/>` o cerrar con la propia etiqueta sin barra (`<option>…<option>`) es inválido y los navegadores terminan "tragando" el resto del formulario.

## El `<form>` y el envío

`<form>` define a dónde y cómo van los datos:

```html
<form action="/api/registro" method="POST" enctype="multipart/form-data">
```

| Atributo | Función |
|---|---|
| `action` | URL que recibe los datos (ausente = la propia URL) |
| `method` | `GET` (datos en la URL) o `POST` (en el cuerpo); `GET` es el defecto |
| `enctype` | Codificación del cuerpo: `application/x-www-form-urlencoded` (defecto), `multipart/form-data` (archivos), `text/plain` (legacy) |
| `name` | Nombre del form (para agrupar campos de forms distintos vía `form="id"`) |
| `novalidate` | Salta la validación nativa (necesario si validás con JS personalizado) |
| `target` | Dónde mostrar la respuesta (legacy de pageserver) |
| `autocomplete` | `on`/`off` a nivel de form |

- **`GET`** va en la URL (`?campo=valor&…`): correcto para búsquedas y filtros — es bookmarkable, cacheable, y no crea nada en el servidor. Límite práctico ~2000 caracteres en algunos servidores.
- **`POST`** va en el cuerpo: creación/actualización de recursos, uploads, datos sensibles (nunca passwords en GET: quedan en historial, logs y referers).
- Campos de forms **distintos** pueden pertenecer a uno por `form="id-del-form"` — útil para botones o campos flotantes fuera del tag.
- El envío dispara el evento `submit`, que se controla con JavaScript — ver [[Eventos]] y [[Acceder a Html]].

### Manejar el submit con JavaScript y `FormData`

Patrón estándar de SPAs: prevenir el envío nativo y mandar con `fetch`:

```js
const form = document.getElementById('registro');

form.addEventListener('submit', async (e) => {
  e.preventDefault();                 // cancela el envío nativo (y su validación se completó)
  if (!form.checkValidity()) return;  // red de seguridad si se usó novalidate

  const datos = new FormData(form);   // recoge todos los campos con name
  // datos.append('extra', 'valor');  // se pueden añadir pares
  // datos.delete('interno');         // o quitar

  const res = await fetch('/api/registro', { method: 'POST', body: datos });
  if (!res.ok) mostrarErrores(await res.json());
});

// Envelope JSON moderno (sin archivos):
form.addEventListener('submit', async (e) => {
  e.preventDefault();
  const datos = Object.fromEntries(new FormData(form));
  await fetch('/api/registro', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(datos),
  });
});
```

| Objetivo | Forma correcta |
|---|---|
| Campos con `name` | `new FormData(form)` los incluye automáticamente (radio solo el seleccionado, checkbox solo si marcado) |
| Archivos | `FormData` los incluye (`form.enctype` debe ser `multipart/form-data` en el nativo) |
| Añadir dato extra | `fd.append('clave', valor)` |
| JSON | `Object.fromEntries(new FormData(form))` + header `Content-Type: application/json` |
| Campos disabled | No viajan ni en FormData ni en el nativo |
| Campos sin `name` | No viajan (ni en FormData) — bug clásico |

`FormData` respeta la asociación `form=` y funciona igual con forms generados dinámicamente.

## Subida de archivos

```html
<input type="file" name="avatar" id="avatar"
       accept="image/jpeg,image/png" multiple>
```

- **`multiple`**: permite seleccionar varios archivos; `.files` es una `FileList`.
- **`accept`**: sugerencia de tipos (`image/*`, `.pdf`, `image/jpeg,image/png`) — **no es validación de seguridad**: el usuario puede cambiarlo; el servidor debe validar siempre.
- **`required`** en file exige al menos un archivo.
- El envío requiere `enctype="multipart/form-data"` en el form nativo (con `fetch` + `FormData` se hace solo).
- Mostrar nombre/tamaño pre-envío:

```js
input.onchange = () => {
  for (const f of input.files) console.log(f.name, f.size, f.type);
};
```

> [!warning] El límite de tamaño no es del navegador
> `accept` y `multiple` son UX; los límites reales (tamaño, cantidad, MIME) los impone y debe validar el **servidor**. Un input file es solo un selector: cualquier curl puede enviar lo que quiera a tu endpoint.

## Controles personalizados (sin reinventar la accesibilidad)

### Checkbox y radio nativos con CSS

Los inputs nativos se pueden estilizar manteniendo toda su accesibilidad:

```css
/* Esconder visualmente pero conservar foco, rol y teclado */
.input-oculto {
  position: absolute;
  opacity: 0;            /* mejor que display:none, que saca del flujo de foco */
}

/* Estilo custom visible hermano del input (patrón clásico) */
.switch {
  position: relative;
  display: inline-block;
  width: 44px; height: 24px;
  background: #ccc;
  border-radius: 12px;
  transition: background .2s;
}
.input-oculto:checked + .switch { background: teal; }
.input-oculto:focus-visible + .switch { outline: 2px solid; outline-offset: 2px; }
```

### El patrón "switch" (interruptor)

HTML no tiene `input type="switch"`; el patrón accesible es un checkbox semántico con apariencia de toggle:

```html
<label class="fila-switch">
  <input type="checkbox" role="switch" checked>
  <span class="switch" aria-hidden="true"></span>
  Notificaciones por correo
</label>
```

`role="switch"` le da al lector de pantalla el anuncio "interruptor, activado/desactivado". Alternativa de diseño: `<button role="switch" aria-checked="true">` con JS (cuando el estado no es un dato del form).

### Menú personalizado, datepickers, etc.

Todo control "fancy" hecho con divs debe reimplementar: teclado (flechas, Enter, Esc), rol (`combobox`, `listbox`), `aria-activedescendant`, foco. **Esa es la razón principal para preferir los controles nativos**: el navegador ya lo hizo y lo mantiene. En formularios modernos de React, librerías de componentes (Radix, MUI) existen precisamente para no cometer esos errores — ver [[ReactJS - Fundamentos]].

## Autocomplete: tokens que ahorran trabajo real

El atributo `autocomplete` con tokens semánticos hace que el navegador rellene con datos guardados — gran ganancia de UX y conversión (y requisito WCAG 1.3.5 para campos que piden datos personales):

| Token | Dato |
|---|---|
| `name`, `honorific-prefix`, `given-name`, `family-name`, `email`, `tel`, `tel-country`, `username`, `new-password`, `current-password` | Identidad |
| `street-address`, `address-line1`, `address-level2` (ciudad), `postal-code`, `country-name` | Dirección |
| `cc-name`, `cc-number`, `cc-exp`, `cc-csc` | Tarjetas (rellenado seguro del navegador) |
| `bday`, `organization`, `off` / `on` | Otros / apagado |

```html
<input name="nombre" autocomplete="given-name">
<input name="apellido" autocomplete="family-name">
<input name="correo" type="email" autocomplete="email">
<input name="dir" autocomplete="street-address">
```

Los nombres deben ser reales y consistentes entre páginas para que el relleno funcione (`name="direccion-calle"` rompe el match).

## Accesibility: checklist de formularios

- [ ] Todo campo tiene **label visible** asociado (`for`/`id` o envolviendo el input).
- [ ] Los grupos de radios/checkboxes usan `fieldset` + `legend`.
- [ ] Errores anunciados: además de la burbuja nativa, un resumen de errores arriba con enlaces a los campos (`aria-live`, `aria-describedby` apuntando al texto de error — ver [[Eventos]]).
- [ ] Campos obligatorios indicados visualmente **y** con `required` (el `*` solo no basta; el lector de pantalla debe anunciarlo).
- [ ] Targets táctiles cómodos: el área del checkbox custom ≥ 24×24px; mejor 44×44px.
- [ ] `autocomplete` en datos personales (WCAG 1.3.5).
- [ ] Zoom al 200% sin pérdida de controles; contraste de bordes y textos ≥ 4.5:1 (WCAG 1.4.3).
- [ ] No se depende del color solo para errores (icono o texto además de rojo).
- [ ] `inputmode` correcto en móvil (numérico para DNI, decimal para precios).
- [ ] El tabulado sigue el orden lógico del formulario (el DOM, no `tabindex` arbitrario).

## Errores comunes

1. **Input sin `name`**: el campo se ve, el usuario lo llena… y no viaja al servidor. Verificar `name` en todos.
2. **`id` duplicados**: el `for` apunta al primer match; labels rotos.
3. **`placeholder` como label**: desaparece al escribir; el campo queda sin nombre visible.
4. **`<button>` sin `type` dentro del form**: envía el formulario cuando el usuario quería otra cosa (bug #1 de botones "calcular envío").
5. **`disabled` cuando querías `readonly`**: el dato no se envía y nadie sabe por qué.
6. **`pattern` sin anclar**… en HTML ya viene anclado, pero recordá: `pattern="abc"` **no** matchea "xxabcxx" — y es válido para validar todo el valor.
7. **Confío en `accept`** como validación de archivos: es solo UX; validar en servidor.
8. **`enctype` faltante en uploads**: sin `multipart/form-data` los archivos no llegan (nativo).
9. **Mensajes de error solo en color o solo en tooltip**: invisibles para varios perfiles de usuario.
10. **Validar solo al enviar**: la UX moderna valida al blur/input yresume; la burbuja nativa llega tarde.
11. **Campos deshabilitados como "estado vacío"**: desaparecen del POST y el backend recibe un objeto incompleto.
12. **GET con datos sensibles**: passwords o tarjetas en la URL quedan en historial y logs.

## Rendimiento

- Los formularios nativos son widgets del navegador: cero JS, cero CSS extra, renderizados con el sistema (y se adaptan a tema oscuro — ver [[Etiquetas Meta]] con `color-scheme`).
- Cargar JS de validación en el `<head>` sin `defer` bloquea el parseo; ver [[Preload]] para hints y [[Prefetch]] para precargar la página destino del envío.
- `fetch` con `FormData` evita recargas completas; para respuestas largas preferir streaming o redirección server-side.

## Relacionadas

- [[Html]] — base del lenguaje
- [[Etiquetas]] — catálogo general de etiquetas
- [[Eventos]] — capturar el `submit` y los cambios de cada campo
- [[Acceder a Html]] — leer valores de los inputs desde JavaScript
- [[CSS Introducción]] — estilar formularios con selectores y pseudo-clases (`:focus`, `:invalid`)
- [[Formato Json]] — serializar formularios como JSON en el cliente
- [[Qué es una API]] — adónde van los datos al enviar con `fetch`
- [[ReactJS - Fundamentos]] — formularios controlados en componentes
