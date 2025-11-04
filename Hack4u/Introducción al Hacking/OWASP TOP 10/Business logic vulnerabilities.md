--------


### Lab: Excessive trust in client-side controls

En este laboratorio no se valida de forma correcta el input del usuario y podemos explotar cierta lógica al momento de pagar algún producto.


![[{57C5A9E1-58D2-424A-8944-75E839643FF4}.png]]

comenzamos con 100 dólares como crédito y vamos a ver que lo que tenemos que comprar no nos alcanza, pero vamos a ver que es lo que esta pasando, podemos agregar el producto al carrito pero interceptar la petición a ver como se esta tramitando:

![[{F2C65109-C3FC-4053-BE0D-33AFB8A8B30B}.png]]

vemos que al agregar el producto al carrito se tramita la cantidad y el precio, podemos intentar algo simple y es ver que pasa si modificamos el precio y dejaos que pase la solicitud:

![[{EAC7EB93-B847-4ACC-BB05-8F4A9E46121F}.png]]

en este caso se cambio por un 1 ya que no nos dejo usar el 0 pero vemos que logramos cambiar el precio, para completar la maquina tenemos que concretar la compra y listo.

![[{E866293E-83C5-43CE-82D5-2D386E020778}.png]]

Lab Terminado.


### Lab: High-level logic vulnerability

En este caso nuevamente no se valida de forma correcta el input del usuario y podemos afectar al flujo del pago con el objetivo de comprar items a otro precio.


De igual forma analicemos como es que esta trabajando por detrás este flujo de pago:

![[{F6DB7A10-EA95-420E-9098-A87F1E8E748A}.png]]

podemos por medio de fuerza bruta como se esta implementado el precio.

vemos que podemos alterar la cantidad veamos que pasa con cantidades negativas:
![[{9DF6CD68-1305-413D-9508-F866466AF7A0}.png]]

vemos que en efecto se aplican, no nos permite pagar pero vamos a ver que no nos deja.

En este punto lo que podemos hacer es jugar con valores positivos y negativos para intentar llegar a un valor total de 0 o 1 pero la verdad es que no podemos obtener `-1` chaquetas por lo que podemos intentar 
mejor generar valores negativos desde otro producto y luego comprar el producto designado:

![[{B36D93FE-CDB3-4115-B8A3-29B34828B336}.png]]

ahora vamos a comprar una chaqueta y `-14` EGGtastic por lo que vamos a ver si esto nos lo permite:

![[{7031C77D-DC00-4AE1-8EF8-C45830952033}.png]]

Lab Terminado.


### Lab: Inconsistent security controls

En este laboratorio lo que vamos a encontrar es que tenemos que acceder a una propiedad administrativa en `/admin` y tenemos que eliminar al usuario `carlos`.

No tenemos acceso a la máquina pero tenemos un panel de registro:

![[{D76D1DFD-4E20-4AD2-82E1-BE7F021EFC80}.png]]

vemos que nos da un dominio si es que pertenecemos a cierto dominio por decirlo de alguna manera.

En este punto lo que vamos a hacer es registrarnos con nuestro correo de atacante para activar la cuenta y ver que nos permite desde dentro:

![[{1AE0E093-0BD0-4A66-895C-8155E7EBFE59}.png]]

perfecto ya estamos como otro usuario pero vemos que podemos actualizar el correo y conocemos el dominio de la empresa externa que permite el que accedamos a paneles de administrador, vamos a intentarlo:

![[{DD39DD78-65EA-474D-B661-C8CFE169F809}.png]]

vemos que la lógica no es correcta debido a que al poder cambiar el correo y además tener el dominio que tiene todos los permisos se activa el panel de admin mediante el cual podemos eliminar al usuario `carlos` que es nuestro objetivo:

![[{4DBC6487-5FDF-4EB4-B339-3D6EEB688584}.png]]

Lab Terminada.


### Lab: Flawed enforcement of business rules

En este caso tenemos que volver a comprar la chaqueta y tenemos que burlar nuevamente el pago.


Vamos a ver que tenemos primero en nuestra web:

![[{9FBB4DB6-76DB-4B58-A742-7C79E24CE1FE}.png]]

al parecer nos esta aplicando un descuento, vemos algo curioso al final vemos que realiza esto:
![[{F4FC88A6-3280-4B7C-A851-C886F3677C32}.png]]

![[{9B5EDBE3-679E-4137-B13A-321288CCA15E}.png]]

Nos da un cupón, esto es bueno, podemos intentar agregar cosas al carrito y ver que nos realiza el cupon:

![[{1344ADC9-842B-4089-AF5F-62173C1BB801}.png]]

vemos que podemos agregar mas cupones pero no los mismo de forma simultanea, pero y si alternamos estos dos cupones, puede que la lógica sea revisar por el ultimo cupón usado, veamos que sucede:

![[{5643E3D2-00E2-40AF-954F-730FE533AE7D}.png]]

perfecto logramos aplicar los mismos cupones de forma consecutiva y llevamos al cero, ya podemos comprar:
![[{021658AE-1352-4D55-8CED-7EC7A8645A3B}.png]]

Lab Terminado.


### Lab: Low-level logic flaw

En este laboratorio tampoco se valida de forma optima los inputs de usuario por lo que vamos a ver que podemos hacer, recordemos que si tenemos permisos para ingresar como `wiener`.

En este laboratorio podemos intentar varias cosas de las anteriores que hemos intentado pero vamos a observar que no funciona.

tenemos que tener claro un concepto y es que en lenguajes como C, C++ y en versiones de PHP vamos a ver que cuando se asigna una variable de tipo entera o numérica en general va en un rango determinado de números que varia desde los negativos y luego los positivos, el problema es que si se supera este numero lo que va a suceder es que comenzara una nueva cuenta desde los negativos.

Tomando esto en cuento con un ataque de fuerza bruta vamos cual es el limite de dinero que se asigna y si este cambia en algún punto:

![[{2FC722E2-C8A9-4F37-9306-8FEF20BCC822}.png]]

vamos a realizar mediante el ataque que suba hasta ver números negativos y luego paramos para calcular un numero:
![[{09DAEDE6-7A92-4DA6-99EE-28F734C1DEB0}.png]]

perfecto cuando estemos cerca del 0 como veremos en la siguiente imagen:

![[{A6520597-DA58-4761-8E3F-575C56E66A5A}.png]]

bueno ya estamos en este caso considerando que nos movemos de a una cantidad de 99 es cerca, ahora podemos calcular un aproximado de cuanto nos falta y volver a hacer el ataque pero mas medido:
![[{B76D5475-DA58-454C-93DA-1C5E1087A466}.png]]

perfecto ya logramos llegar a un valor que podemos pagar veamos si lo logramos:

![[{1698D4DD-F6D6-4A4B-876E-610AA26A661B}.png]]

Lab Terminado.


### Lab: Inconsistent handling of exceptional input

En este laboratorio no se valida de forma correcta el input del usuario y para esto vamos a vulnerar un apartado de registro para ganar acceso a la parte administrativa.

![[{2E22AC79-87CC-4078-A8C5-8846CE5FD475}.png]]

vemos al igual que en otra maquina el formulario para registrarnos. bueno como este es el punto en el que vamos a explotar una cosa que podemos comenzar intentando es ver el limite de caracteres en el  `Email` para determinar que se puede o no hacer:

![[{D04B4542-F65D-463A-A8D5-D8790CEF59EA}.png]]

enviamos 200 A y si llegaron todas, esto nos permitió tener esto activo, vemos si dentro de la web todo normal también:

![[{6EBF14D3-CF65-48A3-8390-085CEB8069AD}.png]]

Vemos que el dominio o se represento completo, podemos intentar ver cuando caracteres tenemos en total que son 255, con esto lo que vamos a intentar es si nosotros intentamos que a partir de estos 255 caracteres los demás puede que no se representen pero si se interpreten, podemos intentarlo, en sentido de cadena seria `255` hasta el dominio con permisos y luego le agregamos el subdominio que al parecer en el post procesado lo elimina:

![[{9C6CBBDC-133D-41DF-9AFD-44FDE9A575FE}.png]]

creamos la cadena donde justo estén los 255 pero luego agregamos como subdominio nuestro email para que funcione lo que pasara es que al representarse se cortara y solo llagar como si lo registráramos con el correo nuestro:

![[{A71BC26D-2932-46C6-B93D-35748A332A4C}.png]]

![[{C30476B3-C6E0-4746-A27F-1AD4D3EF5CE3}.png]]

vemos que nos llega el coreo esto ya es bueno vamos a activar la cuenta e ingresar:

![[{8616885B-05F6-4B74-AF75-110FC431E2F3}.png]]

vemos que dentro de la web se corto el apartado del subdominio provocando que solo quede el dominio que permite el panel de administrador:

![[{7BDECDFC-52D6-4692-8C37-57090E857850}.png]]

Lab Terminado.


### Lab: Weak isolation on dual-use endpoint

En este laboratorio vamos a intentar acceder como usuario administrador y borrar el usuario `carlos`.

Vamos a hacer login a ver que encontramos primero:

![[{92AD9370-A0E8-460A-9798-9B62C1755EE1}.png]]

vemos eso, podemos intentar ver que se tramita por detrás para intentar hacer algún retoque:

![[{4A53CE8C-7081-4F3F-8622-F8F79CF2FF81}.png]]

vemos que estamos tramitando diferentes campos, algo que se me viene a la mente primero es eliminar la validación del current-password:

![[{7AD6EA48-EF30-416D-AE94-CD8571FB354E}.png]]

Pues no se valida y si ahora hacemos lo mismo pero para `administrator`:

![[{F8307FE9-1303-4928-B6B9-7CFD9BAA02C4}.png]]

Vemos ya que al parecer esto funciono:

![[{FE9B548D-3055-4955-A545-B29E71BCDBA7}.png]]

Lab Terminado.


### Lab: Insufficient workflow validation

Este laboratorio asume ciertos eventos al momento de hacer el pago donde partes de estos si logramos golpear y que de asumidas cosas que no son y como en labs anteriores tenemos que comparar la chaqueta.

Vamos directamente a ver como se tramita el pago:

![[{0910664F-07AA-4FEF-9C75-9D314613F3F6}.png]]

vemos como se tramitan ciertas funciones pero como es algo que no podemos pagar no es algo del todo certero por lo que vamos a comprar algo que podemos pagar para ver como se tramita todo:

![[{64C4B66C-34AA-41A6-8032-EDA6488D2195}.png]]

en este caso ya vemos otras peticiones y una que llama mi atención que se tramita `order-confirmation` la cual si revizamos:

![[{C41DC7D2-2A22-43C5-A35E-1966403B7249}.png]]

ahora no tenemos nada en el carro pero si nosotros ahora ponemos algo mas al carrito y luego desde el repeater tramitamos esta solicitud, vemos que sucede:

![[{69B34B6D-3815-4F30-824F-95C637E373BD}.png]]

ahora tramitemos nuevamente la solicitude de confirmación desde el repeater:

![[{74694AEF-00D5-407C-B65E-F63E511FA51D}.png]]

como podemos observar se confirma la compra además de que no nos descuenta dinero ya que recordemos no teníamos suficiente:

![[{8D80CAE7-D86C-4452-88BB-91552E0CDD94}.png]]

Lab Terminada.


### Lab: Authentication bypass via flawed state machine

Primero tenemos que comprender una lo que es una maquina de estado y lo que hacen de forma general es modelar o estructurar algo en la web, en este caso la Authentication, donde los estados serán como partes o secciones ordenadas que seguirá el código desde el cuestionario para hacer login (estado 1), seleccionar rol (estado 2), definir gustos(estado 3), etc.

Al parecer tenemos una maquina de estados y mediante esta tenemos que realizar un bypass al Authentication, el objetivo es eliminar al usuario `carlos`.


![[{2323A5D1-AC1C-4224-BC4D-3A4945E0D24B}.png]]

perfecto, vemos que también tenemos un `role-selector` que es lo siguiente:

![[{2EEDDD0A-3E83-4BC9-B105-491D4CBAC028}.png]]

vemos solo dos opciones pero si capturamos esto y lo modificamos de la siguiente manera:

![[{D14F3DC2-0D5B-4FA3-B1CA-7E1CF1FEE967}.png]]

vemos si funciona:

![[{0BB1DBBC-717F-4AF0-BCC6-A56FA07F1A23}.png]]

pues no no funciono, pero si nosotros lo hacemos ahora paso a paso:

![[{3ED925C6-F00B-4E62-8B1B-3AB4EA02212C}.png]]
Como vemos primero la solicitud del login la cual la dejamos pasar, ahora vamos con la siguiente:
![[{95514747-85F1-44B6-86C8-A61724BFCA01}.png]]
Ahora llega la del rol y que pasa si esta no la dejamos pasar, tiene o debería tener un estado por defecto, vemos que pasa:

![[{EB0D5777-60E6-43AE-91A1-4E62D2C5E2E3}.png]]

al parecer valor por defecto tenemos es como admin al no asignarle nada.

![[{1D9C3F46-9C1D-4E32-B988-234798A7C706}.png]]

Lab Terminado.

###  Lab: Infinite money logic flaw

La idea es como siempre intentar comprar la chaqueta nuevamente, en este caso al igual que en anteriores vamos a contar con las credenciales del usuario `wiener`.

![[{BBD79477-5CBC-4C33-A149-416199E54673}.png]]

vemos esto en el panel, al parecer podemos usar Tarjetas de regalo mediante las cuales podríamos prácticamente canjear dinero, esto es interesante, además conseguimos un cupón de descuento `SIGNUP30` el cual podemos utilizar.

![[{B74B78F9-5DB6-4868-9A27-51DC5C066583}.png]]

al parecer podemos comprar estas Tarjetas de regalo a 10 dólares y son de 10 dólares, pero que pasa si le aplicamos el cupón de descuento:
![[{FDAD89D1-EFE1-4E15-88D8-5121CE330654}.png]]

Lo que estamos viendo es que compramos una tarjeta de 10 dólares en 7 y como estas tarjetas podemos utilizarlas en la misma cuenta podríamos ver que sucede si compramos varias:
![[{CF9A348A-2ED9-444A-979C-458E753D22CB}.png]]

aquí vamos a aplicar un poco de lógica y es que el total es de 14 tarjetas de regalo a 7 dólares cada una, pero cada una de estas nos retorna un total de 10 dólares, entonces si canjeamos esto luego lo que va a suceder es que nos de hasta 140 dólares mas los 2 que ya teníamos 142, veamos si esto funciona, pero por fines prácticos y porque no nos resulta estar poniendo los 14 cupones cuando tenemos que llegar a 1333 con algo seria ilógico por lo que primero comprobemos la teoría:
![[{1892E7A7-42A1-4A09-AD1E-2DAB42F80A76}.png]]

Usemos los códigos a ver que sucede:
![[{020D4098-E311-455C-8C04-FE452C767A83}.png]]

una vez canjeado los dos código vemos que logramos ganar un total de 3 dólares por cupón.

Bueno lo mejor que podemos hacer es automatizar este proceso mediante Burp Suite.

El proceso es el siguiente:

Vamos a la configuración Sesión y en reglas de Sesión agregamos una nueva y se desplegara el siguiente panel:
![[{3A971BFB-23F2-448E-B9D7-D0CBEC07B53E}.png]]

Le damos un nombre y en reglas le agregamos una nueva, una vez que agreguemos los links necesarios lo que vamos a ver es:
![[{40D0DEF8-A086-4110-A3C3-856851190B59}.png]]

donde en nuestro caso modificamos esa petición GET para que se nos seleccione el nuevo cupón que se genere:
![[{175E793A-A447-49A9-AD76-74F223995D08}.png]]

damos en agregar y configuramos lo necesario, en este caso ya esta listo.

Podemos dar a Test Macro y si el valor de dinero cambia pues si funciona:
![[{92D8E269-AE8F-46DD-B392-614B73789FFC}.png]]

Ahora si ya funciona lo que vamos a hacer es dar a ok a todo y obtener una petición al home:
![[{BA665C4A-3180-4141-B9BC-791EECC62387}.png]]

al enviarla ya se automatizara todo:
![[{79B76988-94DE-4530-A820-C825C36C299C}.png]]

vemos que el valor ira aumentado, ahora es cuestión que desde el intruder enviemos un ataque de unas 100 peticiones y ya con esto deveriamos obtener dinero suficiente:
![[{96DB6A96-A78E-4233-BA9C-6705FFFED46F}.png]]

ahora podemos ver que sucede al terminar el ataque:
![[{726DD657-06A5-40A4-92FA-A3BBC4C15857}.png]]

![[{CD8AB84A-5C75-484C-B2B0-BA8DA7342095}.png]]

perfecto, después de casi 500 peticiones y mas vamos aver que tenemos el diner necesario para comprar la chaqueta, vamos a ver si ya podemos completar el laboratorio:

![[{952BF000-252D-4707-A423-AE072301237B}.png]]

Lab Terminado.

### Lab: Authentication bypass via encryption oracle

El objetivo es intentar llegar al panel de administrador y eliminar el usuario `carlos`. 

Bueno vamos a capturar todo el trafico cuando iniciamos sesión y un poco mas para ver que podemos identificar:
![[{479F82E8-30CA-4112-AC75-EEA9CB6C255E}.png]]

vemos que en una de las peticiones logramos obtener datos que viajan cifrados al parecer url-encode y luego en base64 que si intentamos ver en claro:

![[{343617E3-096D-473E-97F4-DE0E3A60B51F}.png]]

vemos que en efecto esto no es legible pero ya con esto suponemos que se esta cifrando datos.

Vemos que podemos crear comentarios dentro de los blogs, vemos como funciona:
![[{36DCF7BA-2E43-49BC-B98C-F4BB90A9D70C}.png]]

creamos y analizamos el como se tramito todo pero no encontramos en si algo que nos pueda ayudar.

Vamos a nivel de sintaxis a intentar generar algún error:

![[{066430F6-494B-48E7-9D50-F6FA41976EBC}.png]]

vemos que nos reporta un error, veamos la petición:

![[{356A72E8-7E3A-4AB6-9357-07BC1936F4B5}.png]]

vemos que antes de la respuesta web tenemos una respuesta por detrás con una cookie notification que si intentamos ver el texto claro no vamos a podemos porque al parecer no es legible.

Puede ser que parte de el mensaje de error este siendo tramitado al servidor mediante la cookie `notification` y que el servidor nos lo descifre para luego mostrarlo.

Podemos intentar algo y es pasarle el contenido de `stay-logged-in` que nosotros no podíamos descifrar a ver si el servidor logra hacerlo:
![[{E1C36B8E-51D7-4BC9-B13D-13DA92A2E466}.png]]

pues si que lo hace y ahora lo que sabemos es que el contenido es `wiener:...`, ahora lo que sabemos es que puede ser que como el mensaje esta en el campo email podríamos enviar algo parecido en email primero a ver si podemos:
![[{DF68F77F-4833-43C2-B6F3-2C06E4D3CF94}.png]]

intentemos mandar eso en email a ver que nos retorna en notification:
![[{2DA70610-2624-47F6-ABE6-004AB48A0504}.png]]

vemos la nueva cadena, podemos intentar descifrar esa cadena a ver como la codifico:
![[{E83BD7DA-5DDF-4965-BB9F-524EB65725B6}.png]]

vemos que hace por defecto impone el valor de `invalid email address:` y luego ingresa el valor que le pasamos.

Bueno entonces nuestro objetivo en este punto es lograr que la primera parte del mensaje encriptado no este, podemos intentar primero lo siguiente.

Vamos a darnos cuenta que el mensaje antes de lo que necesitamos son 24:
![[{61A5F005-8684-4AB8-8D25-4C0B03B471A2}.png]]

podemos ayudarnos de decoder para pasarlo a bytes y eliminar los primeros 24:

![[{0BFAD9AF-A6A2-4318-80A9-932C0D62A3D2}.png]]

y luego ver el resultado y eso pasarlo a base64:
![[{D0021AD6-02FE-4CB2-9332-6988AE25CDA4}.png]]

podemos copiarnos la nueva cadena en base64 e intentar pasarla en la cookie para ver si obtenemos lo que queremos:
![[{153846F3-2085-4C62-963B-A72B769A9516}.png]]

vemos que con eso generamos un error donde dice que la cadena tiene que ser un múltiplo de 16 donde, lo que se me viene a la mente es el cifrado aes el cual se implementa cifrado en bloques a comparación de otros como el string silver que utiliza cifrado byte a byte. Bueno en este cifrado por bloques de 16 bytes se cifra el contenido empleando una key.

El problema radia en que si contamos el numero de bytes actual tenemos lo siguiente:
![[{94771CA2-BF64-4627-9AC1-B908BAF463F7}.png]]


exactamente la ultima línea no tiene los 16 bytes completos y este es el problema, usual mente en las cadenas en automático el sistema al computarlo va a generar lo que se llama padding que es un relleno de los bytes faltantes, la cuestión se da cuando nosotros al eliminar los bytes descompensamos esto y quedo invalido por ahora. Entonces lo que tenemos que hacer es que vamos a tener que cuadrar de forma manual para que cuando eliminemos estos 23 bytes (el espacio no cuenta) quede la cadena compensada y para esto tenemos que hacer es completar los bloques, lo que tenemos que hacer es que nos quede todo un bloque completo para que así al momento de eliminar no sean 23 si no mas de eso donde quedaran dos bloques completos, a partir del 23 nos queda 9 bytes mas los cuales vamos a agregar al comienzo de nuestra cadena:
![[{69B09371-B4B1-4C0E-A643-AF011649FDC9}.png]]

con esto lo que debe pasar es que vamos a eliminar dos bloques completos y no medios provocando que los valores queden exactos como veremos:

![[{508CFC64-E5C1-4A27-8871-537731163856}.png]]

ahora que tenemos el valor con la nueva cadena vamos a eliminar dos bloques completos ya que para completar esos dos bloques es que usamos las x:

![[{EEC221F8-A3F2-4C1C-A39B-ACDA654F9E80} 1.png]]

repetimos como vemos el proceso y tenemos la nueva cadena, vemos que sucede:

![[{7C0A8033-370A-4B5A-9BFD-33CB935B9315}.png]]

ya tenemos la cadena correcta, vemos que pasa si ahora le pasamos esto a la otra cookie:
![[{2D606B49-8D6A-4A7D-B8E0-86AECFC59F1D}.png]]

se despliega el panel de administrador, tenemos que tener en cuenta que no tenemos que tener iniciada la sesión debido a que entra en conflicto, ya con esto podemos resolver el lab:

![[{E11531D1-2ED1-4740-851C-CCCBEB6D25B9}.png]]

Lab Terminado


### Lab: Bypassing access controls using email address parsing discrepancies

El objetivo de esto es lograr entrar como administrado y como ya es costumbre eliminar al usuario `carlos`.

Como lo hemos echo en maquinas anteriores podemos intentar usar el correo y registrar lo que nos da como un subdominio para intentar registrar un usuario con permisos de administrador de la siguiente manera:
![[{10814764-3AD1-40D4-A802-CC2F8939A07D}.png]]

no nos permite pasar por los dos `@` que se están usando y generan un error podemos intentar enviarlos sin el `@` y ponerlo al interceptarlo con el burpsuite a ver que sucede:
![[{9E7643D1-0EA4-423B-A6FE-B9AD5AC6D478}.png]]

vemos que sucede:

![[{12F2CDB4-DDC2-42B7-988A-2109149A0EAE}.png]]

pues no funciona.

Bueno la discrepancia que va a pasar es que le vamos a hacer creer que nos registramos con el dominio que nos pide que es el autorizado, pero en realidad va a ser con el malicioso.

Vamos a dejar claro algunos conceptos aquí y es que los correos electrónicos por detrás tiene una algoritmia muy compleja donde no es solo como lo vemos los usuario sino el post procesado de los mismos es algo complejo y muy bien planeado, es esto lo que vamos a atacar y tenemos muchos tipos de ataques como son:

- Encode word.
- Punnycodes malformados.
- Caracteres nulos y escapados.

para una lectura educativa podemos revisar el siguiente [link](https://portswigger.net/research/splitting-the-email-atom)

Bueno podemos hacer muchos tipos de codificaciones o trucos pero el objetivo siempre es el mismo hacer que el domino aparente se el valido pero por detrás se interpreta el no valido por decirlo de alguna forma.

Entonces lo primero que tenemos que hacer ver que es lo que interpreta y no el servidor y en este caso vamos a jugar con `Encode Word` de la siguiente forma:
![[{36126476-D439-4567-A021-5812D4C9CA9D}.png]]

entonces esa es la estructura general es la que vemos en la imagen y en este caso vamos a usar un `charset=utf-8`, `encodigo=q`, y `encode-text` lo definimos en hexadecimal quedando de la siguiente manera:

![[{477EECDA-816A-4379-8D01-BFBDB5007B84}.png]]

perfecto ahora vamos a usarlo de la siguiente manera:
![[{0F1EBBAA-F967-4885-A783-314D884EE469}.png]]
![[{24418FCC-31F2-42EB-8F99-CDB16A527EA8}.png]]

antes o después de la cadena podemos agregar texto pero vemos que no nos deja. Esto puede ser que este detectando un intento de encode word por lo que nos lo bloquea.

En este punto vamos a tener que buscar otra forma defina en el RFC 2047 que es en lo que se basa este tipo de inyección donde vamos a intentar implementar caracteres no ASCII para que de alguna manera si nos lo interprete.

En si vamos a cambiar quien sabe a `utf-7` la cual es menos usada pero donde solo usa caracteres ASCII de 7bits:
![[{07E5D376-79B1-4727-BEC1-5CC9D5D70B47}.png]]

la estructura general cambia debido al proceso que se emplea en este caso donde vemos que donde va el texto ponemos `&-` ya que la codificación comienza con `&` y termina con `-` ahora para el texto vamos a hacer uso de cyber chef donde vamos a seguir el mismo proceso de `utf-7` donde primero convierte los caracteres a `utf-16` y luego los pasa a base64:
![[{B8605714-33B8-4896-B1A9-DF3F4DE392AE}.png]]

vemos que ya tenemos el valor de la `a` que es `AGE`.

nuestra cadena queda de la siguiente manera:

![[{4F39C21F-BB2D-4D89-B42A-C605D46855F1}.png]]

y vamos a intentarlo nuevamente en la maquina:

![[{6146A4ED-FFA5-4FCF-AE8D-FE9F1AF53317}.png]]

![[{7DE6587D-B65C-4F09-8ECC-27BF00C2EE2E}.png]]

De forma general el concepto se logro el concepto ya que nos permite pasar.

Entonces ahora podemos aprovecharnos de esto de la siguiente manera:
![[{948EBDAD-631D-4657-883A-05BC55C40196}.png]]

estamos ingresando texto claro luego el `@` como `&AEA-` luego el dominio malicioso por ultimo un espacio ques `&ACA-` y al final de todo esto el dominio valido, vemos si esto nos funciona:
![[{299E0291-19B6-4F9E-97AD-0C9F74CF88EF}.png]]

![[{4E838832-60D7-42BC-A0C3-E82A1103C731}.png]]

![[{655C4054-CA37-4B61-80D4-88524744FAF9}.png]]

ya podemos activar la cuenta, vamos a ingresar:
![[{38B6D896-22A0-4BD2-976D-38547D11731C}.png]]

podemos completar el laboratorio:
![[{891AB5FA-BFAE-4410-9026-5847C0342334}.png]]

Lab Terminado.