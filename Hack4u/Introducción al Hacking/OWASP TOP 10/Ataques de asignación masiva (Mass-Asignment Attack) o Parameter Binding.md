------

El ataque de asignación masiva (**Mass Assignment Attack**) se basa en la manipulación de parámetros de entrada de una solicitud HTTP para crear o modificar campos en un objeto de modelo de datos en la aplicación web. En lugar de agregar nuevos parámetros, los atacantes intentan explotar la funcionalidad de los parámetros existentes para modificar campos que no deberían ser accesibles para el usuario.

Por ejemplo, en una aplicación web de gestión de usuarios, un formulario de registro puede tener campos para el nombre de usuario, correo electrónico y contraseña. Sin embargo, si la aplicación utiliza una biblioteca o marco que permite la asignación masiva de parámetros, el atacante podría manipular la solicitud HTTP para agregar un parámetro adicional, como el nivel de privilegio del usuario. De esta manera, el atacante podría registrarse como un usuario con privilegios elevados, simplemente agregando un parámetro adicional a la solicitud HTTP.

## Notas Practicas

![[Pasted image 20260227165202.png]]

tenemos esa web donde podemos registrarnos:

![[Pasted image 20260227165224.png]]

Tenemos que capturar y ver como es que se tramita el registro:

![[Pasted image 20260227165504.png]]

como podemos ver el servidor envía ciertos campos, donde lo que vemos al registrar algunos campos diferentes como son el de `role`, en este punto entre la cuestión de ataque, vamos a intentar enviar una petición similar porque no nos dejara al tener ya al usuario registrado pero podemos en esta nueva petición enviar como `role: administrator` con el objetivo de obtener otro tipo de permisos:

![[Pasted image 20260227165815.png]]

como podemos ver al ingresar y el servidor al confiar en el y es un error grande. Intentemos ingresar como el usuario recordando que ese te creo al final.

![[Pasted image 20260227170132.png]]

