---
aliases:
  - IDOR
  - Insecure Direct Object References
tags:
  - hacking/web
  - hacking/explotacion
  - hacking/laboratorios
estado: 🟢 Terminado
---
Las **Insecure Direct Object References** (**IDOR**) son un tipo de vulnerabilidad de seguridad que se produce cuando una aplicación web utiliza **identificadores internos** (como números o nombres) para identificar y acceder a recursos (como archivos o datos) y no se valida adecuadamente la autorización del usuario para acceder a ellos.

> [!info]
> Las IDOR son una forma de control de acceso roto (Broken Access Control), donde la aplicación no verifica si el usuario autenticado tiene permiso para acceder al recurso solicitado, basándose únicamente en la existencia del identificador. Esto puede llevar a la divulgación de información sensible o a la ejecución de acciones no autorizadas.

Por ejemplo, si una aplicación web utiliza un identificador numérico para identificar un registro en una base de datos, un atacante puede intentar adivinar este identificador y acceder a los registros sin la debida autorización. Esto puede permitir a los atacantes acceder a información confidencial, modificar datos, crear cuentas de usuario no autorizadas y realizar otras acciones maliciosas.

> [!tip]
> Los vectores comunes para IDOR incluyen parámetros en la URL (query strings o path parameters), campos ocultos en formularios, parámetros en cuerpos de peticiones JSON/XML, y encabezados HTTP personalizados. La clave es identificar dónde se utilizan identificadores para referenciar objetos.

La vulnerabilidad IDOR se puede producir cuando una aplicación web **no implementa controles de acceso adecuados** para los recursos que maneja. Por ejemplo, una aplicación puede validar el acceso a través de autenticación y autorización para los recursos que se muestran en la interfaz de usuario, pero no aplicar la misma validación para los recursos que se acceden directamente a través de una URL.

Para explotar una vulnerabilidad IDOR, un atacante puede intentar modificar manualmente el identificador de un objeto en la URL o utilizar una herramienta automatizada para probar diferentes valores. Si el atacante encuentra un identificador que le permite acceder a un recurso que no debería estar disponible, entonces la vulnerabilidad IDOR se ha explotado con éxito.

Por ejemplo, supongamos que un usuario 'A' tiene un pedido con el identificador numérico **123** y el usuario 'B' tiene un pedido con el identificador numérico **124**. Si el atacante intenta acceder a través de la URL `https://example.com/orders/124`, la aplicación web podría permitir el acceso a ese pedido sin validar si el usuario tiene permiso para acceder a él. De esta manera, el atacante podría acceder al pedido del usuario 'B' sin la debida autorización.

Para prevenir la vulnerabilidad IDOR, es importante validar adecuadamente la autorización del usuario para acceder a los recursos, tanto en la interfaz de usuario como en las solicitudes directas a través de URL. Además, se recomienda restringir los permisos de acceso a los recursos y mantener actualizado el software y los sistemas operativos.

> [!warning]
> La prevención de IDOR requiere implementar controles de acceso robustos en el lado del servidor para cada solicitud que involucre un identificador de objeto. Esto significa verificar no solo la autenticación del usuario, sino también su autorización específica para acceder a *ese* objeto en particular. El uso de identificadores impredecibles (UUIDs) puede dificultar la enumeración, pero no reemplaza una validación de autorización adecuada.

## Notas Prácticas

Estas son las referencias inseguras. A continuación, exploraremos esta vulnerabilidad en la siguiente aplicación web:

![[Pasted image 20260227154748.png]]

Tenemos 5 opciones diferentes de cafés a seleccionar, pero ¿son solo 5?

Si nos fijamos en la `URL`, le pasamos el mismo valor de `item` cuando seleccionamos uno y hacemos `submit`:

![[Pasted image 20260227154930.png]]

Al parecer solo son 5, pero ¿qué sucede si intentamos con el identificador `6`?

![[Pasted image 20260227154950.png]]

Como vemos, sí podemos acceder a otro café. Mediante el cambio de los identificadores, podemos verificar en ocasiones información extra que no debería ser accesible, como es este el caso.

> [!tip]
> Este es un ejemplo clásico de IDOR secuencial o enumerable. La aplicación utiliza identificadores numéricos predecibles, lo que permite a un atacante simplemente incrementar o decrementar el valor para acceder a otros recursos. La enumeración puede ser manual o automatizada.

En otro laboratorio, tenemos la siguiente situación:

![[Pasted image 20260227155541.png]]

Tenemos una aplicación web mediante la cual podemos crear un PDF. Veamos qué sucede al hacer clic en 'Crear':

![[Pasted image 20260227155625.png]]

Observamos que nos indica que se creó y nos proporciona un identificador, en este caso `1487`.

Intentemos descargarlo usando este identificador:

![[Pasted image 20260227155710.png]]

En efecto, esto funciona con identificadores directos.

En este caso, ¿qué pasa si ponemos el valor de `1` o cualquier otro identificador:

![[Pasted image 20260227155902.png]]

Cuando hacemos esto, al parecer nos arroja un PDF con el mensaje de 'volver a intentarlo':

![[{6C3C60BE-6B2E-4AC6-A79F-6A8EA4C7B780}.png]]

Al ingresar el valor de `3`, al parecer no existe, y se nos indica que los valores válidos están en el rango del `1` al `1500`.

> [!info]
> Los mensajes de error detallados, como el que indica el rango de IDs válidos (1 al 1500), son extremadamente útiles para los atacantes. Proporcionan información valiosa que facilita la enumeración y el fuzzing, revelando el espacio de búsqueda para encontrar objetos válidos.

Con esta información, podemos intentar realizar `fuzzing` para encontrar archivos válidos en ese rango. Primero, debemos analizar cómo son las peticiones utilizando Burp Suite:

![[Pasted image 20260227160241.png]]

Como observamos, la petición es de tipo `POST` y el parámetro de datos es `pdf_id`. Por lo tanto, nuestro comando en `wfuzz` quedaría de la siguiente manera:

> [!info]
> **Explicación del comando `wfuzz`:**
> - `-c`: Muestra la salida con colores para una mejor legibilidad.
> - `--hh=10703,1233`: Oculta las respuestas con un tamaño de contenido (content-length) de 10703 o 1233 bytes. Esto es útil para filtrar respuestas de error o no válidas y centrarse en las que tienen un tamaño diferente, indicando un contenido válido.
> - `-t 200`: Establece el número de hilos concurrentes a 200, acelerando el proceso de fuzzing.
> - `-z range,1-1500`: Define el payload como un rango numérico desde 1 hasta 1500. `wfuzz` sustituirá `FUZZ` con cada número en este rango.
> - `-d 'pdf_id=FUZZ'`: Especifica que el payload (`FUZZ`) debe ser insertado en el cuerpo de la petición POST, en el parámetro `pdf_id`.
> - `-u http://172.17.0.1:5000/download`: Define la URL objetivo para las peticiones.

```bash
wfuzz -c --hh=10703,1233 -t 200 -z range,1-1500 -d 'pdf_id=FUZZ' -u http://172.17.0.1:5000/download
```

![[Pasted image 20260227160635.png]]

Como podemos observar, hemos encontrado un identificador único válido:

![[Pasted image 20260227160704.png]]

Como vemos, las vulnerabilidades IDOR son relativamente simples de explotar, donde se manipulan identificadores con el objetivo de ver u obtener información extra que no está disponible por defecto, pero tampoco ha sido eliminada o protegida adecuadamente.

> [!warning]
> El impacto de una vulnerabilidad IDOR puede variar desde la divulgación de información confidencial (datos de usuarios, documentos privados) hasta la modificación o eliminación de datos, la toma de control de cuentas de usuario o la ejecución de funciones administrativas. Es crucial tratarlas con la misma seriedad que otras vulnerabilidades de seguridad críticas.