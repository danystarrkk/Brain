---
aliases: ["Burp Suite Targeted Scanning", "Burp Suite Non-Standard Data Structures"]
tags:
  - herramientas/burpsuite
  - hacking/laboratorios
estado: 🟢 Terminado
---
# Descubriendo Vulnerabilidades Rápidamente con Escaneo Dirigido

En realidad, es una funcionalidad de Burp Suite diseñada para identificar vulnerabilidades de manera más eficiente.

> [!info]
> El escaneo dirigido (Targeted Scan) en Burp Suite permite a los pentesters concentrar los esfuerzos de escaneo en peticiones o parámetros específicos, en lugar de escanear toda la aplicación. Esto es útil para validar hipótesis sobre vulnerabilidades en puntos concretos o para acelerar el proceso en entornos de laboratorio.

Primero, ingresamos al laboratorio y capturamos la petición web:

![[Pasted image 20260223212852.png]]

Hacemos clic derecho en la petición capturada y seleccionamos `Scan`:

![[Pasted image 20260223212958.png]]

Seleccionamos las opciones mostradas en la imagen y procedemos con el escaneo:

![[Pasted image 20260223213113.png]]

Confirmamos la selección y comenzamos el escaneo.
![[Pasted image 20260223213156.png]]

Burp Suite comenzará a probar diferentes vectores de ataque; es cuestión de esperar a que finalice el proceso.

![[Pasted image 20260223213542.png]]

Observamos una vulnerabilidad de tipo `External service interaction`, específicamente una Inyección de Entidades Externas (XXE):

![[Pasted image 20260223213557.png]]

> [!warning]
> Una vulnerabilidad XXE (XML External Entity) ocurre cuando un parser XML procesa una entrada XML que contiene referencias a entidades externas. Esto puede llevar a la divulgación de archivos locales, ejecución remota de código (RCE), ataques de denegación de servicio (DoS) o interacciones con servicios externos (SSRF).

Dado que se están manipulando entidades externas, y recordando los conceptos de XXE, podemos utilizar un payload específico para forzar la impresión de recursos internos del sistema, como se muestra a continuación:

> [!tip]
> Un payload común para la divulgación de archivos en XXE es `<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>` seguido de una referencia a la entidad `&xxe;` dentro del XML. Esto instruye al parser a leer el contenido del archivo `/etc/passwd` y mostrarlo en la respuesta.

![[Pasted image 20260223214102.png]]

## Escaneo de Estructuras de Datos No Estándar

A continuación, exploraremos cómo escanear estructuras de datos no estándar.

> [!info]
> Las estructuras de datos no estándar pueden incluir formatos propietarios, serializaciones personalizadas, o datos codificados de formas inusuales (ej. Base64 anidado, XML/JSON dentro de otros formatos). Identificar y probar estas estructuras es crucial, ya que a menudo son pasadas por alto y pueden contener vulnerabilidades.

Comenzamos capturando la petición:

![[Pasted image 20260223214732.png]]

Como observamos, la petición contiene el valor `wiener` y otra estructura de datos cuyo formato exacto desconocemos.

Procederemos a seleccionar las opciones que se muestran en las siguientes imágenes, asegurándonos de tener seleccionado el valor `wiener`:

![[Pasted image 20260223215209.png]]

![[Pasted image 20260223215222.png]]

Iniciamos el `Scan` y esperamos a que Burp Suite complete su análisis.

![[Pasted image 20260223215741.png]]

Al enviar la petición al Repeater, observamos:
![[Pasted image 20260223215904.png]]

Aparentemente, el sistema está intentando escapar de algún tipo de contexto, pero esto nos permite realizar peticiones externas.

> [!info]
> La detección de `External service interaction` en este contexto a menudo indica una vulnerabilidad de Server-Side Request Forgery (SSRF) o una inyección ciega (como XSS o XXE) que permite al servidor realizar peticiones a recursos externos controlados por el atacante. El Burp Collaborator es fundamental para detectar estas interacciones ciegas.

![[Pasted image 20260223220137.png]]

Modificamos la petición para que interactúe con nuestro Burp Collaborator, incluyendo la cookie del usuario objetivo codificada en Base64. Esto nos permitirá exfiltrar información sensible.

> [!tip]
> El Burp Collaborator es una herramienta de Burp Suite que permite detectar vulnerabilidades ciegas. Genera un dominio único que, al ser contactado por la aplicación vulnerable, registra la interacción (DNS, HTTP, SMTP), revelando la existencia y detalles de la vulnerabilidad (ej. SSRF, XXE ciega, XSS ciega, inyección de comandos).

![[Pasted image 20260223220356.png]]

Hemos recibido la interacción en Collaborator, confirmando la exfiltración del identificador deseado. Esto sugiere que el usuario administrador, a nivel de sesión, fue afectado por una inyección en algún panel administrativo (posiblemente un visor de logs) donde los comandos fueron interpretados, enviándonos la información de sus cookies.

Para finalizar el laboratorio, iniciamos sesión como `administrator` utilizando la cookie obtenida:

![[Pasted image 20260223220713.png]]

Una vez dentro, confirmamos la existencia de un apartado de Logs donde se interpretan las entradas. Podemos realizar una prueba adicional inyectando una alerta JavaScript para verificar la vulnerabilidad de XSS:

![[Pasted image 20260223220852.png]]

Enviamos la petición y verificamos si la alerta se genera al acceder a la sección de logs:

![[Pasted image 20260223220924.png]]

Efectivamente, este es el punto vulnerable.

Finalmente, eliminamos al usuario `carlos` para completar el laboratorio:

![[Pasted image 20260223220957.png]]