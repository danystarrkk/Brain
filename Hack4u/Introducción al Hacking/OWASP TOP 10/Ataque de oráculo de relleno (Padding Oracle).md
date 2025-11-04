------
Un ataque de oráculo de relleno (Padding Oracle Attack) es un tipo de ataque contra datos cifrados que permite al atacante descifrar el contenido de los datos sin conocer la clave.

Un oráculo hace referencia a una “indicación” que brinda a un atacante información sobre si la acción que ejecuta es correcta o no. Imagina que estás jugando a un juego de mesa o de cartas con un niño: la cara se le ilumina con una gran sonrisa cuando cree que está a punto de hacer un buen movimiento. Eso es un oráculo. En tu caso, como oponente, puedes usar este oráculo para planear tu próximo movimiento según corresponda.

El relleno es un término criptográfico específico. Algunos cifrados, que son los algoritmos que se usan para cifrar los datos, funcionan en bloques de datos en los que cada bloque tiene un tamaño fijo. Si los datos que deseas cifrar no tienen el tamaño adecuado para rellenar los bloques, los datos se rellenan automáticamente hasta que lo hacen. Muchas formas de relleno requieren que este siempre esté presente, incluso si la entrada original tenía el tamaño correcto. Esto permite que el relleno siempre se quite de manera segura tras el descifrado.

Al combinar ambos elementos, una implementación de software con un oráculo de relleno revela si los datos descifrados tienen un relleno válido. El oráculo podría ser algo tan sencillo como devolver un valor que dice “Relleno no válido”, o bien algo más complicado como tomar un tiempo considerablemente diferente para procesar un bloque válido en lugar de uno no válido.

Los cifrados basados en bloques tienen otra propiedad, denominada “modo“, que determina la relación de los datos del primer bloque con los datos del segundo bloque, y así sucesivamente. Uno de los modos más usados es CBC. CBC presenta un bloque aleatorio inicial, conocido como “vector de inicialización” (IV), y combina el bloque anterior con el resultado del cifrado estático a fin de que cifrar el mismo mensaje con la misma clave no siempre genere la misma salida cifrada.

Un atacante puede usar un oráculo de relleno, en combinación con la manera de estructurar los datos de CBC, para enviar mensajes ligeramente modificados al código que expone el oráculo y seguir enviando datos hasta que el oráculo indique que son correctos. Desde esta respuesta, el atacante puede descifrar el mensaje byte a byte.

Las redes informáticas modernas son de una calidad tan alta que un atacante puede detectar diferencias muy pequeñas (menos de 0,1 ms) en el tiempo de ejecución en sistemas remotos. Las aplicaciones que suponen que un descifrado correcto solo puede ocurrir cuando no se alteran los datos pueden resultar vulnerables a ataques desde herramientas que están diseñadas para observar diferencias en el descifrado correcto e incorrecto. Si bien esta diferencia de temporalización puede ser más significativa en algunos lenguajes o bibliotecas que en otros, ahora se cree que se trata de una amenaza práctica para todos los lenguajes y las bibliotecas cuando se tiene en cuenta la respuesta de la aplicación ante el error.

Este tipo de ataque se basa en la capacidad de cambiar los datos cifrados y probar el resultado con el oráculo. La única manera de mitigar completamente el ataque es detectar los cambios en los datos cifrados y rechazar que se hagan acciones en ellos. La manera estándar de hacerlo es crear una firma para los datos y validarla antes de realizar cualquier operación. La firma debe ser verificable y el atacante no debe poder crearla; de lo contrario, podría modificar los datos cifrados y calcular una firma nueva en función de esos datos cambiados.

Un tipo común de firma adecuada se conoce como “código de autenticación de mensajes hash con clave” (HMAC). Un HMAC difiere de una suma de comprobación en que requiere una clave secreta, que solo conoce la persona que genera el HMAC y la persona que la valida. Si no se tiene esta clave, no se puede generar un HMAC correcto. Cuando recibes los datos, puedes tomar los datos cifrados, calcular de manera independiente el HMAC con la clave secreta que compartes tanto tú como el emisor y, luego, comparar el HMAC que este envía respecto del que calculaste. Esta comparación debe ser de tiempo constante; de lo contrario, habrás agregado otro oráculo detectable, permitiendo así un tipo de ataque distinto.

En resumen, para usar de manera segura los cifrados de bloques de CBC rellenados, es necesario combinarlos con un HMAC (u otra comprobación de integridad de datos) que se valide mediante una comparación de tiempo constante antes de intentar descifrar los datos. Dado que todos los mensajes modificados tardan el mismo tiempo en generar una respuesta, el ataque se evita.

El ataque de oráculo de relleno puede parecer un poco complejo de entender, ya que implica un proceso de retroalimentación para adivinar el contenido cifrado y modificar el relleno. Sin embargo, existen herramientas como PadBuster que pueden automatizar gran parte del proceso.

PadBuster es una herramienta diseñada para automatizar el proceso de descifrado de mensajes cifrados en modo CBC que utilizan relleno PKCS #7. La herramienta permite a los atacantes enviar peticiones HTTP con rellenos maliciosos para determinar si el relleno es válido o no. De esta forma, los atacantes pueden adivinar el contenido cifrado y descifrar todo el mensaje.

A continuación, se proporciona el enlace directo de descarga a la máquina ‘Padding Oracle’ de Vulnhub, la cual estaremos importando en VMWare para practicar esta vulnerabilidad:

Pentester Lab – Padding Oracle: https://www.vulnhub.com/?q=padding+oracle


## Notas de la practica

![[Pasted image 20250826090824.png]]

esta es la web en la que vamos a practicar.

Primero tenemos que entender que es el Cifrado CBC:
![[Pasted image 20250826090918.png]]

En si este modo  de cifrado parte de un texto plano el cual lo separa en diferentes bloques de bytes, en donde cada bloque se somete a un xor con el bloque cifrado anterior, entonces mediante una key cada uno de los bloques se cifra y al final todo se concatena.

El primer bloque no tiene bloque anterior por lo que utiliza un Vector de inicialización con el cual realiza el xor.

![[Pasted image 20250826091519.png]]

En el proceso inverso partimos del texto cifrado el cual diseccionamos en diferentes bytes de longitud y cada cada bloque se descifra y se somete a un xor con el bloque anterior y para el primer bloque utilizamos nuevamente el Vector de inicialización 

Ahora para asegurarnos de que el texto claro encaje en uno o varios bloques se usa lo que es el padding y se suele emplear `PKCS7`:
![[Pasted image 20250826091746.png]]
por ejemplo en la imagen vemos como la primera cadena no completa todo el bloque de 8 bytes por lo que el método se encarga de complementar el relleno con los valores faltantes como vemos faltan dos bytes por lo que usa `0x02` y lo mismo con los demás ejemplos.

Ahora cuando una aplicación realiza el proceso de descifrar un mensaje su flujo es descifrar el mensaje y luego limpiar el relleno. Si mediante la limpieza del relleno tenemos un relleno invalido que provoca un comportamiento detectable ahí es donde decimos que tenemos un Oráculo de Relleno. El comportamiento detectable puede ser un error, una falta de resultados, una respuesta mas lenta y muchos mas, si logramos detectar este comportamiento seremos capaces de descifrar los datos e incluso podemos partir de un texto en claro y cifrarlo.

Un ejemplo seria:

![[Pasted image 20250826092855.png]]

la operatoria es sencilla una vez que pasamos del bloque de encriptación tenemos los valores de I que son `I8 I9 I10` y con esto valores se aplica una operatoria la de `xor` que seria como vemos en la imagen `c15 = E7 xor I15 ....` ahora algo que tenemos que conocer es que esta operatoria de `xor` es conmutativa por lo que  `E7 = c15 xor I15` y `I15 = C15 xor E7` por lo tanto conociendo esto y el como se emplea el padding cuando no se completa un bloque podemos deducir para un bloque de relleno por ejemplo: `\0x1 = valor xor I15` donde obvio se tiene que operar eso para que nos de el bloque de relleno que se uso ahora conociendo que la operatoria `xor` es conmutativa podemos despejar `I5` quedando `I5 = \0x1 xor valor` ese valor mediante fuerza bruta lo podemos obtener pero esto podemos remplazarlo para obtener directamente el texto plano ya que con este valor de I5 podemos remplazarlo en `c15 = E7 xor I15` quedando `c15 = E7 xor \0x1 xor valor` en donde ya podemos aplicar fuerza bruta y encontrar ese valor en texto claro. Esto es complicado de hacer de forma manual por lo que usaremos diferentes herramientas que nos ayudara pero el tener en claro el concepto es muy importante.

Entonces vamos a registrarnos en la web:

![[Pasted image 20250826095747.png]]

![[Pasted image 20250826095800.png]]

podemos ver que estamos ya registrados y que inicio session también.

Ahora el cifrado se aplica en la cookie por lo que vamos a sacarla para analizarla:
![[Pasted image 20250826095940.png]]

ahora detrás de esta cookie tenemos texto claro y podemos intentar obtenerlo y vamos a usar la herramienta [[Padbuster]]:
```bash
padbuster http://192.168.1.105/index.php H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC 8 -cookies 'auth=H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC'
```

![[Pasted image 20250826100401.png]]
seleccionamos la opción recomendada:

![[Pasted image 20250826100443.png]]

como podemos ver obtenemos el valor completo de la información, como dato extra solo podemos dividir los bloques por valores múltiplos de 8.

Ahora el donde radica el problema de vulnerabilidad es que ahora que el programa logro romper el cifrado, podemos recrear cookies validas de otros usuario para suplantar su información de la siguiente manera:

```bash
padbuster http://192.168.1.105/index.php H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC 8 -cookies 'auth=H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC' -plaintext 'user=admin'
```

![[Pasted image 20250826100908.png]]

hemos generado el valor como podemos observar, ya con esto podemos intentar usarlo a ver que si funciona:
![[Pasted image 20250826100944.png]]

ya estamos como admin.
