----

Vamos a centrarnos en ataques específicos a tipo de cabecera HOST en las solicitudes.
## PortSwigger

### Lab: Basic password reset poisoning

El usuario `carlos` de forma descuidada hace click en cualquier link que se le envíe a su correo, tenemos credenciales de wiener y el objetivo es intentar que el usuario `carlos` cierre la sesión.

Si estamos hablando de que el usuario tiene que llegarle un error nuestra mejor opción es utilizar el apartado de forgot password:
![[{3C12D96A-7EA3-4136-BDAD-FEE8D070DF5E}.png]]

podemos ver como trabaja enviando un correo a nuestra cuenta para intentar obtener el link:
![[{CF448E9C-6B6B-406D-8ADF-F55B68AE0C47}.png]]

perfecto vamos a ver todo el trafico que se tramito por detrás para entender que esta pasando:

![[{4A311911-9D2F-4B3D-A906-2669D7733252}.png]]

vemos como se esta tramitando esto por detrás, la solicitud par wiener, podríamos ver si al cambiar a `carlos` esto nos da como correcto y además no nos llega nada a nosotros:

![[{DB93A3B3-9671-44B6-BD43-A53B29A84AE5}.png]]

vemos que es correcto y no nos llega nada al correo por lo que si podemos esto controlarlo a nuestro antojo.

Como el ataque es basada en la cabecera `Host` podemos intentar cambiar el valor para ver si se es valido y si en algún lado esto se representa:

![[{B05B783B-EC00-4358-87E6-CE8D50173F79}.png]]

vemos que esto ya es correcto, algo que no debería, lo que podemos intentar es ver si nos llego a nuestro correo y como nos llego:
![[{0DA3E625-3DCA-499E-B9D9-AF75E9D10B04}.png]]

Como podemos observar esta usando parte de la cabecera `Host` y lo esta incrustando dentro del link para recuperar la contraseña.

Es aquí cuando podemos intentar aprovecharnos de esto, el objetivo es obtener ese `password-token` pero del usuario carlos, si nosotros logramos cambiar el dominio al de nuestro exploit server vamos a ver como podemos obtener mediante los logs este token con el cual podemos cambiar la contraseña al usuario carlos.

![[{5A6F28C4-1446-45DA-8145-D8117A697F22}.png]]

revicemos los logs:

![[{81470263-5F72-4846-96FB-54386FC90086}.png]]
perfecto ahora podemos intentar usar esto para cambiar su contraseña:

![[{77B2204F-9F3F-45E2-B65F-596F1299491E}.png]]

![[{0B63AAE6-4D36-48DB-8023-3BCBF0EC0FBF}.png]]

Lab Terminado.


### Lab: Host header authentication bypass

En este laboratorio se asume el privilegio del usuario basado en la cabecera host, el objetivo es eliminar al usuario `carlos`.

Lo que vamos a hacer en este caso es primero ver como se tramita la petición por detrás:
![[{D16F809D-011C-4238-8163-38767F08F58A}.png]]


lo que tenemos que buscar es el panel administrativo que esta en `/admin` y como tenemos que aprovecharnos de la cabecera host vamos a describirla como `localhost` a ver si así nos permite pasar:

![[{061C43CE-4EA2-464B-96B5-46BB4E9968E1}.png]]

Vemos que si nos permite, busquemos el link con el cual podemos eliminar al usuario `carlos` y vemos si nos deja completar el lab:
![[{FC3330FD-7D88-4C6C-A171-CE9ABA82C959}.png]]

![[{F05B314E-3BA5-485F-B3AB-78F517FD965E}.png]]

Lab Terminado.


### Lab: Web cache poisoning via ambiguous requests

En este laboratorio al parecer es vulnerable a una web cache poisoning debido a una discrepancia entre la cache y el back end, al manejar de forma ambigua las peticiones.  un usuario visita continuamente a la pagina principal pero y tenemos que hacer que se le ejecute una alerta.

![[{068BF842-4DBE-42FF-9770-BC2DDE03CECD}.png]]

al parecer la web almacena en su cache durante 30 segundos la web, si jugamos con parámetros vamos a ver que que se reinicia el contador por lo que la key chache cambia.

Bueno la cabecera en la que estamos centrados es en la de Host que si modificamos no funciona pero si ponemos un poco de atención vamos a ver que nos habla de discrepancias por lo tanto y si duplicamos la cabecera que es lo que pasa?

![[{71987ABF-917B-4D4A-AE20-986FF5CA3E85}.png]]

como podemos observar logramos modificar el dominio en la llamada al script, podemos, esto ya es malo ya que podemos prepara un script que al llamarse se ejecute el alert de la siguiente manera:

![[{15FA3431-FAE7-4026-9339-CB8F5284EC02}.png]]

Podemos cambiar el dominio al maliciosos de la siguiente manera:

![[{FC6CB093-CD59-4846-981F-EA80E045720D}.png]]

y con esto enviar a ver si funciona:

![[{794B86C7-AF0B-4010-BF5B-B43C8C8E6BD4}.png]]

al parecer esto si funciona por lo que veamos si se completo la maquina:

![[{E37FDAF8-3F97-4307-9DD0-6C797A2DA7B0}.png]]

Lab Terminado.


### Lab: Routing-based SSRF

Bueno en este laboratorio tenemos que con un SSRF vamos a tratar de encontrar paneles internos donde en la red interna vamos a tener el panel de administración y el rango es `192.168.0.0/24` y una vez entremos eliminamos el usuario `carlos`.

Vamos a concentrarnos en la cabecera host donde el objetivo es tratar de encontrar en que IP interna esta corriendo el panel de admin, vemos si obtenemos algo mediante el intruder pero primero la petición es la siguiente:
![[{026405B4-FAC2-4086-AA61-BE08CFA94B64}.png]]

Vamos a configurar de la siguiente forma el ataque mediante el intruder:
![[{6A1AE73B-4533-48A6-9A59-727ABA57D205}.png]]

![[{DAF874C2-C59E-4291-A413-D6DA17451D34}.png]]

encontramos una que nos hace un follow redirect a `/admin` por lo tanto vamos a ver desde el repeater a ver que sucede:

![[{03D15016-AC93-46C1-8B07-97778644F77E}.png]]

funciona pero vemos que a diferencia de laboratorios anteriores que son links en este caso  tenemos que tramitar una estructura que es el `username` y el `csrf` por lo tanto vamos a en intentarlo a ver que sucede:

![[{AE072ECF-54FF-43A2-AB45-A027C6207891}.png]]

Pues si parece a ver funcionado, vemos si completamos el laboratorio.

![[{E35DF4A8-DB2F-48A0-B054-2578E6923330}.png]]

Lab Terminado.

### Lab: SSRF via flawed request parsing

En este laboratorio vamos atener que encontrar un panel administrativo interno donde tenemos un error de parsing y el objetivo es eliminar el usuario `carlos`.

Vamos a comenzar interceptando la solicitud al home a ver que vemos de primeras:
![[{7C0FAC15-40FD-4339-8677-AA5C561BBDA8}.png]]

Teniendo en cuenta que la vulnerabilidad esta basada directamente en errores mediante la cabecera host podemos intentar cambiar su valor a ver que sucede:

![[{B1E8E643-B644-4495-B390-8B189466EB79}.png]]

vemos un error 404 por lo que esto de alguna forma el parser mediante la sintaxis no la va a permitir y el objetivo es encontrar que de alguna forma se emplee.

Lo que podemos tratar de hacer es pasarle de forma absoluta el target donde evitamos podemos evitar que el valor de la cabecera `host` tome importancia y nos permite ingresar valores:
![[{4DE40B34-B451-4751-AC51-9BCCBC54A51A}.png]]

vemos que de alguna forma el error ya cambia y es del tipo server error y nos habla de tiempo de conexión agotado por lo podemos intentar en este punto el ataque con el intruder para ver si logramos obtener alguna IP valida en la que cargue la web:

![[{FC88FC41-D1B2-4C86-B2E5-2F5E13A64AD8}.png]]

![[{9284AED8-1585-4E64-9DF1-47FBA4F80313}.png]]

vemos que logramos obtener un 302, esto ya es bueno, vamos a usar en este caso el valor que seria `75` y veamos si con esto logramos avanzar:

![[{583A4101-03DC-465D-8ADD-9A1074FD7189}.png]]

vemos un error pero esto es debido al como se esta tramitando la solicitud, recordemos que estamos hacienda de forma absoluta de alguna forma por lo que vamos a evitar el redirect y directamente ir a `admin`.

![[{31AEF21C-210C-4EC8-B192-22778D52F798}.png]]

Podemos observar que ahora si es valido, vamos aver como se tramita el eliminar al usuario  `carlos` para hacerlo:
![[{9846864B-2FC2-457D-AF6E-EE300BCC96EB}.png]]

vemos si completamos el lab:
![[{8C14C27C-EEE3-4F99-B633-2E088A3EC301}.png]]

Lab Terminado.


### Lab: Host validation bypass via connection state attack

Bueno en este laboratorio vamos a tener que realizar un bypass de la validación de la cabecera host donde el ataque se centra en la cabecera host. El objetivo es que eliminemos al usuario `carlos`.

Comenzamos interceptando una nueva solicitud del home para intentar ver como y que se tramita:

![[{E62B6821-596D-4896-9B22-9786B6DE7CC8}.png]]

Podemos ver algunas cosas como que se esta implementado la version de `HTTP/1.1`, además podemos intentar cambiar el valor del host como en maquinas anteriores a ver que logramos:

![[{F6449095-1BE6-40EB-B41B-E5FAC4C35140}.png]]

Pongamos lo que pongamos vamos a ver como es que nos redirige al home de la web por lo que nos esta indicando de forma especifica lo que tenemos que tener en el host.

Ahora el problema es que la web hace suposiciones donde si enviamos mas de una petición por la misma conexión TCP suponga que la segunda es correcta y nos permita el paso, podemos intentarlo:

![[{02F83F7F-0EEA-4F77-9EE5-A9572397BFBE}.png]]

como vemos definimos un grupo y enviamos la petición en conjunto a otra por una misma conexión lo que provoca esta suposición, ahora vamos a ver que paso con la solicitud maliciosa a ver que encontramos:

![[{98510E77-5CEA-41B1-A509-B378A9AB07B6}.png]]

Perfecto obtenemos un error en el tiempo de conexión, como sabemos este error nos permitirá intentar realizar diferentes conexión y ver si encontramos una web interna que ya nos la dan y es es la de `192.168.0.1/admin`, vamos a intentarlo:

![[{C5AC5256-A7D1-4F19-A748-BD4AD3CEB95F}.png]]

Perfecto, en este punto lo que vamos a hacer es pasarle todo los parámetros necesario para eliminar al usuario  `carlos`:
![[{BBE4D45D-1FA8-4F27-9319-1FD867C9CFFB}.png]]

vemos si se completa el lab:
![[{67ACDA83-7E16-480B-8A8C-11FC2B4A4CF0}.png]]

Lab Terminado.


### Lab: Password reset poisoning via dangling markup

Lo que vamos a tener que hacer en esta maquina es explotar una inyección de alguna forma del tipo HTML que luego se representara en la web y esto nos puede permitir muchas cosas, el objetivo es tratar de entrar como el usuario `carlos`.

Revisando un poco el como se envía y el como se reciben los correos vamos aver lo siguiente:
![[{1ED94442-FFA4-498A-9EF9-D24E06029F1C}.png]]

y tenemos la opción de ver en raw, tomando en cuenta que esto es para recuperar la contraseña:

![[{61C9B03A-3386-4393-B9DB-69B2BB38F76B}.png]]

Nuestro Objetivo es intentar representar primero nuestro exploit server por aquí entonces vamos a ver como se esta tramitando todo por detras:

![[{32A51E20-FE56-42E3-BA06-83108544DACC}.png]]

Encontramos exactamente la petición con al cual nosotros vamos a enviar el correo, podemos intentar ver que sucede al cambiar la cabecera host:
![[{5BE1A436-38D6-4350-BAFD-B11C7B8D6F74}.png]]

vemos que de alguna forma generamos un error familiar pero en este caso no nos sirve de nada, lo mismo pasara duplicando la cabecera por lo tanto pues nada.

Vamos a ver que sucede si aumentamos datos a nuestra cabecera que sucede:

![[{D47FF66F-2DC8-4323-9419-77EF4A886D84}.png]]

eso le gusto y el coreo que llega es el siguiente:
![[{D8C9F5B3-E169-4053-B96F-6F1F53CF4397}.png]]

perfecto, esto es muy bueno, podemos con esto intentar inyectar algún caractér como el cerrar directamente el href y luego la etiqueta `anchor`:

![[{E35009C2-54C0-4EE7-AD55-08752727372A}.png]]

vemos que esto le a gustado, ahora lo que podemos hacer es lo siguiente:
![[{D5D97254-6DD8-45A1-A577-94D77B79023A}.png]]

como vemos estamos intentando crear una nueva solicitud la cual por medio de un nuevo `anchor` intentamos crear nuevo link el cual conste de toda la info de la solicitud ya que no estamos cerrando esa doble comilla, vemos como nos llega el email:
![[{00CBBD18-524E-4B68-A933-B77CBE7C2EF3}.png]]

vemos que ya no se representa de forma correcta y en raw:
![[{2737275D-F547-4D5F-BEA0-B39BB0E3156B}.png]]

perfecto logramos generar una estructura la cual puede que se este tramitando y para probar podemos usar nuestro server exploit de la maquina y revisar si en los logs no llega algo:
![[{E29E321C-F4F9-474F-BA01-124DAB97256F}.png]]

se va sin problemas, veamos los logs a ver que encontramos:
![[{D423CE95-BD58-4EB5-8712-21B2151F06D1}.png]]

perfecto vemos que se esta tramitando toda la información y logramos obtener todo, vamos ahora a intentarlo para el usuario carlos:
![[{5430A4E3-C81B-4FC6-857C-4B25C987B48E}.png]]

![[{BCB36B68-C5BB-4C90-B99A-C5BA8E4CCBD0}.png]]

ya tenemos la contraseña para el usuario caros vamos a intentar acceder:
![[{E05DDD79-35ED-43B2-A72D-F52B3D08D6FB}.png]]

Lab Terminado.