---
aliases: [Laboratorios de Ciberseguridad Ofensiva, Vulnerabilidades Lógicas, Bypass de Autenticación]
tags:
  - hacking/web
  - hacking/explotacion
  - herramientas/burpsuite
  - hacking/laboratorios
estado: 🟢 Terminado
---
# Laboratorios de Vulnerabilidades Lógicas y de Autenticación

### Lab: Excessive trust in client-side controls

En este laboratorio no se valida de forma correcta el input del usuario y podemos explotar cierta lógica al momento de pagar algún producto.

> [!info] Validación en el lado del cliente
> La validación en el lado del cliente (client-side validation) se realiza en el navegador del usuario antes de que los datos se envíen al servidor. Aunque mejora la experiencia del usuario al proporcionar retroalimentación instantánea, **nunca debe ser la única medida de seguridad**. Los atacantes pueden fácilmente omitir esta validación modificando las peticiones HTTP directamente (por ejemplo, con Burp Suite) o deshabilitando JavaScript. Es crucial implementar siempre una validación robusta en el lado del servidor (server-side validation).

![[{57C5A9E1-58D2-424A-8944-75E839643FF4}.png]]

Comenzamos con 100 dólares como crédito y vamos a ver que lo que tenemos que comprar no nos alcanza, pero vamos a ver qué es lo que está pasando. Podemos agregar el producto al carrito, pero interceptar la petición para ver cómo se está tramitando:

![[{F2C65109-C3FC-4053-BE0D-33AFB8A8B30B}.png]]

Vemos que al agregar el producto al carrito se tramita la cantidad y el precio. Podemos intentar algo simple y es ver qué pasa si modificamos el precio y dejamos que pase la solicitud:

![[{EAC7EB93-B847-4ACC-BB05-8F4A9E46121F}.png]]

En este caso se cambió por un 1 ya que no nos dejó usar el 0, pero vemos que logramos cambiar el precio. Para completar la máquina tenemos que concretar la compra y listo.

![[{E866293E-83C5-43CE-82D5-2D386E020778}.png]]

Lab Terminado.

### Lab: High-level logic vulnerability

En este caso, nuevamente no se valida de forma correcta el input del usuario y podemos afectar el flujo del pago con el objetivo de comprar ítems a otro precio.

De igual forma, analicemos cómo es que está trabajando por detrás este flujo de pago:

![[{F6DB7A10-EA95-420E-9098-A87F1E8E748A}.png]]

Podemos, por medio de fuerza bruta, ver cómo se implementa el precio.

Vemos que podemos alterar la cantidad. Veamos qué pasa con cantidades negativas:
![[{9DF6CD68-1305-413D-9508-F866466AF7A0}.png]]

Vemos que, en efecto, se aplican. No nos permite pagar, pero vamos a ver que no nos deja.

> [!warning] Vulnerabilidades de lógica de negocio con cantidades negativas
> Permitir cantidades negativas en transacciones puede llevar a graves vulnerabilidades. Un atacante podría "comprar" una cantidad negativa de un producto, lo que en la lógica de negocio podría interpretarse como una "devolución" o un "crédito", resultando en un saldo positivo para el atacante sin haber pagado. Es fundamental que todas las cantidades sean validadas como positivas y dentro de rangos lógicos en el lado del servidor.

En este punto, lo que podemos hacer es jugar con valores positivos y negativos para intentar llegar a un valor total de 0 o 1, pero la verdad es que no podemos obtener `-1` chaqueta, por lo que podemos intentar mejor generar valores negativos desde otro producto y luego comprar el producto designado:

![[{B36D93FE-CDB3-4115-B8A3-29B34828B336}.png]]

Ahora vamos a comprar una chaqueta y `-14` EGGtastic, por lo que vamos a ver si esto nos lo permite:

![[{7031C77D-DC00-4AE1-8EF8-C45830952033}.png]]


Lab Terminado.

### Lab: Inconsistent security controls

En este laboratorio, lo que vamos a encontrar es que tenemos que acceder a una propiedad administrativa en `/admin` y tenemos que eliminar al usuario `carlos`.

No tenemos acceso a la máquina, pero tenemos un panel de registro:

![[{D76D1DFD-4E20-4AD2-82E1-BE7F021EFC80}.png]]

Vemos que nos da un dominio si es que pertenecemos a cierto dominio, por decirlo de alguna manera.

En este punto, lo que vamos a hacer es registrarnos con nuestro correo de atacante para activar la cuenta y ver qué nos permite desde dentro:

![[{1AE0E093-0BD0-4A66-895C-8155E7EBFE59}.png]]

Perfecto, ya estamos como otro usuario, pero vemos que podemos actualizar el correo y conocemos el dominio de la empresa externa que permite el acceso a paneles de administrador. Vamos a intentarlo:

![[{DD39DD78-65EA-474D-B661-C8CFE169F809}.png]]

Vemos que la lógica no es correcta debido a que, al poder cambiar el correo y además tener el dominio que tiene todos los permisos, se activa el panel de admin mediante el cual podemos eliminar al usuario `carlos`, que es nuestro objetivo:

> [!tip] Spoofing de dominio de correo electrónico para bypass de autenticación
> Algunas aplicaciones web confían en el dominio del correo electrónico para determinar los privilegios del usuario. Si un atacante puede cambiar su dirección de correo electrónico a un dominio de confianza (ej. `admin.example.com`) después del registro inicial, y la aplicación no revalida adecuadamente los permisos, puede escalar privilegios. Esto es común en sistemas que usan un "dominio de confianza" para identificar administradores o usuarios con roles especiales.

![[{4DBC6487-5FDF-4EB4-B339-3D6EEB688584}.png]]

Lab Terminado.

### Lab: Flawed enforcement of business rules

En este caso, tenemos que volver a comprar la chaqueta y tenemos que burlar nuevamente el pago.

Vamos a ver qué tenemos primero en nuestra web:

![[{9FBB4DB6-76DB-4B58-A742-7C79E24CE1FE}.png]]

Al parecer nos está aplicando un descuento. Vemos algo curioso al final, vemos que muestra esto:
![[{F4FC88A6-3280-4B7C-A851-C886F3677C32}.png]]

![[{9B5EDBE3-679E-4137-B13A-321288CCA15E}.png]]

Nos da un cupón, esto es bueno. Podemos intentar agregar cosas al carrito y ver qué nos realiza el cupón:

![[{1344ADC9-842B-4089-AF5F-62173C1BB801}.png]]

Vemos que podemos agregar más cupones, pero no los mismos de forma simultánea. Pero, ¿y si alternamos estos dos cupones? Puede que la lógica sea revisar por el último cupón usado. Veamos qué sucede:

> [!info] Apilamiento de cupones (Coupon Stacking)
> El apilamiento de cupones es una vulnerabilidad de lógica de negocio donde una aplicación permite aplicar múltiples descuentos o cupones de forma consecutiva o simultánea, incluso si no está diseñado para ello. Esto puede llevar a que el precio final de un producto se reduzca drásticamente o incluso a cero, permitiendo compras gratuitas. La validación del lado del servidor debe asegurar que los cupones se apliquen solo una vez o según las reglas de negocio establecidas.

![[{5643E3D2-00E2-40AF-954F-730FE533AE7D}.png]]

Perfecto, logramos aplicar los mismos cupones de forma consecutiva y llegamos al cero. Ya podemos comprar:
![[{021658AE-1352-4D55-8CED-7EC7A8645A3B}.png]]

Lab Terminado.

### Lab: Low-level logic flaw

En este laboratorio tampoco se valida de forma óptima los inputs de usuario, por lo que vamos a ver qué podemos hacer. Recordemos que sí tenemos permisos para ingresar como `wiener`.

En este laboratorio podemos intentar varias cosas de las anteriores que hemos intentado, pero vamos a observar que no funciona.

Tenemos que tener claro un concepto y es que en lenguajes como C, C++ y en versiones de PHP, vamos a ver que cuando se asigna una variable de tipo entera o numérica en general, va en un rango determinado de números que varía desde los negativos y luego los positivos. El problema es que si se supera este número, lo que va a suceder es que comenzará una nueva cuenta desde los negativos.

> [!warning] Desbordamiento de enteros (Integer Overflow)
> Un desbordamiento de enteros ocurre cuando una operación aritmética intenta crear un valor numérico que está fuera del rango que puede representar el tipo de datos entero. En muchos lenguajes, esto puede causar que el valor "envuelva" (wrap around) al extremo opuesto del rango (por ejemplo, un número positivo muy grande se convierte en un número negativo muy grande). Los atacantes pueden explotar esto para manipular precios, saldos o contadores, llevando a resultados inesperados y vulnerabilidades de lógica de negocio.

Teniendo esto en cuenta, con un ataque de fuerza bruta, vamos a ver cuál es el límite de dinero que se asigna y si este cambia en algún punto:

![[{2FC722E2-C8A9-4F37-9306-8FEF20BCC822}.png]]

Vamos a realizar mediante el ataque que suba hasta ver números negativos y luego paramos para calcular un número:
![[{09DAEDE6-7A92-4DA6-99EE-28F734C1DEB0}.png]]

Perfecto, cuando estemos cerca del 0, como veremos en la siguiente imagen:

![[{A6520597-DA58-4761-8E3F-575C56E66A5A}.png]]

Bueno, ya estamos en este caso considerando que nos movemos de a una cantidad de 99, es cerca. Ahora podemos calcular un aproximado de cuánto nos falta y volver a hacer el ataque, pero más medido:
![[{B76D5475-DA58-454C-93DA-1C5E1087A466}.png]]

Perfecto, ya logramos llegar a un valor que podemos pagar. Veamos si lo logramos:

![[{1698D4DD-F6D6-4A4B-876E-610AA26A661B}.png]]

Lab Terminado.

### Lab: Inconsistent handling of exceptional input

En este laboratorio no se valida de forma correcta el input del usuario y para esto vamos a explotar un apartado de registro para ganar acceso a la parte administrativa.

![[{2E22AC79-87CC-4078-A8C5-8846CE5FD475}.png]]

Vemos, al igual que en otra máquina, el formulario para registrarnos. Bueno, como este es el punto en el que vamos a explotar una cosa, podemos comenzar intentando ver el límite de caracteres en el campo 'Email' para determinar qué se puede o no hacer:

![[{D04B4542-F65D-463A-A8D5-D8790CEF59EA}.png]]

Enviamos 200 'A' y sí llegaron todas. Esto nos permitió tener esto activo. Vemos si dentro de la web todo normal también:

![[{6EBF14D3-CF65-48A3-8390-085CEB8069AD}.png]]

Vemos que el dominio no se representó completo. Podemos intentar ver cuántos caracteres tenemos en total, que son 255. Con esto, lo que vamos a intentar es si nosotros intentamos que a partir de estos 255 caracteres los demás puede que no se representen, pero sí se interpreten. Podemos intentarlo, en sentido de cadena sería `255` hasta el dominio con permisos y luego le agregamos el subdominio que, al parecer, en el post-procesado lo elimina:

> [!info] Truncamiento de cadenas y discrepancias de parsing
> Las aplicaciones pueden manejar las cadenas de texto de manera inconsistente en diferentes etapas del procesamiento (entrada, almacenamiento, visualización). Si un campo de entrada tiene un límite de caracteres en la base de datos o en la lógica de negocio que es diferente al límite de validación o visualización, un atacante puede inyectar una cadena larga que se trunca en un punto específico, pero que es interpretada de manera diferente por otra parte del sistema. Esto puede llevar a bypass de autenticación o escalada de privilegios, especialmente en campos como direcciones de correo electrónico o nombres de usuario.

![[{9C6CBBDC-133D-41DF-9AFD-44FDE9A575FE}.png]]

Creamos la cadena donde justo estén los 255, pero luego agregamos como subdominio nuestro email para que funcione. Lo que pasará es que al representarse se cortará y solo llegará como si nos registráramos con nuestro correo:

![[{A71BC26D-2932-46C6-B93D-35748A332A4C}.png]]

![[{C30476B3-C6E0-4746-A27F-1AD4D3EF5CE3}.png]]

Vemos que nos llega el correo, esto ya es bueno. Vamos a activar la cuenta e ingresar:

![[{8616885B-05F6-4B74-AF75-110FC431E2F3}.png]]

Vemos que dentro de la web se cortó el apartado del subdominio, provocando que solo quede el dominio que permite el panel de administrador:

![[{7BDECDFC-52D6-4692-8C37-57090E857850}.png]]

Lab Terminado.

### Lab: Weak isolation on dual-use endpoint

En este laboratorio vamos a intentar acceder como usuario administrador y borrar el usuario `carlos`.

Vamos a hacer login a ver qué encontramos primero:

![[{92AD9370-A0E8-460A-9798-9B62C1755EE1}.png]]

Vemos eso. Podemos intentar ver qué se tramita por detrás para intentar hacer alguna modificación:

![[{4A53CE8C-7081-4F3F-8622-F8F79CF2FF81}.png]]

Vemos que estamos tramitando diferentes campos. Algo que se me viene a la mente primero es eliminar la validación del `current-password`:

> [!tip] Bypass de validación de contraseña actual
> En funciones de cambio de contraseña o actualización de perfil, a menudo se solicita la "contraseña actual" para verificar la identidad del usuario. Si esta validación se realiza únicamente en el lado del cliente o es débil en el lado del servidor, un atacante puede interceptar la petición y eliminar el parámetro `current-password` o enviar un valor vacío, permitiendo cambiar la contraseña o el rol sin conocer la credencial original.

![[{7AD6EA48-EF30-416D-AE94-CD8571FB354E}.png]]

Pues no se valida. Y si ahora hacemos lo mismo, pero para `administrator`:

![[{F8307FE9-1303-4928-B6B9-7CFD9BAA02C4}.png]]

Vemos ya que, al parecer, esto funcionó:

![[{FE9B548D-3055-4955-A545-B29E71BCDBA7}.png]]

Lab Terminado.

### Lab: Insufficient workflow validation

Este laboratorio asume ciertos eventos al momento de hacer el pago donde, si logramos golpear partes de estos, asumirá cosas que no son. Como en labs anteriores, tenemos que comprar la chaqueta.

Vamos directamente a ver cómo se tramita el pago:

![[{0910664F-07AA-4FEF-9C75-9D314613F3F6}.png]]

Vemos cómo se tramitan ciertas funciones, pero como es algo que no podemos pagar, no es algo del todo certero, por lo que vamos a comprar algo que podemos pagar para ver cómo se tramita todo:

![[{64C4B66C-34AA-41A6-8032-EDA6488D2195}.png]]

En este caso ya vemos otras peticiones y una que llama mi atención que se tramita `order-confirmation`, la cual si revisamos:

![[{C41DC7D2-2A22-43C5-A35E-1966403B7249}.png]]

Ahora no tenemos nada en el carrito, pero si nosotros ahora ponemos algo más al carrito y luego, desde el repeater, tramitamos esta solicitud, vemos qué sucede:

> [!warning] Bypass de flujo de trabajo (Workflow Bypass)
> Las vulnerabilidades de bypass de flujo de trabajo ocurren cuando una aplicación no valida adecuadamente la secuencia de pasos que un usuario debe seguir para completar una tarea. Un atacante puede saltarse pasos críticos (como el pago o la validación de existencias) enviando directamente peticiones a endpoints posteriores en el flujo, asumiendo que los pasos anteriores ya se han completado correctamente. Esto puede llevar a compras gratuitas, acceso no autorizado o manipulación de datos.

![[{69B34B6D-3815-4F30-824F-95C637E373BD}.png]]

Ahora tramitemos nuevamente la solicitud de confirmación desde el repeater:

![[{74694AEF-00D5-407C-B65E-F63E511FA51D}.png]]

Como podemos observar, se confirma la compra, además de que no nos descuenta dinero, ya que recordemos que no teníamos suficiente:

![[{8D80CAE7-D86C-4452-88BB-91552E0CDD94}.png]]

Lab Terminado.

### Lab: Authentication bypass via flawed state machine

Primero tenemos que comprender qué es una máquina de estado y lo que hacen de forma general es modelar o estructurar algo en la web, en este caso la autenticación, donde los estados serán como partes o secciones ordenadas que seguirá el código desde el formulario para hacer login (estado 1), seleccionar rol (estado 2), definir gustos (estado 3), etc.

Al parecer tenemos una máquina de estados y mediante esta tenemos que realizar un bypass a la autenticación. El objetivo es eliminar al usuario `carlos`.

![[{2323A5D1-AC1C-4224-BC4D-3A4945E0D24B}.png]]

Perfecto, vemos que también tenemos un `role-selector`, que es lo siguiente:

![[{2EEDDD0A-3E83-4BC9-B105-491D4CBAC028}.png]]

Vemos solo dos opciones, pero si capturamos esto y lo modificamos de la siguiente manera:

![[{D14F3DC2-0D5B-4FA3-B1CA-7E1CF1FEE967}.png]]

Vemos si funciona:

![[{0BB1DBBC-717F-4AF0-BCC6-A56FA07F1A23}.png]]

Pues no, no funcionó. Pero si nosotros lo hacemos ahora paso a paso:

![[{3ED925C6-F00B-4E62-8B1B-3AB4EA02212C}.png]]
Como vemos, primero la solicitud del login, la cual la dejamos pasar. Ahora vamos con la siguiente:
![[{95514747-85F1-44B6-86C8-A61724BFCA01}.png]]
Ahora llega la del rol, ¿y qué pasa si esta no la dejamos pasar? Tiene o debería tener un estado por defecto. Veamos qué pasa:

> [!info] Bypass de máquina de estados en autenticación
> Una máquina de estados define las transiciones válidas entre diferentes etapas de un proceso (ej. autenticación, compra). Una vulnerabilidad ocurre si la aplicación no valida que el usuario ha pasado por todos los estados requeridos o si asume un estado por defecto inseguro cuando una transición es omitida. Al interceptar y omitir una petición de selección de rol, el sistema podría asignar un rol por defecto (como `admin`) si no hay una validación estricta de la secuencia de estados.

![[{EB0D5777-60E6-43AE-91A1-4E62D2C5E2E3}.png]]

Al parecer, el valor por defecto que tenemos es como admin al no asignarle nada.

![[{1D9C3F46-9C1D-4E32-B988-234798A7C706}.png]]

Lab Terminado.

### Lab: Infinite money logic flaw

La idea es, como siempre, intentar comprar la chaqueta nuevamente. En este caso, al igual que en anteriores, vamos a contar con las credenciales del usuario `wiener`.

![[{BBD79477-5CBC-4C33-A149-416199E54673}.png]]

Vemos esto en el panel. Al parecer, podemos usar Tarjetas de regalo mediante las cuales podríamos canjear dinero. Esto es interesante. Además, conseguimos un cupón de descuento `SIGNUP30`, el cual podemos utilizar.

![[{B74B78F9-5DB6-4868-9A27-51DC5C066583}.png]]

Al parecer, podemos comprar estas Tarjetas de regalo a 10 dólares y son de 10 dólares. Pero, ¿qué pasa si le aplicamos el cupón de descuento?
![[{FDAD89D1-EFE1-4E15-88D8-5121CE330654}.png]]

Lo que estamos viendo es que compramos una tarjeta de 10 dólares en 7, y como estas tarjetas podemos utilizarlas en la misma cuenta, podríamos ver qué sucede si compramos varias:
![[{CF9A348A-2ED9-444A-979C-458E753D22CB}.png]]

Aquí vamos a aplicar un poco de lógica y es que el total es de 14 tarjetas de regalo a 7 dólares cada una, pero cada una de estas nos retorna un total de 10 dólares. Entonces, si canjeamos esto, luego lo que va a suceder es que nos dará hasta 140 dólares más los 2 que ya teníamos, 142. Veamos si esto funciona, pero por fines prácticos y porque no resulta práctico estar poniendo los 14 cupones cuando tenemos que llegar a 1333 con algo, sería ilógico, por lo que primero comprobemos la teoría:
![[{1892E7A7-42A1-4A09-AD1E-2DAB42F80A76}.png]]

Usemos los códigos a ver qué sucede:
![[{020D4098-E311-455C-8C04-FE452C767A83}.png]]

Una vez canjeados los dos códigos, vemos que logramos ganar un total de 3 dólares por cupón.

Bueno, lo mejor que podemos hacer es automatizar este proceso mediante Burp Suite.

El proceso es el siguiente:

Vamos a la configuración de Sesión y en reglas de Sesión agregamos una nueva y se desplegará el siguiente panel:
![[{3A971BFB-23F2-448E-B9D7-D0CBEC07B53E}.png]]

Le damos un nombre y en reglas le agregamos una nueva. Una vez que agreguemos los links necesarios, lo que vamos a ver es:
![[{40D0DEF8-A086-4110-A3C3-856851190B59}.png]]

Donde en nuestro caso modificamos esa petición GET para que se nos seleccione el nuevo cupón que se genere:

> [!tip] Automatización con Burp Suite Session Handling Rules
> Las reglas de manejo de sesión de Burp Suite permiten automatizar tareas repetitivas durante las pruebas de penetración. Se pueden configurar para:
> 1.  **Actualizar tokens**: Extraer nuevos tokens CSRF o de sesión de respuestas y aplicarlos a peticiones subsiguientes.
> 2.  **Realizar logins**: Ejecutar una secuencia de login si la sesión actual expira.
> 3.  **Modificar peticiones**: Insertar o modificar parámetros en peticiones salientes basándose en la lógica definida.
> Esto es invaluable para ataques de fuerza bruta, fuzzing o para mantener una sesión activa durante pruebas prolongadas.

![[{175E793A-A447-49A9-AD76-74F223995D08}.png]]

Damos en agregar y configuramos lo necesario. En este caso ya está listo.

Podemos dar a Test Macro y si el valor de dinero cambia, pues sí funciona:
![[{92D8E269-AE8F-46DD-B392-614B73789FFC}.png]]

Ahora sí ya funciona, lo que vamos a hacer es dar a OK a todo y obtener una petición al home:
![[{BA665C4A-3180-4141-B9BC-791EECC62387}.png]]

Al enviarla ya se automatizará todo:
![[{79B76988-94DE-4530-A820-C825C36C299C}.png]]

Vemos que el valor irá aumentando. Ahora es cuestión de que desde el Intruder enviemos un ataque de unas 100 peticiones y ya con esto deberíamos obtener dinero suficiente:
![[{96DB6A96-A78E-4233-BA9C-6705FFFED46F}.png]]

Ahora podemos ver qué sucede al terminar el ataque:
![[{726DD657-06A5-40A4-92FA-A3BBC4C15857}.png]]

![[{CD8AB84A-5C75-484C-B2B0-BA8DA7342095}.png]]

Perfecto, después de casi 500 peticiones y más, vamos a ver que tenemos el dinero necesario para comprar la chaqueta. Vamos a ver si ya podemos completar el laboratorio:

![[{952BF000-252D-4707-A423-AE072301237B}.png]]

Lab Terminado.

### Lab: Authentication bypass via encryption oracle

El objetivo es intentar llegar al panel de administrador y eliminar el usuario `carlos`.

Bueno, vamos a capturar todo el tráfico cuando iniciamos sesión y un poco más para ver qué podemos identificar:
![[{479F82E8-30CA-4112-AC75-EEA9CB6C255E}.png]]

Vemos que en una de las peticiones logramos obtener datos que viajan cifrados, al parecer URL-encoded y luego en Base64, que si intentamos ver en claro:

![[{343617E3-096D-473E-97F4-DE0E3A60B51F}.png]]

Vemos que, en efecto, esto no es legible, pero ya con esto suponemos que se están cifrando datos.

Vemos que podemos crear comentarios dentro de los blogs. Vemos cómo funciona:
![[{36DCF7BA-2E43-49BC-B98C-F4BB90A9D70C}.png]]

Creamos y analizamos cómo se tramitó todo, pero no encontramos en sí algo que nos pueda ayudar.

Vamos a nivel de sintaxis a intentar generar algún error:

![[{066430F6-494B-48E7-9D50-F6FA41976EBC}.png]]

Vemos que nos reporta un error. Veamos la petición:

![[{356A72E8-7E3A-4AB6-9357-07BC1936F4B5}.png]]

Vemos que antes de la respuesta web tenemos una respuesta por detrás con una cookie `notification` que, si intentamos ver el texto claro, no vamos a poder porque al parecer no es legible.

Puede ser que parte del mensaje de error esté siendo tramitado al servidor mediante la cookie `notification` y que el servidor nos lo descifre para luego mostrarlo.

Podemos intentar algo y es pasarle el contenido de `stay-logged-in` que nosotros no podíamos descifrar a ver si el servidor logra hacerlo:
![[{E1C36B8E-51D7-4BC9-B13D-13DA92A2E466}.png]]

Pues sí que lo hace, y ahora lo que sabemos es que el contenido es `wiener:...`. Ahora lo que sabemos es que puede ser que como el mensaje está en el campo email, podríamos enviar algo parecido en email primero a ver si podemos:
![[{DF68F77F-4833-43C2-B6F3-2C06E4D3CF94}.png]]

Intentemos mandar eso en email a ver qué nos retorna en `notification`:
![[{2DA70610-2624-47F6-ABE6-004AB48A0504}.png]]

Vemos la nueva cadena. Podemos intentar descifrar esa cadena a ver cómo la codificó:
![[{E83BD7DA-5DDF-4965-BB9F-524EB65725B6}.png]]

Vemos que, por defecto, impone el valor de `invalid email address:` y luego ingresa el valor que le pasamos.

Bueno, entonces nuestro objetivo en este punto es lograr que la primera parte del mensaje encriptado no esté. Podemos intentar primero lo siguiente.

Vamos a darnos cuenta de que el mensaje antes de lo que necesitamos son 24 caracteres:
![[{61A5F005-8684-4AB8-8D25-4C0B03B471A2}.png]]

Podemos ayudarnos de Decoder para pasarlo a bytes y eliminar los primeros 24:

![[{0BFAD9AF-A6A2-4318-80A9-932C0D62A3D2}.png]]

Y luego ver el resultado y eso pasarlo a Base64:
![[{D0021AD6-02FE-4CB2-9332-6988AE25CDA4}.png]]

Podemos copiarnos la nueva cadena en Base64 e intentar pasarla en la cookie para ver si obtenemos lo que queremos:
![[{153846F3-2085-4C62-963B-A72B769A9516}.png]]

Vemos que con eso generamos un error donde dice que la cadena tiene que ser un múltiplo de 16. Lo que se me viene a la mente es el cifrado AES, el cual se implementa cifrado en bloques a comparación de otros como el stream cipher que utiliza cifrado byte a byte. Bueno, en este cifrado por bloques de 16 bytes se cifra el contenido empleando una clave.

> [!warning] Oráculo de cifrado (Encryption Oracle) y Padding Oracle
> Un oráculo de cifrado es una vulnerabilidad donde un atacante puede obtener información sobre el texto plano cifrado al observar cómo el servidor responde a diferentes textos cifrados manipulados. En el caso de un "padding oracle", el servidor revela si el padding de un bloque cifrado es válido o no. Esto permite a un atacante descifrar el texto cifrado o cifrar texto plano arbitrario sin conocer la clave de cifrado, explotando la forma en que el sistema maneja los errores de padding en cifrados por bloques como AES en modo CBC.

El problema radica en que si contamos el número de bytes actual, tenemos lo siguiente:
![[{94771CA2-BF64-4627-9AC1-B908BAF463F7}.png]]

Exactamente, la última línea no tiene los 16 bytes completos y este es el problema. Usualmente, en las cadenas, en automático el sistema al computarlo va a generar lo que se llama padding, que es un relleno de los bytes faltantes. La cuestión se da cuando nosotros, al eliminar los bytes, descompensamos esto y quedó inválido por ahora. Entonces, lo que tenemos que hacer es cuadrar manualmente para que cuando eliminemos estos 23 bytes (el espacio no cuenta), quede la cadena compensada. Para esto, tenemos que hacer que nos quede todo un bloque completo para que así, al momento de eliminar, no sean 23, sino más de eso, donde quedarán dos bloques completos. A partir del 23, nos quedan 9 bytes más, los cuales vamos a agregar al comienzo de nuestra cadena:
![[{69B09371-B4B1-4C0E-A643-AF011649FDC9}.png]]

Con esto, lo que debe pasar es que vamos a eliminar dos bloques completos y no a medias, provocando que los valores queden exactos como veremos:

![[{508CFC64-E5C1-4A27-8871-537731163856}.png]]

Ahora que tenemos el valor con la nueva cadena, vamos a eliminar dos bloques completos, ya que para completar esos dos bloques es que usamos las 'x':

![[{EEC221F8-A3F2-4C1C-A39B-ACDA654F9E80} 1.png]]

Repetimos, como vemos, el proceso y tenemos la nueva cadena. Vemos qué sucede:

![[{7C0A8033-370A-4B5A-9BFD-33CB935B9315}.png]]

Ya tenemos la cadena correcta. Vemos qué pasa si ahora le pasamos esto a la otra cookie:
![[{2D606B49-8D6A-4A7D-B8E0-86AECFC59F1D}.png]]

Se despliega el panel de administrador. Tenemos que tener en cuenta que no tenemos que tener iniciada la sesión debido a que entra en conflicto. Ya con esto podemos resolver el lab:

![[{E11531D1-2ED1-4740-851C-CCCBEB6D25B9}.png]]

Lab Terminado.

### Lab: Bypassing access controls using email address parsing discrepancies

El objetivo de esto es lograr entrar como administrador y, como ya es costumbre, eliminar al usuario `carlos`.

Como lo hemos hecho en máquinas anteriores, podemos intentar usar el correo y registrar lo que nos da como un subdominio para intentar registrar un usuario con permisos de administrador de la siguiente manera:
![[{10814764-3AD1-40D4-A802-CC2F8939A07D}.png]]

No nos permite pasar por los dos `@` que se están usando y generan un error. Podemos intentar enviarlos sin el `@` y ponerlo al interceptarlo con Burp Suite a ver qué sucede:
![[{9E7643D1-0EA4-423B-A6FE-B9AD5AC6D478}.png]]

Vemos qué sucede:

![[{12F2CDB4-DDC2-42B7-988A-2109149A0EAE}.png]]

Pues no funciona.

Bueno, la discrepancia que va a pasar es que le vamos a hacer creer que nos registramos con el dominio que nos pide, que es el autorizado, pero en realidad va a ser con el malicioso.

Vamos a dejar claros algunos conceptos aquí y es que los correos electrónicos por detrás tienen una algoritmia muy compleja donde no es solo como lo vemos los usuarios, sino que el post-procesado de los mismos es algo complejo y muy bien planeado. Es esto lo que vamos a atacar y tenemos muchos tipos de ataques como son:

-   Encoded-Word.
-   Punnycodes malformados.
-   Caracteres nulos y escapados.

Para una lectura educativa, podemos revisar el siguiente [link](https://portswigger.net/research/splitting-the-email-atom)

Bueno, podemos hacer muchos tipos de codificaciones o trucos, pero el objetivo siempre es el mismo: hacer que el dominio aparente ser el válido, pero por detrás se interpreta el no válido, por decirlo de alguna forma.

Entonces, lo primero que tenemos que hacer es ver qué es lo que interpreta y no el servidor, y en este caso vamos a jugar con `Encoded-Word` de la siguiente forma:
![[{36126476-D439-4567-A021-5812D4C9CA9D}.png]]

Entonces, esa es la estructura general que vemos en la imagen, y en este caso vamos a usar un `charset=utf-8`, `encoding=q`, y `encode-text` lo definimos en hexadecimal, quedando de la siguiente manera:

![[{477EECDA-816A-4379-8D01-BFBDB5007B84}.png]]

Perfecto, ahora vamos a usarlo de la siguiente manera:
![[{0F1EBBAA-F967-4885-A783-314D884EE469}.png]]
![[{24418FCC-31F2-42EB-8F99-CDB16A527EA8}.png]]

Antes o después de la cadena podemos agregar texto, pero vemos que no nos deja. Esto puede ser que esté detectando un intento de `Encoded-Word`, por lo que nos lo bloquea.

En este punto, vamos a tener que buscar otra forma definida en el RFC 2047, que es en lo que se basa este tipo de inyección, donde vamos a intentar implementar caracteres no ASCII para que de alguna manera sí nos lo interprete.

En sí, vamos a cambiar, quién sabe, a `UTF-7`, la cual es menos usada, pero donde solo usa caracteres ASCII de 7 bits:
![[{07E5D376-79B1-4727-BEC1-5CC9D5D70B47}.png]]

La estructura general cambia debido al proceso que se emplea en este caso, donde vemos que donde va el texto ponemos `&-` ya que la codificación comienza con `&` y termina con `-`. Ahora, para el texto, vamos a hacer uso de CyberChef donde vamos a seguir el mismo proceso de `UTF-7` donde primero convierte los caracteres a `UTF-16` y luego los pasa a Base64:
![[{B8605714-33B8-4896-B1A9-DF3F4DE392AE}.png]]

Vemos que ya tenemos el valor de la `a` que es `AGE`.

Nuestra cadena queda de la siguiente manera:

> [!info] Discrepancias de parsing de direcciones de correo electrónico (RFC 2047)
> Las direcciones de correo electrónico pueden ser interpretadas de manera diferente por distintos componentes de una aplicación debido a la complejidad de los estándares (como RFC 2047 para "Encoded-Word" o RFC 6531 para internacionalización). Un atacante puede explotar estas discrepancias para hacer que una dirección de correo electrónico parezca válida para un sistema de validación (ej. un filtro de entrada), pero que sea interpretada como una dirección diferente (ej. con un dominio de administrador) por otro componente (ej. el sistema de autenticación o autorización). Esto puede llevar a bypass de controles de acceso.

![[{4F39C21F-BB2D-4D89-B42A-C605D46855F1}.png]]

Y vamos a intentarlo nuevamente en la máquina:

![[{6146A4ED-FFA5-4FCF-AE8D-FE9F1AF53317}.png]]

![[{7DE6587D-B65C-4F09-8ECC-27BF00C2EE2E}.png]]

De forma general, el concepto se logró, ya que nos permite pasar.

Entonces, ahora podemos aprovecharnos de esto de la siguiente manera:
![[{948EBDAD-631D-4657-883A-05BC55C40196}.png]]

Estamos ingresando texto claro, luego el `@` como `&AEA-`, luego el dominio malicioso, por último un espacio que es `&ACA-` y al final de todo esto el dominio válido. Vemos si esto nos funciona:
![[{299E0291-19B6-4F9E-97AD-0C9F74CF88EF}.png]]

![[{4E838832-60D7-42BC-A0C3-E82A1103C731}.png]]

![[{655C4054-CA37-4B61-80D4-88524744FAF9}.png]]

Ya podemos activar la cuenta. Vamos a ingresar:
![[{38B6D896-22A0-4BD2-976D-38547D11731C}.png]]

Podemos completar el laboratorio:
![[{891AB5FA-BFAE-4410-9026-5847C0342334}.png]]

Lab Terminado.