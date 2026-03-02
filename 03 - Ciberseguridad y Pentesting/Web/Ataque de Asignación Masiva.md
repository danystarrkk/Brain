---
aliases: ["Mass Assignment Attack", "Asignación Masiva"]
tags: ["hacking/web", "hacking/explotacion"]
estado: 🟢 Terminado
---
# Ataque de Asignación Masiva (Mass Assignment Attack)

El ataque de asignación masiva (**Mass Assignment Attack**) se basa en la manipulación de parámetros de entrada de una solicitud HTTP para crear o modificar campos en un objeto de modelo de datos en la aplicación web. En lugar de agregar nuevos parámetros, los atacantes intentan explotar la funcionalidad de los parámetros existentes para modificar campos que no deberían ser accesibles para el usuario.

> [!info] Mecanismo de Asignación Masiva
> Muchos frameworks web (como Ruby on Rails, Laravel, ASP.NET MVC, Spring) ofrecen funcionalidades de "binding" o "mass assignment" para mapear automáticamente los parámetros de una solicitud HTTP a los atributos de un objeto de modelo. Esto simplifica el desarrollo, pero si no se configura correctamente (por ejemplo, usando listas blancas de atributos permitidos o "fillable" en Laravel, o "strong parameters" en Rails), puede permitir que un atacante modifique atributos no deseados.

Por ejemplo, en una aplicación web de gestión de usuarios, un formulario de registro puede tener campos para el nombre de usuario, correo electrónico y contraseña. Sin embargo, si la aplicación utiliza una biblioteca o marco que permite la asignación masiva de parámetros, el atacante podría manipular la solicitud HTTP para agregar un parámetro adicional, como el nivel de privilegio del usuario. De esta manera, el atacante podría registrarse como un usuario con privilegios elevados, simplemente agregando un parámetro adicional a la solicitud HTTP.

## Notas Prácticas

![[Pasted image 20260227165202.png]]

Tenemos esa web donde podemos registrarnos:

![[Pasted image 20260227165224.png]]

Tenemos que capturar y ver cómo se tramita el registro:

![[Pasted image 20260227165504.png]]

> [!tip] Interceptación de Solicitudes
> Herramientas como Burp Suite o OWASP ZAP son esenciales para interceptar y modificar solicitudes HTTP. Permiten examinar los parámetros enviados por el cliente y los recibidos por el servidor, facilitando la identificación de posibles puntos de manipulación para ataques de asignación masiva.

Como podemos ver, el servidor envía ciertos campos, entre los cuales observamos al registrar algunos campos diferentes como el de `role`. En este punto, entra en juego la cuestión del ataque: vamos a intentar enviar una petición similar. Aunque no nos dejará registrar al mismo usuario, podemos en esta nueva petición enviar `role: administrator` con el objetivo de obtener otro tipo de permisos:

![[Pasted image 20260227165815.png]]

> [!warning] Impacto de la Asignación Masiva
> La explotación exitosa de una vulnerabilidad de asignación masiva puede llevar a la escalada de privilegios (como en este ejemplo), la modificación de datos sensibles, la elusión de controles de seguridad o incluso la ejecución remota de código si se manipulan atributos que controlan la lógica de la aplicación o la deserialización de objetos.

Como podemos ver, al ingresar, el servidor confía en el parámetro enviado, lo cual es un error grave. Intentemos ingresar como el usuario, recordando que este se creó al final.

![[Pasted image 20260227170132.png]]