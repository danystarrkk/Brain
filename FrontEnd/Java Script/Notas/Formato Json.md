---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[JAVASCRIPT]]"
---
----------
# Formato Json
## Qué es Json?
json es un formato el cual nos permite estructurar datos mediante la cual podemos compartir información.

Nosotros en Json vamos a tener propiedades y valores de la siguiente manera:

```json
{
"propiedad": "valor",
"precio": 12,
"estado": true,
"arreglo": ["pera", "fresa"]
}
```

Nosotros en el formato json también nos podremos encontrar con la anidación la cual se vería de la siguiente manera:

```json
{
	"propiedad":{
		"precio": 34,
	},
	"perfil": [
		{"id":3},
		{"url": "https://"}
	]
}
```