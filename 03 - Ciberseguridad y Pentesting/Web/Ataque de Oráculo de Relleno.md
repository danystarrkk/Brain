---
aliases:
  - Padding Oracle Attack
  - Ataque de Oráculo de Relleno
tags:
  - hacking/explotacion
  - hacking/laboratorios
  - hacking/herramientas
estado: 🟢 Terminado
---
Un ataque de oráculo de relleno (Padding Oracle Attack) es un tipo de ataque contra datos cifrados que permite al atacante descifrar el contenido de los datos sin conocer la clave de cifrado.

Un oráculo hace referencia a una "indicación" que brinda a un atacante información sobre si la acción que ejecuta es correcta o no. Imagina que estás jugando a un juego de mesa o de cartas con un niño: la cara se le ilumina con una gran sonrisa cuando cree que está a punto de hacer un buen movimiento. Eso es un oráculo. En tu caso, como oponente, puedes usar este oráculo para planear tu próximo movimiento según corresponda.

> [!info]
> En criptografía, un oráculo es una función o servicio que, al ser consultado con datos cifrados, revela información sobre la validez o propiedades de los datos descifrados, sin exponer la clave de cifrado. Esta "indicación" puede ser explícita (un mensaje de error) o implícita (diferencias de tiempo de respuesta).

El relleno (padding) es un término criptográfico específico. Algunos cifrados, que son los algoritmos que se usan para cifrar los datos, funcionan en bloques de datos, donde cada bloque tiene un tamaño fijo. Si los datos que deseas cifrar no tienen el tamaño adecuado para rellenar los bloques, los datos se rellenan automáticamente hasta que lo hacen. Muchas formas de relleno requieren que este siempre esté presente, incluso si la entrada original tenía el tamaño correcto. Esto permite que el relleno siempre se quite de manera segura tras el descifrado.

> [!tip]
> Los esquemas de relleno más comunes incluyen PKCS#7 (el más extendido), ANSI X9.23 e ISO 10126. Cada uno define reglas específicas sobre cómo se añaden los bytes de relleno y qué valores deben tener, lo que es crucial para su correcta eliminación tras el descifrado.

Al combinar ambos elementos, una implementación de software con un oráculo de relleno revela si los datos descifrados tienen un relleno válido. El oráculo podría ser algo tan sencillo como devolver un valor que dice "Relleno no válido", o bien algo más complicado como tomar un tiempo considerablemente diferente para procesar un bloque válido en lugar de uno no válido.

> [!warning]
> Los ataques de temporización (timing attacks) son una forma común de oráculo de relleno. Si el procesamiento de un relleno válido toma un tiempo diferente (aunque sea mínimo) al de un relleno inválido, un atacante puede explotar esta diferencia para inferir información byte a byte.

Los cifrados basados en bloques tienen otra propiedad, denominada "modo de operación", que determina la relación de los datos del primer bloque con los datos del segundo bloque, y así sucesivamente. Uno de los modos más usados es CBC (Cipher Block Chaining). CBC presenta un bloque aleatorio inicial, conocido como "vector de inicialización" (IV), y combina el bloque cifrado anterior con el resultado del cifrado estático a fin de que cifrar el mismo mensaje con la misma clave no siempre genere la misma salida cifrada.

> [!info]
> El modo CBC proporciona difusión, lo que significa que un cambio en un bit del texto plano o del IV afectará a todos los bloques cifrados subsiguientes. Esto dificulta la manipulación directa de bloques individuales sin afectar la validez del relleno, pero es precisamente esta propiedad la que el ataque de oráculo de relleno explota.

Un atacante puede usar un oráculo de relleno, en combinación con la manera en que CBC estructura los datos, para enviar mensajes ligeramente modificados al código que expone el oráculo y seguir enviando datos hasta que el oráculo indique que son correctos. A partir de esta respuesta, el atacante puede descifrar el mensaje byte a byte.

Las redes informáticas modernas son de una calidad tan alta que un atacante puede detectar diferencias muy pequeñas (menos de 0.1 ms) en el tiempo de ejecución en sistemas remotos. Las aplicaciones que suponen que un descifrado correcto solo puede ocurrir cuando no se alteran los datos pueden resultar vulnerables a ataques desde herramientas que están diseñadas para observar diferencias en el descifrado correcto e incorrecto. Si bien esta diferencia de temporización puede ser más significativa en algunos lenguajes o bibliotecas que en otros, ahora se cree que se trata de una amenaza práctica para todos los lenguajes y las bibliotecas cuando se tiene en cuenta la respuesta de la aplicación ante el error.

Este tipo de ataque se basa en la capacidad de cambiar los datos cifrados y probar el resultado con el oráculo. La única manera de mitigar completamente el ataque es detectar los cambios en los datos cifrados y rechazar cualquier acción sobre ellos. La manera estándar de hacerlo es crear una firma para los datos y validarla antes de realizar cualquier operación de descifrado. La firma debe ser verificable y el atacante no debe poder crearla; de lo contrario, podría modificar los datos cifrados y calcular una firma nueva en función de esos datos cambiados.

Un tipo común de firma adecuada se conoce como "código de autenticación de mensajes hash con clave" (HMAC). Un HMAC difiere de una suma de comprobación en que requiere una clave secreta, que solo conoce la persona que genera el HMAC y la persona que la valida. Si no se tiene esta clave, no se puede generar un HMAC correcto. Cuando recibes los datos, puedes tomar los datos cifrados, calcular de manera independiente el HMAC con la clave secreta que compartes tanto tú como el emisor y, luego, comparar el HMAC que este envía respecto del que calculaste. Esta comparación debe ser de tiempo constante; de lo contrario, se habrá agregado otro oráculo detectable, permitiendo así un tipo de ataque distinto (un ataque de temporización en la verificación del HMAC).

> [!info]
> La implementación correcta de la autenticación de mensajes es crucial. El consenso actual favorece el enfoque "Encrypt-then-MAC" (cifrar y luego autenticar), donde primero se cifra el mensaje y luego se calcula el HMAC sobre el texto cifrado. Esto garantiza la confidencialidad y la integridad, y previene ataques como los de oráculo de relleno y otros ataques de manipulación de texto cifrado.

En resumen, para usar de manera segura los cifrados de bloques de CBC rellenados, es necesario combinarlos con un HMAC (u otra comprobación de integridad de datos) que se valide mediante una comparación de tiempo constante antes de intentar descifrar los datos. Dado que todos los mensajes modificados tardan el mismo tiempo en generar una respuesta (debido a la verificación temprana del HMAC), el ataque se evita.

El ataque de oráculo de relleno puede parecer un poco complejo de entender, ya que implica un proceso de retroalimentación iterativo para adivinar el contenido cifrado y modificar el relleno. Sin embargo, existen herramientas como PadBuster que pueden automatizar gran parte del proceso.

PadBuster es una herramienta diseñada para automatizar el proceso de descifrado de mensajes cifrados en modo CBC que utilizan relleno PKCS#7. La herramienta permite a los atacantes enviar peticiones HTTP con rellenos maliciosos para determinar si el relleno es válido o no. De esta forma, los atacantes pueden adivinar el contenido cifrado y descifrar todo el mensaje.

A continuación, se proporciona el enlace directo de descarga a la máquina 'Padding Oracle' de Vulnhub, la cual estaremos importando en VMWare para practicar esta vulnerabilidad:

Pentester Lab – Padding Oracle: `https://www.vulnhub.com/?q=padding+oracle`

## Práctica con PadBuster

![[Pasted image 20250826090824.png]]

Esta es la interfaz web en la que vamos a practicar.

### Entendiendo el Cifrado CBC
![[Pasted image 20250826090918.png]]

En sí, este modo de cifrado parte de un texto plano, el cual lo separa en diferentes bloques de bytes. Cada bloque se somete a una operación XOR con el bloque cifrado anterior. Luego, mediante una clave (key), cada uno de los bloques se cifra y, al final, todo se concatena.

El primer bloque no tiene un bloque cifrado anterior, por lo que utiliza un Vector de Inicialización (IV) con el cual realiza la operación XOR.

![[Pasted image 20250826091519.png]]

En el proceso inverso, partimos del texto cifrado, el cual diseccionamos en diferentes bloques de bytes de longitud fija. Cada bloque se descifra y se somete a una operación XOR con el bloque cifrado anterior. Para el primer bloque, utilizamos nuevamente el Vector de Inicialización (IV).

### Relleno PKCS#7
Ahora, para asegurarnos de que el texto claro encaje en uno o varios bloques, se usa lo que es el relleno (padding), y se suele emplear `PKCS#7`:
![[Pasted image 20250826091746.png]]
Por ejemplo, en la imagen vemos cómo la primera cadena no completa todo el bloque de 8 bytes, por lo que el método se encarga de complementar el relleno con los valores faltantes. Como vemos, faltan dos bytes, por lo que usa `0x02`, y lo mismo ocurre con los demás ejemplos.

> [!info]
> El esquema de relleno PKCS#7 funciona de la siguiente manera: si se necesitan `N` bytes de relleno para completar el último bloque, se añaden `N` bytes, cada uno con el valor `N`. Por ejemplo, si se necesitan 2 bytes, se añaden `0x02 0x02`. Si se necesitan 8 bytes (un bloque completo), se añaden `0x08 0x08 0x08 0x08 0x08 0x08 0x08 0x08`.

Ahora, cuando una aplicación realiza el proceso de descifrar un mensaje, su flujo es descifrar el mensaje y luego limpiar el relleno. Si, durante la limpieza del relleno, se detecta un relleno inválido que provoca un comportamiento detectable, ahí es donde decimos que tenemos un Oráculo de Relleno. El comportamiento detectable puede ser un error, una falta de resultados, una respuesta más lenta, entre otros. Si logramos detectar este comportamiento, seremos capaces de descifrar los datos e incluso podemos partir de un texto en claro y cifrarlo.

### Ejemplo de Operatoria

![[Pasted image 20250826092855.png]]

La operatoria es sencilla: una vez que pasamos del bloque de encriptación, obtenemos los valores intermedios `I` (por ejemplo, `I8, I9, I10`). Con estos valores, se aplica una operación XOR, como se ve en la imagen: `c15 = E7 XOR I15`. Algo fundamental que debemos conocer es que la operación XOR es conmutativa, por lo que `E7 = c15 XOR I15` y `I15 = C15 XOR E7`. Conociendo esto y cómo se emplea el relleno (padding) cuando no se completa un bloque, podemos deducir para un bloque de relleno, por ejemplo: `0x01 = valor_conocido XOR I15`. Aquí, `valor_conocido` es el byte que estamos probando en el bloque anterior para que el último byte del bloque descifrado sea `0x01`. Conociendo que la operación XOR es conmutativa, podemos despejar `I15` (o el byte `I` correspondiente) quedando `I15 = 0x01 XOR valor_conocido`. Este valor `valor_conocido` se puede obtener mediante fuerza bruta, probando todos los posibles bytes (0-255) hasta que el oráculo indique un relleno válido. Una vez que tenemos `I15`, podemos reemplazarlo en la ecuación original de descifrado para obtener el texto plano: `P15 = D(C15) XOR C14` (donde `D(C15)` es el bloque descifrado y `C14` es el bloque cifrado anterior, o el IV para el primer bloque). El ataque de oráculo de relleno manipula el bloque cifrado anterior (`C14` o IV) para controlar el resultado del XOR y forzar un relleno válido, revelando así los bytes del texto plano uno por uno. Esto es complicado de hacer de forma manual, por lo que usaremos diferentes herramientas que nos ayudarán, pero tener claro el concepto es muy importante.

> [!info]
> El proceso de descifrado byte a byte en un ataque de oráculo de relleno implica iterar sobre cada byte del último bloque del texto cifrado. Para cada byte, se manipula el byte correspondiente en el bloque cifrado *anterior* (o el IV si es el primer bloque) y se envía al oráculo. Se prueban todos los 256 posibles valores para el byte manipulado hasta que el oráculo indica un relleno válido. Una vez que se encuentra un valor que produce un relleno válido, se puede inferir el valor del byte del texto plano. Este proceso se repite para cada byte, trabajando desde el final del bloque hacia el principio, y luego se extiende a los bloques anteriores.

Entonces, vamos a registrarnos en la web:

![[Pasted image 20250826095747.png]]

![[Pasted image 20250826095800.png]]

Podemos ver que ya estamos registrados y que la sesión ha iniciado.

Ahora, el cifrado se aplica en la cookie, por lo que vamos a extraerla para analizarla:
![[Pasted image 20250826095940.png]]

Ahora, detrás de esta cookie tenemos texto claro y podemos intentar obtenerlo. Para ello, usaremos la herramienta `PadBuster`:
```bash
padbuster http://192.168.1.105/index.php H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC 8 -cookies 'auth=H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC'
```

![[Pasted image 20250826100401.png]]
Seleccionamos la opción recomendada:

![[Pasted image 20250826100443.png]]

Como podemos ver, obtenemos el valor completo de la información. Como dato extra, solo podemos dividir los bloques por valores múltiplos de 8 (el tamaño del bloque del cifrado).

Ahora, la vulnerabilidad radica en que, una vez que el programa logró romper el cifrado, podemos recrear cookies válidas de otros usuarios para suplantar su información de la siguiente manera:

```bash
padbuster http://192.168.1.105/index.php H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC 8 -cookies 'auth=H2er5oQSzYRp5tkgCCyBbYr2UmIu3TJC' -plaintext 'user=admin'
```

![[Pasted image 20250826100908.png]]

Hemos generado el valor, como podemos observar. Con esto, podemos intentar usarlo para verificar si funciona:
![[Pasted image 20250826100944.png]]

¡Ya estamos como admin!
