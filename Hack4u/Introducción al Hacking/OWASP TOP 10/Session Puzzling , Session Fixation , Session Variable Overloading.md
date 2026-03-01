--------

Session Puzzling, Session Fixation y Session Variable Overloading son diferentes nombres correspondientes a vulnerabilidades de seguridad que afectan la **gestión de sesiones** en una aplicación web.

La vulnerabilidad de **Session Fixation** se produce cuando un atacante establece un identificador de sesión válido para un usuario y luego espera a que el usuario inicie sesión. Si el usuario inicia sesión con ese identificador de sesión, el atacante podría acceder a la sesión del usuario y realizar acciones maliciosas en su nombre. Para lograr esto, el atacante puede engañar al usuario para que haga clic en un enlace que incluye un identificador de sesión válido o explotar una debilidad en la aplicación web para establecer el identificador de sesión.

El término “**Session Puzzling**” se utiliza a veces para referirse a la misma vulnerabilidad, pero desde el punto de vista del atacante que intenta **adivinar** o **generar identificadores** de sesión válidos.

Por último, el término “**Session Variable Overloading**” se refiere a un tipo específico de ataque de Session Fixation en el que el atacante envía una gran cantidad de datos a la aplicación web con el objetivo de sobrecargar las variables de sesión. Si la aplicación web no valida adecuadamente la cantidad de datos que se pueden almacenar en las variables de sesión, el atacante podría sobrecargarlas con datos maliciosos y causar problemas en el rendimiento de la aplicación.

Para prevenir estas vulnerabilidades, es importante utilizar identificadores de sesión aleatorios y seguros, validar la autenticación y autorización del usuario antes de establecer una sesión y limitar la cantidad de datos que se pueden almacenar en las variables de sesión.

## Notas Practica

![[Pasted image 20260227173616.png]]

vamos a acceder con `admin:admin` :

![[Pasted image 20260227173746.png]]

Como observamos tenemos una cookie de sessión la cual parece ser un [[Enumeración y explotación de Json Web Tokens (JWT)]], veamos que es lo que se puede hacer:

![[Pasted image 20260227173957.png]]

observamos que tenemos un panel para cambiar la contraseña donde vamos a capturar la petición primero:

![[Pasted image 20260227174157.png]]

vemos como se tramita y al cargar nos dice:

![[Pasted image 20260227174213.png]]

nos tiene que llegar un correo para cambiar la contraseña, pero si nosotros en el Burp Suite capturamos la respueta:

![[Pasted image 20260227174305.png]]

Estamos observando una cookie de sesión y como esta la incrusta en nosotros podemos intentar movernos a algún apartado especifico como `dashboard` y si lo arrastra podríamos acceder:

![[Pasted image 20260227174452.png]]