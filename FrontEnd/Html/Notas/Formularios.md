---
tags:
  - FrontEnd
Creado: 2025-01-24
Relacionado:
  - "[[Brain/FrontEnd/Html/HTML|HTML]]"
---
----------
# Formularios en HTML

Los formularios siempre van a iniciar con la etiqueta form, esto como una buena practica, y a partir de allí vamos a hacer uso de la etiqueta input la cual a través de los atributos nos permitirán crear diferentes inputs, además de estas tenemos label, esta etiqueta nos permite crear el indicador del input, el cual lleva su descripción, además el atributo for es importante ya que facilita la navegación y este se complementa con un id que se defina en nuestro input correspondiente:

```jsx
<form>
	<label for="nombre">Nombre: </label>
	<input id="nombre" type="Tipo de input" placeholder="Indicador" >
</form>
```

Tenemos mas atributos para las etiquetas como son:

- value: Nos permite ingresar información en el input.
- readonly: Nos indica que en ese campo input no se pueda ingresar información o modificar.
- tel: Nos permite que en los dispositivos móviles se despliegue el panel numérico, además de ayudar en la verificación de números telefónicos.
- email: Permite que nos ingrese el arroba en el campo.
- time: Nos permite tener desplegar el calendario.
- file: Nos permite en la subida de archivos.
- password: Le quita la visibilidad al texto.
- radio: es un tipo de input el vual permite crear opciones múltiples las cuales mediante diferentes atributos mas podemos manipula.
- submit: esto nos permite crear un input el cual nos permite enviar la información en el caso de los formularios.
- required: Esto permite realizar una validación si el campo a sido llenado.

Otra etiqueta que se usa para complementar nuestro formularios es fieldset esta nos agrega recuadros con el objetivo de agrupar el formulario, además de esta tenemos otra la cual es legend que nos permitirá agregar una especie de titulo, entonces nuestra estructura final quedaría de la siguiente manera:

```jsx
<form>
	<fieldset>
		<legend>Titulo del formulario<legend/>
		<label for="nombre">Nombre: </label>
		<input id="nombre" type="Tipo de input" placeholder="Indicador" >
	<fieldset/>
</form>
```

### TextArea
Una etiqueta extra para esas áreas de texto grande va a ser textarea, esta etiqueta nos permitirá crear estas áreas de texto y con los siguientes atributos podemos modificarlas:

- row: Definimos el tamaño de líneas.
- cols: Definimos el tamaño de columnas.
- A comparación de los inputs si queremos definir valores por defecto los escribimos dentro de las etiquetas.

```jsx
<textarea row="20" cols="40"></textarea>
```

### Select
Esta etiqueta nos permite crear una barra de opciones de las cuales se puede escoger. esta a diferencia de otras tiene etiquetas internas y la mas usada es option, nos permite crear las opciones del menú, esta etiqueta también tiene atributos como:

- selected: Nos permite dejar una opción seleccionada por defecto.
- disabled: Nos da la posibilidad de deshabilitar una de las opciones.
- value: Nos permite asignarle un valor a la computadora para poder tramitarlo con algún lenguaje de programación.

```jsx
<select>
	<option selected disbled>0<option/>
	<option>1<option/>
	<option>2<option/>
	<option>3<option/>
	<option>4<option/>
</select>
```