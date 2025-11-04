---
tags:
  - FrontEnd
Creado: 2025-01-25
Relacionado:
  - "[[SASS]]"
---
----------
# Sintaxis de SASS
## Variables
```sass
$color: #e1e1e1;
$separacion: 3rem;
$fuente: Arial, Helvetica;
```
## Anidación
```scss
div
	display: flex;
	h1
		margin-top: 10rem;
	p
		margin-top: 10rem;
```

# Extensions de Archivo
archivos .sass
```scss
.header
	display: flex;
	.log
		margin-top: 10rem;
```

archivos .scss
```scss
.header{
	display: flex;
	.logo{
		margin-top: 10rem;
	}
}
```


