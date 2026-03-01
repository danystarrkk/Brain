-----

Las **Insecure Direct Object References** (**IDOR**) son un tipo de vulnerabilidad de seguridad que se produce cuando una aplicación web utiliza **identificadores internos** (como números o nombres) para identificar y acceder a recursos (como archivos o datos) y no se valida adecuadamente la autorización del usuario para acceder a ellos.

Por ejemplo, si una aplicación web utiliza un identificador numérico para identificar un registro en una base de datos, un atacante puede intentar adivinar este identificador y acceder a los registros sin la debida autorización. Esto puede permitir a los atacantes acceder a información confidencial, modificar datos, crear cuentas de usuario no autorizadas y realizar otras acciones maliciosas.

La vulnerabilidad IDOR se puede producir cuando una aplicación web **no implementa controles de acceso adecuados** para los recursos que maneja. Por ejemplo, una aplicación puede validar el acceso a través de autenticación y autorización para los recursos que se muestran en la interfaz de usuario, pero no aplicar la misma validación para los recursos que se acceden directamente a través de una URL.

Para explotar una vulnerabilidad IDOR, un atacante puede intentar modificar manualmente el identificador de un objeto en la URL o utilizar una herramienta automatizada para probar diferentes valores. Si el atacante encuentra un identificador que le permite acceder a un recurso que no debería estar disponible, entonces la vulnerabilidad IDOR se ha explotado con éxito.

Por ejemplo, supongamos que un usuario ‘**A**‘ tiene un pedido con el identificador numérico **123** y el usuario ‘**B**‘ tiene un pedido con el identificador numérico **124**. Si el atacante intenta acceder a través de la URL “**https://example.com/orders/124**“, la aplicación web podría permitir el acceso a ese pedido sin validar si el usuario tiene permiso para acceder a él. De esta manera, el atacante podría acceder al pedido del usuario ‘**B**‘ sin la debida autorización.

Para prevenir la vulnerabilidad IDOR, es importante validar adecuadamente la autorización del usuario para acceder a los recursos, tanto en la interfaz de usuario como en las solicitudes directas a través de URL. Además, se recomienda restringir los permisos de acceso a los recursos y mantener actualizado el software y los sistemas operativos.

## Notas Practica

Bueno estas son las referencias inseguras, esta vulnerabilidad vamos a verla en la siguiente web:

![[Pasted image 20260227154748.png]]

tenemos 5 opciones diferentes de cafés a seleccionar, pero solo 5?

Si nos fijamos en la `url` nosotros le pasamos el mismo valor de item cuando seleccionamos uno y le damos submit:

![[Pasted image 20260227154930.png]]

Al parecer solo son 5, pero y si ponemos el 6?

![[Pasted image 20260227154950.png]]

Como vemos si podemos ver otro café, en si nosotros mediante el cabio de los identificadores podemos directamente verificar en ocasiones información extra que no debería, como es este el caso.

En otro laboratorio tenemos lo siguiente.

![[Pasted image 20260227155541.png]]

Tenemos una web mediante la cual podemos crear un pdf, veamos que sucede al darle crear:

![[Pasted image 20260227155625.png]]

Observamos que nos dice que se creo y nos da un identificador, en este caso `1487`.

Intentemos descargarlo usando el identificador:

![[Pasted image 20260227155710.png]]

En efecto esto funciona con identificadores.

En este caso que pasa si ponemos el valor de `1` o cualquier otro identificador:

![[Pasted image 20260227155902.png]]

Cuando hacemos esto al parecer nos arroja un pdf con el mensaje de volver a intentarlo:

![[{6C3C60BE-6B2E-4AC6-A79F-6A8EA4C7B780}.png]]

al ingresar el valor de 3 al parecer no se tiene y pues se pide que son valores del 1 al 1500.

Con esto claro podemos intentar fuzzing para encontrar archivos validos en ese rango pero primero tenemos que ver como es que son las peticiones y esto mediante Burp Suite:

![[Pasted image 20260227160241.png]]

como observamos tenemos que por POST y además como data es `pdf_id` por lo tanto nuestro comando en wfuzz quedaría :

```bash
wfuzz -c --hh=10703,1233 -t 200 -z range,1-1500 -d 'pdf_id=FUZZ' -u http://172.17.0.1:5000/download
```

![[Pasted image 20260227160635.png]]

Como podemos observar tenemos el identificador único:

![[Pasted image 20260227160704.png]]

Como vemos los IDORs son vulnerabilidades muy simples donde se juga con identificadores con el objetivo de ver o obtener información extra que no esta disponible por defecto pero no se encuentra dada de baja tampoco.